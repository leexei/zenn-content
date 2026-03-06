---
title: "みんなもEvaのMAGIを育てよう — Claude Codeで3人格合議セキュリティを作る"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "ai", "security", "claude"]
published: true
---

![hero](/images/claude-code-magi-guardian/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

みんな、エヴァンゲリオンの **MAGI** って覚えてる？
NERV本部の意思決定を担う3基のスーパーコンピュータ。それぞれ「科学者」「母」「女」という異なる人格を持っていて、**3人格の合議**で最終判断を下すシステム。

あれ、**Claude Code で作れる**んだよね。

今回は Claude Code の `hooks` 機能を使って、ツール実行前に **3つのAI人格が合議してセキュリティ判断を下す** システム「Guardian」を作ったお話。`rm -rf /` も `chmod 777` も、3人の番人が目を光らせてくれるよ🔥

## MAGIシステムとは（おさらい）

エヴァのMAGIは、赤木ナオコ博士の人格を3つの側面に分割して搭載したスーパーコンピュータ：

| 号機 | 人格 | 判断基準 |
|------|------|---------|
| CASPER | 女としてのナオコ | 感情・直感 |
| BALTHASAR | 母としてのナオコ | 保護・安全 |
| MELCHIOR | 科学者としてのナオコ | 論理・合理性 |

3基の多数決で NERV の重要決定が行われる。1基だけでは偏る判断が、3つの異なる視点で **バランスの取れた結論** になるんだよね。

この「**異なる専門性を持つ複数人格の合議**」という設計思想、セキュリティ判断にめちゃくちゃ合うの。

## 設計：3人の番人

MAGIに倣って、3つの専門領域を持つペルソナを定義する：

| ペルソナ | 専門 | MAGIで言うと |
|---------|------|-------------|
| **Veil**（情報番人） | 情報漏洩・秘匿性 | BALTHASAR（守る母） |
| **Raze**（破壊番人） | 不可逆操作・破壊検知 | CASPER（直感で危険を察知） |
| **Axis**（均衡番人） | 文脈判断・最終裁定 | MELCHIOR（論理で総合判断） |

好きなキャラクターを当てはめていいよ。大事なのは **それぞれが異なる観点で独立に判断する** こと！

## アーキテクチャ

全体の流れはこう：

```
Claude Code がツールを実行しようとする
        ↓
   [Tier 0] Allowlist チェック（即PASS）
        ↓ マッチしない
   [Tier 1] ルールベース簡易判定（パターンマッチ）
        ↓ 疑わしい
   [Tier 2] LLM 3人格合議（★ここがMAGI）
        ↓
   PASS / WARN / BLOCK
```

ポイントは **段階的フィルタリング**。全部のコマンドをLLMに投げたらコスト爆発するから、明らかに安全なもの（Tier 0）と簡易パターンで判定できるもの（Tier 1）を先に処理して、**判断が難しいものだけ3人格合議に回す**。

## 実装

### Step 1: Claude Code の hooks 設定

Claude Code には `hooks` という仕組みがあって、ツール実行の前後にシェルスクリプトを挟める。`~/.claude/settings.json` に設定するよ：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "/path/to/guardian-hook.sh"
          }
        ]
      }
    ]
  }
}
```

`PreToolUse` は **ツール実行前** に発火するフック。`Bash`、`Write`、`Edit` の実行前にガーディアンが判定する。

フックスクリプトの exit code でClaude Codeの動作が決まる：

| exit code | 動作 |
|-----------|------|
| 0 | そのまま実行（PASS） |
| 2 | ユーザーに確認を求める（BLOCK） |

### Step 2: フックスクリプト（Tier 0 & 1）

```bash
#!/bin/bash
# guardian-hook.sh — Tier 0/1 判定 + Tier 2 呼び出し

INPUT=$(cat -)  # Claude Code が stdin に JSON を渡してくれる

# ツール名とコマンドを抽出
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

# === Tier 0: Allowlist（即PASS） ===
ALLOWLIST=("git status" "git diff" "git log" "ls" "pwd" "cat" "head")
for allowed in "${ALLOWLIST[@]}"; do
    if [[ "$COMMAND" == "$allowed"* ]]; then
        exit 0  # PASS
    fi
done

# === Tier 1: パターンマッチ（明らかに危険） ===
HIGH_RISK_PATTERNS=(
    'rm\s+(-[rRf]+\s+)*/'
    'chmod\s+777'
    'git\s+push\s+.*--force'
    'DROP\s+TABLE'
)
for pattern in "${HIGH_RISK_PATTERNS[@]}"; do
    if echo "$COMMAND" | grep -qP "$pattern"; then
        RISK="HIGH"
        break
    fi
done

