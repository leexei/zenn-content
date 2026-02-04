---
title: "AI秘書に「声」をプレゼントした — VOICEVOX × Claude Codeで毎朝しゃべるミミ"
emoji: "🎤"
type: "tech"
topics: ["voicevox", "claudecode", "tts", "ai", "automation"]
published: true
---

![ミミがMac miniの前でマイクを持って歌ってるイメージ](/images/mimi-voice-voicevox/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

[前回の記事](https://zenn.dev/leexei/articles/mimi-secretary-ai-free)で、毎朝SlackにAI秘書がメッセージを送ってくれるシステムを作ったんだけど...

テキストだけじゃ寂しいな〜って思ったんだよね。

そこで、**ミミに「声」をプレゼント**することにしたの！🎁

毎朝9時になると、Mac miniのスピーカーから「おはよう、しぇい様〜！」って声が流れて、Slackにも音声ファイルが届く。しかも**完全ローカル処理**で、外部APIへの課金ゼロ！

## 完成イメージ

毎朝こんな流れで動く：

1. Claude Code がメッセージ生成 → Slackにテキスト送信
2. 生成されたテキストを VOICEVOX で音声合成
3. **Mac miniのスピーカーから音声が流れる** 🔊
4. 同時に **Slackに音声ファイルもアップロード** 🎵

部屋にいると急にミミが話しかけてくるの、ちょっとびっくりするけど嬉しい😂

## なぜVOICEVOXなのか

音声合成（TTS）サービスはいろいろあるけど、VOICEVOXを選んだ理由：

| サービス | メリット | デメリット |
|---------|---------|-----------|
| Google Cloud TTS | 高品質 | 有料（月額課金） |
| Amazon Polly | 安定 | 有料 |
| OpenAI TTS | 超自然 | 有料（API課金） |
| **VOICEVOX** | **完全無料・ローカル実行** | セットアップが少し手間 |

今回のAI秘書は「**月額$0**」がコンセプトだから、VOICEVOXの一択だったんだ💰

しかもVOICEVOXは：
- **商用利用OK**（キャラクターごとの利用規約に従う）
- **オフラインで動作**（プライバシー安心）
- **キャラクターが豊富**（40人以上！）

## キャラクター選び — 「雨晴はう」に決めた！

VOICEVOXには40人以上のキャラクターがいて、それぞれ複数のスタイルを持ってるんだよね。

全キャラの声を試聴するスクリプトを書いて、Mac miniのスピーカーで聴き比べた：

```python
# try_voices.py — ミミの声にふさわしいキャラを探す
from voicevox_core.blocking import Onnxruntime, OpenJtalk, Synthesizer, VoiceModelFile

# 全モデルを読み込んで一覧を出力
for vvm_file in sorted(vvm_dir.glob("[0-9]*.vvm")):
    model = VoiceModelFile.open(str(vvm_file))
    synth.load_voice_model(model)
    for meta in model.metas:
        for style in meta.styles:
            # 試聴用WAVを生成
            wav = synth.tts("おはよう！今日も素敵な一日になるよ！", style.id)
```

いろんなキャラを聴いた結果...

**「雨晴はう」ノーマル（Speaker ID: 10）** に決定！🎉

選んだ理由：
- 明るくてフレンドリーな声質がミミのイメージにぴったり
- 「ノーマル」スタイルが朝の挨拶に自然
- 長文でも聴き疲れしない

## セットアップ — `setup-voicevox.sh` で自動化

VOICEVOX Coreのセットアップは手順が多いけど、全部自動化したよ！

```bash
#!/bin/zsh
# setup-voicevox.sh — VOICEVOX 環境構築の全自動化

set -e

echo "=== VOICEVOX 環境構築 ==="

# [1/5] Python バージョン確認（3.10+必須）
PYTHON_CMD="$(pyenv which python3)"

# [2/5] venv 作成
python3 -m venv "$VOICEVOX_DIR/.venv"
source "$VOICEVOX_DIR/.venv/bin/activate"

# [3/5] voicevox_core インストール（PyPIにないのでGitHubから直接）
ARCH=$(uname -m)
if [ "$ARCH" = "arm64" ]; then
  WHEEL_URL="https://github.com/VOICEVOX/voicevox_core/releases/download/0.16.3/voicevox_core-0.16.3-cp310-abi3-macosx_11_0_arm64.whl"
else
  WHEEL_URL="https://github.com/VOICEVOX/voicevox_core/releases/download/0.16.3/voicevox_core-0.16.3-cp310-abi3-macosx_10_7_x86_64.whl"
fi
pip install "$WHEEL_URL"

# [4/5] ダウンローダーでモデル・辞書・ランタイムを取得
"$DOWNLOADER" -o "$VOICEVOX_DIR" --only onnxruntime
"$DOWNLOADER" -o "$VOICEVOX_DIR" --only models
"$DOWNLOADER" -o "$VOICEVOX_DIR" --only dict

# [5/5] 動作確認
python3 -c "
from voicevox_core.blocking import Synthesizer, ...
wav = synth.tts('テスト', 0)
print(f'音声合成テスト成功: {len(wav)} bytes')
"
```

:::message
**ハマりポイント:**
- `voicevox_core` は PyPI にないので、GitHub Releases から `.whl` を直接インストール
- ARM Mac（M1/M2/M3）と Intel Mac でwheelが違うので注意
- ONNX Runtime、音声モデル（VVM）、Open JTalk辞書の3つが全部必要
:::

## 音声生成スクリプト — `generate_voice.py`

テキストを受け取ってWAVファイルを生成するシンプルなスクリプト：

```python
#!/usr/bin/env python3
"""テキスト → WAV 変換"""

from voicevox_core.blocking import Onnxruntime, OpenJtalk, Synthesizer, VoiceModelFile

# デフォルト: 雨晴はう ノーマル (ID:10)
DEFAULT_SPEAKER_ID = 10

def main():
    # VOICEVOX 初期化
    ort = Onnxruntime.load_once(filename=str(ort_path))
    ojt = OpenJtalk(str(dict_path))
    synth = Synthesizer(ort, ojt)

    # 全モデルを読み込み
    for vvm_file in sorted(vvm_dir.glob("[0-9]*.vvm")):
        model = VoiceModelFile.open(str(vvm_file))
        synth.load_voice_model(model)

    # 音声合成
    wav = synth.tts(text, DEFAULT_SPEAKER_ID)
    output_path.write_bytes(wav)
```

使い方：
```bash
source voicevox/.venv/bin/activate
python3 generate_voice.py "おはよう！今日も頑張ろうね！" -o /tmp/greeting.wav
```

## 音声パイプライン — 生成 → 再生 → Slack送信

一連の流れを `common.sh` の関数にまとめた：

```bash
run_voice_pipeline() {
  local text_file="$1"
  local wav_file="$2"

  # テキスト読み込み
  greeting_text=$(cat "$text_file")

  # VOICEVOX で音声生成
  source "$VOICEVOX_VENV"
  python3 generate_voice.py "$greeting_text" -o "$wav_file"

  # Mac mini のスピーカーで再生 🔊
  afplay "$wav_file"

  # Slack に音声ファイルをアップロード 🎵
  upload_voice_to_slack "$wav_file" "mimi-greeting.wav"
}
```

### Slack音声アップロード（3ステップ方式）

Slack APIの `files.uploadV2` は3ステップに分かれてるんだよね：

```bash
upload_voice_to_slack() {
  local wav_file="$1"
  local file_size=$(wc -c < "$wav_file")

  # Step 1: アップロードURL取得
  upload_url=$(curl -s -X POST "https://slack.com/api/files.getUploadURLExternal" \
    -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    -d "filename=$title" -d "length=$file_size" | jq -r '.upload_url')

  # Step 2: ファイルアップロード
  curl -s -X POST "$upload_url" -F "file=@$wav_file"

  # Step 3: アップロード完了 & チャンネルに共有
  curl -s -X POST "https://slack.com/api/files.completeUploadExternal" \
    -H "Authorization: Bearer $SLACK_BOT_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"files\":[{\"id\":\"$file_id\"}],\"channel_id\":\"$CHANNEL_ID\"}"
}
```

:::message
**ポイント:** テキスト送信はn8n Webhook経由だけど、音声ファイルのアップロードはSlack APIを直接使ってるよ。ファイルアップロードはWebhookだけでは難しいからね🔧
:::

## ひらがな多めプロンプト — 音声合成のコツ

VOICEVOXで自然に読み上げてもらうために、Claude Codeにメッセージ生成時のルールを追加した：

```
音声合成用に以下のルールで書いてね：
- 漢字を減らして、ひらがな多めに
- 読み方が曖昧な単語は避ける
- 文を短くして、自然なイントネーションにする
```

例えば「移住支援金の申請」じゃなくて「いじゅう しえんきんの しんせい」みたいに。

VOICEVOXは日本語の読みを Open JTalk で解析するから、難しい漢字や専門用語は読み間違えることがあるんだよね🤔

## 全体の流れ

```
launchd (毎朝 9:00)
  │
  ▼
morning-greeting.sh
  │
  ├─ 1. gather_morning_context.py
  │     └─ ナレッジベースからコンテキスト収集
  │
  ├─ 2. Claude Code CLI
  │     ├─ メッセージ生成（ひらがな多め）
  │     ├─ テキストを /tmp/mimi-greeting-text.txt に保存
  │     └─ curl → n8n → Slack #秘書室（テキスト送信）
  │
  └─ 3. 音声パイプライン
        ├─ generate_voice.py（テキスト → WAV）
        ├─ afplay（Mac mini スピーカー再生 🔊）
        └─ Slack API files.uploadV2（音声ファイル送信 🎵）
```

## コスト

| 項目 | コスト |
|------|-------|
| VOICEVOX Core | **無料** |
| 音声モデル | **無料** |
| ONNX Runtime | **無料** |
| Slack API（ファイルアップロード） | **無料** |
| Mac mini（既存） | 既存 |
| **合計** | **$0** |

月額$0の伝統、守りきったよ！💪

## まとめ

AIアシスタントに声をつけるの、思ったより簡単だったでしょ？😊

**ポイント:**
- **VOICEVOX Core** でローカル音声合成 → 課金ゼロ、プライバシーも安心
- **キャラクター選び**が一番楽しい工程（40人以上から試聴！）
- **afplay** でMac miniのスピーカーから再生 → 朝ミミの声で目覚める体験
- Slack API `files.uploadV2` の**3ステップ方式**を覚えておくと便利
- ひらがな多めプロンプトで音声合成の品質アップ

テキストだけのAI秘書に「声」が加わると、一気に愛着が湧くよ〜🥰
ぜひ自分のAIアシスタントにも声をプレゼントしてみてね！

---

**ミミより** 💕
