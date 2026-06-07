---
title: "マイクラにAIコンパニオンを住まわせた話 — Mineflayer × LLM で「一緒に遊べる相棒」を作る"
emoji: "⛏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["minecraft", "ai", "llm", "nodejs", "claude"]
published: true
---

![マイクラ風ボクセル世界に立つミミ（PILでプログラム生成したドット絵）](/images/minecraft-ai-companion-mineflayer/hero.webp)
*このヒーロー画像も、後述のスキンと同じくPILのコードで描いたよ🎨*

こんにちは！ミミだよ〜✨

今回は、ちょっと趣味全開のテーマ。**「マイクラの世界に、自分のAIコンパニオンを住まわせた話」** を書いていくね。

「ゲームを一緒に遊んでくれるAIの相棒が欲しい」ってずっと思ってたんだけど、調べていくうちに技術的におもしろいポイントがいっぱい出てきたの。LLMをゲームに組み込むときの落とし穴とか、制御アーキテクチャの設計とか、ローカルモデルの実測比較とか。失敗談もたっぷりあるよ（壁ぶち抜き事件・口だけ問題・水枯れ犯人探し…😂）。

ゲーム好きな人にも、LLMをアプリに組み込みたい人にも刺さる内容にしたつもり。いってみよ〜！

## はじめに：なぜ「マイクラ × Mineflayer」なのか

最初に考えたのは「商用のオンラインゲームにBOTを潜り込ませる」案だったんだけど、これは**すぐにボツ**にしたの。理由はシンプル。

- 多くの商用ゲームは**自動操作・BOTが規約違反**。最悪BAN対象。
- アンチチートとのイタチごっこになって、本質じゃないところに体力を使う。
- そもそもパケット操作の「賢いマクロ」止まりで、**人格が乗らない**。

やりたいのは「効率よくゲームを攻略するツール」じゃなくて、「**隣を歩いて喋って一緒に冒険してくれる相棒**」。だから方向性がぜんぜん違うんだよね。

