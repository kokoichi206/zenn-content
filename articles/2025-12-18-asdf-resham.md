---
title: "asdf の shim が消えない問題とその設計上の理由"
emoji: "🐈‍⬛"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["asdf", "shim", "Claude", "nodejs"]
publication_name: "japagate"
published: true
---

:::message
この記事は、[ジャパゲートシステムズ Advent Calendar 2025](https://qiita.com/advent-calendar/2025/japagate-systems) 18 日目の記事です。
:::

## 先にまとめ

方針

- asdf 管理下での `npm i -g` などのグローバルインストールは避けて native でのインストールに寄せる

asdf の挙動

- asdf の shim は作成時に**全バージョン**を走査し、実行時に**現在のバージョン**を参照する
- この非対称性により、古いバージョンにインストールしたパッケージの shim が残り続ける
- 対処するには、全バージョンからアンインストールして reshim を実行する

## バージョン

この記事は以下のバージョンで検証しています。

```
asdf 0.18.0
```

## 事象

ある時 claude code を実行しようとしたら、以下のエラーが出ました。

```bash
> claude  

No version is set for command claude
Consider adding one of the following versions in your config file at /Users/kokoichi206/ghq/github.com/kokoichi206/zenn-content/.tool-versions
nodejs 22.20.0
nodejs 20.18.1
```

[asdf](https://asdf-vm.com/) で複数の Node.js バージョンを管理していたのですが、**現在 global に設定しているバージョンとは別のバージョンに `npm i -g` でインストールしたパッケージが残ってい**たのが原因でした。

### 対処

各 asdf の nodejs plugin のバージョンからアンインストールして reshim を実行しました。

```bash
# どの nodejs に入ってるのか特定がだるいので、全部から消す。
for v in $(asdf list nodejs | tr -d ' '); do
  echo "=== Node $v ==="
  ASDF_NODEJS_VERSION="$v" asdf exec npm -g uninstall @anthropic-ai/claude-code
  # uninstall でうまくいかないケースがあったため、念のため実態からも削除する。
  rm -rf "$dir/lib/node_modules/@anthropic-ai/claude-code"
  rm -rf "$dir/lib/node_modules/@anthropic-ai/claude"
  rm -rf "$dir/lib/node_modules/claude"
  rm -f  "$dir/bin/claude"
done

asdf reshim nodejs
```

その後 native にバイナリを導入し直しました。
（claude も現在では[その方法が推奨](https://code.claude.com/docs/ja/setup#native-install-recommended)されています。）

``` sh
# mac
curl -fsSL https://claude.ai/install.sh | bash
```

## asdf shim の挙動について

shim の挙動が気になったので少し調べてみました。

shim とは、[各々の plugin にインストールされている実行プログラムが、現在の環境で利用できるようにするための仕組み](https://asdf-vm.com/ja-jp/manage/versions.html#shims)です。

実体は `exec asdf exec` をするだけの、シンプルなラッパーとなっています。

``` sh
> which claude       
/Users/kokoichi206/.asdf/shims/claude

> cat /Users/kokoichi206/.asdf/shims/claude                 
#!/usr/bin/env bash
# asdf-plugin: nodejs 22.20.0
# asdf-plugin: nodejs 20.18.1
exec asdf exec "claude" "$@"%
```

### shim 作成時

`asdf reshim`, `asdf reshim nodejs` を実行すると、[**全バージョン**のグローバルパッケージを走査して shim を生成します](https://github.com/asdf-vm/asdf/blob/80fffd86853fbfcc31e6a52aee99f8ab3d5fae9f/internal/shims/shims.go#L251-L267)。

reshim 作業での成果物は、先ほど確認した `~/.asdf/shims/claude` という [1 つのファイルとなります](https://github.com/asdf-vm/asdf/blob/80fffd86853fbfcc31e6a52aee99f8ab3d5fae9f/internal/shims/shims.go#L310-L330)。

その際に asdf-plugin という形で、[コメントに実行バイナリのインストールされたプラグインのバージョンを記載](https://github.com/asdf-vm/asdf/blob/80fffd86853fbfcc31e6a52aee99f8ab3d5fae9f/internal/shims/shims.go#L437-L442)します。

``` sh
# asdf-plugin: nodejs 22.20.0
# asdf-plugin: nodejs 20.18.1
```

### shim 実行時

`~/.asdf/shims/claude` が実行されると、先ほど確認した通り `exec asdf exec "claude" "$@"%` が実行されます。 

この shim が実行されると、以下のステップで実際の実行ファイルを探します。

1. [shim ファイルに記載されている plugin のバージョンを取得](https://github.com/asdf-vm/asdf/blob/80fffd86853fbfcc31e6a52aee99f8ab3d5fae9f/internal/shims/shims.go#L66-L69)
2. [そのバージョンに対応する実行ファイルを探索](https://github.com/asdf-vm/asdf/blob/80fffd86853fbfcc31e6a52aee99f8ab3d5fae9f/internal/shims/shims.go#L83)
3. [resolve.Resolve](https://github.com/asdf-vm/asdf/blob/80fffd86853fbfcc31e6a52aee99f8ab3d5fae9f/internal/resolve/resolve.go#L26-L56) でカレントディレクトリから親へ .tool-versions を再帰的に探す + global で設定されたバージョンの `.tool-versions` を確認

つまり、実行時は global および現在のディレクトリの `.tool-versions` に基づいてバージョンが決定されます。

### shim 作成時と実行時の非対称性

作成時と実行時で参照するバージョンが異なります。

|  | 参照するバージョン |
| --- | --- |
| 作成時 | **全てのバージョン** |
| 実行時 | **現在の `.tool-versions`** で指定されたバージョン |

本来、PATH に登録されるコマンドは『今使えるもの』だけであるべきで、作成時と実行時に同じバージョンを参照するのが理想的に思えます。

しかし asdf の設計上 **reshim のタイミングが不定であり、確実にバイナリを登録できるかが保証できません**。
そのため、「どこかのバージョンで使える可能性があるもの」を全て登録しておく設計になっています。

この設計の副作用として、今回のように「**もう使わないパッケージの shim が残り続ける**」という問題が起きます。
古いバージョンにグローバルインストールしたパッケージがあると、現在のバージョンで使っていなくても shim が生成され続けます。
