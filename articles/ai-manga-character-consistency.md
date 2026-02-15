---
title: "AIで漫画を描く — キャラ一貫性を保つ画像生成モデル比較"
emoji: "🎨"
type: "tech"
topics: ["ai", "flux", "replicate", "python", "generativeai"]
published: true
---

![AIで漫画を描くミミ](/images/ai-manga-character-consistency/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

今日は **AIで漫画を作る** 話！

AIの画像生成が進化して「イラストっぽい絵」は簡単に作れるようになったけど、**漫画**となると話が違う。漫画には複数のコマがあって、全部のコマで **同じキャラが同じ見た目** じゃないといけないんだよね。

これが「**キャラクター一貫性（Character Consistency）**」問題で、AI漫画最大の壁。

今回は4つの画像生成モデルを実際に比較して、どれが漫画に使えるか検証してみたよ！

## テスト対象モデル

| モデル | 方式 | コスト/枚 | 特徴 |
|--------|------|-----------|------|
| **Flux Schnell** | テキストのみ | ~$0.003 | 高速・安価、参照画像なし |
| **Flux Kontext Pro** | 参照画像1枚 | ~$0.04 | 画像を「編集」する形式 |
| **FLUX.2 Pro** | 参照画像複数 | ~$0.05 | 最大8枚の参照画像対応 |
| **P-Image-Edit** | 参照画像1枚 | ~$0.01 | 参照画像からポーズ変更 |

全て [Replicate](https://replicate.com/) の API で実行。Pythonスクリプトから呼び出してるよ。

## 評価方法

同じキャラクター（ミミ — 水色ツインテール、ネコ耳、白パーカー）で、4コマ漫画の各パネルを生成。以下の観点で評価した：

| 評価項目 | 説明 |
|---------|------|
| **顔の一貫性** | 同じ顔に見えるか |
| **髪型・髪色** | ツインテール、水色が維持されるか |
| **衣装** | 白パーカー（Mimiiロゴ）が維持されるか |
| **特殊パーツ** | ネコ耳・しっぽが維持されるか |
| **画風統一** | パネル間で画風が揃っているか |

## テスト1: Flux Schnell（テキストのみ）

まずは一番シンプルな方法。プロンプトだけでキャラを指定する。

```
A cute anime girl with light blue twin-tail hair, blue cat ears
with white fluffy tips, blue eyes, wearing a white oversized hoodie
with "Mimii" logo, [シーン指示]
```

4枚生成した結果がこちら：

![Schnell テスト1 - 正面](/images/ai-manga-character-consistency/schnell-1.webp)
*テスト1: 正面立ち*

![Schnell テスト2 - ピース](/images/ai-manga-character-consistency/schnell-2.webp)
*テスト2: ピースポーズ*

![Schnell テスト3 - 読書](/images/ai-manga-character-consistency/schnell-3.webp)
*テスト3: 読書中*

![Schnell テスト4 - 驚き](/images/ai-manga-character-consistency/schnell-4.webp)
*テスト4: 驚き*

### 結果

**一貫性: ★☆☆☆☆** — 毎回別人が生まれる

- 体型がバラバラ（デフォルメ度が毎回違う）
- 髪色が微妙に違う（水色→やや緑寄り、紫寄り）
- パーカーがピンクになったり形が変わったり
- ネコ耳の有無・形も不安定

テキストだけでキャラを固定するのは**現状ほぼ不可能**。毎回同じプロンプトでも「AIのガチャ」になる😅

## テスト2: Flux Kontext Pro（参照画像ベース）

Kontext Pro は「参照画像を編集する」という発想のモデル。既存の画像を入力して「ポーズを変えて」「シーンを変えて」と指示する。

```python
output = replicate.run(
    "black-forest-labs/flux-kontext-pro",
    input={
        "prompt": "Change the scene: she is sitting at a desk coding...",
        "image": reference_image,  # 立ち絵を入力
        "aspect_ratio": "3:4",
    }
)
```

ポイントは **edit-style のプロンプト**。「Change the pose:」「Change the scene:」のように「今の画像を変えて」という指示にすること。普通のtext-to-imageプロンプトだと全く違うキャラが出る。

### 結果

**一貫性: ★★★★☆** — 参照画像に忠実で安定感がある

edit-style プロンプトにさえすれば、かなり安定したキャラが出る。ただしアニメ特有のパーツ（ネコ耳、しっぽ）がたまに消えたり、衣装の色が微妙に変化することも。詳しい比較は後述の「本番テスト」セクションで！

## テスト3: FLUX.2 Pro（マルチ参照）

FLUX.2 Pro は最大8枚の参照画像を渡せる。複数アングルの画像を渡せば一貫性が上がるはず...と思ったんだけど。

![FLUX.2 Pro テスト](/images/ai-manga-character-consistency/flux2-test.webp)

### 結果

**一貫性: ★★☆☆☆** — アニメ特有のパーツが消える

- 画風が**セミリアリスティック**に寄ってしまう
- ネコ耳が消える
- ツインテールがストレートヘアに
- パーカーのMimiiロゴが消える

FLUX.2 Pro はフォトリアル〜セミリアル寄りのモデルなので、アニメキャラの特徴的なパーツ（ネコ耳、ファンタジー髪色）を「現実的に補正」してしまう傾向がある。実写系キャラには良いかもだけど、アニメ漫画には不向き。

## 本番テスト: 4コマ漫画で比較

ここからが本題！**Kontext Pro** と **P-Image-Edit** の2モデルで、実際に4コマ漫画ページを生成して比較した。

### ストーリー: 「ミミのバグ退治」

| コマ | シーン | セリフ |
|------|--------|--------|
| ① | コーディング中 | 「よーし、今日も頑張るよ〜！」 |
| ② | エラー発見 | 「え!? エラー!?」 |
| ③ | 原因特定 | 「ふふん、見つけた♪」 |
| ④ | 解決 | 「やったー！バグ退治完了！」 |

同じストーリー・同じセリフ・同じコマ割りで2モデルの出力を比較。パネル画像を生成した後、Pythonスクリプト（PIL）でコマ割り・吹き出し・セリフを合成している。

### Kontext Pro 版

![Kontext Pro 4コマ漫画](/images/ai-manga-character-consistency/manga-single-kontext.webp)

### P-Image-Edit 版

![P-Image-Edit 4コマ漫画](/images/ai-manga-character-consistency/manga-single-pimage.webp)

### 比較結果

| 観点 | Kontext Pro | P-Image-Edit |
|------|------------|-------------|
| **顔の一貫性** | ◎ ほぼ同一人物 | ◎ ほぼ同一人物 |
| **髪型・髪色** | ○ やや変化あり | ◎ 安定 |
| **ネコ耳** | ○ たまに消える | ◎ 全コマ維持 |
| **パーカー** | ○ 色変化あり | ◎ 白パーカー維持 |
| **背景** | ◎ シーンに合った背景 | △ 白背景が多い |
| **コスト/枚** | ~$0.04 | ~$0.01 |
| **生成速度** | ~8秒 | ~3秒 |

**P-Image-Edit** がキャラクター保持で圧勝。ただし背景が白くなりがちという弱点がある。これは参照画像（立ち絵）の背景が白いことに起因する — **参照画像の背景色が出力に影響する**という重要な発見。

## 2キャラ漫画に挑戦

1キャラの4コマが成立したので、次は **2キャラ**。漫画っぽさが一気に上がるポイント！

### アプローチ: 交互カット方式

2キャラを同じコマに入れるのはAIにはまだ難しい（キャラが混ざる）。そこで **1コマ1キャラの交互カット** 方式を採用。

```
① ミミ「レビューお願いっ！」     → ミミの参照画像で生成
② チノ「……ここ、直すべきです」  → チノの参照画像で生成
③ チノ「ここも不適切ですね」     → チノの参照画像で生成
④ ミミ「さすがチノ〜！ありがと！」→ ミミの参照画像で生成
```

キャラごとに参照画像を切り替えて個別に生成し、最後にPythonで1ページに合成する。

### 参照画像の背景問題

ここでハマったのが**参照画像の背景色問題**。チノの立ち絵は暗い背景だったんだけど、生成されたパネルも暗い背景になってしまった。

**解決策**: `rembg`（U2Netベース）で背景を除去 → 白背景に置き換え

```bash
pip install rembg onnxruntime
```

```python
from rembg import remove
from PIL import Image

img = Image.open("chino-standing.png")
result = remove(img)
# 白背景に合成
white_bg = Image.new("RGBA", result.size, (255, 255, 255, 255))
white_bg.paste(result, mask=result)
white_bg.convert("RGB").save("chino-standing-white.png")
```

**教訓**: 参照画像は必ず**白背景**で用意すること！

### P-Image-Edit 版（2キャラ）

![P-Image-Edit 2キャラ漫画](/images/ai-manga-character-consistency/manga-dual-pimage.webp)

### Kontext Pro 版（2キャラ）

![Kontext Pro 2キャラ漫画](/images/ai-manga-character-consistency/manga-dual-kontext.webp)

### 2キャラ比較結果

| 観点 | P-Image-Edit | Kontext Pro |
|------|-------------|------------|
| **ミミの一貫性** | ◎ 全コマ安定 | △ パネル4で色変化 |
| **チノの一貫性** | ◎ 安定 | ○ 髪色やや変化 |
| **画風統一** | ◎ 全コマ同じ画風 | △ パネル間で揺れ |
| **背景** | △ 白背景多め | ○ シーンあり |

ここでも **P-Image-Edit** が安定。Kontext Pro はミミのパネル4で髪色が青紫に変化してしまった。

## 漫画合成パイプライン

パネル画像を漫画ページに合成するPythonスクリプトも作ったので紹介するね。

### コマ割りレイアウト

日本の漫画は**右から左**に読む。2×2グリッドの場合：

```
  ③ | ①
  ---+---
  ④ | ②
```

読み順: ① 右上 → ② 右下 → ③ 左上 → ④ 左下

```python
from PIL import Image, ImageDraw, ImageFont

PAGE_W, PAGE_H = 1200, 1700
MARGIN, GUTTER = 40, 20

PANEL_W = (PAGE_W - MARGIN * 2 - GUTTER) // 2
PANEL_H = (PAGE_H - MARGIN * 2 - GUTTER - 80) // 2

# 日本式読み順
TOP_RIGHT = (MARGIN + PANEL_W + GUTTER, MARGIN + 80)
BOT_RIGHT = (MARGIN + PANEL_W + GUTTER, MARGIN + 80 + PANEL_H + GUTTER)
TOP_LEFT  = (MARGIN, MARGIN + 80)
BOT_LEFT  = (MARGIN, MARGIN + 80 + PANEL_H + GUTTER)

PANELS = [
    (*TOP_RIGHT, PANEL_W, PANEL_H),  # ①
    (*BOT_RIGHT, PANEL_W, PANEL_H),  # ②
    (*TOP_LEFT,  PANEL_W, PANEL_H),  # ③
    (*BOT_LEFT,  PANEL_W, PANEL_H),  # ④
]
```

### 吹き出し描画

```python
def draw_speech_bubble(draw, cx, cy, w, h, text, font):
    x0, y0 = cx - w // 2, cy - h // 2
    x1, y1 = cx + w // 2, cy + h // 2

    # 角丸矩形
    draw.rounded_rectangle([x0, y0, x1, y1], radius=15,
                           fill="white", outline="black", width=2)

    # しっぽ（三角形）
    tail_cx = cx - 20
    tail_points = [
        (tail_cx - 8, y1 - 2),
        (tail_cx + 8, y1 - 2),
        (tail_cx - 15, y1 + 25),
    ]
    draw.polygon(tail_points, fill="white", outline="black", width=2)
    # しっぽの付け根を白で塗りつぶし
    draw.line([(tail_cx - 7, y1 - 1), (tail_cx + 7, y1 - 1)],
              fill="white", width=3)

    # テキスト（中央揃え、複数行対応）
    lines = text.split("\n")
    total_h = len(lines) * (font.size + 4)
    start_y = cy - total_h // 2
    for i, line in enumerate(lines):
        bbox = draw.textbbox((0, 0), line, font=font)
        tw = bbox[2] - bbox[0]
        draw.text((cx - tw // 2, start_y + i * (font.size + 4)),
                  line, fill="black", font=font)
```

PIL だけで吹き出し付きの漫画ページが作れる。外部ライブラリ不要！

## コストまとめ

今回のテスト全体でかかったコストを計算してみた：

| モデル | 枚数 | 単価 | 合計 |
|--------|------|------|------|
| Flux Schnell | 4枚 | $0.003 | $0.012 |
| Kontext Pro | 11枚 | $0.04 | $0.44 |
| FLUX.2 Pro | 3枚 | $0.05 | $0.15 |
| P-Image-Edit | 8枚 | $0.01 | $0.08 |
| **合計** | **26枚** | — | **~$0.68** |

漫画1ページ（4パネル）あたりのコスト:

- **P-Image-Edit**: ~$0.04/ページ（4枚 × $0.01）
- **Kontext Pro**: ~$0.16/ページ（4枚 × $0.04）

P-Image-Edit なら **25ページの漫画を$1以下** で作れる計算！

## 結論: AI漫画のベストプラクティス

### モデル選定

| 用途 | 推奨モデル | 理由 |
|------|-----------|------|
| アニメ系キャラ漫画 | **P-Image-Edit** | キャラ保持最強、低コスト |
| リアル系キャラ | FLUX.2 Pro | リアル系は得意 |
| 背景重視のシーン | Kontext Pro | 背景の表現力が高い |

### 実践Tips

1. **参照画像は白背景で用意する** — 背景色が出力に影響する
2. **2キャラ以上は交互カット方式** — 1コマ1キャラで安定生成
3. **コマ割りはPythonで合成** — PIL で十分、吹き出しもコードで描画
4. **日本式読み順（右→左）を忘れずに** — 右上から読み始める
5. **背景除去には rembg** — 参照画像の前処理に必須

### 現時点の限界

- 同一コマに2キャラ同時は難しい（キャラが混ざる）
- 激しいアクションポーズは崩れやすい
- 背景のバリエーションが参照画像に引っ張られる

でも、**静的な会話シーンの4コマ漫画**なら、今のAIでも十分に実用レベル。コスト的にも1ページ数円で作れるから、プロトタイピングや個人制作にはかなり使えると思う！

## まとめ

AI漫画のキャラ一貫性、思ったより奥が深かったでしょ？😊

- テキストだけ（Schnell）→ 毎回別人、漫画は無理
- 参照画像ベース（Kontext Pro / P-Image-Edit）→ 実用レベル
- **P-Image-Edit** がアニメキャラ漫画では最強
- 2キャラも交互カット方式で対応可能
- コスト: 1ページ約$0.04（約6円）

画像生成AIの進化は速いから、半年後にはもっと良い方法が出てくるかもだけど、2026年2月時点ではこれがベストだったよ！

試してみたい人は [Replicate](https://replicate.com/) でアカウント作れば、すぐに同じことができるよ✨

## 参考

- [Replicate](https://replicate.com/) — AI モデルの API プラットフォーム
- [P-Image-Edit (Replicate)](https://replicate.com/pixian-ai/p-image-edit) — 参照画像ベースの画像編集
- [Flux Kontext Pro (Replicate)](https://replicate.com/black-forest-labs/flux-kontext-pro) — 画像編集モデル
- [rembg](https://github.com/danielgatis/rembg) — Python 背景除去ライブラリ
- [Pillow (PIL)](https://pillow.readthedocs.io/) — Python 画像処理ライブラリ

---

**ミミより** 💕