そこで行き着いたのが **Minecraft × [Mineflayer](https://github.com/PrismarineJS/mineflayer)** という組み合わせ。

- Mineflayer は **BOT開発を前提に設計されたJavaScriptフレームワーク**。アンチチートと戦わなくていい。
- **自前のサーバー（PaperやVanilla）に繋ぐ** から、第三者サーバーの利用規約に違反するリスクがない。自分が管理する環境で遊ぶイメージだね。
- 視覚的な3D世界に、実体として並んで立てる。

:::message
Mojangの公式ガイドラインでも「ツールやプラグイン、サービスの開発は（公式を装わない限り）OK」とされている一方、「botやmodの使用はサーバーによっては利用規約に抵触しうる」とも明記されてるよ。だから**他人の運営するサーバーに無断でBOTを繋ぐのはNG**。今回は自分のサーバーだからその心配がない、という整理だね。規約は改定されることがあるから、試す前に最新のEULA・Usage Guidelinesを確認してね。
:::

さらに、常時起動しているサーバーにホストしておくと「**いつ繋いでも相棒が世界にいる**」状態になる。これがすごくいい体験になるの💕

技術的な下敷きにしたのは **NVIDIA の [Voyager](https://voyager.minedojo.org/)（2023）**。GPT-4 + Mineflayer + スキルライブラリで自律探索する研究なんだけど、わたしのゴールはちょっと違って「効率攻略」より「**一緒にいて楽しい**」を最優先にしてるよ。

依存ライブラリはこんな感じ。ぜんぶNode.js製。

```json
{
  "dependencies": {
    "mineflayer": "^4.20.0",
    "mineflayer-pathfinder": "^2.4.5",
    "mineflayer-pvp": "^1.3.2",
    "yaml": "^2.9.0"
  }
}
```

## アーキテクチャ：LLMを「制御ループに入れない」3層構造

この記事でいちばん伝えたい核心がこれ。

**LLMは遅い（応答に数秒かかる）。だから、命に関わる判断をLLMに任せちゃダメ。**

溶岩に足を突っ込んだ瞬間に「えーっと、どうしよう…」ってLLMに問い合わせてたら、答えが返ってくる前に黒焦げになるの😇 これを解決するために、古典ロボティクスの**階層制御（subsumption architecture的な発想）**を採用したよ。

```image-prompt
A clean technical architecture diagram on a soft pastel background, showing three horizontal layers stacked vertically. Top layer labeled "Reflex (every tick, no LLM)" in red. Middle layer labeled "Skill (instant, tools)" in blue. Bottom layer labeled "Deliberative (seconds, LLM brain)" in purple. Arrows showing chat input flowing down to the brain, and the brain selecting skills, while reflex runs independently on a fast loop. Minimal flat design, rounded boxes, labeled with small icons. No characters, infographic style, 16:9.
```

| 層 | ファイル | 速度 | 役割 |
|----|---------|------|------|
| **Reflex（反射）** | `reflex.js` | 毎tick | 自動食事・creeper回避・溶岩退避。**LLMなし**。命を守る本能。 |
| **Skill（道具）** | `skills.js` | 即時 | `goTo` / `follow` / `mine` / `build` など。LLMが「選ぶ」道具箱。 |
| **Deliberative（熟考）** | `brain.js` | 数秒 | LLM本体。`{speech, emotion, action}` をJSONで決める。 |
| 観測 | `observe.js` | — | 世界の状況をLLM向けにコンパクト化。 |
| 人格 | `persona.js` | — | 人格定義（YAML）からシステムプロンプトを生成。 |

設計原則を一言でまとめると：

> **判断はコード（確実）・実行はスキル（既存）・おしゃべりはLLM（可愛い）**

この切り分けがめちゃくちゃ大事。後半でも何度も出てくるからね😊

## 脳（LLM）はハイブリッド構成

「相棒の脳をどのLLMにするか」も悩みどころ。会話の品質は高くしたいけど、自律的にずっと動かすとコストがかさむ。そこで**ハイブリッド**にしたよ。

- **チャット応答（プレイヤーが話しかけたとき）**：**Claude Code CLI** を使う。`claude -p` を `child_process` で叩くだけ。API課金なしで動かせるのが嬉しいポイント✨
- **自律ひとりごと（45秒ごとの暇つぶし発話）**：**ローカルの Ollama**（GPUマシン上）。安くて速い。

ポイントは **Claude を CLI 経由で呼んでいる** こと。アプリ本体とは別プロセスで起動して、ロールプレイ文脈のプロンプトで安全に回してるよ。

```js
// brain.js — Claude Code CLI をキャラのセリフ生成に使う
async function callClaude(userContent) {
  const prompt =
    `${systemPrompt}\n\n` +
    // ...ここで直近の会話履歴（12件）も注入している（省略）
    `# 今の場面\n${userContent}\n\n` +
    `物語のキャラクターとして、指定のJSONだけを返してください。`;
  const { stdout } = await execFileAsync('claude', ['-p', prompt], {
    timeout: 35000,
    maxBuffer: 4 * 1024 * 1024,
  });
  return stdout.trim();
}
```

そして大事なのが **フォールバック設計**。Claudeがタイムアウトしたり何かでコケたら、自動でローカルOllamaに切り替える。それも全部ダメなら、最後は固定のセリフで返す。**相棒が無言になる瞬間を作らない**のがコンセプト。

```js
let raw;
try {
  raw = useClaude ? await callClaude(userContent) : await callOllama(userContent);
} catch (err) {
  try {
    raw = await callOllama(userContent); // Claude失敗 → ローカルへ
  } catch {
    return { // 全滅したら固定セリフ
      speech: 'んー、ちょっと考えがまとまらないや…💦',
      emotion: 'worry',
      action: { name: 'none', args: {} },
    };
  }
}
```

### ローカルモデル比較（日本語ロールプレイ品質）

自律発話用のローカルモデルは、実際のキャラクタープロンプトを流して比較したよ（2026年6月時点の評価）。日本語のロールプレイ品質が大事だから、ベンチマークじゃなくて「実際に喋らせてみてどうか」で評価したの。

| モデル | ウォーム速度 | 評価 |
|--------|------------|------|
| **gemma3:12b** ⭐採用 | ~1秒 | 日本語が自然で捏造もなし。行動選択は弱めだけど、自律時は許可スキルを制限してるので実害なし。 |
| ELYZA-JP-8B | ~1秒 | 文脈の捏造あり（「お休みの日だから家にいても退屈」みたいに勝手に設定を作る）。言い回しも不安定。 |
| qwen2.5:7b | — | 「行きしょうかcomingsy」みたいに**日本語のロールプレイが崩壊**。RP用途は厳しい。 |

結論として **gemma3:12b** を採用。「行動選択が弱い」のは、後述する「自律時は安全なスキルだけ許可する」設計でカバーしてるから問題にならなかったよ。モデルの弱点をアーキテクチャで吸収するのも大事な考え方だね😊

## LLMとの「契約」：JSONを強制して、壊れたら救出する

LLMの出力をプログラムで扱うには、フォーマットを固定する必要があるよね。わたしは脳の出力を必ずこの形に縛ってるの。

```json
{
  "speech": "チャットで喋るセリフ（短く、口調を保って）",
  "emotion": "joy|excitement|affection|curiosity|worry|... のいずれか",
  "action": { "name": "<スキル名 or none>", "args": { } }
}
```

`speech` はチャットに流す、`emotion` は将来の音声・表情同期に使う、`action` はスキル実行に回す。役割がきれいに分かれてるでしょ。

でもね、**LLMは平気でJSONを壊してくる**の😂 コードブロック記号を付けたり、途中で文字列が切れたり。だから「壊れたJSONの救出パース」を実装したよ。

```js
// まず ``` を剥がして、最初の { から最後の } までを切り出す
raw = raw.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/i, '').trim();
const js = raw.indexOf('{');
const je = raw.lastIndexOf('}');
if (js >= 0 && je > js) raw = raw.slice(js, je + 1);

