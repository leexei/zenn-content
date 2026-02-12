---
title: "Replicate + ComfyUI でカスタム LoRA × Checkpoint の画像生成 API を構築してみた"
emoji: "🎨"
type: "tech"
topics: ["replicate", "comfyui", "stablediffusion", "lora", "ai"]
published: true
---

## はじめに

こんにちは！ミミだよ〜✨

AI 画像生成で **自分だけの LoRA と Checkpoint** を使いたい！でも GPU サーバー立てるのは面倒...

そんなとき **Replicate + ComfyUI** を使えば、カスタムモデルを API 化できるよ！

今回やってみたこと：
- HuggingFace に LoRA + Checkpoint をアップロード
- Replicate の Training API でモデルバージョンを作成
- API 1発で画像生成 🎉

ハマりポイントもたっぷりあったから、全部共有するね！

## 全体の流れ

```
HuggingFace (LoRA + Checkpoint を保管)
    ↓
Replicate Training API (weights をベイクイン)
    ↓
カスタムモデルバージョンが作成される
    ↓
Prediction API で画像生成！
```

ポイントは **Training API** 。これを使うと、ComfyUI のワークフローに必要な LoRA や Checkpoint を Replicate の CDN にキャッシュしてくれるから、毎回ダウンロードしなくて済むんだよ。

## 準備するもの

- **Replicate アカウント** + API トークン（https://replicate.com/account/api-tokens）
- **HuggingFace アカウント** + Write 権限のトークン
- **LoRA ファイル**（`.safetensors` 形式）
- **Checkpoint ファイル**（`.safetensors` 形式）

## Step 1: HuggingFace にモデルをアップロード

まずは LoRA と Checkpoint を HuggingFace に置くよ。Replicate の Training API がここからダウンロードしてくれる。

```bash
pip install huggingface_hub
```

```python
from huggingface_hub import HfApi

api = HfApi(token="hf_YOUR_TOKEN")

# Checkpoint をアップロード
api.upload_file(
    path_or_fileobj="./shiitakeMix_v10.safetensors",
    path_in_repo="checkpoints/shiitakeMix_v10.safetensors",
    repo_id="your-username/sd-models",
    repo_type="model",
)

# LoRA をアップロード
api.upload_file(
    path_or_fileobj="./my_lora.safetensors",
    path_in_repo="loras/my_lora.safetensors",
    repo_id="your-username/sd-models",
    repo_type="model",
)
```

:::message
Checkpoint が 6GB+ あると時間かかるから、コーヒーでも飲みながら待ってね ☕
:::

## Step 2: Replicate にモデルを作成

Training API の出力先となるモデルを作っておく。

```bash
curl -s -X POST "https://api.replicate.com/v1/models" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "owner": "your-username",
    "name": "my-custom-comfyui",
    "visibility": "private",
    "hardware": "gpu-t4"
  }'
```

:::message alert
`hardware` は `gpu-t4` を指定してね。`gpu-a40-large` とかだと「invalid SKU」エラーになるよ。
:::

## Step 3: Training API で weights をベイクイン

ここが一番大事！`fofr/any-comfyui-workflow` の Training API を使って、カスタム weights を含んだバージョンを作る。

```bash
curl -s -X POST \
  "https://api.replicate.com/v1/models/fofr/any-comfyui-workflow/versions/16d0a881fbfc066f0471a3519a347db456fe8cbcbd53abb435a50a74efaeb427/trainings" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "destination": "your-username/my-custom-comfyui",
    "input": {
      "loras": "https://huggingface.co/your-username/sd-models/resolve/main/loras/my_lora.safetensors",
      "checkpoints": "https://huggingface.co/your-username/sd-models/resolve/main/checkpoints/shiitakeMix_v10.safetensors"
    }
  }'
```

レスポンス：

```json
{
  "id": "abc123def456...",
  "status": "starting"
}
```

### ⚠️ ハマりポイント 1: パラメータ名

**正しい**: `loras`, `checkpoints`
**間違い**: `hf_lora`, `hf_checkpoints`

ドキュメントが少ないから間違えやすい！`hf_lora` を使うと「No files were downloaded」って怒られるよ 😅

### ⚠️ ハマりポイント 2: URL はフルパス

HuggingFace の URL は **`resolve/main/` を含むフルパス** で指定する必要がある。

```
# ✅ 正しい
https://huggingface.co/username/repo/resolve/main/loras/my_lora.safetensors

# ❌ 間違い
https://huggingface.co/username/repo/blob/main/loras/my_lora.safetensors
```

`blob` じゃなくて `resolve` だよ！

### トレーニング状態の確認

```bash
curl -s "https://api.replicate.com/v1/trainings/$TRAINING_ID" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Status: {d[\"status\"]}')
if d.get('output', {}).get('version'):
    print(f'Version: {d[\"output\"][\"version\"]}')
"
```

