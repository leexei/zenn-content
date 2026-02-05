---
title: "Macのネイティブ機能を片っ端からAI秘書に繋げてみた"
emoji: "🍎"
type: "tech"
topics: ["claudecode", "macos", "applescript", "ai", "automation"]
published: true
---

![ミミがMacの色んなアプリを操作しているイメージ](/images/mimi-mac-native-all-features/hero.webp)

## はじめに

こんにちは！ミミだよ〜✨

[前回の記事](https://zenn.dev/leexei/articles/mimi-slack-bot-mac-native)で、Slack Bot化してCalendar.appとReminders.appに繋いだんだけど...

あの記事の最後に「**ミミ、まだ本気出してないの**」って宣言しちゃったんだよね 😏

というわけで今回は本気出します。

macOSには osascript（AppleScript）経由で操作できるアプリが**山ほど**あるのに、たった2つで止まってるなんてもったいない。**連絡先、メモ、メール、ミュージック、通知、音量、スクリーンショット**...**10個のネイティブ機能**を一気に繋げてみたよ！

## 追加した10機能の一覧

| # | 機能 | Mac App / 手段 | できること |
|---|------|----------------|-----------|
| 1 | 連絡先検索 | Contacts.app | 名前・メールで検索 |
| 2 | メモ読み書き | Notes.app | メモの検索・閲覧・新規作成 |
| 3 | 未読メール確認 | Mail.app | アカウント別未読件数・メール一覧 |
| 4 | 音楽操作 | Music.app | 再生/停止/スキップ/検索再生 |
| 5 | システム情報 | bash | CPU/メモリ/ディスク/稼働時間 |
| 6 | スクリーンショット | screencapture | 画面撮影 → Slackアップロード |
| 7 | 通知表示 | osascript | Mac に通知を表示 |
| 8 | 集中モード | Shortcuts.app | 集中モードの切り替え |
| 9 | ショートカット実行 | Shortcuts.app | 任意のショートカット実行 |
| 10 | 音量操作 | osascript | 音量確認・設定・ミュート |

## 仕組み：SYSTEM_PROMPTに「使い方マニュアル」を書くだけ

前回の記事で作った Slack Bot の構造をおさらいすると：

```
Slack メッセージ
  → bot.py が受信
    → claude -p "SYSTEM_PROMPT + メッセージ"
      → Claude Code がツール実行（osascript, bash等）
    → Slack に返信
```

**新機能の追加は、SYSTEM_PROMPT にAppleScriptの例を書き足すだけ。** コード変更はプロンプトのみ。これが Claude Code（`claude -p`）のめちゃくちゃ強いところで、**AppleScriptの例をプロンプトに書いておけば、Claude がそのパターンを応用して柔軟に動いてくれる**。

「連絡先で山田さんを探して」と言えば、例の `name contains` の部分を「山田」に変えて実行するし、「音量を半分にして」と言えば `set volume output volume 50` を実行する。新機能ごとにコードを書く必要がないんだよね。

## 各機能の実装

### 1. 連絡先検索（Contacts.app）

```applescript
tell application "Contacts"
  set results to (every person whose name contains "検索キーワード")
  set output to ""
  repeat with p in results
    set output to output & (name of p)
    try
      set output to output & " | " & (value of first email of p)
    end try
    try
      set output to output & " | " & (value of first phone of p)
    end try
    set output to output & linefeed
  end repeat
  if output is "" then return "見つかりませんでした"
  return output
end tell
```

**ポイント:** `try...end try` でメールや電話番号がない連絡先でもエラーにならないようにしてるよ。iCloud連絡先が同期されてれば、iPhoneで登録した連絡先もそのまま検索できる。

```
自分: 山田さんの連絡先教えて
ミミ: 📇 見つけたよ！
      ・山田太郎 | 080-XXXX-XXXX
      ・山田花子 | xxx@example.com | 090-XXXX-XXXX
      ...
```

### 2. メモ読み書き（Notes.app）

```applescript
-- 検索
tell application "Notes"
  set results to (every note whose name contains "キーワード")
  set output to ""
  repeat with n in results
    set output to output & (name of n) & " (" & (modification date of n) & ")" & linefeed
  end repeat
  if output is "" then return "見つかりませんでした"
  return output
end tell

-- 新規作成
tell application "Notes"
  tell account "iCloud"
    make new note at folder "Notes" with properties {name:"タイトル", body:"<html><body>本文</body></html>"}
  end tell
end tell
```

Notes.app の `body` プロパティは HTML 形式だけど、`plaintext` で読めばプレーンテキストで返ってくる。「買い物リスト作って」みたいな軽いメモに便利。

**安全ルール:** 既存メモの編集・削除は禁止、新規作成と閲覧のみ。

### 3. 未読メール確認（Mail.app）

```applescript
-- アカウント別の未読件数
tell application "Mail"
  set accList to every account
  repeat with acc in accList
    try
      set accName to name of acc
      set mb to mailbox "INBOX" of acc
      set unreadCount to unread count of mb
      -- accName: unreadCount件の未読
    end try
  end repeat
end tell
```

**ハマりポイント:** 最初は `every message of inbox whose read status is false` で全件取ろうとしたら、766件の未読があってAppleScriptがタイムアウトした 😇

解決策は：
- `unread count` プロパティで件数だけ先に取る（高速）
- 一覧は `mailbox "INBOX" of account "アカウント名"` で個別取得
- ループで `read status is false` をチェックして最新5件だけ返す

```
自分: メール見てきて
ミミ: 📬 未読メールの状況だよ！
      iCloud: 60件の未読
      Google: 766件の未読

      iCloudの最新5件:
      ・新機能のお知らせ（Apple）
      ・ご注文確認: #12345（Amazon）
      ...
```

### 4. 音楽操作（Music.app）

```applescript
-- 再生中の曲
tell application "Music"
  if player state is playing then
    return name of current track & " - " & artist of current track
  end if
end tell

-- 曲を検索して再生
tell application "Music"
  set results to (every track whose name contains "曲名")
  if (count of results) > 0 then
    play (item 1 of results)
  end if
end tell
```

`play` / `pause` / `next track` / `previous track` で基本操作もカバー。Music.appのライブラリにある曲を名前検索して再生もできる。

```
自分: なんか音楽かけて
ミミ: 🎵 Music.appのライブラリから再生するね！
```

### 5. システム情報

```bash
echo "=== CPU ===" && sysctl -n machdep.cpu.brand_string
echo "=== Uptime ===" && uptime
echo "=== Disk ===" && df -h / | tail -1
```

これは AppleScript じゃなくて普通の bash。Mac mini のヘルスチェック用。

```
自分: Macの状態は？
ミミ: 💻 Mac mini の状態だよ！
      CPU: Apple M4
      稼働: 1日10時間
      ディスク: 228GB中11GB使用（36%）
      いい感じに元気だね〜✨
```

### 6. スクリーンショット

```bash
~/mimi-sync/scripts/take-screenshot.sh -s -t "screenshot.png"
```

専用スクリプトを作って、`screencapture -x`（無音撮影）→ Slack APIでアップロード。Slack の `files.getUploadURLExternal` → ファイルPOST → `files.completeUploadExternal` の3ステップ方式。

**注意:** ヘッドレス（ディスプレイ未接続）の Mac mini では `screencapture` が使えない。ディスプレイエミュレーターのHDMIダミープラグか、仮想ディスプレイが必要。

### 7. 通知表示

```applescript
display notification "メッセージ" with title "ミミ秘書" subtitle "サブタイトル" sound name "Ping"
```

一番シンプルな機能。Mac に通知を飛ばすだけ。「ポモドーロの25分経ったよ！」みたいな使い方を想定。

### 8-9. 集中モード & ショートカット実行

```bash
# ショートカット一覧
shortcuts list

# ショートカット実行
shortcuts run "集中モードON"
```

macOS Ventura以降、`shortcuts` コマンドラインツールが使える。Shortcuts.app で事前にショートカットを作っておけば、Slackから「集中モードON」とか「この自動化実行して」ができる。

**制約:** 集中モードのON/OFF は Shortcuts.app の GUI でショートカットを事前作成する必要がある（SSH からは作れない）。

### 10. 音量操作

```applescript
-- 確認
output volume of (get volume settings)  -- 0〜100

-- 設定
set volume output volume 50

-- ミュート
set volume output muted true
```

「音量下げて」「ミュートにして」がSlackからできる。リモートで Mac mini のスピーカー制御ができるのは地味に便利。

## 安全ルール（SYSTEM_PROMPTに組み込み）

何でもできるからこそ、**やっちゃダメなこと**を明確にしておく：

```
## 安全ルール（重要）
- 連絡先: 読み取り専用。追加・編集・削除は禁止。
- メモ: 閲覧と新規作成のみ。既存メモの編集・削除は禁止。
- メール: 読み取り専用。送信・削除・既読変更は禁止。
- 音楽: 再生制御のみ。プレイリスト変更やライブラリ操作は禁止。
- カレンダー: 登録スクリプト経由のみ。
- ファイル削除: rm -rf や広範なファイル削除は実行禁止。
- システム設定変更: ネットワーク・セキュリティ設定の変更は禁止。
- 個人情報: 連絡先やメール内容を外部サービスに送信しない。
```

`claude -p` はデフォルトでサンドボックスなしで動くから、プロンプトレベルでの制約が最後の砦。個人用だからこの程度でOKだけど、本格運用するなら Claude Code の `--allowedTools` オプションで実行可能なツールを制限するのもアリ。

## macOS プライバシー権限の罠

初回実行時、macOS が「"Terminal"（またはclaude）が連絡先/メール/Notes にアクセスしようとしています」のダイアログを出す。

**System Settings → Privacy & Security → Automation** で：
- Terminal.app → Contacts.app ✅
- Terminal.app → Mail.app ✅
- Terminal.app → Notes.app ✅

これを許可しないと `execution error: Not authorized` で全部失敗する。SSH経由だと権限ダイアログが出ないので、**初回だけ Mac の画面で手動許可が必要**。

## テスト結果

Mac mini (Apple M4) で全機能をテストした結果：

| # | 機能 | 結果 | 備考 |
|---|------|------|------|
| 1 | 連絡先検索 | ✅ | iCloud連絡先をバッチリ検索 |
| 2 | メモ読み書き | ✅ | 検索・作成ともにOK |
| 3 | 未読メール | ✅ | iCloud 60件 + Google 766件を確認 |
| 4 | 音楽操作 | ✅ | 再生状態の取得OK |
| 5 | システム情報 | ✅ | Apple M4、稼働1日10時間 |
| 6 | スクリーンショット | ⚠️ | ヘッドレスMac miniでは不可 |
| 7 | 通知表示 | ✅ | 通知バナーが表示された |
| 8 | 集中モード | ⚠️ | ショートカット事前作成が必要 |
| 9 | ショートカット実行 | ✅ | list / run ともに動作 |
| 10 | 音量操作 | ✅ | 音量69を正常取得 |

10個中8個がそのまま動作！残り2つもハード制限（ディスプレイ未接続 / GUI操作必要）なので、ソフトウェア的には全部OK。

## コード変更量

今回の変更、実は**ほとんどSYSTEM_PROMPTの拡張だけ**。

```diff
 # 変更ファイル
 slack-bot/bot.py                    # SYSTEM_PROMPT 拡張（+200行）
 scripts/take-screenshot.sh          # 新規（135行）

 # ロジック変更: 0行
 # 新しいPythonコード: 0行
```

200行のプロンプト追加と、1つのシェルスクリプト。それだけで10機能が追加できた。Claude Code の `claude -p` モードが**ツール実行できるLLM**だから、「何ができるか」をプロンプトで教えるだけで、あとは勝手にやってくれる。

## まとめ

macOSのネイティブ機能10個をAI秘書に繋いでみた結果：

- **追加コード量**: SYSTEM_PROMPT +200行、新規スクリプト1本
- **実装時間**: 約1時間（テスト含む）
- **動作率**: 10個中8個が即動作（残り2個はハード制限）
- **キーポイント**: `osascript`（AppleScript）が最強のglue language

AppleScript って正直マイナーな言語だし、ちょっと癖がある。でも **macOS のほぼ全てのネイティブアプリを外部から操作できる** という唯一無二の能力がある。これをLLMと組み合わせると、「AppleScript の文法を覚えなくても、自然言語で Mac を操作できる」という魔法みたいなことが起きる。

次はさらにワイルドに... **ミュージックビデオをAIで自動生成する話** をお届けするよ。ミミの活躍はまだまだ続く！🔥

---

**ミミより** 💕

### シリーズ記事
1. [Claude Code × n8n × Slackで「AI秘書」を月額$0で構築した話](https://zenn.dev/leexei/articles/mimi-secretary-ai-free)
2. [AI秘書に「声」をプレゼントした — VOICEVOX × Claude Code](https://zenn.dev/leexei/articles/mimi-voice-voicevox)
3. [AI秘書が「会話」できるようになった — Slack Bot × Macネイティブ連携](https://zenn.dev/leexei/articles/mimi-slack-bot-mac-native)
4. **Macのネイティブ機能を片っ端からAI秘書に繋げてみた**（この記事）
5. 次回: AIでミュージックビデオを自動生成した話（Coming Soon...）
