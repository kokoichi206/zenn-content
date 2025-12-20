---
title: "GitHub Actions で Terraform がキャンセルされない問題を深掘りする"
emoji: "🐈‍⬛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHubActions", "Terraform", "CI/CD"]
publication_name: "japagate"
published: true
---

:::message
この記事は、[ジャパゲートシステムズ Advent Calendar 2025](https://qiita.com/advent-calendar/2025/japagate-systems) 19 日目の記事です。
:::

GitHub Actions で `terraform apply` を実行中にワークフローをキャンセルしても Terraform のプロセスが止まらず、deploy に影響を与えてしまったことがありました。
この問題の原因の深掘りと対策を残しておきます。

## 先にまとめ

- GitHub Actions で terraform apply がハングし続けていた
  - 必須の terraform variable が設定されていなかったことが原因
  - [Cancel Workflow](https://docs.github.com/ja/actions/how-tos/manage-workflow-runs/cancel-a-workflow-run) が何故か効かない
- `hashicorp/setup-terraform` の `terraform_wrapper` による node process での wrap が原因
  - Node.js ラッパーがシグナルを Terraform に転送しない
- 解決策
  - `terraform_wrapper: false` を設定
  - `-input=false` をつける

```yaml
- uses: hashicorp/setup-terraform@v3
  with:
	# Node.js wrapper を無効化。
	# (他にどんな影響が出るかは確認しきれてないです。)
    terraform_wrapper: false
- name: Terraform Apply
  run: |
    # --input=false で入力待ちを回避し、未指定の場合に即時で落ちるようにする。
    terraform apply -auto-approve -input=false
```

## 問題の発端

GitHub Actions で Terraform を使った CI/CD パイプラインを運用している時に、以下のような事象に遭遇しました。

1. `terraform apply -auto-approve` が実行される
2. 必須の variable が設定されておらず標準入力からの input を待ち受ける文章が出る
3. Action の実行ログから [workflow をキャンセル](https://docs.github.com/ja/actions/how-tos/manage-workflow-runs/cancel-a-workflow-run)
4. **Terraform の実行が止まらない**
5. タイムアウトまで待つしかない

「キャンセルボタンを押したのに、なぜ止まらないのか？」という疑問から調査を開始しました。

## 調査1: シグナルは届いているのか?

まず、GitHub Actions のキャンセル時にどんなシグナルが送られるのかを調べました。

### シグナル監視プログラムを作成

:::details Go でシグナルを監視するプログラムを作成

main.go

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	fmt.Fprintf(os.Stderr, "PID: %d\n", os.Getpid())
	fmt.Fprintf(os.Stderr, "PPID: %d\n", os.Getppid())

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh) // すべてのシグナルを受け取る

	// 生存確認を出力
	go func() {
		ticker := time.NewTicker(1 * time.Second)
		for t := range ticker.C {
			fmt.Fprintf(os.Stderr, "Still alive at %s (PID=%d, PPID=%d)\n",
				t.Format("15:04:05"), os.Getpid(), os.Getppid())
		}
	}()

	for {
		sig := <-sigCh
		fmt.Fprintf(os.Stderr, "Received signal: %v at %s\n",
			sig, time.Now().Format("2006-01-02 15:04:05"))
	}
}
```

:::

```yaml
# .github/workflows/cancel-test.yml
- name: Run app
  run: go run main.go
```

### 実行結果1.1: シグナルが届かない

```
Run go run main.go
PID: 2378
Waiting for signals...
Still alive at 11:21:37 (PID=2378, PPID=2377)
Still alive at 11:21:38 (PID=2378, PPID=2377)
...
Error: The operation was canceled.
```

キャンセルしても何のシグナルも表示されずに終了しました。

SIGINT or SIGTERM が送信されるのだろうと推測していたのですが、どうやらそうではないようです。

### 実行結果1.2: 親プロセスの死を検知

PPID (親プロセス ID) を監視してみると、興味深い発見がありました。

```
Still alive at 11:24:21 (PID=2261, PPID=2260)
Still alive at 11:24:22 (PID=2261, PPID=1)     # PPID が 1 に変わった!
Still alive at 11:24:23 (PID=2261, PPID=1)
Error: The operation was canceled.
```

cancel workflow を押したちょっと後で PPID が 2260 から 1 に変わっています。

これは以下のようなことを意味し、シグナルは子プロセスに直接送られていないと推測されます。

1. キャンセル時に親プロセス (bash) だけが kill される
2. 子プロセスが orphan となり init (PID=1) に引き取られる

## 調査2: exec を使って bash を置き換える

bash を経由せずに直接プロセスを実行すれば、シグナルが届くのでは?

```yaml
# .github/workflows/cancel-test.yml
- name: Build app
  run: go build -o app main.go
- name: Run app
  run: exec ./app  # exec で bash を置き換える
```

### 結果2.1: シグナルが Go application に届いた

```
Still alive at 11:34:24 (PID=2268, PPID=1976)
Still alive at 11:34:25 (PID=2268, PPID=1976)
Received signal: interrupt at 2025-12-20 11:34:26     # SIGINT!
Still alive at 11:34:27 (PID=2268, PPID=1976)
...
Received signal: terminated at 2025-12-20 11:34:34    # SIGTERM!
```

またこの結果から、GitHub Actions のキャンセル処理は以下の通りであることがわかります。

1. 親プロセスに **SIGINT** を送信
2. 数秒待機 (7, 8 秒くらい)
3. 親プロセスに **SIGTERM** を送信
4. 最後に **SIGKILL** を送信 (Go application では捕捉できない)

## 調査3: terraform process を exec する

調査 2 より exec を使って terraform を bash を経由せずに実行すればいいのではないかと推測されますが、実際にはうまくいきませんでした。
（結果略）

## 調査4: なぜ Terraform には届かないのか

Terraform がキャンセルされない問題にあたり、プロセスツリーを確認してみました。

:::details プロセスツリーの確認 job step と結果詳細

``` yaml
- name: Terraform apply with process monitoring
  run: |
    # バックグラウンドで定期的にプロセスツリーを表示
    (
      while true; do
        echo "=== Process tree at $(date) ==="
        ps -ef --forest 2>/dev/null || ps -ef
        echo ""
        sleep 5
      done
    ) &
    MONITOR_PID=$!

    trap 'echo "SIGINT received at $(date)"; kill -INT $TF_PID 2>/dev/null' INT
    trap 'echo "SIGTERM received at $(date)"; kill -TERM $TF_PID 2>/dev/null; kill $MONITOR_PID 2>/dev/null' TERMExpand commentComment on lines R143 to R144ResolvedCode has comments. Press enter to view.

    terraform apply -auto-approve &

    TF_PID=$!
    echo "Terraform PID: $TF_PID"
```

```sh
runner      1774       1  0 11:46 ?        00:00:00 /opt/hca/hosted-compute-agent
root        1786    1774  0 11:46 ?        00:00:00  \_ sudo -E -n /tmp/provjobd2590389568
root        1789    1786  0 11:46 ?        00:00:00  |   \_ /tmp/provjobd2590389568
runner      1807    1774  4 11:47 ?        00:00:01  \_ /home/runner/actions-runner/cached/bin/Runner.Listener run
runner      1822    1807 12 11:47 ?        00:00:03      \_ /home/runner/actions-runner/cached/bin/Runner.Worker spawnclient 142 145
runner      1983    1822  0 11:47 ?        00:00:00          \_ /usr/bin/bash -e /home/runner/work/_temp/c313a579-3e4f-4f65-8485-c3d85c3007a2.sh
runner      1984    1983  0 11:47 ?        00:00:00              \_ /usr/bin/bash -e /home/runner/work/_temp/c313a579-3e4f-4f65-8485-c3d85c3007a2.sh
runner      2013    1984  0 11:47 ?        00:00:00              |   \_ ps -ef --forest
runner      1985    1983  0 11:47 ?        00:00:00              \_ node /home/runner/work/_temp/42d3384c-c4ec-401a-8f11-4cf7bdee2414/terraform apply -target=module.lambda.aws_ecr_repository.lambda -target=module.lambda.aws_ecr_lifecycle_policy.lambda -target=module.lambda.aws_ecr_repository.python_lambda -target=module.lambda.aws_ecr_lifecycle_policy.python_lambda
```

:::

```
runner      1983    1822  \_ /usr/bin/bash -e /home/runner/work/_temp/c313a579-3e4f-4f65-8485-c3d85c3007a2.sh
runner      1984    1983      \_ /usr/bin/bash -e /home/runner/work/_temp/c313a579-3e4f-4f65-8485-c3d85c3007a2.sh
runner      2013    1984      |   \_ ps -ef --forest
runner      1985    1983      \_ node /home/runner/work/_temp/42d3384c-c4ec-401a-8f11-4cf7bdee2414/terraform apply -target=module.lambda.aws_ecr_repository.lambda -target=module.lambda.aws_ecr_lifecycle_policy.lambda -target=module.lambda.aws_ecr_repository.python_lambda -target=module.lambda.aws_ecr_lifecycle_policy.python_lambda
```

最後が terraform apply に関連するプロセスなのですが、**Terraform が node プロセスとして動いている**ことがわかります。

実はこれは local の terraform cli では存在しない仕組みで、GitHub Action で terraform を setup 時に使う Action [`hashicorp/setup-terraform`](https://github.com/hashicorp/setup-terraform) の `terraform_wrapper` 機能によって作られたものです。

## 調査5: terraform_wrapper を試す

### terraform_wrapper とは

`hashicorp/setup-terraform` は [terraform を Node.js ラッパースクリプトとして実行する](https://github.com/hashicorp/setup-terraform/blob/v3.1.2/wrapper/terraform.js)フラグが用意されており、[default では true](https://github.com/hashicorp/setup-terraform#inputs) となっています。

後続のステップで Terraform の出力を使えるようにするための仕組みらしいのですが、`@actions/exec` のシグナル伝播がうまくいかない既知の挙動があるようで #441 で報告されてます。

- [Terraform state doesn't unlock if you cancel workflow running a terraform plan #441](https://github.com/hashicorp/setup-terraform/issues/441)

### terraform_wrapper false にして実行してみる

```yaml
- name: Setup Terraform
  uses: hashicorp/setup-terraform@b9cd54a3c349d3f38e8881555d616ced269862dd # v3.1.2
  with:
    terraform_version: '1.13.0'
    terraform_wrapper: false
```

:::details 調査 4 と同じ実行の結果詳細

```sh
runner      1773       1  0 11:49 ?        00:00:00 /opt/hca/hosted-compute-agent
root        1785    1773  0 11:49 ?        00:00:00  \_ sudo -E -n /tmp/provjobd2638103238
root        1787    1785  0 11:49 ?        00:00:00  |   \_ /tmp/provjobd2638103238
runner      1808    1773  6 11:50 ?        00:00:01  \_ /home/runner/actions-runner/cached/bin/Runner.Listener run
runner      1823    1808 22 11:50 ?        00:00:03      \_ /home/runner/actions-runner/cached/bin/Runner.Worker spawnclient 142 145
runner      1976    1823  0 11:50 ?        00:00:00          \_ /usr/bin/bash -e /home/runner/work/_temp/e6c0bf6b-54bc-4fdc-900b-feb8307ed658.sh
runner      1977    1976  0 11:50 ?        00:00:00              \_ /usr/bin/bash -e /home/runner/work/_temp/e6c0bf6b-54bc-4fdc-900b-feb8307ed658.sh
runner      1980    1977  0 11:50 ?        00:00:00              |   \_ ps -ef --forest
runner      1978    1976  0 11:50 ?        00:00:00              \_ terraform apply -target=module.lambda.aws_ecr_repository.lambda -target=module.lambda.aws_ecr_lifecycle_policy.lambda -target=module.lambda.aws_ecr_repository.python_lambda -target=module.lambda.aws_ecr_lifecycle_policy.python_lambda
```

:::

結果は以下のようになり、無事 bash の子プロセスとして terraform が実行されました。

```sh
runner      1976    1823  \_ /usr/bin/bash -e /home/runner/work/_temp/e6c0bf6b-54bc-4fdc-900b-feb8307ed658.sh
runner      1977    1976      \_ /usr/bin/bash -e /home/runner/work/_temp/e6c0bf6b-54bc-4fdc-900b-feb8307ed658.sh
runner      1980    1977      |   \_ ps -ef --forest
runner      1978    1976      \_ terraform apply -target=module.lambda.aws_ecr_repository.lambda -target=module.lambda.aws_ecr_lifecycle_policy.lambda -target=module.lambda.aws_ecr_repository.python_lambda -target=module.lambda.aws_ecr_lifecycle_policy.python_lambda
```

この場合に terraform apply をすると input 待ちが表示され、プロセスが即時終了することがわかります。
（input 待ちでハングしないのは CI 環境では tty に接続しておらず stdin がパイプとして扱われ EOF を返すためと思われますが、調査の詳細は割愛します。）

## 解決策

### 方法1: terraform_wrapper を無効化

```yaml
- uses: hashicorp/setup-terraform@v3
  with:
    terraform_wrapper: false
```

これにより `terraform` コマンドが直接バイナリを実行するようになり terraform のキャンセルが可能になります。

### 方法2: input=false フラグをつける（推奨）

[#440](https://github.com/hashicorp/setup-terraform/pull/440) でも指摘されている通り、workflow 実行中に入力待ちでハングするのを防ぐには `input=false` フラグを指定するのが適切です。
（README の例も、全部そのように変わってた。）

## Links

- [hashicorp/setup-terraform](https://github.com/hashicorp/setup-terraform)
  - [Issue #441 - State lock on workflow cancellation](https://github.com/hashicorp/setup-terraform/issues/441)
  - [Issue #395 - Real-time output](https://github.com/hashicorp/setup-terraform/issues/395)
- [hashicorp/terraform](https://github.com/hashicorp/terraform)
  - [Issue #10459 - SIGTERM support in Terraform](https://github.com/hashicorp/terraform/issues/10459)