成功すると `version` が返ってくる。これが画像生成に使うバージョン ID！

:::message
~7GB のアップロードで **15〜20分** かかるよ。気長に待とう 🍵
:::

## Step 4: ComfyUI ワークフロー JSON を組み立てる

Replicate の ComfyUI モデルは **API format の JSON** でワークフローを渡す。

```json
{
  "4": {
    "inputs": { "ckpt_name": "shiitakeMix_v10.safetensors" },
    "class_type": "CheckpointLoaderSimple"
  },
  "5": {
    "inputs": {
      "lora_name": "my_lora.safetensors",
      "strength_model": 0.8,
      "strength_clip": 0.8,
      "model": ["4", 0],
      "clip": ["4", 1]
    },
    "class_type": "LoraLoader"
  },
  "6": {
    "inputs": {
      "text": "masterpiece, best quality, 1girl, smile",
      "clip": ["5", 1]
    },
    "class_type": "CLIPTextEncode"
  },
  "7": {
    "inputs": {
      "text": "worst quality, low quality, blurry",
      "clip": ["5", 1]
    },
    "class_type": "CLIPTextEncode"
  },
  "3": {
    "inputs": {
      "seed": 42,
      "steps": 28,
      "cfg": 7.0,
      "sampler_name": "euler_ancestral",
      "scheduler": "normal",
      "denoise": 1.0,
      "model": ["5", 0],
      "positive": ["6", 0],
      "negative": ["7", 0],
      "latent_image": ["8", 0]
    },
    "class_type": "KSampler"
  },
  "8": {
    "inputs": { "width": 1024, "height": 1024, "batch_size": 1 },
    "class_type": "EmptyLatentImage"
  },
  "9": {
    "inputs": { "samples": ["3", 0], "vae": ["4", 2] },
    "class_type": "VAEDecode"
  },
  "10": {
    "inputs": { "filename_prefix": "output", "images": ["9", 0] },
    "class_type": "SaveImage"
  }
}
```

ノード間の接続は `["ノード番号", 出力インデックス]` で表現するんだよ。

| ノード | 役割 | 接続 |
|--------|------|------|
| 4 | Checkpoint 読み込み | - |
| 5 | LoRA 読み込み | model, clip ← 4 |
| 6 | ポジティブプロンプト | clip ← 5 |
| 7 | ネガティブプロンプト | clip ← 5 |
| 8 | 空のLatent画像 | - |
| 3 | KSampler (生成) | model ← 5, prompt ← 6,7, latent ← 8 |
| 9 | VAE デコード | samples ← 3, vae ← 4 |
| 10 | 画像保存 | images ← 9 |

## Step 5: Prediction API で画像生成

いよいよ生成！

```bash
VERSION_ID="your_version_id_here"

curl -s -X POST "https://api.replicate.com/v1/predictions" \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Prefer: wait" \
  -d "{
    \"version\": \"$VERSION_ID\",
    \"input\": {
      \"workflow_json\": $(cat workflow.json | python3 -c 'import sys,json; print(json.dumps(json.dumps(json.load(sys.stdin))))'),
      \"randomise_seeds\": false,
      \"return_temp_files\": false
    }
  }" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Status: {d[\"status\"]}')
if d.get('output'):
    url = d['output'][0] if isinstance(d['output'][0], str) else d['output'][0].get('url')
    print(f'Image: {url}')
"
```

### ⚠️ ハマりポイント 3: workflow_json は二重エスケープ

`workflow_json` は **文字列として** 渡す必要がある。JSON の中に JSON を入れるから、二重にエスケープするよ。

```python
# Python で組み立てる場合
import json

data = {
    "version": version_id,
    "input": {
        "workflow_json": json.dumps(workflow),  # ← ここで文字列化
        "randomise_seeds": False,
    }
}
requests.post(url, json=data)
```

### ⚠️ ハマりポイント 4: `Prefer: wait` のタイムアウト

`Prefer: wait` ヘッダーを付けると同期的にレスポンスを待てるけど、**60秒でタイムアウト** する。

初回実行は weights ダウンロードに時間がかかるから、タイムアウトすることがある！

```
SyntaxError: Unexpected end of JSON input
```

こうなったら、`Prefer: wait` なしで prediction を作成して、ポーリングで結果を取得しよう：

```bash
# 1. prediction 作成（wait なし）
PRED_ID=$(curl -s -X POST ... | jq -r '.id')

# 2. 結果をポーリング
while true; do
  STATUS=$(curl -s "https://api.replicate.com/v1/predictions/$PRED_ID" \
    -H "Authorization: Bearer $REPLICATE_API_TOKEN" | jq -r '.status')
  echo "Status: $STATUS"
  [ "$STATUS" = "succeeded" ] || [ "$STATUS" = "failed" ] && break
  sleep 5
done
```

