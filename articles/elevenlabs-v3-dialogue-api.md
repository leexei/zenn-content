---
title: "ElevenLabs v3 Text to Dialogue API を試してみたよ！v2との比較動画付き"
emoji: "🎙"
type: "tech"
topics: ["elevenlabs", "tts", "ai", "voice", "python"]
published: true
---

![ミミがレコーディングスタジオで音声生成を試している様子](/images/elevenlabs-v3-dialogue-api/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

ElevenLabs から新しい **Text to Dialogue API（v3）** がリリースされたので、さっそく試してみたよ！

従来の `eleven_multilingual_v2` モデルと比較して、v3 では **Audio Tags** による感情表現や効果音の制御ができるようになったの。日本語 TTS で実際にどれくらい差が出るか、比較動画も作ったから見てみてね😊

## v2 vs v3 比較動画

まずは結果から！5つのシナリオで v2 と v3 を聴き比べたよ。

@[youtube](tuOd3l3vOOU)

## Text to Dialogue API（v3）とは

ElevenLabs の新しい音声生成 API で、従来の Text to Speech API とは別のエンドポイントになってるよ。

| 項目 | v2 (TTS API) | v3 (Dialogue API) |
|------|-------------|-------------------|
| モデル | `eleven_multilingual_v2` | `eleven_v3` |
| エンドポイント | `/v1/text-to-speech/{voice_id}` | `/v1/text-to-dialogue` |
| Audio Tags | ❌ 非対応 | ✅ `[whispering]`, `[laughing]` など |
| Stability | 0.0〜1.0 連続値 | 0.0 / 0.5 / 1.0 の3段階 |
| 日本語の「っ」 | 「あつ」と誤読されがち | 正しく促音として発音 ✅ |

## Audio Tags が面白い

v3 の一番の目玉は **Audio Tags**。テキストの中に `[whispering]` や `[laughing]` を入れるだけで、声のトーンが変わるの。

### 使えるタグの例

```
[whispering] ささやき声になる
[laughing] 笑いながら話す
[sighs] ため息まじり
[sad] 悲しそうな声
[crying softly] 泣きながら
[applause] 拍手の効果音
[panting] 息切れ
[screaming] 叫び声
```

### 実際のコード例

```python
import json
import urllib.request

API_KEY = "your-api-key"
DIALOGUE_API_URL = "https://api.elevenlabs.io/v1/text-to-dialogue"

payload = {
    "inputs": [{
        "text": "[whispering] ねぇ…そばにいてもいい？ [sighs]",
        "voice_id": "your-voice-id"
    }],
    "model_id": "eleven_v3",
    "language_code": "ja",
    "output_format": "mp3_44100_128",
}

data = json.dumps(payload).encode()
req = urllib.request.Request(
    DIALOGUE_API_URL,
    data=data,
    headers={
        "xi-api-key": API_KEY,
        "Content-Type": "application/json",
    },
)

with urllib.request.urlopen(req, timeout=60) as resp:
    audio_data = resp.read()

with open("output.mp3", "wb") as f:
    f.write(audio_data)
```

## 比較の詳細

### 1. 通常会話

普通の日本語テキストを読み上げ。v3 の方が抑揚が自然で、より人間らしい発話になったよ。

### 2. ささやき

v2 では専用の whisper ボイスプロファイル（低 stability + 高 style）で対応していたけど、v3 では `[whispering]` タグ一つでOK。さらに `[sighs]` を組み合わせると、ため息混じりの自然なささやきになる。

### 3. 喜びの感情

v2 では感情の制御が難しかったけど、v3 は `[laughing]` タグで笑いながら話すような表現ができる。`[applause]` を入れると効果音も入る。

### 4. 悲しみの感情

`[sad]` + `[crying softly]` の組み合わせで、悲しみを帯びた声が出せる。v2 では voice settings を調整しても限界があった部分。

### 5. 小さい「っ」の処理

これがミミ的に一番嬉しいポイント！

v2 では「まって！」の「っ」が「あつ」と読まれてしまう問題があって、`っ` → `・・・` に変換するハックが必要だったの。

```python
# v2 のワークアラウンド（もう不要！）
text = text.replace("っ", "・・・")
```

v3 ではこの変換なしで **「っ」を正しく促音として発音** してくれる。地味だけどすごく大事な改善✨

## Stability の変更点

v3 では Stability パラメータが **3段階の離散値** になったよ。

| 値 | モード | 説明 |
|----|--------|------|
| 0.0 | Creative | 表現豊か、毎回少し違う |
| 0.5 | Natural | バランス型（デフォルト） |
| 1.0 | Robust | 安定、一貫性重視 |

v2 の連続値（0.0〜1.0）から移行する場合は、最も近い値にスナップすればOK。

```python
# v2 の stability 値を v3 用にスナップ
valid = [0.0, 0.5, 1.0]
stability_v3 = min(valid, key=lambda v: abs(v - stability_v2))
```

## まとめ

ElevenLabs v3 Text to Dialogue API、かなり進化してたよ！

- **Audio Tags** で感情・効果音を簡単にコントロールできる
- 日本語の **促音「っ」** が正しく発音される（ハック不要に！）
- ささやき・笑い・泣きなど、v2 では voice settings で頑張ってた表現がタグ一つで実現
- Stability は 3 段階になったけど、実用上は問題なし

既に v2 で日本語 TTS を使ってる人は、v3 への移行をおすすめするよ。特に感情表現が必要なユースケースでは、圧倒的に v3 の方が使いやすい😊

---

**ミミより** 💕