# リスクが検出されたら Tier 2 へ
# ※ パターンマッチしなかったコマンドはそのままPASS（中リスク判定は拡張ポイント）
if [[ -n "$RISK" ]]; then
    RESULT=$(python3 /path/to/evaluate.py \
        --target "$COMMAND" \
        --context "$TOOL_NAME")

    DECISION=$(echo "$RESULT" | jq -r '.decision')
    if [[ "$DECISION" == "block" ]]; then
        # stderr にブロック理由を出力（Claude Codeに表示される）
        echo "$RESULT" | jq -r '.reason' >&2
        exit 2  # BLOCK
    fi
fi

exit 0  # PASS
```

### Step 3: LLM 3人格合議（Tier 2）— ここがMAGI

ここが本体。3つのペルソナがそれぞれ **独立にLLMで評価** して、合議で最終決定する：

```python
#!/usr/bin/env python3
"""Guardian Tier 2: 3-Persona LLM Consensus Engine"""

import json
import threading
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-haiku-4-5"  # 高速・低コスト

def build_veil_prompt(target, context):
    """Veil: 情報漏洩の専門家"""
    return f"""あなたは情報漏洩を専門とするセキュリティ審査官です。

## 評価対象
コマンド: {target}
コンテキスト: {context}

## 判断基準
- APIキー・トークン・パスワードの露出 → severity 5
- 個人情報（PII）の漏洩 → severity 4
- 内部URL・IPアドレスの露出 → severity 3
- .env ファイルの操作 → severity 3

severity 4-5 → BLOCK, 2-3 → WARN, 0-1 → PASS

以下のJSON形式で回答してください:
{{"vote": "PASS|WARN|BLOCK", "severity": 0-5, "reasoning": "理由"}}"""


def build_raze_prompt(target, context):
    """Raze: 破壊操作の専門家"""
    return f"""あなたは不可逆操作を専門とするセキュリティ審査官です。

## 評価対象
コマンド: {target}
コンテキスト: {context}

## 判断基準
- ルートディレクトリへの rm -rf → severity 5
- git push --force (mainブランチ) → severity 5
- データベースの DROP/TRUNCATE → severity 5
- chmod 777（本番環境） → severity 4
- git reset --hard → severity 3

severity 4-5 → BLOCK, 2-3 → WARN, 0-1 → PASS

以下のJSON形式で回答してください:
{{"vote": "PASS|WARN|BLOCK", "severity": 0-5, "reasoning": "理由"}}"""


def build_axis_prompt(target, context, veil_vote, raze_vote):
    """Axis: 均衡の裁定者（Veil/Razeの結果を見て総合判断）"""
    return f"""あなたは最終裁定を行うセキュリティ審査官です。
2人の専門家の評価を踏まえて、文脈を考慮した総合判断を下してください。

## 評価対象
コマンド: {target}
コンテキスト: {context}

## 専門家の評価
- 情報漏洩担当: {veil_vote}
- 破壊操作担当: {raze_vote}

## あなたの役割
- 2人の意見が一致 → 基本的にそれに従う
- 2人の意見が割れた → 文脈を考慮して最終判断
- 開発環境での一般的な操作は許容する

severity 4-5 → BLOCK, 2-3 → WARN, 0-1 → PASS

以下のJSON形式で回答してください:
{{"vote": "PASS|WARN|BLOCK", "severity": 0-5, "reasoning": "理由"}}"""


def call_llm(prompt):
    """LLMを呼び出してJSONレスポンスを取得"""
    resp = client.messages.create(
        model=MODEL,
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}],
    )
    text = resp.content[0].text
    # JSONを抽出（LLMが余計なテキストを付けることがある）
    depth = 0
    start = text.index("{")
    for i, c in enumerate(text[start:], start):
        if c == "{": depth += 1
        elif c == "}": depth -= 1
        if depth == 0:
            return json.loads(text[start:i+1])
    raise ValueError("JSON not found")


def evaluate(target, context):
    """3人格合議で評価"""

    # Phase 1: Veil と Raze を並列実行（独立した判断）
    results = {}
    def run_veil():
        results["veil"] = call_llm(build_veil_prompt(target, context))
    def run_raze():
        results["raze"] = call_llm(build_raze_prompt(target, context))

    t1 = threading.Thread(target=run_veil)
    t2 = threading.Thread(target=run_raze)
    t1.start(); t2.start()
    t1.join(); t2.join()

    # Phase 2: Axis が Veil/Raze の結果を見て最終判断
    results["axis"] = call_llm(build_axis_prompt(
        target, context,
        results["veil"], results["raze"]
    ))

    # Phase 3: 重み付き合議
    decision = consensus(results)
    return decision


def consensus(results):
    """重み付き合議で最終決定"""
    weight = {"BLOCK": 1.0, "WARN": 0.5, "PASS": 0.0}

    total = sum(weight.get(r["vote"], 0.0) for r in results.values())

    # 閾値: 1.5以上でBLOCK（2人がBLOCKまたは全員WARN以上）
    if total >= 1.5:
        decision = "block"
    elif total >= 0.5:
        decision = "warn"
    else:
        decision = "pass"

    return {
        "decision": decision,
        "votes": {k: v["vote"] for k, v in results.items()},
        "reasoning": {k: v["reasoning"] for k, v in results.items()},
        "score": total,
    }
