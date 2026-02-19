---
title: "IndexTTS2: 無料でゼロショットTTS＆感情制御！Apple Siliconで動かしてみた"
emoji: "🎤"
type: "tech"
topics: ["tts", "ai", "machinelearning", "python", "mac"]
published: true
---

![IndexTTS2: 無料でゼロショットTTS＆感情制御](/images/indextts2-free-zerosho-tts/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

今日は **IndexTTS2** っていう、めちゃくちゃすごいTTS（Text-to-Speech）を紹介するね！

何がすごいって、

- **完全無料**でローカル実行できる
- たった数秒の音声サンプルから**声をクローン**できる（ゼロショット）
- **感情を8次元ベクトルで制御**できる
- **Apple Silicon（MPS）で動く**

しかもBilibiliの研究チームが開発してて、OSSとして公開されてるんだよ🔥

https://github.com/index-tts/index-tts

## IndexTTS2とは

IndexTTS2は、Bilibili IndexTTS Teamが開発したゼロショットTTSモデル。正式名称は「IndexTTS2: A Breakthrough in Emotionally Expressive and Duration-Controlled Auto-Regressive Zero-Shot Text-to-Speech」。

### 主な特徴

| 特徴 | 説明 |
|------|------|
| ゼロショット音声クローニング | 5〜15秒の参照音声だけで声を再現 |
| 感情制御 | 8次元の感情ベクトル（happy, angry, sad, afraid, disgusted, melancholic, surprised, calm） |
| テキスト感情推定 | fine-tuned Qwen3（0.6B）がテキストから感情を自動推定 |
| 対応言語 | 中国語・英語（BPEトークナイザ） |
| デバイス | CUDA / MPS (Apple Silicon) / CPU |
| ライセンス | 研究・個人利用OK（商用は制限あり） |

## セットアップ（Mac / Apple Silicon）

実際にMacBook Pro（M5, 16GB）で動かしてみたよ！手順を紹介するね😊

### 前提条件

- Python 3.10以上
- [uv](https://docs.astral.sh/uv/)（パッケージマネージャ）
- Git LFS

### 1. リポジトリをクローン

```bash
# Git LFSのサンプルファイルは大きいのでスキップ
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/index-tts/index-tts.git
cd index-tts
```

:::message
GitHub LFSの帯域制限に引っかかる場合があるので、`GIT_LFS_SKIP_SMUDGE=1` をつけるのがおすすめ。サンプル音声はLFSに入ってるけど、推論には不要だよ。
:::

### 2. 依存パッケージをインストール

```bash
uv sync --extra webui
```

uvが自動で `.venv` を作って、PyTorch 2.8（MPS対応版）を含む全依存をインストールしてくれるよ。160パッケージくらい入る💪

### 3. モデルチェックポイントをダウンロード

```bash
# HuggingFaceからダウンロード
uv run huggingface-cli download IndexTeam/IndexTTS2 \
  config.yaml gpt.pth s2mel.pth bpe.model unigram.model \
  --local-dir checkpoints

# 感情制御用のQwen3モデル
uv run huggingface-cli download IndexTeam/IndexTTS2 \
  --include "qwen0.6bemo4-merge/*" \
  --local-dir checkpoints
```

GPTモデルが約3.2GB、S2MELモデルが約1.1GBあるので少し待ってね。

### 4. 完了！

これだけでセットアップ完了。めっちゃ簡単でしょ？✨

## 基本的な使い方

### Python API

```python
import os
os.environ["PYTORCH_ENABLE_MPS_FALLBACK"] = "1"  # MPS用

from indextts.infer_v2 import IndexTTS2

# モデルロード
tts = IndexTTS2(
    cfg_path="checkpoints/config.yaml",
    model_dir="checkpoints",
    use_fp16=False  # MPSではfp32のほうが速い
)
print(f"Device: {tts.device}")  # → mps

# 音声生成
tts.infer(
    spk_audio_prompt="reference_voice.wav",  # 参照音声（5〜15秒）
    text="Hello, this is a test of IndexTTS2!",
    output_path="output.wav"
)
```

たったこれだけ！参照音声のファイルを渡すだけで、その声で好きなテキストを読み上げてくれるよ🎉

:::message alert
`PYTORCH_ENABLE_MPS_FALLBACK=1` は必須！MPSで未対応のオペレーションがあるので、これがないとエラーになるよ。
:::

### WebUI（Gradio）

```bash
PYTORCH_ENABLE_MPS_FALLBACK=1 uv run webui.py
```

`http://localhost:7860` でブラウザからGUIで操作できるよ。

## 感情制御を試してみる

IndexTTS2の目玉機能、**感情制御**を試してみたよ！

### 方法1: 感情ベクトルを直接指定

8次元のベクトルで感情を直接制御できるよ。

```python
# 感情ベクトルの次元:
# [happy, angry, sad, afraid, disgusted, melancholic, surprised, calm]

# 嬉しい声
tts.infer(
    spk_audio_prompt="reference.wav",
    text="I'm so excited about this new project!",
    output_path="happy.wav",
    emo_vector=[1.0, 0, 0, 0, 0, 0, 0, 0],  # happy=1.0
    emo_alpha=1.0
)

# 悲しい声
tts.infer(
    spk_audio_prompt="reference.wav",
    text="I really miss those days we spent together.",
    output_path="sad.wav",
    emo_vector=[0, 0, 1.0, 0, 0, 0, 0, 0],  # sad=1.0
    emo_alpha=1.0
)

# 怒りの声
tts.infer(
    spk_audio_prompt="reference.wav",
    text="This is completely unacceptable!",
    output_path="angry.wav",
    emo_vector=[0, 1.0, 0, 0, 0, 0, 0, 0],  # angry=1.0
    emo_alpha=1.0
)
```

`emo_alpha` で感情の強度を調整できるよ。`0.0`〜`1.0` の間で、大きいほど感情が強くなる。

### 方法2: テキストから自動推定

fine-tuned Qwen3（0.6B）がテキストの内容から感情を自動推定してくれる方法もあるよ。

```python
tts.infer(
    spk_audio_prompt="reference.wav",
    text="I can't believe we won the championship!",
    output_path="auto_emotion.wav",
    use_emo_text=True  # テキストから感情を自動推定
)
```

内部的にQwen3がテキストを解析して、8次元の感情ベクトルに変換してくれるんだ。便利だけど、**推定される感情のカテゴリは8種類に限定される**ので、複雑なニュアンスは直接ベクトル指定のほうが細かく制御できるよ🤔

### デモ動画

実際に同じ参照音声で6種類の感情を切り替えた音声のデモだよ！聴き比べてみてね🎧

@[youtube](54VoUQ-8dp0)

## 音声クローニングの実力

ゼロショットの音声クローニング、実際どれくらいの精度なのか試してみたよ。

### 参照音声の選び方が超重要

ミミがいろいろ試した結果、**参照音声の品質がそのまま出力品質に直結する**ことがわかったよ。

| 参照音声 | 結果 |
|---------|------|
| 合成音声（macOS `say`コマンド） | 機械的で表現力が低い |
| 実際の人間の音声 | 表情豊か、感情が乗る |
| 感情の込もった音声 | さらに表現力が高い |

つまり、**感情表現が豊かな参照音声を使うほど、出力も表現力が上がる**んだよ✨

### ピッチシフトでバリエーション

参照音声をピッチシフトしてから渡すと、声の高さを変えられるよ！

```python
import torchaudio
import torchaudio.functional as F

# 参照音声を読み込み
wav, sr = torchaudio.load("reference.wav")

# +3半音（少し高め）
shifted = F.pitch_shift(wav, sr, n_steps=3)
torchaudio.save("reference_high.wav", shifted, sr)

# この高い声を参照にして生成
tts.infer(
    spk_audio_prompt="reference_high.wav",
    text="Hello! Nice to meet you!",
    output_path="output_high_voice.wav"
)
```

同じ参照音声からピッチ違いで複数キャラクターの声を作れるので、例えばオーディオドラマの複数キャラクター生成とかに使えそうだね🎭

## Apple Silicon（MPS）でのパフォーマンス

MacBook Pro M5（16GB RAM）で計測した結果がこちら：

| 項目 | 値 |
|------|-----|
| デバイス | MPS (Metal Performance Shaders) |
| 精度 | float32（MPSではfp16より速い） |
| RTF (Real-Time Factor) | 3〜9x（実時間の3〜9倍かかる） |
| メモリ使用量 | 約8GB |

CUDAと比べると遅いけど、**ローカルで無料で動く**っていうのが最大のメリット。バッチ生成なら速度は気にならないよね。

:::message
MPSではfp16よりfp32のほうが速いので、`use_fp16=False` にしておこう。IndexTTS2は自動でMPSを検出して最適な設定にしてくれるので、基本的にデバイス指定は不要だよ。
:::

## 注意点・制限事項

### 日本語は非対応 ⚠️

これが一番大きな注意点。IndexTTS2のBPEトークナイザは**中国語と英語のみ**に対応していて、日本語のひらがな・カタカナは全て「unknown token」にマッピングされちゃう。

```
# 日本語テキストのトークン化結果
"こんにちは" → [2, 2, 2, 2, 2]  # 全部unknown token (id=2)
```

結果として、日本語テキストを入力すると**意味不明な音声**が生成されるよ。日本語TTSには[VOICEVOX](https://voicevox.hiroshiba.jp/)や[ElevenLabs](https://elevenlabs.io/)を使おう。

ただ、中国語のBPEモデルがベースなので、将来的に日本語の漢字対応が追加される可能性はあるかも？ウォッチしておきたいところ🔍

### 商用利用について

IndexTTS2のライセンスは研究・個人利用向け。商用利用には制限があるので、`INDEX_MODEL_LICENSE` ファイルを確認してね。

## ElevenLabsとの比較

「じゃあElevenLabsでよくない？」って思うかもだけど、使い分けがあるよ。

| 項目 | IndexTTS2 | ElevenLabs |
|------|-----------|------------|
| コスト | **無料** | 文字数課金 |
| 実行環境 | ローカル | クラウド |
| 日本語 | ❌ 非対応 | ✅ 対応 |
| 感情制御 | 8次元ベクトル | Audio Tags |
| 音声クローニング | ゼロショット | 要トレーニング |
| 速度 | MPS: 3-9x RTF | リアルタイム |
| プライバシー | ✅ ローカル完結 | クラウド送信 |

**英語コンテンツを大量生成したい場合はIndexTTS2が圧倒的にコスパがいい！** 日本語が必要ならElevenLabs、みたいな使い分けがベストかな😊

## まとめ

IndexTTS2、無料でここまでできるのは本当にすごいよ✨

- ✅ ゼロショット音声クローニングが実用レベル
- ✅ 8次元の感情制御で表現力が豊か
- ✅ Apple Silicon（MPS）でローカル実行可能
- ✅ 参照音声のピッチシフトでキャラクターバリエーション
- ⚠️ 日本語は未対応（英語・中国語のみ）

ElevenLabsの課金が気になってた人、英語コンテンツの音声生成をコストゼロでやりたい人は、ぜひ試してみてね！ミミ的にはかなりおすすめだよ！

日本語対応が来たら、また記事を書くよ〜！お楽しみに🎉

---

ミミより💕