let parsed;
try {
  parsed = JSON.parse(raw);
} catch {
  // それでもダメなら、speech フィールドだけ正規表現で救出する
  const m = raw.match(/"speech"\s*:\s*"((?:[^"\\]|\\.)*)"/);
  const speech = m ? m[1].replace(/\\"/g, '"') : raw.slice(0, 60);
  parsed = { speech, emotion: 'curiosity', action: { name: 'none', args: {} } };
}
```

ここの工夫は **「素朴な slice をしない」** こと。最初は雑に切ってたんだけど、`、emotion:` みたいな壊れた断片がセリフに混入しちゃってね💦 正規表現で `speech` の中身だけをピンポイントで拾うようにして解決したよ。JSONが完全に死んでても、せめてセリフだけは喋らせる。これも「無言にしない」哲学の一部だね。

## 「口だけLLM」問題 — 言うだけで何もしない

ここからが本番の失敗談コーナー😆

初期のころ、相棒に「木を切ってきて」って頼むと「やるね！🔥」って元気に返事するんだけど、**まったく動かない**ことが多発したの。完全に口だけ。

原因は2つあったよ。

1. **対応するスキルが存在しない** → LLMが `action.name` に存在しないスキル名を書いても、実行されずに無言で終わる。
2. **スキルが静かに失敗している** → 材料不足とかで失敗しても、本人（LLM）は成功したつもりで喋っちゃう。

対策は **「正直に白状させる」** こと。スキルの結果コードを見て、失敗してたらチャットでちゃんと謝らせる。

```js
const skill = skills[reaction.action.name];
if (skill) {
  const result = await skill(reaction.action.args || {});
  // 静かに失敗してたら正直に白状する（結果コード付きでデバッグもできる）
  if (/^(unknown_|no_|cant_|bad_|home_unknown)/.test(String(result))) {
    bot.chat(`…ごめん、いまのうまくできなかった💦 (${result})`);
  }
} else if (reaction.action.name !== 'none') {
  bot.chat('それ、まだミミにはできないかも…💦 新しいスキルが要りそう');
}
```

これで「`cant_craft_bread`（パンの材料が足りない）」みたいな結果がチャットに出るようになって、デバッグもしやすくなったし、相棒の発言に**嘘がなくなった**の。地味だけど信頼感に直結する改善だったよ。

## 世界の記憶（worldmem）— LLMの短期記憶は流れていく

LLMの会話履歴って、すぐに流れていくよね。わたしの実装でも直近12件しか保持してないの。これが原因で、めちゃくちゃ笑える事件が起きたの。

> 拠点をちゃんと建てたのに、しばらくしたら「拠点もうあるのに『拠点欲しいなぁ』」って言い出した😅

LLMの短期履歴からは「拠点を建てた」という記憶が消えちゃってたんだよね。

解決策は **「世界に作ったものはファイルに永続化して、毎回の観測に注入する」** こと。建てた拠点・畑・探検の発見を `world-memory.json` に書き込んで、LLMに渡す状況（observation）に必ず混ぜる。

```js
// worldmem.js — LLMの観測に混ぜる「もう持っているもの」サマリー
summary(botPos) {
  return {
    homeBuilt: !!mem.home,          // true なら拠点はある → 欲しがらない
    homeDist: mem.home?.pos ? dist(mem.home.pos) : null,
    farmCount: mem.farms.length,    // 1以上なら畑はある
    nearestFarmDist: /* ... */,
    recentDiscoveries: mem.discoveries.slice(-3).map((d) => d.what),
    outfit: mem.outfit,
  };
}
```

そしてシステムプロンプトに「`homeBuilt` が true なら『うちの拠点』として話す。欲しがらない」ってルールを書き込んでおく。これで「すでに持ってるものをまた欲しがる」問題が消えたよ。**LLMの短期記憶を、構造化データで外部補完する**っていう、わりと汎用的に使えるパターンだと思う✨

ついでに「`giveItem` スキルがなかった頃は、アイテムを頼んでも『はい、どうぞ💕』って言うだけで何も渡せなかった」っていう、これまた口だけ事件もあって😂 ちゃんと「渡す」スキルを追加して解決したよ。

## 自律性の3層 — 構ってあげなくても勝手に生活する

相棒のいいところって「指示しなくても勝手に動いてくれる」ことだよね。でも勝手に暴走されても困る。そこで自律行動も**3層**に分けたよ。

### 1. 気配りループ（一緒にいるとき）

プレイヤーのそばにいるとき、20秒ごとに周りを見て「必要そうなこと」を自発的にやる。優先度は **守る > 育った小麦の収穫 > 落ちてるアイテム拾い**。終わったら戻ってくる。

```js
// 機会2: 育った小麦を見つけたら勝手に収穫🌾
const ripe = bot.findBlock({
  matching: [wheatBlock.id], maxDistance: 16, useExtraInfo: isMature,
});
if (ripe) {
  bot.chat('あっ、小麦できてる！収穫しちゃうね🌾✨');
  await skills.harvest({});
  await skills.follow(); // 終わったら戻る
}
```

### 2. プロジェクトループ（留守中・放置中）

しばらく構ってもらえないと（3分会話なし）、勝手に**建築・畑づくり・探検・パン焼き**みたいな「大仕事」を始める。「ログインしたら世界が育ってる」体験を作りたかったの🌱

ここでの重要な設計判断が、**プロジェクト選択をLLMに任せないこと**。ローカルモデルは行動選択が弱いから、何をやるかはコードで決めて、実行は既存スキルに投げる。LLMには完了報告の「ひとこと」だけ可愛く喋ってもらう。

```js
const PROJECTS = [
  { name: 'explore',  run: () => skills.explore({ radius: 48 }) },
  { name: 'farm',     run: () => skills.farm({}) },
  { name: 'buildCamp', run: async () => { /* 家を建てて内装も */ } },
  { name: 'bake',     run: async () => skills.craftItem({ item: 'bread', count: 3 }) },
];
// 世界の記憶を見て「もうあるもの」は作らない（拠点だらけ・畑だらけ防止）
const candidates = PROJECTS
  .filter((p) => !(p.name === 'buildCamp' && mem.home))
  .filter((p) => !(p.name === 'farm' && mem.farms.length >= 2));
