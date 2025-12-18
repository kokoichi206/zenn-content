---
title: "GitHub Actions のデプロイで環境が壊れる前に設定しておきたい concurrency"
emoji: "🐈‍⬛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHubActions", "CD"]
published: true
---

[concurrency](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs) は、ワークフローやジョブの同時実行を制御する機能です。
同じグループ内では、同時に 1 つのワークフロー (or ジョブ) のみが実行されることが保証されます。

```yaml
name: 簡単なサンプル

on:
  push:
    branches: [main]
  pull_request:

# ワークフロー単位で制御する場合。
concurrency:
  # group に ref 等を入れると、ブランチや PR 単位で制御できる。
  group: ${{ github.workflow }}
  # true に設定すると、新しい実行が開始された際に古い実行をキャンセルする。
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    ...
```

並列で deploy が走ってしまい環境が壊れる前に設定しておきましょう！
