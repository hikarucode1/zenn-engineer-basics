---
title: "AIに『3人の専門家会議』をさせる ― マルチエージェントAI鏡『トリアージュ』を支える4つの実装"
emoji: "🪞"
type: "tech"
topics: ["claude", "nextjs", "ai", "youcam", "マルチエージェント"]
published: true
---

> 本記事は [Zenn Fes Spring 2026 / YouCam API コンテスト](https://zenn.dev/contests/zennfes-spring-2026-perfect) 応募作品 **「トリアージュ (Triage)」** の技術解説です。

🔗 **実際に動くデモ**: https://mirror-council.vercel.app/ （スマホのカメラで顔を撮ると、3人の専門家AIがその場で会議します。サインアップ不要）

## TL;DR

カメラに映った顔を解析し、**3人の専門家AIエージェント**（美容皮膚科医・内科医・東洋医学師）が**同時に**議論、統合エージェントが優先度付きアクションプランへまとめる Web アプリ **「トリアージュ」** を作りました。

本記事のテーマは **「1つのLLMに複数の人格で会議をさせる」マルチエージェント実装** です。次の4つを実コードとともに解説します。

1. **人格の分離** ― 1本の system prompt と `<role>` タグで3人格を出し分け、混線を防ぐ
2. **Prompt Caching** ― 共有 system prompt を4コールで使い回し、2回目以降のコストを大幅削減
3. **SSE で"会議"をライブ配信** ― 3エージェントを並列実行し、発言を逐次ストリーム
4. **tool_use で構造化出力を強制** ― 統合結果を壊れない JSON で受け取り、検証＋リトライ

---

## なぜ「会議型」なのか

肌や健康の悩みに対して、1人の専門家が出す答えは **1つの視点** でしかありません。

- 美容皮膚科医は「シミにはトラネキサム酸」と**成分・施術**で答える
- 内科医は「クマは睡眠と鉄不足かも」と**生活習慣**から見る
- 東洋医学師は「乾燥は肺の弱り、白い食材を」と**体質・食養生**で捉える

同じ顔を見ても、立つ場所が違えば助言は変わる。本来この「重なり」の中に答えがあるのに、普通のAIアプリは1つの声しか返しません。

**トリアージュ**は、この複数視点を **明示的な「会議UI」** として可視化します。名前の由来は医療の **トリアージュ（優先度判定）** ―― 3つの視点で肌の課題を振り分け、「今夜やること／今週／今月」に優先度付けする、というコンセプトに直結しています。

```
[撮影] → [3賢者が同時に観察] → [会議(チャットUI)] → [統合: アクションプラン]
```

| エージェント | 役割 | 担当指標 |
|---|---|---|
| 👩‍⚕️ Dr. Western | 西洋的・データドリブン | wrinkle / pore / texture / acne / firmness / age_spot |
| 🩺 Dr. Internal | 全身の健康サインから推定 | dark_circle / eye_bag / radiance / tear_trough |
| 🍵 Master Toyo | 五臓と肌、食養生・漢方 | moisture / oiliness / redness / skin_type / 上下まぶた |
| 🪞 統合エージェント | 議論を整理し優先度付きプランへ | (3者の所見を集約) |

肌の解析には **YouCam Skin Analysis API（HDモード）** を使い、14の健康度スコア + 肌質 + 推定肌年齢を取得しています。

### 🎬 デモ（実機フル疎通・58秒）

撮影 → 3賢者の並列会議 → アクションプランまでの実機フローです。

@[youtube](qlyhhvTCy1o)

---

## 実装1: 1つの system prompt で3人格を出し分ける

マルチエージェントというと「エージェントごとに別々のプロンプト」を思い浮かべますが、トリアージュは **全役割で system prompt を1本に統一** しています。理由は後述の Prompt Caching ですが、その代わり **人格の混線（クロストーク）** という品質リスクが生まれます。Dr. Western が「気の巡りが…」と言い出したら台無しです。

これを防ぐため、共有 system prompt の中に **役割ごとのトーン基準表** を明文化しました。

```ts
// src/lib/claude/system.ts (抜粋)
## Dr. Western (美容皮膚科医) の役割
- トーン: 簡潔・定量的。エビデンスベース。一人称は「私」。
- 専有語彙: レチノール、ナイアシンアミド、トラネキサム酸、エビデンス、ターンオーバー…
- 禁止語彙: 気 / 養生 / 自律神経 / 五臓 / 漢方 / 巡り / 陰陽

## Master Toyo (東洋医学師) の役割
- トーン: 比喩的・ゆったり。一人称は「私」。
- 専有語彙: 五臓(肝/心/脾/肺/腎)、気血水、陰陽、食養生、薬膳、白湯、巡り
- 禁止語彙: ガイドライン / 自律神経 / 処方箋 / レチノール / エビデンス / ホルモン
```

ポイントは **「専有語彙」と「禁止語彙」をペアで持たせる** こと。「東洋医学っぽく」のような曖昧な指示ではなく、*使ってよい単語* と *他役割の領域なので使ってはいけない単語* を具体語で線引きすると、人格境界が安定します。

実際に演じる役割は **user message 側の `<role>` タグ** で切り替えます。

```ts
// src/lib/claude/sage.ts (抜粋)
return `<role>${role}</role>
<skin_analysis>
${skinAnalysis}
</skin_analysis>
<my_focus>${focus}</my_focus>

上記の解析結果について、あなたの役割で観察→解釈→提案の順に120字以内で述べてください。`;
```

system prompt（＝人格カタログ）は固定のまま、user message の `<role>` だけが変わる。**変動部分を user 側に寄せる**この分離が、次の Prompt Caching を効かせる鍵になります。

---

## 実装2: 共有 system prompt を Prompt Caching で使い回す

トリアージュは1回の会議で **Claude を4回**呼びます（3賢者 + 統合）。4回とも同じ system prompt（肌指標の解説 + 3人格定義、約2,300トークン）を送るので、ここは Prompt Caching の出番です。

```ts
system: [
  {
    type: "text",
    text: SAGE_BASE_SYSTEM,
    cache_control: { type: "ephemeral" },  // ← これ
  },
],
```

1人目の賢者がキャッシュを書き込み（`cache_creation`）、続く2〜4人目は同じ system prompt を**読むだけ**（`cache_read`）になります。実測では2回目以降の呼び出しで **`cache_read = 13,257 トークン`** を確認できました。

:::message
**ハマりどころ: キャッシュには最小トークン数がある**
Sonnet 系のプロンプトキャッシュは **最低 2,048 トークン** ないと、`cache_control` を付けても**黙って無効化**されコストが跳ね上がります。日本語は概ね1.5文字/トークンなので、system prompt が短くなりすぎないよう **文字数カナリア** を仕込みました。
:::

```ts
// src/lib/claude/system.ts
const MIN_CHARS_FOR_CACHE = 3500; // ≒2,300 tokens（2,048の閾値にマージン）

if (SAGE_BASE_SYSTEM.length < MIN_CHARS_FOR_CACHE) {
  console.warn(`[SAGE_BASE_SYSTEM] length=... < ${MIN_CHARS_FOR_CACHE} — `
    + `cache_control が無効化され単価が約3倍になります`);
}
```

さらに**ランタイム側にも二重の保険**を入れています。最初の賢者のレスポンスで `cache_read === 0 && cache_creation === 0`（＝キャッシュが一切効いていない）なら、プロセスに1回だけ警告を出す ―― 静的な文字数チェックを、API が返す**権威ある usage 値**でバックストップする構成です。

```ts
// src/app/api/council/route.ts (抜粋)
if (!cacheMissWarned && evt.result.cacheRead === 0 && evt.result.cacheCreation === 0) {
  cacheMissWarned = true;
  console.warn(`[council] prompt cache inactive on first sage — `
    + `system prompt が 2048 トークン未満か、無効化要因が混入`);
}
```

「キャッシュが効かなくなっても気づけない」のがこの手の罠の怖いところなので、**静的見積もり＋実測値の二段構え**で検知できるようにしました。

---

## 実装3: SSE で"会議"をライブ配信する

会議型UIの肝は **「3人が同時に考えて、書けた人から喋り出す」** 体験です。これを Server-Sent Events で実現しました。

![3賢者が並列にストリーミングされる会議画面（実機デモ）](/images/triage-council.gif)
*↑ 3つの吹き出しがバラバラのタイミングで書き上がっていくのが SSE 並列配信の証拠。ソースは `トリアージュ_会議ハイライト.gif`*

3賢者は `Promise.all` で**並列実行**し、各賢者のストリーミング delta を `western_delta` / `internal_delta` / `toyo_delta` という**役割ごとのイベント名**でクライアントへ流します。

```ts
// src/app/api/council/route.ts (抜粋・簡略化)
await Promise.all(
  SAGES.map(async (role) => {
    write(`${role}_start`, {});
    for await (const evt of runSage(client, role, results, signal)) {
      if (evt.type === "delta") {
        write(`${role}_delta`, { text: evt.text });   // ← 賢者ごとに逐次push
      } else {
        write(`${role}_done`, { text: evt.result.text, usage: {...} });
      }
    }
  }),
);
```

役割ごとにイベント名を分けておくと、クライアントは「どの吹き出しに追記するか」を**イベント名だけで**判別でき、UI 実装がシンプルになります。

設計上こだわった点：

- **graceful degradation**: 3賢者の `try/catch` は独立。1人が失敗しても `*_error` を流して残り2人で会議は続行。3人**全員**失敗したときだけ統合をスキップする
- **クライアント切断のレース対策**: `controller.enqueue` を `closed` フラグと `try/catch` でガード。ユーザーが途中で離脱しても enqueue/close の二重呼びで落ちない
- **`maxDuration = 60` / Vercel**: 会議全体の実測は約20秒。Hobby プランでも fluid compute で上限まで回るため、これで足ります

3人が並走して書き上がった順に喋り、最後に統合役が登場する ―― この**時間軸そのものが会議の演出**になります。

---

## 実装4: tool_use で「壊れない構造化出力」を受け取る

統合エージェントの出力は **UI が描画する構造化データ**（総括 + 優先度付きアクション配列 + トレードオフ + 受診フラグ）です。ここでLLMに自由テキストの JSON を書かせると、ときどき壊れます。

そこで **tool_use を強制** しました。`tool_choice` で `emit_council_summary` ツールの呼び出しを固定し、出力スキーマを JSON Schema で縛ります。

```ts
// src/lib/claude/integrator.ts (抜粋)
const stream = client.messages.stream({
  model: "claude-sonnet-4-6",
  tools: [EMIT_COUNCIL_SUMMARY_TOOL],
  tool_choice: { type: "tool", name: "emit_council_summary" }, // ← 必ずこのツールを呼ばせる
  system: [{ type: "text", text: SAGE_BASE_SYSTEM, cache_control: { type: "ephemeral" } }],
  messages: [{ role: "user", content: userMessage }],
});
```

スキーマ側では `actions` を3〜5件、`id` を `^act-[1-9]$`、`timeframe` を `today|this_week|this_month|long_term` の enum…と細かく制約しています。

```ts
// src/lib/claude/tools.ts (抜粋)
actions: {
  type: "array", minItems: 3, maxItems: 5,
  items: { /* id / priority / timeframe / category / task / from / rationale */ },
},
```

それでもLLM出力は信用しきれないので、**受け取った後に自前バリデータで再検証**し、失敗したら**温度を下げてリトライ**します。

```ts
// 1回目: temperature 0.3
const attempt1 = await runOneAttempt(client, userMessage, 0.3, onPartial, signal);
if (attempt1.ok) { yield { type: "done", summary: attempt1.summary, attempts: 1 }; return; }

// 2回目: temperature 0.1 でリトライ
const attempt2 = await runOneAttempt(client, userMessage, 0.1, null, signal);
if (attempt2.ok) { yield { type: "done", summary: attempt2.summary, attempts: 2 }; return; }

// それでもダメなら fallback（賢者の所見を生で見せる）
yield { type: "fallback", reason: `validation failed twice`, observations };
```

**「スキーマで縛る → それでも検証する → ダメなら温度を下げて1回だけ再挑戦 → 最後は素の所見にフォールバック」** の4段構え。tool_use があっても**アプリ側の検証を省かない**のが、壊れないAI機能のコツだと思います。

なお統合の `partial_json` も SSE で流しているので、UIでは**アクションプランが組み上がっていく様子**もストリーム表示できます。

---

## おまけ: YouCam 実機連携で踏んだ罠

マルチエージェントが主役の記事ですが、**入力（肌解析）側**でも実機検証で何度か転びました。同じ API を使う人向けにメモを残します。

- **エラーは文字列で返る**: 解析失敗は `data.error` に `error_no_face` / `error_below_min_image_size` のような**文字列**で来る。オブジェクトの `error.code` だけ見ていると全エラーが unknown に潰れる
- **最小画像サイズは「短辺」に効く**: 短辺 1036px は却下、1450px は通過。**短辺 ≥ 1500px** なら確実に通る（アップスケールでも顔検出OK）
- **iOS Safari の WYSIWYG 崩れ**: プレビューは 3:4 表示なのに、`getUserMedia` のスナップショットはカメラ全フレーム（横長）を撮るため**顔比率が縮小**し `error_src_face_too_small` に。**表示と同じ 3:4 にクロップ**して解消

カメラUIは「見えている通りに撮れている」とは限らない、という良い教訓でした。

---

## 技術スタック

| カテゴリ | 採用 |
|---|---|
| Framework | Next.js 16 (App Router) + React 19 |
| 言語 | TypeScript 5.9 (strict) |
| UI | Tailwind CSS 4 + shadcn/ui |
| 状態 | Zustand 5 + localStorage（サインアップ不要・local-first） |
| PWA | Serwist 9.5 |
| 肌解析 | YouCam Skin Analysis API (HD) |
| AI | Claude Sonnet 4.6 × 並列3 + 統合1（Prompt Caching + SSE + tool_use） |
| デプロイ | Vercel |

---

## まとめ

「複数の専門家に同時に診てもらう」という体験を、**1つのLLMに複数人格で会議させる**ことで実装しました。鍵は4つ：

1. **人格分離** ― system prompt 1本 + `<role>` タグ + 専有/禁止語彙でクロストークを防ぐ
2. **Prompt Caching** ― 共有 system を4コールで使い回し、二重カナリアで無効化を検知
3. **SSE 並列ストリーム** ― `Promise.all` + 役割別イベントで"会議"を時間軸ごと配信
4. **tool_use + 自前検証 + 温度下げリトライ** ― 壊れない構造化出力

LLMアプリは「賢いプロンプト1発」になりがちですが、**役割を分けて議論させ、その過程を見せる**だけで、体験の説得力が大きく変わります。医療×ビューティのように、本来複数の視点が重なって成立する領域とは特に相性が良いと感じました。

:::message alert
トリアージュは医療行為ではありません。表示される助言は一般的な情報の整理であり、診断・治療の代替にはなりません。
:::