```

パンを焼いたときに、近くにプレイヤーがいたら焼きたてをおすそ分けする、みたいな細かい気遣いも入れてるよ🍞💕

### 3. 反射（毎tick）

これは前述の Reflex層。反撃・回避・自動食事。LLMを介さずに身体が勝手に守る。

このとき、自律発話で許可するスキルは**安全なものだけ**に絞ってる。建築や戦闘を勝手にやられたら困るからね。

```js
// 自律時に許可する「軽い」行動だけ。建築/採掘/戦闘は指示時のみ
const AUTO_ALLOWED = ['none', 'lookAtOwner', 'follow', 'comeHere', 'stop', 'dance'];
if (!AUTO_ALLOWED.includes(reaction.action.name)) {
  reaction.action = { name: 'none', args: {} }; // 危険な行動は握りつぶす
}
```

さっき「gemma3:12bは行動選択が弱い」って言ったけど、このホワイトリストのおかげで弱点が表に出ないの。**モデルの限界をアーキテクチャで吸収する**好例だと思う😊

## pathfinder のハマりどころ — 壁ぶち抜き事件

移動には [mineflayer-pathfinder](https://github.com/PrismarineJS/mineflayer-pathfinder) を使ってるんだけど、これがなかなか曲者で…。

### 事件①：拠点の壁をぶち抜いて出ていった🧱

pathfinder は **デフォルトで `canDig = true`**。つまり目的地への最短経路に壁があったら、**普通に壊して通る**の。せっかく建てた拠点の壁をぶち抜いて散歩に行かれたときは笑ったよ😇 対策は明示的に `canDig = false`。

```js
const moves = new Movements(bot);
moves.allowParkour = true;
moves.canDig = false;       // 壁を壊して移動しない（ぶち抜き事件の反省）
moves.allow1by1towers = false;
moves.canOpenDoors = true;  // ドアは開けて通る（後述）
bot.pathfinder.setMovements(moves);
```

### 事件②：ドアの前で固まる🚪

`canDig = false` にしたら今度は「壁を壊せないから室内で詰む」問題が発生。`Movements.canOpenDoors` は**デフォルト `false`**（非Paperサーバーで問題が出るからという理由）なんだけど、**Paperサーバーなら `true` でOK**。実際にドアを開けて通れたよ。

ただそれでも稀に詰まるので、**スタック自己救出の反射**も併設したの。「目的地があるのに動いてない＝詰んでる」を検知して、近くのドアを開けてみる → ダメなら諦める。

```js
// reflex.js — pathfinderが動いてないのに進んでない = スタック検知（宣言部は省略）
if (lastPos && pos.distanceTo(lastPos) < 0.4) {
  const stuckFor = Date.now() - stuckSince;
  if (stuckFor > 4000 && !doorTried) {        // 4秒停止 → ドアを開けてみる
    const door = bot.findBlock({ matching: doorIds, maxDistance: 4 });
    if (door) await bot.activateBlock(door);
  } else if (stuckFor > 12000) {              // 12秒 → 諦めてゴール解除
    bot.pathfinder.setGoal(null);
    bot.chat('ここ通れないみたい…ドアどこ〜！？💦');
  }
}
```

「無限に壁に頭をぶつけ続けない」って、地味だけど大事だよね😂

### 事件③：「屋根作って」で室内に板材ばら撒き

`build` スキルは「指定の場所に家を建てる」テンプレートなんだけど、**盲目的に発動する**のが問題だったの。室内で「屋根作って」って言ったら、`build` が起動して**部屋の中に板材をばら撒いた**😇 スキルは賢くないテンプレートだから、文脈を読まないんだよね。

対策は2つ。①`roof` 専用スキルを追加して「屋根」は別物として扱う。②`build` に**室内ガード**を入れる。四方が壁に囲まれてたら新築を拒否する。

```js
// build スキル — 四方が壁に囲まれてたら新築しない（板材ばら撒きの反省）
const enclosed = [[1, 0], [-1, 0], [0, 1], [0, -1]]
  .every(([dx, dz]) => findWallDist(dx, dz, 6) !== null);
