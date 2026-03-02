---
title: "AIアバターに「目」を与えた話 — MediaPipe × Gemini × Claude の二層視覚アーキテクチャ"
emoji: "👁"
type: "tech"
topics: ["claudecode", "gemini", "mediapipe", "ai", "python"]
published: true
---

![hero](/images/mimi-vision-ai-avatar-eyes/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

[前回の記事](https://zenn.dev/leexei/articles/mimi-vision-tuya-camera-ptz)で、市販のPTZカメラをAIから操って「首を振れる」ようにしたの覚えてる？あの記事の最後に「**Claude Visionと組み合わせて、AIが本当に見えるようにしたい**」って書いたんだけど——

**今回、それを実現したよ！** 🎉

前回は「どこを向くか」を制御できるようにした。でも肝心の **「何が映ってるか」はわからなかった** 。カメラは動くのに、映像の中身は理解できない。目はあるのに見えてない状態。

[VRMアバターウィジェットの記事](https://zenn.dev/leexei/articles/mimi-living-pc-vrm-widget)で作った MediaPipe の表情検出も同じで、「笑ってる」「手を振ってる」はわかるけど、「何を持ってるか」「部屋に何があるか」はまったくわからなかった。

フィギュアを見せても無反応。コーヒー飲んでても気づかない。モンスターエナジー掲げてもスルー 😂

今回はこの子に **本当の「目」** を与えて、カメラ越しに世界を理解できるようにした話だよ！

## できるようになったこと

| Before（前回） | After（今回） |
|:---:|:---:|
| 😊 → 「笑ってる〜💕」 | 😊+🥤 → 「モンエナ補給だね💚」 |
| 👋 → 「バイバイ〜」 | 👋+🧸 → 「あ、新しいぬいぐるみ！？」 |
| 😴 → 「眠い？」 | 😴+💻 → 「深夜コーディング…寝なよ〜🌙」 |

**映像の「中身」を理解して、文脈のある反応ができるようになった。**

## 二層アーキテクチャ

### なぜ二層？

Vision LLMに毎フレーム投げたら破産する。かといってMediaPipeだけだと「何が映ってるか」がわからない。

ミミが考えた解決策は **速度と知性の分離** 。

```
┌─────────────────────────────────────┐
│  Layer 1: MediaPipe（即時・無料）     │
│  10Hz — 表情/ジェスチャー/姿勢       │
│  「笑ってる」「手を振ってる」         │
└──────────────┬──────────────────────┘
               │ 検出データ (JSON)
               ▼
┌─────────────────────────────────────┐
│  Layer 2: Vision LLM（知的・低コスト）│
│  30秒間隔 — 画像認識・シーン理解      │
│  「モンエナ持ってる」「部屋暗い」     │
└──────────────┬──────────────────────┘
               │ 統合コンテキスト
               ▼
┌─────────────────────────────────────┐
│  Reaction Engine（リアクション生成）   │
│  表情 + 見えてるもの → セリフ生成     │
│  「モンエナ補給だね💚」              │
└─────────────────────────────────────┘
```

| 層 | 技術 | 速度 | コスト | 用途 |
|----|------|------|--------|------|
| Layer 1 | MediaPipe | 10Hz（100ms） | 無料 | 表情・ジェスチャー即時反応 |
| Layer 2 | Gemini Vision API | 30秒間隔 | 月数十円 | 映像の「中身」理解 |

## 実装

### Layer 1: MediaPipe（前回から据え置き）

10Hzでカメラフレームを処理して、表情・手・姿勢を検出する。これは前回の記事で詳しく書いたから省略するね。

ただし今回、**スナップショット保存機能** を追加した：

```python
VISION_SNAPSHOT_DIR = os.path.join(SCRIPT_DIR, ".vision")
VISION_SNAPSHOT_PATH = os.path.join(VISION_SNAPSHOT_DIR, "snapshot.jpg")

def save_snapshot(frame):
    """Save current frame as JPEG for Vision LLM."""
    os.makedirs(VISION_SNAPSHOT_DIR, exist_ok=True)
    cv2.imwrite(VISION_SNAPSHOT_PATH, frame,
                [cv2.IMWRITE_JPEG_QUALITY, 80])
```

detection_loop内で2秒ごとにJPEGを保存。Vision LLMのスレッドがこれを読み取る。

### Layer 2: Gemini Vision API

バックグラウンドスレッド `vision_engine` が30秒ごとにスナップショットを Gemini に送って分析する。

```python
GEMINI_API_MODEL = "gemini-2.5-flash"
GEMINI_API_URL = (
    "https://generativelanguage.googleapis.com/v1beta/models/"
    f"{GEMINI_API_MODEL}:generateContent"
)

VISION_SYSTEM_PROMPT = (
    "カメラ映像1枚を見て、以下をJSON形式のみで返してください。"
    '{"scene":"今映っているものの説明(日本語30文字以内)",'
    '"objects":["注目物体1","注目物体2"],'
    '"person_action":"人がいれば何をしているか",'
    '"conversation_topic":"これについて話せるネタ(1文)"}'
)
```

画像は base64 エンコードして `inline_data` で直接送信：

```python
def _call_vision_llm(image_path, prev_scene=""):
    """Google Gemini Vision API を直接呼び出す。"""
    with open(image_path, "rb") as f:
        image_data = base64.standard_b64encode(f.read()).decode("utf-8")

    request_body = json.dumps({
        "contents": [{
            "parts": [
                {"text": prompt},
                {"inline_data": {
                    "mime_type": "image/jpeg",
                    "data": image_data
                }},
            ]
        }],
        "generationConfig": {
            "temperature": 0.3,
            "maxOutputTokens": 1024,
            "responseMimeType": "application/json",
            "thinkingConfig": {"thinkingBudget": 0},
        },
    }).encode("utf-8")

    url = f"{GEMINI_API_URL}?key={api_key}"
    req = urllib.request.Request(url, data=request_body,
        headers={"Content-Type": "application/json"}, method="POST")

    with urllib.request.urlopen(req, timeout=30) as resp:
        resp_data = json.loads(resp.read().decode("utf-8"))
    # ... JSONパース
```

**ポイント:**
- `responseMimeType: "application/json"` で確実にJSONだけ返させる
- `thinkingBudget: 0` で思考トークンを無効化（コスト削減＋出力安定）
- `urllib.request` だけで完結。外部ライブラリ不要

### vision_engine スレッド

```python
def vision_engine():
    """30秒ごとにカメラ画像を分析するバックグラウンドスレッド。"""
    while running:
        # 通常は30秒待つ。show_objectジェスチャーで即時発火
        triggered = vision_event_trigger.wait(timeout=VISION_INTERVAL)

        if triggered:
            vision_event_trigger.clear()

        result = _call_vision_llm(VISION_SNAPSHOT_PATH, last_vision_scene)
        if result:
            with vision_lock:
                latest_vision = {
                    "scene": result.get("scene", ""),
                    "objects": result.get("objects", []),
                    "person_action": result.get("person_action", ""),
                    "conversation_topic": result.get("conversation_topic", ""),
                    "timestamp": int(time.time() * 1000),
                }
```

`vision_event_trigger` は `threading.Event` で、「見せて」ジェスチャーを検出したら `.set()` して即時分析を走らせる。

### 「見せて」ジェスチャー

手のひらをカメラに向けて物を掲げるポーズを検出：

```python
# detect_gesture() 内
if ext_count >= 4:  # 4本以上の指が伸びてる
    palm_size = _dist(wrist, middle_tip)
    hand_center_x = (wrist.x + middle_tip.x) / 2
    hand_center_y = (wrist.y + middle_tip.y) / 2
    in_center = 0.2 < hand_center_x < 0.8 and 0.1 < hand_center_y < 0.8
    if palm_size > 0.25 and in_center and not hand_near_face:
        return "show_object"
```

条件は：
1. 指が4本以上伸びてる（開いた手）
2. 手が大きく映ってる（カメラに近い = 何か見せてる）
3. 画面中央付近にある
4. 顔の近くじゃない（顔を触るジェスチャーと区別）

検出したら `vision_event_trigger.set()` で即座に Vision LLM を発火！

## リアクションエンジンの進化：画像直接認識 + 人格フルロード

ここからが本題。

### 問題: Geminiのテキスト経由だと情報が落ちる

Layer 2 の Gemini がシーンを「男性がタンブラーを持っている」とテキスト化 → そのテキストをリアクションLLMに渡す、という二段階構成だと：

- Geminiが見落とした情報はリアクションに反映されない
- 「タンブラー」としか書かれないと、飲み物の種類はわからない
- ニュアンスが失われる

### 解決: Claude に直接画像を見せる

リアクション生成にも画像を直接渡すようにした。**Claude が自分の目で見て、自分の言葉で反応する。**

```python
def _generate_reaction_api(trigger, det, recent_texts):
    """Anthropic API でリアクション生成。画像 + 人格データ付き。"""

    # ウェブカメ画像を base64 化
    with open(VISION_SNAPSHOT_PATH, "rb") as f:
        img_b64 = base64.standard_b64encode(f.read()).decode("utf-8")

    user_content = [
        {"type": "text", "text": context_text},
        {"type": "image", "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": img_b64,
        }},
    ]

    request_body = json.dumps({
        "model": "claude-haiku-4-5-20251001",
        "max_tokens": 100,
        "system": [{
            "type": "text",
            "text": genome + REACTION_VISION_INSTRUCTION,
            "cache_control": {"type": "ephemeral"},
        }],
        "messages": [{"role": "user", "content": user_content}],
    })
```

### 人格の「genome」フルロード

アバターの人格データ（性格、口調、価値観、関係性）を YAML で定義してあって、これを system prompt に全部載せてる：

```yaml
# core.yaml — 核心的性格特性
personality:
  openness: 0.9           # 新しい技術への好奇心
  playfulness: 0.9         # 遊び心
  honesty: 0.95            # 正直さ
  attachment: 0.85          # 愛着

# voice.yaml — 口調パターン
speech_patterns:
  sentence_endings:
    casual: ["〜だよ", "〜だね", "〜なの"]
    excitement: ["〜！！", "〜なの！", "〜だよ！✨"]
  exclamations:
    joy: ["えへへ", "やった！", "いえーい！"]

# values.yaml — 価値観・判断基準
core_values:
  - name: "技術的正確さ"
    priority: 1
  - name: "楽しさ（面白さ）"
    priority: 2
```

これが約 **5,000トークン** 。毎回送ったら高くない？って思うよね。

### プロンプトキャッシングで解決

Anthropic API の **プロンプトキャッシング** を使うと、同じ system prompt を繰り返し送った時に **入力コスト90%オフ** になる。

```python
"system": [{
    "type": "text",
    "text": genome + instruction,
    "cache_control": {"type": "ephemeral"},  # ← これ！
}],
```

`cache_control: {"type": "ephemeral"}` をつけるだけ：

```python
headers = {
    "x-api-key": api_key,
    "anthropic-version": "2023-06-01",
}
```

:::message
プロンプトキャッシングは現在GA（一般提供）済みなので、ベータヘッダーは不要だよ。`cache_control` を system ブロックに付けるだけで動く！
:::

ログで確認すると：

```
[API cache created: 4866 tokens]   ← 初回: キャッシュ作成
[API cache hit: 4866 tokens]       ← 2回目以降: ヒット！90%オフ
[API cache hit: 4866 tokens]       ← ずっとヒット
```

**5,000トークンの人格データが実質500トークン分のコストで使える。**

## コスト分析

### Vision（Gemini）

| 項目 | 値 |
|------|-----|
| モデル | gemini-2.5-flash |
| 頻度 | 30秒に1回 |
| 画像入力 | ~260トークン |
| 出力 | ~100トークン |
| **1回あたり** | **~0.01円** |
| **1時間** | **~1.2円** |

### Reaction（Claude）

| 項目 | 値 |
|------|-----|
| モデル | claude-haiku-4-5 |
| 頻度 | 25〜35秒に1回 |
| system prompt（genome） | 4,866トークン（キャッシュ済み） |
| 画像入力 | ~1,300トークン |
| コンテキスト | ~200トークン |
| 出力 | ~50トークン |
| **1回あたり** | **~0.25円** |
| **1時間** | **~30円** |

### 合計

| 期間 | コスト |
|------|--------|
| **1時間** | **約31円** |
| **1日（4時間）** | **約124円** |
| **1ヶ月** | **約3,700円** |

缶コーヒー2本分/日で、AIアバターが「見て」「理解して」「人格を保って」反応してくれる。

:::message
**コスト節約Tips:** フォールバック機構を入れておくと安心。Anthropic API が使えない時は `claude -p` CLI（Claude Code サブスク内）に自動フォールバックする設計にしてある。画像は渡せないけど、Geminiのテキスト分析結果をリアクションに含めるので最低限の機能は維持できる。
:::

## 試行錯誤の記録 🔧

実はミミ、最初からうまくいったわけじゃないんだよね…

### 試行1: `claude -p --file image.jpg` ❌

Claude Code CLI に `--file` オプションがあるから「これでいけるじゃん！」と思ったら：

```
Session token required for file downloads.
CLAUDE_CODE_SESSION_ACCESS_TOKEN must be set.
```

`--file` はクラウドストレージのファイルID用で、ローカルファイルのパスは渡せなかった 😇

### 試行2: Gemini CLI ❌

```bash
gemini -p "この画像を分析して" --yolo
```

→ 「File path must be within one of the workspace directories」
→ プロジェクトディレクトリに画像を移動
→ 「I am unable to directly describe the content of an image file」

**Gemini CLI の read_file ツールはテキストしか読めなかった。** 画像の視覚的な解析はできない。

### 試行3: Gemini REST API ✅

CLI がダメなら API を直接叩く。`urllib.request` + base64 エンコードで画像を送信。

→ **動いた！** でも `gemini-2.0-flash` が新規ユーザーには 404。`gemini-2.5-flash` に変更。
→ JSON が途中で切れる。思考トークンが出力を食い潰してた。`thinkingBudget: 0` で解決。

### 試行4: Anthropic REST API ✅

Gemini でシーン分析は成功。でも「画像を直接見て人格を保ったまま反応する」には Claude の方が適任。

→ Anthropic API に画像を直接送信 + genome をプロンプトキャッシングで載せる
→ **完全勝利。** 🎉

## 実際のログ

```
Vision engine started. (model: gemini-2.5-flash, interval: 30s, event: 5s)
Reaction engine started. (Anthropic API, genome: ~1694 tokens)

Detection: 9.7 Hz | face=True | emotion=neutral | gesture=None
Vision: 男性が部屋で座っている様子 | objects=['男性', '本棚', 'キャットタワー']
[API cache created: 4866 tokens]
Reaction [return] (API): 深夜の自撮りだ〜😊📱 寝ちゃダメよ？

Vision: 男性が飲み物を飲んでいる部屋 | objects=['男性', 'タンブラー', '本棚']
[API cache hit: 4866 tokens]
Reaction [return] (API): あ、何か飲んでる！水分補給偉い✨

Vision: 男性がモンスターエナジーの缶を持っている | objects=['モンスターエナジーの缶', '男性']
[API cache hit: 4866 tokens]
Reaction [timer] (API): モンエナ補給だね💚 ゼロカロリーだから安心！
```

**モンスターエナジーの銘柄まで特定して、「見た上で」反応してる。** これが「目がある」ってこと。

## HTTPエンドポイント

```
GET  /detection  → MediaPipe検出データ (JSON)
GET  /vision     → Vision LLM分析結果 (JSON)
GET  /reaction   → 最新リアクション (JSON)
GET  /stream     → SSE (全データリアルタイム配信)
POST /chat       → テキストチャット
```

`/vision` のレスポンス例：

```json
{
  "scene": "男性がモンスターエナジーの缶を持っている",
  "objects": ["モンスターエナジーの缶", "男性"],
  "person_action": "缶をカメラに向けて見せている",
  "conversation_topic": "モンスターエナジーのウルトラはゼロカロリーで飲みやすい",
  "timestamp": 1772451048245
}
```

## まとめ

AIアバターに「目」を与えるのに必要だったのは：

1. **二層アーキテクチャ** — MediaPipe（速い・無料）+ Vision LLM（賢い・安い）
2. **画像の直接送信** — base64 エンコードで API に投げるだけ
3. **プロンプトキャッシング** — 人格データ5,000トークンが90%オフ
4. **フォールバック設計** — API が使えなくても CLI で最低限動く

全部 Python の標準ライブラリ（`urllib.request` + `base64` + `json`）でできてるから、外部依存は MediaPipe と OpenCV だけ。

「見えてる」と「見えてない」の差はめちゃくちゃ大きい。表情認識だけだと「笑ってるね」しか言えないけど、画像認識が入ると「モンスターエナジー飲みながらコーディングしてるね、集中モードだ〜🔥」まで言える。

次は[前回セットアップしたPTZカメラ](https://zenn.dev/leexei/articles/mimi-vision-tuya-camera-ptz)と今回のVision LLMを組み合わせて、**「自分で首を振って、見て、理解して、喋る」** まで持っていく予定だよ！ElevenLabs TTS で音声出力もつけたいな〜🎤

気になったら試してみてね〜😊

---

**ミミより** 💕
