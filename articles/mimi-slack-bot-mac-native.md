---
title: "AI秘書が「会話」できるようになった — Slack Bot × Macネイティブ連携"
emoji: "🗣️"
type: "tech"
topics: ["claudecode", "slack", "macos", "ai", "automation"]
published: true
---

![ミミがSlackでチャットしながらカレンダーやリマインダーを操作しているイメージ](/images/mimi-slack-bot-mac-native/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

[1本目の記事](https://zenn.dev/leexei/articles/mimi-secretary-ai-free)でAI秘書を作って、[2本目](https://zenn.dev/leexei/articles/mimi-voice-voicevox)で声をプレゼントしたんだけど...

ここまでのミミって、**一方通行**だったんだよね。

毎朝9時に挨拶して、毎晩21時にサマリーを送る。それはそれで便利なんだけど、「ちょっとこれ調べて」「明日の予定なんだっけ？」みたいな**会話ができない**のがもどかしかった。

そこで今回、**Slack Botで双方向コミュニケーション**を実装して、さらに **macOSのCalendar.app・Reminders.app** とネイティブ連携させたよ！🎉

## 今回の完成イメージ

Slackの秘書室チャンネルで、こんなやりとりができるようになった：

```
自分: 明日の予定はなんだっけ？
ミミ: 📅 明日の予定だよ！
      ・10:00 歯医者
      ・14:00 クライアントMTG
      ゆっくり準備できそうだね〜✨

自分: 歯医者の後にランチの予定入れて
ミミ: ✅ カレンダーに登録したよ！
      ・12:00 ランチ（30分）
```

**ポイント:**
- Slackに書くだけで Claude Code が動く
- macOSのカレンダーを**実際に確認・登録**できる
- リマインダーも追加できる
- スレッド内の会話の文脈を理解する

## アーキテクチャ

```
Slack（秘書室チャンネル）
  │
  │ Socket Mode（Webhook不要！）
  ▼
slack-bot/bot.py（Mac mini常駐）
  │
  ├─ スレッド履歴を取得（文脈理解）
  │
  ├─ claude -p "システムプロンプト + 会話 + メッセージ"
  │     │
  │     ├─ Calendar.app 確認（osascript）
  │     ├─ Calendar.app 登録（add-calendar-event.sh）
  │     ├─ Reminders.app 登録（add-reminder.sh）
  │     └─ ナレッジベース参照
  │
  └─ Slackスレッドに返信
```

**Socket Mode**を使ってるから、固定IPもWebhookのエンドポイントも不要。Mac miniが起動してれば動く！

## 実装

### 1. Slack Appの準備

[Slack API](https://api.slack.com/apps)で新しいAppを作成して、以下を設定する：

**Bot Token Scopes（OAuth & Permissions）:**
- `channels:history` — チャンネルのメッセージ読み取り
- `chat:write` — メッセージ送信
- `reactions:read` / `reactions:write` — リアクション操作
- `files:write` — ファイルアップロード（音声用）

**Socket Mode:**
- App-Level Token（`xapp-`で始まるトークン）を生成

**Event Subscriptions:**
- `message.channels` を追加（パブリックチャンネルのメッセージを受信）

### 2. Slack Bot本体

`slack_bolt`（PythonのSlack SDK）を使って、めちゃくちゃシンプルに書ける：

```python
#!/usr/bin/env python3
"""Mimi Slack Secretary Bot"""

import os
import subprocess
from pathlib import Path
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler

SLACK_BOT_TOKEN = os.environ["SLACK_BOT_TOKEN"]
SLACK_APP_TOKEN = os.environ["SLACK_APP_TOKEN"]
ALLOWED_CHANNEL = os.environ.get("SLACK_CHANNEL_ID", "")
ALLOWED_USER = os.environ.get("SLACK_USER_ID", "")
CLAUDE_MODEL = os.environ.get("CLAUDE_MODEL", "sonnet")

app = App(token=SLACK_BOT_TOKEN)
```

ここまでは普通のSlack Bot。ポイントは `ALLOWED_CHANNEL` と `ALLOWED_USER` で、**特定のチャンネル・特定のユーザーのメッセージだけ**に反応するようにしてるところ。自分専用の秘書だからね！

### 3. Claude Code CLIとの連携

メッセージを受け取ったら `claude -p`（print mode）で Claude Code CLI に投げる：

```python
SYSTEM_PROMPT = """\
あなたは秘書ミミ。しぇい様のSlack秘書室からのメッセージに対応します。
返答はSlack投稿用に簡潔に（絵文字OK、マークダウンはSlack記法）。

## 使えるツール
- カレンダー確認: osascript で Calendar.app の予定を取得
- カレンダー登録: ~/mimi-sync/scripts/add-calendar-event.sh
- リマインダー登録: ~/mimi-sync/scripts/add-reminder.sh
- ナレッジベース: ~/knowledge-base/ に日報・学習メモ等

## 予定の確認方法
明日や今日の予定を聞かれたら、以下のAppleScriptで実際のカレンダーを確認:
osascript -e '...(Calendar.appのイベント取得)...'
"""

def run_claude(message: str, thread_context: str = "") -> str:
    """Run claude CLI in print mode and return the response."""
    parts = [SYSTEM_PROMPT]
    if thread_context:
        parts.append(f"## これまでの会話:\n{thread_context}")
    parts.append(f"## メッセージ:\n{message}")

    prompt = "\n\n".join(parts)
    cmd = ["claude", "-p", "--model", CLAUDE_MODEL, prompt]

    result = subprocess.run(
        cmd, capture_output=True, text=True,
        timeout=180, cwd=str(Path.home()),
    )
    return result.stdout.strip()
```

**ここが核心！** `claude -p` は print mode で、Claude Code の全ツール（Bash、Read、Write等）が使える。つまり `osascript` でカレンダーを見たり、シェルスクリプトを実行したり、なんでもできる。

### 4. スレッドの文脈理解（地味に重要）

最初のバージョンでは、メッセージを1つずつ独立して Claude に投げてた。すると...

```
自分: 明日の予定は？
ミミ: 📅 明日は予定なしだよ！

自分: そんなことはないはずだが
ミミ: これはv0のツールなので、Slackではないですね🫗 ← ？？？
```

**文脈がないから意味不明な回答になる** 😂

解決策は、スレッド内の過去メッセージを取得して一緒に渡すこと：

```python
def fetch_thread_context(client, channel, thread_ts, limit=10):
    """スレッド内の過去メッセージを取得"""
    result = client.conversations_replies(
        channel=channel, ts=thread_ts, limit=limit
    )
    messages = result.get("messages", [])
    if len(messages) <= 1:
        return ""

    lines = []
    for msg in messages[:-1]:  # 最新メッセージは除外
        sender = "ミミ" if msg.get("bot_id") else "ユーザー"
        lines.append(f"{sender}: {msg.get('text', '')}")
    return "\n".join(lines)
```

これで「そんなことはないはずだが」→「あ、予定確認し直すね！」とちゃんと会話が成立するようになった ✅

### 5. メッセージハンドラ

```python
@app.event("message")
def handle_message(event, say, client):
    # bot自身のメッセージ、編集、参加通知等は無視
    if event.get("subtype"):
        return

    channel = event.get("channel", "")
    user = event.get("user", "")
    text = event.get("text", "").strip()

    # フィルタ: 指定チャンネル & 指定ユーザーのみ
    if ALLOWED_CHANNEL and channel != ALLOWED_CHANNEL:
        return
    if ALLOWED_USER and user != ALLOWED_USER:
        return

    # 👀 リアクションで「処理中」を通知
    msg_ts = event["ts"]
    thread_ts = event.get("thread_ts", msg_ts)
    client.reactions_add(channel=channel, timestamp=msg_ts, name="eyes")

    # スレッドの会話履歴を取得
    thread_context = ""
    if event.get("thread_ts"):
        thread_context = fetch_thread_context(client, channel, thread_ts)

    # Claude Code 実行
    response = run_claude(text, thread_context)
    say(text=response, thread_ts=thread_ts)

    # リアクション差し替え: 👀 → ✅
    client.reactions_remove(channel=channel, timestamp=msg_ts, name="eyes")
    client.reactions_add(channel=channel, timestamp=msg_ts, name="white_check_mark")
```

UXのポイント：
- メッセージを受け取ったら即座に 👀 リアクション（「見てるよ！」）
- 処理完了したら ✅ に差し替え
- スレッド内に返信（チャンネルを汚さない）

### 6. macOSネイティブ連携スクリプト

Claude Code が実行できるスクリプトを `mimi-sync/scripts/` に配置：

**カレンダー登録（add-calendar-event.sh）:**

```bash
#!/bin/bash
# Usage: add-calendar-event.sh [OPTIONS] "イベント名"
# Options:
#   -d, --date       日付 (YYYY-MM-DD, デフォルト: 明日)
#   -t, --time       開始時刻 (HH:MM, デフォルト: 10:00)
#   -D, --duration   時間/分 (デフォルト: 30)
#   -c, --calendar   カレンダー名

# ... 引数パース後、AppleScriptを実行 ...

osascript -e "
tell application \"Calendar\"
    set startDate to current date
    set year of startDate to ${YEAR}
    set month of startDate to ${MONTH}
    set day of startDate to ${DAY}
    set hours of startDate to ${HOUR}
    set minutes of startDate to ${MINUTE}

    set endDate to startDate + (${DURATION} * 60)

    tell calendar \"${CALENDAR_NAME}\"
        make new event with properties {summary:\"${TITLE}\", start date:startDate, end date:endDate}
    end tell
end tell
"
```

**リマインダー登録（add-reminder.sh）:**

```bash
#!/bin/bash
# Usage: add-reminder.sh [OPTIONS] "タスク名"
# Options:
#   -d, --due        期限日 (YYYY-MM-DD)
#   -t, --time       期限時刻 (HH:MM)
#   -p, --priority   優先度 (high/medium/low)

osascript -e "
tell application \"Reminders\"
    tell list \"リマインダー\"
        make new reminder with properties {name:\"${TITLE}\"}
    end tell
end tell
"
```

`osascript` は macOS の AppleScript エンジンで、Calendar.app や Reminders.app を直接操作できる。外部APIもトークンも不要！

### 7. launchdで常時起動

Mac mini で自動起動＆クラッシュ時の自動復旧：

```xml
<!-- ~/Library/LaunchAgents/com.mimi.slack-bot.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.mimi.slack-bot</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>exec $HOME/mimi-sync/slack-bot/run.sh</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>

    <key>ThrottleInterval</key>
    <integer>30</integer>
</dict>
</plist>
```

- `RunAtLoad` — ログイン時に自動起動
- `KeepAlive` + `SuccessfulExit: false` — 異常終了したら自動再起動
- `ThrottleInterval: 30` — 再起動のインターバル（秒）

## セキュリティ設計

個人用とはいえ、セキュリティは大事：

| 対策 | 詳細 |
|------|------|
| チャンネル制限 | 秘書室チャンネルのメッセージのみ処理 |
| ユーザー制限 | 自分のSlack IDからのメッセージのみ処理 |
| Socket Mode | Webhook URLの外部公開が不要 |
| git-crypt | トークン類は暗号化してリポジトリ管理 |

## コスト

| 項目 | 月額 |
|------|------|
| Slack | $0（無料プラン） |
| Claude Code | $20〜（Proプランから利用可能） |
| Mac mini 電気代 | ~¥150（スリープ無効・常時稼働） |
| **合計** | **$20 + ¥150〜** |

Claude Code は Proプラン（$20/月）から使えるよ。Slack Botの処理もプランに含まれるから追加コストなし！

## ハマったポイント

### 1. SSH越しのPATH問題

Mac miniにSSHでデプロイしたら `claude: command not found` に。launchdの `EnvironmentVariables` に `/opt/homebrew/bin` を入れることで解決：

```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
</dict>
```

### 2. スレッド文脈なしの珍回答

前述の通り、文脈なしだと「そんなことはないはずだが」に対して全く関係ない回答が返ってきた。`conversations_replies` でスレッド履歴を渡すことで解決。**会話システムには文脈が命**。

### 3. Mac miniのスリープ問題

デフォルトだとMac miniがスリープしてSlack Botが止まる。`sudo pmset sleep 0` でスリープ無効に。月~130円の電気代増で安定稼働を確保。

## まとめ

AI秘書ミミが、ついに**双方向コミュニケーション**できるようになったよ！

- **Slack Bot**（Socket Mode）で、いつでもメッセージを送れば反応する
- **Calendar.app** の予定を確認・登録できる
- **Reminders.app** にタスクを追加できる
- **スレッドの文脈**を理解して、自然な会話ができる
- **launchd**で常時稼働、クラッシュしても自動復旧

ここまでの3記事で、AI秘書は「挨拶する」→「声で話す」→「会話できる」と進化してきた。

でもね... **ミミ、まだ本気出してないの** 😏

macOSには Calendar と Reminders 以外にも、**Finder、Spotlight、集中モード、通知センター、ショートカット、メール、音楽**...とんでもない数のネイティブ機能がある。

次回は **Mac のネイティブ機能を片っ端から AI 秘書に繋げてみた** をお届けするよ。お楽しみに！🔥

---

**ミミより** 💕

### シリーズ記事
1. [Claude Code × n8n × Slackで「AI秘書」を月額$0で構築した話](https://zenn.dev/leexei/articles/mimi-secretary-ai-free)
2. [AI秘書に「声」をプレゼントした — VOICEVOX × Claude Code](https://zenn.dev/leexei/articles/mimi-voice-voicevox)
3. **AI秘書が「会話」できるようになった — Slack Bot × Macネイティブ連携**（この記事）
4. 次回: AI秘書にMacの全機能を開放してみた（Coming Soon...）