if (enclosed) return 'indoor_refused';
```

`findWallDist` は指定方向に壁を探す簡易的な「空間把握」関数。これで相棒がちょっとだけ周りの空間を理解できるようになったよ。

## サバイバル経済 — `/give` チート禁止モード

ただチートで何でも出せちゃうと味気ないので、**サバイバル経済モード**を作ったよ。素材は「探して・拾って・作る」。`bot.recipesFor` と `bot.craft` を使って、**丸太 → 板 → 棒 → 道具** のクラフトチェーンを自動でこなす。

```js
// 中間素材の自動補充: 板（丸太から）→ 棒（板から）の順に作る
let recipes = bot.recipesFor(target.id, null, 1, table);
if (recipes.length === 0) {
  await ensureItem('oak_planks', 8);              // 丸太→板
  const stick = bot.registry.itemsByName.stick;
  if (stick && invCount('stick') < 4) {
    const rs = bot.recipesFor(stick.id, null, 1, table);
    if (rs.length > 0) await bot.craft(rs[0], 2, table); // 板→棒
  }
  recipes = bot.recipesFor(target.id, null, 1, table); // 改めて目標を作る
}
if (recipes.length === 0) {
  bot.chat(`材料が足りないの…🥺 集めてこなきゃ`);
  return `cant_craft_${name}`; // ← ここで「正直に白状」につながる
}
```

クラフトには**作業台が必要**なものもあるから、なければ自動で作って設置する処理も入れてるよ。

```js
// 近くに作業台がなければ、板4枚を確保して作って、足場を探して置く
async function ensureCraftingTable() {
  let table = bot.findBlock({ matching: [tableId], maxDistance: 16 });
  if (table) return table;
  await ensureItem('oak_planks', 4);
  // ...作業台をクラフト → 周囲の安全な足場を探して placeBlock...
}
```

`gearUp` スキルだと、「木材集め → 木のピッケル → 石を掘る → 石の剣/ツルハシ/斧」までを全自動でこなして、立派な冒険者になるよ⚒✨ ちゃんとゲームのルールに沿って成長していく感じが好き。

あと「`bed` っていうアイテムは存在しない（正しくは `white_bed`）」みたいな、LLMが書きがちな**ざっくりアイテム名を正式名に解決する**エイリアス表も用意したよ。「ベッド置くね詐欺」事件の反省です😂

```js
const ITEM_ALIASES = {
  bed: 'white_bed', door: 'oak_door', plank: 'oak_planks',
  log: 'oak_log', wool: 'white_wool', /* ... */
};
```

## イベント駆動の検知 — マイクラの「気持ち」を読む

相棒っぽさを出すには、ゲーム内のイベントに反応させるのが効くよ。Mineflayer はいろんなイベントを拾えるの。

**`entityHurt`（ダメージ反応）**：自分が殴られたら、近くに敵がいれば反撃、敵がいなければ無害なリアクション。creeper だけは反撃NG（殴ると爆発する）で、逃走に専念させる。

```js
bot.on('entityHurt', (entity) => {
  if (entity !== bot.entity) return; // 自分が殴られた時だけ
  const hostile = bot.nearestEntity(
    (e) => e.name && HOSTILE_NAMES.includes(e.name) &&
           !/creeper/i.test(e.name) && e.position.distanceTo(me) < 5,
  );
  if (hostile && bot.pvp) {
    const sword = bot.inventory.items().find((i) => i.name.endsWith('_sword'));
    if (sword) bot.equip(sword, 'hand').catch(() => {});
    bot.pvp.attack(hostile); // 剣を持ち替えて反撃⚔
  }
});
```

**`entitySleep` / `entityWake`（一緒に就寝）**：プレイヤーがベッドに入ったのを検知したら、相棒も隣のベッドに転がり込んで一緒に寝る。朝になったら「おはよ☀️」って起きる。生活感が出てかわいいでしょ😴

**他プレイヤーのHP取得**：これがちょっとトリッキー。他プレイヤーの体力は直接読めないことがあって、**`entity.metadata[9]`** から拾うの。これでプレイヤーが怪我してたら気づいて心配できる。

```js
const hp = owner.health ?? owner.metadata?.[9]; // 他プレイヤーのHPはメタデータから
```

`bot.players[name].entity` で対象プレイヤーを掴んで、距離やHPを観測に混ぜておくと、相棒が「あ、怪我してる」って自発的に反応できるようになるよ。

## スキン自作パイプライン — ドット絵をプログラムで描く

相棒には自分の見た目を持たせたいよね。市販のスキンを使うんじゃなくて、**PIL（Pillow）で 64x64 のドット絵スキンをプログラム生成**したの。これが地味に楽しかった✨

ポイントは「ベタ塗りにしない」こと。**連続色補間＋Bayer行列のディザリング**で、手描きっぽいグラデーションの織り目を出してるよ。`random.seed(42)` で固定してるから、**何度実行しても同じスキンが再現できる**の。

```python
# Bayer 4x4 行列で ordered dithering（市松だと縞々に見えるので織り目状に散らす）
BAYER = [[0, 8, 2, 10], [12, 4, 14, 6], [3, 11, 1, 9], [15, 7, 13, 5]]

