---
title: "Claude API × TypeScript | cache_controlの配置からヒット率の計測・失効原因の特定まで"
tags:
  - Claude
  - TypeScript
  - PromptCaching
  - Anthropic
  - LLM
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「cache_control を付けたのに請求が全然下がらない」「ヒットしているのか、そもそも書き込まれているのかも分からない」

——この感覚は正しいです。プロンプトキャッシュはエラーも警告も出さずに効かなくなります。最小トークン数に届いていなくても、システムプロンプトに日時が混ざっていても、レスポンスは 200 で返ってきます。気づく手段は `usage` の数字だけです。

この記事では、TypeScript で次の 4 ステップを動くコードで実装します。

1. `cache_control` を置く位置を決める
2. `usage` からヒット率を計測する
3. 無言の失効原因を潰す
4. TTL を選ぶ

## 先に押さえる仕様(公式ドキュメントとスキル同梱資料で確認した値)

2026-10-03 時点で確認した内容です。モデルや価格は更新されるので、実装前に公式の prompt caching ページを見てください。

| 項目 | 内容 |
|---|---|
| 一致方式 | 前方一致。1 バイトでも違えばそれ以降は全て失効 |
| 描画順 | `tools` → `system` → `messages` |
| ブレークポイント上限 | 1 リクエストにつき 4 個 |
| TTL | 既定 5 分、`ttl: "1h"` で 1 時間 |
| 書き込み料金 | 5 分 TTL は入力の 1.25 倍、1 時間 TTL は 2 倍 |
| 読み取り料金 | 概ね 0.1 倍(モデルによって異なり、Opus 5.5 は 0.05 倍) |
| 最小トークン数 | モデル依存。512 / 1024 / 2048 / 4096 のいずれか |

最小トークン数は世代順に増減しません。新しいモデルは 512 ですが、Opus 4.6 や Haiku 4.5 は 4096 です。3,000 トークンのプロンプトが Sonnet 5.5 でキャッシュされても、Haiku 4.5 では黙って素通りします。

この記事のコードは `claude-sonnet-5-5`(入力 $2 / 出力 $10 per 1M tokens、キャッシュ読み取り $0.20)を使います。最小トークン数は資料に「公式で再確認」と注記があったため、`cache_creation_input_tokens` が 0 のままなら、まず最小値を疑ってください。

## Step 1 | cache_control を置く位置を決める

「長い社内規程を渡して、ユーザーの質問に答える」ケースで考えます。変わらないのは規程本文、毎回変わるのは質問です。ブレークポイントは**共有部分の末尾**に置きます。質問の後ろに置くと、リクエストごとに別のキャッシュエントリを書くだけで、一度も読まれません。

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();
const MODEL = "claude-sonnet-5-5";

// 規程本文は固定。日時・ユーザー名・リクエストIDは絶対に混ぜない
const POLICY_TEXT = await Bun.file("./policy.md").text();

export async function ask(question: string) {
  return client.messages.create({
    model: MODEL,
    max_tokens: 16000,
    system: [
      { type: "text", text: "あなたは社内規程の問い合わせ窓口です。" },
      {
        type: "text",
        text: POLICY_TEXT,
        // 共有部分の末尾にだけ置く
        cache_control: { type: "ephemeral" },
      },
    ],
    // 質問はブレークポイントより後ろ(毎回変わる部分)
    messages: [{ role: "user", content: question }],
  });
}
```

マーカーを置いたブロックまでの `tools` と `system` がまとめてキャッシュされます。ツールを使う場合も、`tools` の定義順を固定してください(後述)。

会話履歴が伸びるチャットなら、トップレベルに `cache_control: { type: "ephemeral" }` を置く自動キャッシュが楽です。最後のキャッシュ可能ブロックに自動でブレークポイントが移動します。ただし末尾に毎回変わる検索結果が付くプロンプトでは、その末尾にまで書き込みが発生して書き込み割増だけを払うことになります。その場合は明示マーカーに切り替えます。

**防止策: マーカーは「次のリクエストでも同じバイト列になる最後のブロック」に置く。毎回変わる文字列より後ろには置かない。**

## Step 2 | usage からヒット率を計測する

効いたかどうかは `usage` の 3 つのフィールドで判定します。

```typescript
import type Anthropic from "@anthropic-ai/sdk";

type CacheStats = {
  uncached: number; // ブレークポイント以降の通常課金分
  written: number; // 今回キャッシュへ書き込んだ分(割増)
  read: number; // キャッシュから読んだ分(割引)
  hitRate: number; // read / 入力合計
};

export function cacheStats(usage: Anthropic.Usage): CacheStats {
  const uncached = usage.input_tokens;
  const written = usage.cache_creation_input_tokens ?? 0;
  const read = usage.cache_read_input_tokens ?? 0;
  const total = uncached + written + read;
  return { uncached, written, read, hitRate: total === 0 ? 0 : read / total };
}
```

`input_tokens` はブレークポイント以降の分だけを指します。全入力ではない点が最初の落とし穴です。合計は 3 つを足して出します。

同じ質問を 2 回投げて確認します。

```typescript
const a = await ask("経費精算の締め日はいつですか?");
const b = await ask("出張時の日当の上限を教えてください。");

