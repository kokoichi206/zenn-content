---
title: "GitHub Actions の paths trigger には 300 ファイルの壁がある"
emoji: "🐈‍⬛"
type: "tech"
topics: ["GitHubActions", "CI/CD", "Monorepo"]
published: true
---

## TL;DR

GitHub Actions の `on.push.paths` や `on.pull_request.paths` トリガーは、差分の最初の **300 ファイルしか評価しません**。300 件を超える差分を含む push や PR では、マッチするファイルが 301 番目以降にあるとワークフローが **サイレントにスキップ** されます。エラーも警告も出ません。

本記事では、この仕様の挙動と、運用上押さえておきたい回避策を整理します。

## 実際に確認した挙動

モノレポで release ブランチを main にマージしたところ、差分は 482 ファイルでした。CD ワークフローには以下の paths フィルタが設定されていました。

```yaml
on:
  push:
    branches:
      - main
    paths:
      - 'infrastructure/terraform/**'
      - 'workers/node/**'
```

`infrastructure/terraform/` や `workers/node/` に大量の変更があったにもかかわらず、ワークフローはトリガーされませんでした。

一方、同じ push で `apps/web/**` や `**/*.md` をトリガーとする別のワークフローは正常に実行されました。

原因は、GitHub が返す差分ファイルの **先頭 300 件だけ** が paths フィルタで評価される仕様にありました。`.claude/`, `.github/`, `api/`, `apps/` などが先に 300 枠を埋め、`infrastructure/` や `workers/` は評価対象外になっていました（この並びは今回の観測ではパス順でした）。

## 公式ドキュメントの記載

[GitHub 公式ドキュメント](https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-syntax#git-diff-comparisons)には以下のように記載されています。

> Diffs are limited to 300 files. If there are files changed that aren't matched in the first 300 files returned by the filter, the workflow will not run. You may need to create more specific filters so that the workflow will run automatically.

つまり、これは仕様でありバグではありません。

## 仕様として押さえておきたいポイント

- **見え方に注意が必要**: フィルタ条件に合わないと workflow run 自体が作られないため、Actions の実行履歴だけでは追いにくいです。PR では required check が Pending のまま残ることもあります
- **モノレポでは発生しやすい**: ディレクトリが多いモノレポほど差分が 300 ファイルを超えやすいです
- **返却順の影響を受ける**: 先頭側のパスに変更が偏ると、後ろのディレクトリが評価対象から漏れます
- **小さい変更では表面化しにくい**: 普段の小さな PR では問題なく動くため、大きなリリース PR で表面化しやすいです

## git diff の比較方法

GitHub がファイル変更リストを生成する方法はイベントによって異なります。

| イベント              | 比較方法                                                                            |
| --------------------- | ----------------------------------------------------------------------------------- |
| Pull request          | 3 点比較 (three-dot diff): トピックブランチとベースブランチの最終同期コミットの差分 |
| 既存ブランチへの push | 2 点比較 (two-dot diff): head SHA と base SHA の直接比較                            |
| 新規ブランチへの push | プッシュされた最も深いコミットの祖先の親との 2 点比較                               |

いずれの方式でも **300 ファイルの上限は共通** で適用されます。

なお、1,000 コミットを超える push や、差分生成がタイムアウトした場合は、paths フィルタに関係なくワークフローが実行されます。

> If you push more than 1,000 commits, or if GitHub does not generate the diff due to a timeout, the workflow will always run.

[公式ドキュメント: ワークフローのトリガー > Git diffの比較](https://docs.github.com/ja/actions/how-tos/writing-workflows/choosing-when-your-workflow-runs/triggering-a-workflow)

## 対策

### 1. dorny/paths-filter を使う

paths フィルタをトリガーレベルから外し、ワークフロー内のジョブで [dorny/paths-filter](https://github.com/dorny/paths-filter) を使って判定します。  

```yaml
on:
  push:
    branches:
      - main
  # paths フィルタは使わない

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      infra: ${{ steps.filter.outputs.infra }}
      workers: ${{ steps.filter.outputs.workers }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            infra:
              - 'infrastructure/terraform/**'
            workers:
              - 'workers/node/**'
              - 'workers/python/**'

  deploy:
    needs: detect-changes
    if: needs.detect-changes.outputs.infra == 'true' || needs.detect-changes.outputs.workers == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying..."
```

### 2. git diff を直接使う

外部アクションに依存したくない場合は、`push` イベントの `before` と `sha` を使って `git diff` を自前でチェックします。

```yaml
jobs:
  check-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Check for relevant changes
        id: check
        run: |
          BASE_SHA="${{ github.event.before }}"
          HEAD_SHA="${{ github.sha }}"
          CHANGED=$(git diff --name-only "$BASE_SHA" "$HEAD_SHA" -- \
            'infrastructure/terraform/' \
            'workers/node/' \
            'workers/python/' \
          )
          if [ -n "$CHANGED" ]; then
            echo "changed=true" >> "$GITHUB_OUTPUT"
          else
            echo "changed=false" >> "$GITHUB_OUTPUT"
          fi
      - name: Deploy
        if: steps.check.outputs.changed == 'true'
        run: echo "deploying..."
```

## 参考リンク

- [Workflow syntax for GitHub Actions - GitHub Docs](https://docs.github.com/ja/actions/reference/workflows-and-actions/workflow-syntax#git-diff-comparisons)
- [Improve error messaging on the 300 file git diff limit for triggering actions - GitHub Community Discussion #53831](https://github.com/orgs/community/discussions/53831)
- [dorny/paths-filter](https://github.com/dorny/paths-filter)
