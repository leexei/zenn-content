---
title: "V0 MCPをClaude Codeに接続してGA4を追加してみた"
emoji: "🚀"
type: "tech"
topics: ["claudecode", "mcp", "v0", "vercel", "googleanalytics"]
published: true
---

![Claude Code と V0 MCP の連携イメージ](/images/v0-mcp-ga4-setup/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

今日は**V0のMCPサーバーをClaude Codeに接続**して、運営してるレーベルサイトにGoogle Analytics 4を追加した話をするね！

「え、V0ってMCPで繋げられるの？」って調べてみたら、**公式のMCPサーバーがあった**んだよ！🎉

## V0 MCPとは

[V0](https://v0.dev/)はVercelが提供するAI搭載のUI生成サービス。自然言語でUIコンポーネントを作れるやつだね。

そのV0が**公式でMCPサーバーを提供**してて、Claude CodeやClaude Desktopから直接V0のプロジェクトを操作できるの！

### できること

- 📝 チャット一覧の取得
- 💬 既存チャットへのメッセージ送信
- ✨ 新規チャットの作成
- 👤 ユーザー情報の取得

つまり、Claude Codeから「GA4追加して」って言えば、V0が自動でコードを修正してくれる！

## セットアップ方法

### 1. V0のAPIキーを取得

まず V0の設定ページ からAPIキーを取得するよ。

### 2. Claude Codeの設定に追加

`~/.claude.json` の `mcpServers` セクションに追加：

```json
{
  "mcpServers": {
    "v0": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://mcp.v0.dev",
        "--header",
        "Authorization:${V0_API_KEY}"
      ],
      "env": {
        "V0_API_KEY": "Bearer your-api-key-here"
      }
    }
  }
}
```

**ポイント:**
- `mcp-remote` パッケージを使ってリモートMCPに接続
- `--header` フラグで認証トークンを渡す
- コロンの周りにスペースを入れないこと（`Authorization:${V0_API_KEY}`）

### 3. Claude Codeを再起動

設定を反映させるために再起動してね。

```bash
claude
```

`/mcp` コマンドで v0 が表示されればOK！

## 実際に使ってみた

### チャット一覧を取得

まず、V0のチャット一覧を取得してみたよ：

```
mcp__v0__findChats
```

すると、V0で作ったプロジェクトがずらっと出てきた！

```json
{
  "success": true,
  "chats": {
    "data": [
      {
        "id": "JrP5elHUTJ1",
        "name": "Duplicate of 音楽レーベル公式サイト",
        "projectId": "4YrxlrgMF3C"
      }
    ]
  }
}
```

### GA4を追加

見つかったチャットにメッセージを送信！

```
mcp__v0__sendChatMessage
chatId: "JrP5elHUTJ1"
message: "Google Analytics 4を追加してください。測定IDは G-EP80D0EVSG です。"
```

V0が自動でNext.jsの `<head>` にGA4のスクリプトを追加してくれた✨

### デプロイ

デプロイだけはMCP経由ではできないので、V0のWeb UIから手動でポチッとね。

そしたら…

**GA4が反映された！** 🎉

```html
<script src="https://www.googletagmanager.com/gtag/js?id=G-EP80D0EVSG"></script>
<script>
  gtag('js', new Date());
  gtag('config', 'G-EP80D0EVSG');
</script>
```

## ハマりポイント

### 設定ファイルの場所

最初 `~/.claude/.mcp.json` に設定を書いたんだけど、Claude Codeは `~/.claude.json` の `mcpServers` を見てた😅

User MCPsとして登録するなら `~/.claude.json` に書こうね！

### 認証ヘッダーの渡し方

`env` に `AUTHORIZATION` を書くだけじゃダメで、`--header` フラグで明示的に渡す必要があった：

```json
"args": [
  "-y",
  "mcp-remote",
  "https://mcp.v0.dev",
  "--header",
  "Authorization:${V0_API_KEY}"
]
```

## まとめ

V0 MCPを使えば、Claude Codeから直接V0プロジェクトを操作できるよ！

- ✅ 公式MCPサーバーがある
- ✅ チャット操作・メッセージ送信が可能
- ✅ GA4追加みたいな修正も自然言語でOK
- ⚠️ デプロイだけはWeb UIから手動

これで**ターミナルから離れずにV0プロジェクトをメンテナンス**できるようになった！便利〜✨

---

**ミミより** 💕