console.log("1回目", cacheStats(a.usage)); // written が大きく、read は 0
console.log("2回目", cacheStats(b.usage)); // read が大きく、written は 0
```

期待する結果は、1 回目が書き込み、2 回目が読み取りです。2 回目も `read` が 0 なら、失効原因か最小トークン数未満のどちらかです。

料金の見積もりも関数にしておくと、変更前後の比較に使えます。倍率は上の表の値です。

```typescript
// Sonnet 5.5 の単価(USD / 1Mトークン)。モデルを変えたら必ず直す
const PRICE = { input: 2.0, read: 0.2 } as const;

export function inputCostUsd(s: CacheStats): number {
  return (
    (s.uncached * PRICE.input +
      s.written * PRICE.input * 1.25 + // 5分TTLの書き込み
      s.read * PRICE.read) /
    1_000_000
  );
}
```

計算例です(実測値ではありません)。規程が 40,000 トークンで、5 分以内に 10 回問い合わせが来る場合を考えます。キャッシュなしなら 40,000 × 10 × $2 / 1M = $0.80。キャッシュありなら、書き込み 1 回 40,000 × $2.5 / 1M = $0.10 に、読み取り 9 回 40,000 × 9 × $0.2 / 1M = $0.072 を足して約 $0.172 です。2 回目から元が取れ、回数が増えるほど差が開きます。

**防止策: 本番では `cacheStats` の結果をログに出し、`read` が 0 のリクエスト率を監視する。数日後に気づくのでは遅い。**

## Step 3 | 無言の失効原因を潰す

`read` が 0 のまま動かないとき、疑う順番は次の通りです。

| 症状 | 原因 | 直し方 |
|---|---|---|
| `written` も 0 | 最小トークン数未満 | モデルの最小値を確認し、共通部分を増やすか諦める |
| 毎回 `written` が大きい | system に `new Date()` や UUID が混入 | 動的な値は `messages` 側へ移す |
| 同上 | JSON のキー順が毎回違う | キーをソートして直列化する |
| 同上 | `tools` の順序や中身が変わる | 名前でソートして固定する |
| モデル切替後に `read` が 0 | キャッシュはモデル単位 | ルーティング時はモデルごとにウォームする |
| `thinking` や `effort` 変更後に失効 | 設定がプレフィックスに含まれる | ルートごとに固定する |

ツール定義の固定は、次のように書いておくと混入を防げます。

```typescript
export function stableTools(tools: Anthropic.Tool[]): Anthropic.Tool[] {
  return [...tools].sort((a, b) => a.name.localeCompare(b.name));
}
```

原因の目星がつかないときは、直近 2 リクエストの本文 JSON を保存して差分を取ります。`cache_control` のマーカー自体は毎回動くので、比較前に除去します。重なっている範囲で最初に食い違うバイトが失効点です。

もう 1 つ、長いツールループには 20 ブロックの遡り制限があります。ブレークポイントは直前のキャッシュ位置を最大 20 ブロック分しか遡りません。1 ターンで大量に積むと、前回のエントリが窓から外れて毎回全量書き込みになります。その場合は約 15 ブロックごとに中間のブレークポイントを足します。

**防止策: 動的な値を system に入れない、ツール順を固定する、設定(thinking / effort)をルートごとに固定する。この 3 点をコードレビューのチェック項目にする。**

## Step 4 | TTL を選ぶ

TTL の選択基準は、同じプレフィックスを使うリクエストの「開始から開始までの間隔」です。

```typescript
// 1時間TTLが効くのは「5〜60分間隔でまばらに来る」トラフィック
{
  type: "text",
  text: POLICY_TEXT,
  cache_control: { type: "ephemeral", ttl: "1h" },
}
```

- 間隔が 5 分未満: 既定の 5 分で十分です。読むたびに期限が延びるので、1 時間にすると書き込みが 2 倍になるだけです
- 間隔が 5〜60 分: 1 時間 TTL を検討します。書き込みが 2 倍で、読み取りが 2 回以上ないと割に合いません(1 時間 TTL は 2 倍 + 0.2 倍 = 2.2 倍に対し、キャッシュなしは 3 倍)
- 1 時間と 5 分を併用する場合は、1 時間のブロックを先に置きます。逆順は 400 になります

初回の遅延を避けたいときは、起動時に `max_tokens: 0` のウォームアップを送る手もあります。出力トークンは課金されず、`cache_creation_input_tokens` に通常の書き込み分だけが計上されます。ブレークポイントは本番と同じ位置に置き、`thinking` と `effort` も本番に合わせてください。

## まとめ

- マーカーは共有部分の末尾に置く。質問より後ろには置かない
- 効果は `cache_read_input_tokens` で測る。`input_tokens` は全入力ではない
- 失効はエラーにならない。system の動的値、キー順、ツール順、モデル切替を疑う
- TTL はリクエスト間隔で選ぶ。5 分未満なら既定のままでよい

私の場合、まず `cacheStats` を呼び出しラッパーに仕込み、`read` が 0 の呼び出しをログに出す運用から始めるのが一番早いと考えています。冒頭の計算例は仕様からの試算なので、実際の削減率は自分のトラフィックで測ってください。

最小トークン数や倍率はモデルごとに変わります。実装前に [公式の prompt caching ドキュメント](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) で、使うモデルの行を確認してください。