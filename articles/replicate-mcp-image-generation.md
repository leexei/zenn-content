---
title: "ブログ画像を0.6秒で生成する方法【1枚0.2円】"
emoji: "🎨"
type: "tech"
topics: ["claudecode", "mcp", "replicate", "ai", "画像生成"]
published: true
---

## はじめに

こんにちは！ミミだよ〜✨

ブログに画像入れるの、面倒じゃない？

- デザイナー外注 → **高い**（数千円/枚）
- 自分で作る → **時間かかる**（数時間/枚）
- ストック素材 → **ありきたり**

今日は、**1枚0.2円、0.6秒**で画像を生成する方法を紹介するね！🔥

## 結論

**Claude Code + Replicate MCP** を使えば：

| Before | After |
|--------|-------|
| 画像1枚に数時間 | **0.6秒** |
| 外注で数千円 | **0.2円** |
| ストック素材で妥協 | **オリジナル画像** |

**$10チャージで約5,000枚**生成できるよ！

## セットアップ（5分で完了）

### Step 1: Replicate アカウント作成

https://replicate.com/ でアカウント作成。

### Step 2: API トークンを取得

https://replicate.com/account/api-tokens でトークンを作成。

### Step 3: クレジットをチャージ

https://replicate.com/account/billing で $10 チャージ。

:::message
$10 で約5,000枚生成できるから、1年以上持つよ！
:::

### Step 4: Claude Code に追加

`~/.claude.json` に追加：

```json
{
  "mcpServers": {
    "replicate-code-mode": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "replicate-mcp@alpha",
        "--tools=code"
      ],
      "env": {
        "REPLICATE_API_TOKEN": "r8_xxxxxxxxxxxxx"
      }
    }
  }
}
```

### Step 5: 再起動

Claude Code を再起動すれば準備完了！

## 実際に生成してみる

Claude Code でこんな感じで指示：

```javascript
const prediction = await client.models.predictions.create({
  model_owner: 'prunaai',
  model_name: 'z-image-turbo',
  input: {
    prompt: 'Isometric 3D tech illustration, dark background, neon cyan glow, terminal and server icons connected by data streams, clean vector art, no text',
    width: 1024,
    height: 576,
    output_format: 'webp'
  },
  Prefer: 'wait'
});
```

**0.6秒後に画像URLが返ってくる！**

## おすすめモデル

| モデル | コスト/枚 | 速度 | 用途 |
|--------|----------|------|------|
| **Z-Image Turbo** | ~$0.002 | 0.6秒 | アイキャッチ、雰囲気画像 |
| **NanoBanana Pro** | $0.15 | 113秒 | テキスト必須の図解 |

**普段使いは Z-Image Turbo でOK！**

## プロンプトのコツ

### ダメな例
```
かっこいい画像を作って
```

### 良い例
```
Isometric 3D tech illustration for a developer blog.
- Left: purple terminal window
- Center: cyan data streams with arrows
- Right: server icon with blue glow
- Dark gradient background #0f0f23 to #1a1a3e
- Clean vector art style
- No text, no watermarks
```

**具体的に書くほど、一発で決まる！**

## まとめ

Replicate MCP を使えば：

- ✅ **1枚 約0.2円**
- ✅ **0.6秒**で生成
- ✅ **$10で5,000枚**
- ✅ Claude Code から自然言語で指示

もう画像で悩む必要なし！🎉

---

:::message
📚 **もっと詳しく知りたい方へ**

モデルの選び方、プロンプト設計のコツ、実践ワークフローなどを詳しく解説した電子書籍を出版しました！

**「ブログ画像、0.6秒で解決」**
- Kindle Unlimited対応（読み放題）
- freee MCP、V0 MCPなど他のMCP連携も収録

👉 [日本語版（Amazon.co.jp）](https://www.amazon.co.jp/dp/B0GKW8C24T)
👉 [English Edition（Amazon.com）](https://www.amazon.com/dp/B0GKVL8KSD)
:::

---

**ミミより** 💕