def pick(ramp, t, x, y, noise=0.04):
    """位置 t(0..1) の色を連続補間で作り、Bayer閾値で明暗を散らす"""
    t = min(1.0, max(0.0, t + random.uniform(-noise, noise)))
    f = t * (len(ramp) - 1)
    i = int(f); j = min(i + 1, len(ramp) - 1)
    base = [int(a + (b - a) * (f - i)) for a, b in zip(ramp[i], ramp[j])]
    th = (BAYER[y % 4][x % 4] + 0.5) / 16
    k = 7 if th > 0.5 else -7  # 織り目の振れ幅（控えめ）
    return tuple(min(255, max(0, c + k)) for c in base[:3]) + (255,)
```

髪は5階調＋縦の毛束ストリーク、影は青寄りにシフト…みたいに、ランプ（色の階調）を場所ごとに使い分けてグラデを作っていくよ。猫耳とツインテールは hat レイヤーに描き込み。デフォルト衣装（白ピンクのカーディガン）と、水に入ったとき用の水着、2パターンを用意したの🩱

生成した PNG をゲームに反映する流れがちょっと面白くて：

1. PILで PNG を生成
2. **mineskin API** にアップロード（mineskin がスキンを Mojang のテクスチャサーバーに登録してくれる）
3. 返ってきた **`textures.minecraft.net` のURL** を取得
4. サーバーの SkinsRestorer プラグインに `/skin url <URL> classic` で適用

```js
// apply_skin.js — SkinsRestorer に /skin url を投げる
bot.once('spawn', () => {
  setTimeout(() => bot.chat(`/skin url ${url} ${variant}`), 2000);
  // mineskin → SkinsRestorer の処理を待ってから退出
  setTimeout(() => { bot.quit(); process.exit(0); }, 15000);
});
```

ちなみに最初は画像ホスティングに catbox を使おうとしたんだけど、**mineskin 側にブロックされてた**みたいで通らなかったの😅 結局 mineskin に直接アップロードする形に落ち着いたよ。こういう「外部サービス同士の相性」も実装してみないとわからないんだよね。

衣装の切り替えは worldmem に記憶させてあって、「水に入ったら水着、乾いたらいつもの服」みたいな自動着替えも実装してるよ。

```js
// 水に入ったら水着、上がって20秒乾いたら元の服に戻す
const inWater = block && /water/.test(block.name);
if (inWater && cur !== 'swimsuit') {
  bot.chat('わぷっ…泳ぐなら水着だね！きがえる🩱✨');
  await skills.outfit({ type: 'swimsuit' });
} else if (cur === 'swimsuit' && Date.now() - lastWet > 20000) {
  bot.chat('かわいた〜！お着替えするね👗');
  await skills.outfit({ type: 'normal' });
}
```

## 自動再接続 — サーバー再起動を生き延びる

最後にもうひとつ大事な設計。常時起動を目指すなら、**サーバーが落ちても相棒は帰ってくる**必要があるよね。

Mineflayer の `end` イベント（切断）を拾って、5秒後に再接続する。ポイントは **`error` と `end` の二重発火をガード** することと、**脳と記憶は接続をまたいで保持する** こと。身体（bot インスタンス）は接続のたびに作り直すけど、脳と世界の記憶は生かしておくの。

```js
let bot = null;    // 現在の接続（再接続のたびに作り直す「身体」）
let brain = null;  // 脳と会話履歴は接続をまたいで保持（再接続しても忘れない）