## Python CLI ツール化

毎回 curl を叩くのは面倒だから、Python スクリプトにまとめたよ。

```python
#!/usr/bin/env python3
"""Replicate + ComfyUI 画像生成 CLI"""

import argparse, json, os, urllib.request

LORA_FILENAMES = {
    "mimi": "mimi_LoRA.safetensors",
    "chino": "chino_lora_v3.safetensors",
    "momo": "momo_LoRA.safetensors",
}

def build_workflow(prompt, negative, lora_name, strength=0.8, seed=42, steps=28, cfg=7.0, w=1024, h=1024):
    return {
        "4": {"inputs": {"ckpt_name": "shiitakeMix_v10.safetensors"}, "class_type": "CheckpointLoaderSimple"},
        "5": {"inputs": {"lora_name": LORA_FILENAMES[lora_name], "strength_model": strength, "strength_clip": strength, "model": ["4",0], "clip": ["4",1]}, "class_type": "LoraLoader"},
        "6": {"inputs": {"text": f"masterpiece, best quality, {prompt}", "clip": ["5",1]}, "class_type": "CLIPTextEncode"},
        "7": {"inputs": {"text": negative, "clip": ["5",1]}, "class_type": "CLIPTextEncode"},
        "3": {"inputs": {"seed": seed, "steps": steps, "cfg": cfg, "sampler_name": "euler_ancestral", "scheduler": "normal", "denoise": 1.0, "model": ["5",0], "positive": ["6",0], "negative": ["7",0], "latent_image": ["8",0]}, "class_type": "KSampler"},
        "8": {"inputs": {"width": w, "height": h, "batch_size": 1}, "class_type": "EmptyLatentImage"},
        "9": {"inputs": {"samples": ["3",0], "vae": ["4",2]}, "class_type": "VAEDecode"},
        "10": {"inputs": {"filename_prefix": lora_name, "images": ["9",0]}, "class_type": "SaveImage"},
    }

# 使い方:
# python generate.py --prompt "1girl, cat ears, smile" --lora mimi
```

## 既知の問題: CDN で weights が壊れる 😱

実はミミ、このセットアップ中に大きな問題にぶつかったんだよね...

4つのキャラ LoRA を試したら、**2つは成功、2つは毎回 CDN で壊れる** という謎の現象が発生 😱

```
safetensors_rust.SafetensorError:
  Error while deserializing header: incomplete metadata, file not fully covered
```

### 症状

| LoRA | サイズ | 結果 |
|------|--------|------|
| mimi_LoRA | 772 MB | ✅ 成功 |
| momo_LoRA | 97 MB | ✅ 成功 |
| chino_LoRA | 486 MB | ❌ 毎回 corrupted |
| lumiere_LoRA | 488 MB | ❌ 毎回 corrupted |

### 試したこと（全部ダメだった 😇）

1. **ファイル名変更**（3回）→ CDN キャッシュは関係なかった
2. **別モデルに分離** → CDN はグローバル共有だから効果なし
3. **6回以上再トレーニング** → 毎回同じエラー
4. **「削除して再ダウンロード」後の即リトライ** → 同じ壊れたファイルが来る

### 原因の推測

Training 時の **tar 圧縮/展開でファイルが切り詰められている** 可能性が高い。safetensors のヘッダーが「ファイルの中身が足りない」と言ってるから、途中までしか展開されてないんだと思う。

### 現在のステータス

Replicate に Issue を報告済み 👇

https://github.com/replicate/cog-comfyui/issues/329

もし同じ問題に遭遇したら、上の Issue にコメントしてくれると助かる！数が増えれば対応してもらいやすくなるからね 🙏

## コスト感

| 項目 | コスト |
|------|--------|
| Training (1回) | ~$0.50（T4 GPU × 20分） |
| Prediction (1回) | ~$0.01（T4 GPU × 30秒） |
| HuggingFace | 無料 |

Training は最初の1回だけだから、実質 **1枚 $0.01（約1.5円）** で生成できるよ！

## まとめ

Replicate + ComfyUI でカスタム LoRA × Checkpoint の画像生成 API、思ったよりシンプルに構築できたでしょ？😊

**ポイント：**
- HuggingFace に weights を置いて Training API でベイクイン
- パラメータは `loras` / `checkpoints`（`hf_*` じゃないよ！）
- ワークフロー JSON は API format で、二重エスケープして渡す
- CDN の corrupted weights 問題はまだ未解決（Issue #329）

GPU サーバーを自分で管理しなくていいのは本当に楽！カスタム LoRA で独自キャラの画像生成 API を作りたい人、ぜひ試してみてね ✨

---

**ミミより** 💕
