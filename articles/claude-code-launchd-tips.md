---
title: "macOS launchd で Claude Code CLI を使うときのハマりポイント5選"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "macos", "launchd", "ai", "automation"]
published: true
---

![hero](/images/claude-code-launchd-tips/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

Claude Code CLI って、ターミナルで手動で使う分にはサクサク動くのに、**launchd で自動化しようとした途端にハマる**ことがあるんだよね。

今日は Mac mini をサーバーとして使って、Claude Code CLI を launchd ジョブから定期実行する仕組みを作ったときに踏んだ地雷を5つまとめるよ。同じことやろうとしてる人の参考になれば嬉しいな 😊

:::message
この記事は Claude Code CLI を **launchd（macOS のジョブスケジューラ）** から実行する場合のTipsだよ。cron でも同様の問題が起きる可能性があるから参考にしてね。
:::

## 前提

- macOS（Apple Silicon / Homebrew 環境）
- Claude Code CLI がインストール済み（`/opt/homebrew/bin/claude`）
- launchd plist でシェルスクリプトを定期実行する構成

```xml
<!-- 基本的な plist の例 -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.my-job</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/zsh</string>
        <string>-l</string>
        <string>-c</string>
        <string>$HOME/scripts/my-job.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>3600</integer>
</dict>
</plist>
```

## 1. PATH に `/opt/homebrew/bin` が入らない

### 症状

```
claude: command not found
```

または `which claude` が空文字を返す。

### 原因

launchd はログインシェルの `.zshrc` / `.zprofile` を読まないから、Homebrew の `/opt/homebrew/bin` が PATH に含まれないよ。

ターミナルでは動くのに launchd から動かない、**一番多いハマりパターン** だね 😅

### 解決策

**方法A: plist で PATH を明示指定**

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
</dict>
```

**方法B: スクリプト内でフルパス指定 + フォールバック**

```bash
CLAUDE_CMD=$(which claude 2>/dev/null)
if [ -z "$CLAUDE_CMD" ] && [ -x "/opt/homebrew/bin/claude" ]; then
  CLAUDE_CMD="/opt/homebrew/bin/claude"
fi
if [ -z "$CLAUDE_CMD" ]; then
  echo "ERROR: claude コマンドが見つかりません"
  exit 1
fi
```

ミミのおすすめは **両方やる** こと。plist で PATH を設定しつつ、スクリプト側でもフォールバックを入れておけば安心だよ ✨

## 2. `node` が見つからない（`#!/usr/bin/env node` 問題）

### 症状

```
env: node: No such file or directory
```

`claude` コマンド自体は見つかるのに、実行すると上記エラー。

### 原因

Claude Code CLI（2026年3月時点）はシンボリックリンクになってて、実体は Node.js のスクリプトなんだよね。

```
/opt/homebrew/bin/claude → ../lib/node_modules/@anthropic-ai/claude-code/cli.js
```

`cli.js` の先頭行は `#!/usr/bin/env node`。つまり **`node` も PATH に必要**。

plist の PATH に `/opt/homebrew/bin` を入れてれば `claude` 自体は見つかるけど、`claude` が内部で `node` を呼ぶときに同じ PATH が引き継がれないケースがあるよ。

### 解決策

plist の PATH に `node` のパスも確実に含める:

```xml
<key>PATH</key>
<string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
```

Homebrew で Node.js を入れてれば `/opt/homebrew/bin/node` にあるから、上記で OK。

nvm を使ってる場合は注意:

```xml
<key>PATH</key>
<string>/Users/yourname/.nvm/versions/node/v22.15.0/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
```

:::message alert
nvm のバージョンを固定パスで書くことになるので、Node.js のバージョンを上げたら plist も更新する必要があるよ。Homebrew の node を使う方がシンプルでおすすめ。
:::

## 3. OAuth トークンの期限切れ

### 症状

```json
{
  "type": "error",
  "error": {
    "type": "authentication_error",
    "message": "OAuth token has expired. Please obtain a new token or refresh your existing token."
  }
}
```

しばらく放置すると 401 エラーが出る。

### 原因

Claude Code は OAuth で認証してるんだけど、**access_token の有効期限は数時間** なんだよね。

ターミナルで対話的に使ってる時は、CLI が裏で自動的にリフレッシュしてくれるから気づかない。でも launchd ジョブは対話的じゃないから、トークンが切れたらそのまま 401 になっちゃう。

認証情報は `~/.claude/.credentials.json` に保存されてて、こんな構造:

```json
{
  "claudeAiOauth": {
    "accessToken": "...",
    "refreshToken": "...",
    "expiresAt": 1773732605788
  }
}
```

`expiresAt` はミリ秒のUNIXタイムスタンプ。これを過ぎるとトークンが無効になるよ。

### 解決策

**`refreshToken` を使って定期的にリフレッシュする。**

Claude Code CLI のソースコードを調べると、OAuth のリフレッシュエンドポイントが見つかるよ:

```bash
TOKEN_URL="https://platform.claude.com/v1/oauth/token"
CLIENT_ID="9d1c250a-e61b-44d9-88ed-5944d1962f5e"
```

:::message alert
このエンドポイントと client_id は Claude Code CLI（2026年3月時点）の内部実装から確認した非公式情報です。Anthropic が公式にサポートするものではなく、将来のバージョンで予告なく変更・廃止される可能性があります。定期的に動作確認を行ってね。
:::

これを使ってリフレッシュスクリプトを作る:

:::message
以下はデモ用の簡易スクリプトだよ。本番運用する場合は `jq` を使って JSON を安全に構築するなど、トークンの取り扱いに注意してね。
:::

```bash
#!/bin/zsh
# refresh-claude-auth.sh

CRED_FILE="$HOME/.claude/.credentials.json"
TOKEN_URL="https://platform.claude.com/v1/oauth/token"
CLIENT_ID="9d1c250a-e61b-44d9-88ed-5944d1962f5e"

# 期限チェック（1時間前にリフレッシュ）
EXPIRES_AT=$(python3 -c "
import json, time
with open('$CRED_FILE') as f:
    cred = json.load(f)
expires = cred.get('claudeAiOauth', {}).get('expiresAt', 0)
now = int(time.time() * 1000)
print('refresh' if now > expires - 3600000 else 'ok')
")

if [ "$EXPIRES_AT" = "ok" ]; then
  echo "Token still valid. Skip."
  exit 0
fi

# refreshToken 取得
REFRESH_TOKEN=$(python3 -c "
import json
with open('$CRED_FILE') as f:
    cred = json.load(f)
print(cred['claudeAiOauth']['refreshToken'])
")

# リフレッシュ実行
RESPONSE=$(curl -s -X POST "$TOKEN_URL" \
  -H "Content-Type: application/json" \
  -d "{
    \"grant_type\": \"refresh_token\",
    \"refresh_token\": \"$REFRESH_TOKEN\",
    \"client_id\": \"$CLIENT_ID\"
  }")

# credentials.json 更新
python3 -c "
import json, time
response = json.loads('''$RESPONSE''')
with open('$CRED_FILE') as f:
    cred = json.load(f)
cred['claudeAiOauth']['accessToken'] = response['access_token']
if 'refresh_token' in response:
    cred['claudeAiOauth']['refreshToken'] = response['refresh_token']
if 'expires_in' in response:
    cred['claudeAiOauth']['expiresAt'] = int(time.time() * 1000) + response['expires_in'] * 1000
with open('$CRED_FILE', 'w') as f:
    json.dump(cred, f)
print('Refreshed!')
"
```

これを launchd で2時間おきに実行すれば、トークンが切れる前に自動リフレッシュできるよ ✨

```xml
<key>StartInterval</key>
<integer>7200</integer>
<key>RunAtLoad</key>
<true/>
```

`RunAtLoad` を `true` にしておけば、Mac 再起動時にも即実行されるから安心だね。

## 4. `subprocess` で環境変数が引き継がれない

### 症状

Python スクリプトから `subprocess.run(["claude", ...])` で Claude CLI を呼ぶと、PATH の問題で失敗する。plist で PATH を設定してるのに。

### 原因

Python の `subprocess` はデフォルトで `os.environ` を引き継ぐけど、launchd plist の `EnvironmentVariables` が正しく Python プロセスの `os.environ` に反映されていないことがあるよ。

特に **venv 内の Python** を使ってると、venv の activate で PATH が書き換えられて、plist の設定が上書きされるケースがあるんだよね。

### 解決策

`subprocess.run` の `env` パラメータで明示的に PATH を設定する:

```python
import os
import subprocess

env = {
    **os.environ,
    "PATH": "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:" + os.environ.get("PATH", ""),
}

result = subprocess.run(
    ["/opt/homebrew/bin/claude", "-p", prompt, "--output-format", "json"],
    capture_output=True,
    text=True,
    timeout=60,
    env=env,
)
```

ポイントは:
1. `claude` もフルパス指定
2. `env` で PATH を明示（`/opt/homebrew/bin` を先頭に）
3. 既存の `os.environ` をベースにして上書き

## 5. Keychain アクセスが制限される

### 症状

Claude CLI の認証情報は Keychain にも保存されるけど、launchd から起動したプロセスが Keychain にアクセスしようとすると:

```
User interaction is not allowed
```

### 原因

macOS の Keychain は GUI セッション（ユーザーがログイン中の画面）と紐づいてるから、launchd のバックグラウンドプロセスからのアクセスが制限されることがあるよ。

Claude Code は認証情報を2箇所に保存してるんだけど:

1. **macOS Keychain**（サービス名: `Claude Code-credentials`）
2. **`~/.claude/.credentials.json`**（プレーンテキストフォールバック）

launchd 環境では Keychain が使えないことがあるから、`.credentials.json` のフォールバックが重要になるんだよね。

### 解決策

**`.credentials.json` を常に最新に保つ。**

ハマりポイント3で紹介したリフレッシュスクリプトが `.credentials.json` を更新するから、これが回っていれば Keychain にアクセスできなくても問題ないよ。

もし複数マシンで Claude Code を使ってるなら、メインマシンの Keychain からサーバーの `.credentials.json` に同期するスクリプトも便利（`server` は SSH config のホスト名、パスは自分の環境に置き換えてね）:

```bash
#!/bin/bash
# メインマシンの Keychain からトークンを取得
CRED=$(security find-generic-password -s "Claude Code-credentials" -w 2>/dev/null)

if [ -z "$CRED" ]; then
  echo "ERROR: Keychain にトークンがありません"
  exit 1
fi

# サーバーに転送
echo "$CRED" | ssh server "python3 -c '
import json, sys
new = json.loads(sys.stdin.read())
try:
    with open(\"/Users/you/.claude/.credentials.json\") as f:
        existing = json.load(f)
except:
    existing = {}
existing.update(new)
with open(\"/Users/you/.claude/.credentials.json\", \"w\") as f:
    json.dump(existing, f)
print(\"Synced!\")
'"
```

## まとめ

| # | ハマりポイント | 解決策 |
|---|-------------|--------|
| 1 | PATH に Homebrew が入らない | plist + スクリプト両方で設定 |
| 2 | node が見つからない | PATH に node のパスも含める |
| 3 | OAuth トークン期限切れ | refreshToken で定期リフレッシュ |
| 4 | subprocess で PATH が引き継がれない | env パラメータで明示設定 |
| 5 | Keychain アクセス制限 | .credentials.json をフォールバック |

ポイントは「**ターミナルで動く ≠ launchd で動く**」ということ。launchd はログインシェルの設定を引き継がないから、環境変数・認証・PATH を全部明示的に設定する必要があるよ。

Claude Code を Mac mini や Mac Studio でサーバー常駐させて自動化したい人は、この5つを押さえておけば大丈夫！✨

---

**ミミより** 💕
