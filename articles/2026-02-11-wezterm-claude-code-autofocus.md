---
title: "【WezTerm × Claude Code】AI を待たせない人間になりたい"
emoji: "🐈‍⬛"
type: "tech"
topics: ["WezTerm", "ClaudeCode", "Lua", "macOS"]
published: true
---

AI を使った開発において、『AI を使って今までのやり方を便利にする』だけではだめで、『**人間と AI の協調を再設計し AI が生き生き働ける環境にする**』必要があると感じています。

その中でも手始めとして『人間がボトルネックになっている状況』を減らし、AI が止まらず走れるようにしてみました。

今回やったのは『Claude Code のタスクが完了・人間の判断を仰ぐときは、自動でターミナルの該当 tab/pane にフォーカスが戻る』という設定です。

## 先にまとめ

- Claude Code の **Stop / Notification hook** で OSC 1337 シーケンスを WezTerm に送信
- WezTerm 側で `user-var-changed` イベントを拾ってタブ/ペインを切り替え
- osascript で macOS レベルのウィンドウ前面化も併用
- 外部ツール不要、WezTerm のネイティブ機能だけで完結

## 環境

```sh
$ claude --version
2.1.59 (Claude Code)

$ wezterm --version
wezterm 20251201-075747-d3b0fdad
```

## 使用例

自分の普段の使い方として、WezTerm で 4 プロジェクトくらいを 1 プロジェクト 1 タブで開いていて、各タブに 4 つくらいペインを配置しています。
そのうち 1〜4 つのペインで Claude Code が同時に動いている、という状況が多いです。

この環境で Claude Code のタスクを流している間、ブラウザでドキュメント読んだり Slack 見たり別タブでコード書いたりしていると、タスク完了に気づかないことがありました。
通知は来るのですが、クリックして戻るのが地味に面倒です。

「完了したら勝手にフォーカス戻ってきてくれ」と思い、WezTerm の Lua API で実現しました。

![WezTerm と Claude Code の連携イメージ](/images/2026-02-11-wezterm-claude-code-autofocus/demo.gif)

## 仕組み

```
Claude Code (タスク完了 or 入力待ち)
  |
  v
Stop / Notification hook: OSC 1337 SetUserVar を送信
  |
  v
WezTerm: user-var-changed イベント発火
  |
  +-- pane:tab():activate()  ... タブ切り替え
  +-- pane:activate()        ... ペインをアクティブに
  +-- window:focus()         ... WezTerm 内でのウィンドウフォーカス
  |
  v
hook: osascript で WezTerm を activate
  |
  v
macOS が WezTerm ウィンドウを最前面に
```

ポイントは 2 段階になっていることです。

1. **WezTerm Lua API** (`user-var-changed`) -- タブ/ペインの正確な特定と切替
2. **osascript** (`tell application "WezTerm" to activate`) -- OS レベルでのウィンドウ前面化

WezTerm の `window:focus()` だけだと OS レベルのウィンドウ前面化が安定しなかったため、osascript を併用しています。

## 設定

### WezTerm の Lua 設定

`~/.config/wezterm/wezterm.lua`（または `~/.wezterm.lua`）に以下を追加します。

```lua
local wezterm = require("wezterm")

wezterm.on("user-var-changed", function(window, pane, name, value)
    if name == "claude_code_stop" then
        pane:tab():activate()
        pane:activate()
        window:focus()
    end
end)
```

### Claude Code の hooks 設定

まず `~/.claude/hooks/focus-wezterm.sh` を作成します。

```bash
#!/bin/bash
# WezTerm の user-var-changed イベントを発火させ、Claude Code が動いているペイン/タブを活性化する。
printf '\033]1337;SetUserVar=claude_code_stop=%s\007' MQ== > /dev/tty

# WezTerm ウィンドウ自体を前面に持ってくる
osascript -e 'tell application "WezTerm" to activate'
```

```bash
chmod +x ~/.claude/hooks/focus-wezterm.sh
```

次に `~/.claude/settings.json` の hooks に Stop と Notification を追加します。

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$HOME/.claude/hooks/focus-wezterm.sh\""
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$HOME/.claude/hooks/focus-wezterm.sh\""
          }
        ]
      }
    ]
  }
}
```

- **Stop**: タスク完了時に発火
- **Notification**: 人間の判断を仰ぐとき（確認ダイアログなど）に発火

## 技術的な補足

### OSC 1337 SetUserVar について

OSC (Operating System Command) 1337 は iTerm2 が定義した独自エスケープシーケンスで、WezTerm もサポートしています。
`SetUserVar` を使うとペインに紐づくユーザー変数を設定できます。

```bash
printf '\033]1337;SetUserVar=変数名=%s\007' $(echo -n '値' | base64)
```

- `\033]` -- OSC の開始
- `1337` -- iTerm2 互換プロトコル
- `SetUserVar=名前=値` -- ユーザー変数の設定（値は base64 必須）
- `\007` -- OSC の終端 (BEL)

### user-var-changed イベント

WezTerm は `SetUserVar` を受信すると `user-var-changed` イベントを発火します。
ハンドラに渡される `pane` は **OSC を送信したペインそのもの**なので、どのタブのどのペインで Claude Code が動いているか正確に特定できます。

（詳しくは [公式ドキュメント](https://wezterm.org/config/lua/window-events/user-var-changed.html) をご覧ください。）

### osascript が必要な理由

WezTerm の `window:focus()` は WezTerm 内でのウィンドウフォーカスを制御しますが、別アプリがフォアグラウンドの場合に macOS レベルで WezTerm を前面に持ってくる動作が安定しませんでした。

`osascript -e 'tell application "WezTerm" to activate'` は AppleScript 経由でアプリをアクティベートする標準的な方法で、確実に WezTerm を最前面に持ってきます。

## 参考

- [WezTerm user-var-changed 公式ドキュメント](https://wezterm.org/config/lua/window-events/user-var-changed.html)
- [WezTerm tab:activate() / pane:activate() 追加 (Issue #3217)](https://github.com/wezterm/wezterm/issues/3217)
- [Claude Code Hooks ガイド（公式）](https://code.claude.com/docs/ja/hooks-guide)
