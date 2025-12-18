---
title: "『GitHub Actions で誰でも本番環境を壊せる』状態を潰す"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub Actions", "CI/CD", "AWS", "セキュリティ"]
publication_name: "japagate"
published: true
---

:::message
この記事は、[ジャパゲートシステムズ Advent Calendar](https://qiita.com/advent-calendar/2025/japagate-systems) 2025 15 日目の記事です。  
:::

GitHub Actions は非常に便利な CI/CD 基盤ですが、設定次第では**本番デプロイの権限が想像以上に緩くなってしまう**ことがあります。

本記事では、GitHub Actions と AWS OIDC を利用した既存のデプロイフローについて、以下の内容を整理してご紹介します。

* どの点にリスクを感じていたか
* どのような対策をしたか
* 実際の設定例

同じような構成を採用している方の参考になれば幸いです。

## 既存の GitHub Actions 構成と懸念点

以下のように、`main` ブランチへの push または `workflow_dispatch` による手動実行で、本番環境へデプロイが走る構成をしていました。

``` yml
name: cd-prod

on:
  workflow_dispatch:
  push:
    branches:
      - main

env:
  AWS_ROLE_TO_ASSUME: 'arn:aws:iam::000000000000:role/gh-action-role-production'

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production
    ...
    steps:
      - name: Check out
        uses: actions/checkout@08eba0b27e820071cde6df949e0beb9ba4906955 # v4.3.0
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@7474bc4690e29a8392af63c5b98e7449536d5c3a # v4.3.1
        with:
          aws-region: ap-northeast-1
          role-to-assume: ${{ env.AWS_ROLE_TO_ASSUME }}
      ...
```

当初から以下のような懸念が頭の中にありつつも、メンバーが少なかったこともあり放置していました。

- workflow_dispatch が誰でも実行可
- workflow_dispatch で任意のブランチで実行可
- workflow の書き換えで意図しないデプロイが可
  - trigger を変更・消してしまう
  - step で意図しないコマンドを実行してしまう
- 新規の workflow を作成しデプロイが可

## 対策

### GitHub Environment によるデプロイ制御

まず [GitHub Environment](https://docs.github.com/ja/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments) を使った対策をおこないました。

Environments を使うことで環境ごとに env/secrets を設定できたり、[Deploy 時のレビュワーの必須にでき](https://docs.github.com/ja/actions/reference/workflows-and-actions/deployments-and-environments#required-reviewers)たりします。

具体的には、**Repository Settings → Environments** から `production` Environment を作成し、
以下の設定しました。

![](/images/2025-12-15-prod-release/env-settings.png)

これにより指定したレビュワーが承認するまでデプロイできなくなります。
（↓ レビュー承認待ちの workflow）

![レビュー承認待ちの workflow](/images/2025-12-15-prod-release/waiting-deploy-review.png)

ここで注意なのが、**必要な secret を作成した環境の secret として登録しないと意味**がないということです。

secret には [repository secret](https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets#creating-secrets-for-a-repository) と [environment secret](https://docs.github.com/ja/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets#creating-secrets-for-an-environment) がありますが、repository の方に登録してしまうと `environment: production` といった環境を設定せずに workflow を作成・実行してしまう恐れがあります。

### AWS OIDC によるデプロイブランチ制御

GitHub Actions から AWS リソースを操作するときは [OIDC を使う](https://docs.github.com/ja/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)ことが推奨されていますが、その設定を見直すことで**特定ブランチでしか assume role できないよう**にしました。

具体的には、OIDC ID トークンの sub クレームを『対象リポジトリの main ブランチ』に限定するよう、AWS 側の信頼ポリシー（Trust policy）を設定しました。

``` json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::000000000000:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
				},
				"StringLike": {
					"token.actions.githubusercontent.com:sub": "repo:<organization>/<repository>:ref:refs/heads/main"
				}
			}
		}
	]
}
```

ちなみに、複数ブランチ設定したければ以下のように配列で指定できます。

```
"token.actions.githubusercontent.com:sub": [
  "repo:<organization>/<repository>:ref:refs/heads/main",
  "repo:<organization>/<repository>:ref:refs/heads/hotfix/*"
]
```

## 対応後の挙動

最初にあげた懸念に対して、対応後は以下のような挙動になります。


対応後は、以下のような動作になります。

| 操作                    | 結果                   |
| --------------------- | -------------------- |
| workflow_dispatch が誰でも実行可 | Environment 承認待ちで停止  |
| workflow_dispatch で任意のブランチで実行可 | AWS 側で AssumeRole 失敗 |
| workflow の書き換えで意図しないデプロイが可 | Environment 承認待ちで停止 |
| 新規の workflow を作成しデプロイが可 | Environment 承認待ちで停止<br />（Environment 未指定だと secret にアクセスできず失敗） |

意図しないデプロイが起きにくい構成になりました。

## まとめ

インフラ等で本番環境のリソースへのアクセスを制限するのと同様に、CI/CD の設定においても**誰が・どのように本番環境へデプロイできるか**をしっかり設計することが重要だと感じています。
その点については、GitHub Actions の標準機能だけでも多くのことが実現できるため、仕組みを正しく理解した上で継続的にキャッチアップしていきたいと考えています。