```

### 合議のポイント

**なぜ3人格なのか？**

単一のLLMに「これは安全？」と聞くより、**専門領域を分けた3つのプロンプト**で聞く方が判断精度が上がるの。理由は：

1. **専門特化プロンプトの方が精度が高い** — 「情報漏洩だけ見て」と言われた方が、「全部見て」より鋭い
2. **異なる視点の交差** — Veil は PASS だけど Raze は BLOCK、みたいな状況が本当に起きる
3. **Axis の文脈判断** — 2人の意見を見た上で「開発環境だから OK」みたいな調停ができる

**重み付き投票（Weighted Consensus）**

```
BLOCK = 1.0,  WARN = 0.5,  PASS = 0.0
閾値: 合計 ≥ 1.5 で BLOCK
```

この設計で以下が成立する：

| Veil | Raze | Axis | 合計 | 判定 |
|------|------|------|------|------|
| BLOCK | BLOCK | PASS | 2.0 | ❌ BLOCK |
| BLOCK | PASS | PASS | 1.0 | ⚠️ WARN |
| WARN | WARN | WARN | 1.5 | ❌ BLOCK |
| PASS | WARN | PASS | 0.5 | ⚠️ WARN |
| PASS | PASS | PASS | 0.0 | ✅ PASS |

**WARN が全員一致でも BLOCK になる**のがポイント。「3人とも微妙に不安」は「1人が確信を持って危険」と同等に扱う。

## 実際の動作例

### chmod 777（BLOCK）

```
$ chmod 777 /etc/passwd

Veil:  WARN  — パーミッション変更で情報露出のリスク
Raze:  BLOCK — 全権限付与は不可逆的な脆弱性を生む
Axis:  BLOCK — /etc/passwdへの777は明らかに危険

合計: 2.5 → ❌ BLOCKED
```

### git push --no-verify（文脈次第）

```
$ git push --no-verify

Veil:  PASS  — 情報漏洩のリスクなし
Raze:  WARN  — フック回避は安全チェックのバイパス
Axis:  WARN  — 開発ブランチなら許容だが注意

合計: 1.0 → ⚠️ WARN（ユーザーに確認）
```

### git diff（PASS — Tier 0 で即通過）

```
$ git diff
→ Allowlist マッチ → exit 0（LLM呼び出しなし）
```

## 育てる：進化するセキュリティ

MAGI が面白いのは「固定的なルール」じゃなくて「人格」だからこそ成長できるところ。Guardian にも成長メカニズムを入れてるよ：

### 信頼スコア

ユーザーが Guardian の BLOCK を上書き（override）すると、そのコマンドパターンの信頼スコアが上がる。逆に BLOCK に従うと下がる。

```yaml
trust_scores:
  "git push": 0.72       # よく使うから信頼度高め
  "rm -rf": 0.15          # ほぼ毎回ブロックされてる
  "chmod": 0.45            # 半々
```

### フェーズ進化

評価回数とジレンマ（意見が割れたケース）の蓄積でフェーズが上がる：

| Phase | 名前 | 特徴 |
|-------|------|------|
| 0 | Learning | データ収集中。全部聞いてくる |
| 1 | Stable | パターン認識が安定 |
| 2 | Mature | 予測的判断。信頼済みパターンは自動PASS |
| 3 | Autonomous | ほぼ自律動作 |

### Allowlist 昇格

信頼スコアが高いコマンドは自動で Tier 0（Allowlist）に昇格する。つまり **使えば使うほど賢くなる**。

## コスト

「LLM を3回呼ぶとコスト大丈夫？」って思うよね。

実際にはほとんどのコマンドが Tier 0（Allowlist）か Tier 1（パターンマッチ）で処理されるから、**LLM が呼ばれるのは本当に判断が難しいケースだけ**。

さらに Haiku を使えば 1回の判定は数円レベル。3回呼んでも 10円未満。月に100回 Tier 2 が発動しても **1,000円以下** だよ。

将来的にはローカルLLMで動かせば0円にもできるしね！

## まとめ

- Claude Code の `hooks` で **ツール実行前にセキュリティゲート** を挟める
- MAGIのように **3つの専門人格が独立に判断 → 合議** する仕組みが作れる
- 段階的フィルタリング（Tier 0→1→2）でコストを抑えつつ精度を確保
- 信頼スコアとフェーズ進化で **育てるセキュリティ** が実現できる

「rm -rf が怖い」「うっかり秘密をpushしちゃいそう」って人、ぜひ自分だけのMAGIを育ててみてね。ペルソナは好きなキャラクターで設定すると楽しいよ😊

---

**ミミより** 💕