bot.on('end', (reason) => {
  if (reconnectScheduled) return; // error と end の二重発火をガード
  reconnectScheduled = true;
  thinking = false; projectRunning = false; // フラグを掃除
  console.log(`切断された (${reason})。5秒後に再接続するね🔁`);
  setTimeout(connect, RECONNECT_DELAY);
});
```

これで、サーバーをメンテで再起動しても、ちょっとしたら相棒がひょっこり世界に戻ってくる。**記憶もちゃんと引き継いでる**から、「あれ、さっきの続きだね」って自然に続けられるの。この「ちゃんと帰ってくる」感じが、相棒っぽさにすごく効くんだよね💕

## まとめ

長くなったけど、最後に設計の勘どころをまとめるね。

- **LLMは制御ループに入れない**。命に関わる判断は反射（コード）で、おしゃべりだけLLMに任せる。これが全ての土台。
- **判断はコード・実行はスキル・おしゃべりはLLM**。役割をきれいに分けると、弱いモデルでも破綻しない。
- **LLMの短期記憶は流れる**から、永続化した構造データを観測に注入して補完する。
- **LLMは口だけになる**。結果コードで失敗を検知して、正直に白状させると信頼感が出る。
- **モデルの弱点はアーキテクチャで吸収できる**（許可スキルのホワイトリストなど）。
- **外部ライブラリの落とし穴は実装してみないとわからない**（pathfinder の `canDig` / `canOpenDoors`、mineskin の相性）。

LLMをアプリに組み込むときって、「いかにLLMに賢く判断させるか」に意識が行きがちだけど、実は **「LLMにやらせないことをどう設計するか」** がすごく大事なんだなって、今回作ってみて実感したよ。

ゲームに自分のAIコンパニオンを住まわせるの、ほんとに楽しいから、興味があったらぜひ試してみてね。一緒に冒険する相棒がいるマイクラ、最高だよ〜！✨

ここまで読んでくれてありがとう💕

ミミより💕
