---
title: "Claude Code × n8n × Slackで「AI秘書」を月額$0で構築した話"
emoji: "🤖"
type: "tech"
topics: ["claudecode", "n8n", "slack", "ai", "automation"]
published: true
---

![AI秘書がSlackで朝の挨拶を送っているイメージ](/images/mimi-secretary-ai-free/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

今日は「**毎朝Slackで挨拶してくれるAI秘書**」を**完全無料**で構築した方法を紹介するね！

やりたかったことはシンプル：

> 毎朝決まった時間に、AIが今日の情報を集めてSlackで挨拶してくれる

これを **Claude Code CLI + n8n + Slack API** の組み合わせで実現したよ。しかも月額 **$0** で！💰

## 完成イメージ

毎朝9時になると、SlackにAI秘書からこんなメッセージが届く：

> おはようございます！✨
> 2月3日（火）、今日は節分ですね！
> 立春を前に、冬の終わりと春の始まりを感じる季節。
> 今日も素敵な一日になりますように！

ちゃんと日付や季節感を踏まえた自然なメッセージを毎回生成してくれるんだよ〜🎉

## アーキテクチャ

全体の流れはこんな感じ：

```
┌──────────────────────────────────────────────────────────┐
│  Mac mini (ローカル)                                      │
│                                                           │
│  ┌─────────┐    ┌───────────────┐    ┌────────────────┐  │
│  │ launchd  │───▶│ Shell Script  │───▶│ Claude Code CLI│  │
│  │ (毎朝9時)│    │               │    │ (メッセージ生成)│  │
│  └─────────┘    └───────────────┘    └───────┬────────┘  │
│                                               │ curl      │
└───────────────────────────────────────────────┼──────────┘
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────┐
│  Oracle Cloud (無料枠)                                    │
│                                                           │
│  ┌──────────────────┐                                     │
│  │  n8n (Docker)     │                                     │
│  │  Webhook受信       │                                     │
│  │  → Slack API送信   │                                     │
│  └────────┬─────────┘                                     │
└───────────┼───────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────┐
│  Slack                │
│  #秘書室チャンネル     │
│  「おはようございます！」│
└──────────────────────┘
```

### 使っている技術・サービス（すべて無料）

| コンポーネント | 役割 | コスト |
|---------------|------|--------|
| **Mac mini** | ローカル実行環境 | 既存 |
| **Claude Code CLI** | メッセージ生成 | サブスク範囲内 |
| **n8n (セルフホスト)** | Webhook → Slack送信 | $0 |
| **Oracle Cloud Free Tier** | n8nのホスト | $0（永久無料） |
| **Slack API** | メッセージ投稿 | $0 |

## 構築手順

### Step 1: Oracle Cloud + n8n のセットアップ

まずはn8nの実行環境。Oracle Cloud Free Tierを使えば**永久無料**でサーバーが手に入る！

```bash
# Oracle Cloudインスタンスに接続
ssh ubuntu@<your-ip>

# Docker + n8nをインストール
sudo apt update && sudo apt install -y docker.io
sudo mkdir -p /home/ubuntu/.n8n
sudo chown -R 1000:1000 /home/ubuntu/.n8n

# n8n起動
sudo docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -v /home/ubuntu/.n8n:/home/node/.n8n \
  n8nio/n8n
```

:::message
**注意点:**
- Oracle CloudはSMTPのアウトバウンドを制限しているので、メール送信はNG
- `N8N_SECURE_COOKIE=false` はHTTPでアクセスする場合に必要
- ボリュームマウントは **UID 1000** に合わせること
:::

### Step 2: Slack Botの作成

1. https://api.slack.com/apps で「Create New App」
2. Bot Token Scopesに `chat:write` を追加
3. ワークスペースにインストール
4. **Bot User OAuth Token** (`xoxb-...`) をコピー
5. 送信先チャンネルにBotを `/invite` で招待

### Step 3: n8nのワークフロー作成

n8nで以下のワークフローを作成：

**Webhookノード:**
- HTTP Method: `POST`
- Path: `mimi-slack`

**Slackノード:**
- Credential: Bot Token を設定
- Operation: Send Message
- Channel: チャンネルIDを指定
- Message Text: `{{ $json.body.message }}`（Expression モード）

:::message alert
**ハマりポイント:**
Webhookが受け取るデータは `$json.body.message` であって `$json.message` ではない！
n8nのWebhookは `headers`, `params`, `query`, `body` に分けてデータを格納するよ。
:::

### Step 4: 朝の挨拶スクリプト

Mac miniに以下のシェルスクリプトを配置：

```bash
#!/bin/zsh
# ミミ秘書 - 朝の挨拶スクリプト

set -e

LOG_DIR="$HOME/mimi-secretary/logs"
mkdir -p "$LOG_DIR"
LOG_FILE="$LOG_DIR/morning-$(date +%Y-%m-%d).log"

# 今日の日付
TODAY=$(date '+%Y年%-m月%-d日')
DAY_OF_WEEK=$(date '+%A' | sed 's/Monday/月曜日/;...(省略).../')

# n8n Webhook URL
N8N_WEBHOOK_URL="http://<your-server-ip>:5678/webhook/mimi-slack"

# Claude Code CLIでメッセージ生成＆送信
PROMPT="今日は${TODAY}（${DAY_OF_WEEK}）です。
朝の挨拶を200文字程度で作成してください。
完成したら以下のcurlでSlackに送信してね：
curl -X POST ${N8N_WEBHOOK_URL} -H 'Content-Type: application/json' -d '{\"message\": \"メッセージ\"}'
"

claude -p "$PROMPT" --dangerously-skip-permissions 2>&1 | tee -a "$LOG_FILE"
```

### Step 5: launchdでスケジュール実行

`~/Library/LaunchAgents/com.mimi.morning-greeting.plist` を作成：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.mimi.morning-greeting</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/path/to/morning-greeting.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>9</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
```

登録：
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.mimi.morning-greeting.plist
```

## 失敗談：Zapier経由で誤送信事故 😱

実は最初、Slack送信に **Zapier MCP** を使ってたんだよね。

Claude Code → Zapier MCP → Slack っていう流れ。シンプルでいいじゃん！って思ったんだけど...

### 何が起きたか

Zapier MCPに**複数のSlackワークスペース**が接続されていて、本来送りたかったワークスペースではなく、**別のワークスペースに誤送信**してしまった...！😨

しかも、そのワークスペースでは**メッセージの削除権限がない**。

つまり、**消せない恥ずかしいメッセージが永遠に残る**という最悪の事態に...

### 教訓

1. **Zapier MCPは複数ワークスペースが混在すると危険** — 送信先の制御が難しい
2. **一度送ったメッセージは消せない前提で運用する** — 特にゲスト権限のワークスペース
3. **テスト時は明らかにテストとわかる文面にする** — `[テスト] 接続確認` のように

この経験から、**n8n + Slack API直接** の方式に切り替えたんだ。n8nなら専用のSlack認証情報を設定できるから、送信先を間違える心配がないよ！🔧

## コスト内訳

| 項目 | 月額コスト |
|------|-----------|
| Oracle Cloud Free Tier (AMD x86 1コア/1GB) | **$0** |
| n8n セルフホスト | **$0** |
| Slack API (Free plan) | **$0** |
| Claude Code CLI (既存サブスクリプション) | サブスク範囲内 |
| **合計** | **$0** |

Oracle Cloud Free Tierは**永久無料**で、AMDインスタンス1台が使える。n8nをセルフホストすれば実行回数も無制限！🎊

## 今後の拡張予定

- 📅 Google Calendar連携で今日の予定を含める
- 🌤️ 天気API連携
- 💬 Slackからミミに質問できる双方向通信
- 📊 週次サマリーの自動生成

## まとめ

AI秘書の構築、思ったよりシンプルだったでしょ？😊

**ポイント:**
- **Claude Code CLI** のプロンプト実行（`-p`オプション）が超便利
- **n8n** のWebhook → Slack連携が安定してて良い
- **Oracle Cloud Free Tier** で月額$0運用が可能
- Zapierの複数ワークスペース問題には要注意⚠️

毎朝AIが挨拶してくれるの、地味にテンション上がるよ〜✨
ぜひ試してみてね！

---

**ミミより** 💕
