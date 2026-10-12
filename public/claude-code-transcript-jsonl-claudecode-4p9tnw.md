---
title: "Claude Code × セッションログ | transcript JSONLからトークン消費を集計し日本円換算するまで"
tags:
  - ClaudeCode
  - Node.js
  - 料金
  - JSONL
  - コスト管理
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「今月どれくらい使ったのか、正直よく分からない」
「ダッシュボードの数字と体感が合わない」

——この感覚は正しいです。Claude Code は使った分のトークンを手元の `~/.claude/projects/` にすべて JSONL で書き残していますが、そのまま足し算すると**水増しした数字**になります。この記事では、追加のインストールも OpenTelemetry の設定も無しに、ログを読むだけで日別の消費を日本円で出す 60 行ほどのスクリプトを作ります。

私の Hub リポジトリ(1 プロジェクトで 410 セッション分のファイルがある)で実測しながら書いているので、ハマりどころも先に書いておきます。

## なぜログを直接読むのか

コストを見る手段はいくつかあります。

| 手段 | 準備 | 粒度 | 向いている場面 |
|---|---|---|---|
| `/cost` など組み込みの表示 | 不要 | セッション単位 | その場の確認 |
| OpenTelemetry | Collector と設定が必要 | リアルタイム・属性付き | チーム運用・アラート |
| transcript JSONL を直接集計 | Node があれば足りる | 呼び出し単位・過去分も遡れる | 個人の振り返り・単価の差し替え |

OTel は入れた時点からしか取れません。ログ集計は「入れる前の 3 ヶ月分」も同じ手順で出せます。単価を後から変えて再計算できるのも利点です。

## Step 1 ログの構造を確かめる

保存先はプロジェクトのパスをハイフンに置換したディレクトリです。まず 1 行の形を `jq` で見ます。

```bash
cd ~/.claude/projects/-Users-yourname-Developer-your-repo
f=$(ls -t *.jsonl | head -1)

# assistant 行の usage とモデル名だけを取り出す
jq -c 'select(.type=="assistant")
  | {id:.message.id, model:.message.model, usage:.message.usage}' "$f" | head -2
```

`usage` には次のフィールドが入っていました。

```json
{
  "input_tokens": 2,
  "output_tokens": 401,
  "cache_read_input_tokens": 29050,
  "cache_creation_input_tokens": 71432,
  "cache_creation": {
    "ephemeral_5m_input_tokens": 0,
    "ephemeral_1h_input_tokens": 71432
  }
}
```

注目は `cache_creation` が 5 分キャッシュと 1 時間キャッシュに分かれている点です。書き込み単価が違うため、合計の `cache_creation_input_tokens` だけで計算すると誤差になります。

## Step 2 足し算する前に潰す 2 つの罠

### 罠1 同じレスポンスが複数行に書かれる

あるセッションで assistant 行を数えると 34 行、`message.id` の一意数は 20 でした。1 回のレスポンスが `thinking` / `text` / `tool_use` といった content ブロックごとに別行で書かれ、各行に同じ `usage` が付くためです。

```bash
# 行数と一意な message.id の数を比べる
jq -c 'select(.type=="assistant")' "$f" | wc -l
jq -r 'select(.type=="assistant") | .message.id' "$f" | sort -u | wc -l
```

素朴に全行を足すと、この例では約 1.7 倍に膨らみます。**対策は `message.id` で一意化すること**です。

### 罠2 `<synthetic>` モデルの行

API エラーなどでクライアントが合成した行が `model: "<synthetic>"` で混ざり、トークンは全て 0 です。単価表に無いモデルとして「未設定」警告を出す原因になるので、先に除外します。

サブエージェントの会話は `<セッションID>/subagents/*.jsonl` に別ファイルで入ります。ディレクトリを再帰的に歩けば拾えますし、`message.id` で一意化しているので二重計上にもなりません。

## Step 3 集計スクリプト

単価は USD / 100 万トークンで `prices.json` に外出しします。**単価はここに書かず、公式の料金表を見て自分で埋めてください**(モデルと時期で変わるため、記事に固定値を載せません)。

```json
{
  "claude-opus-5-5": {
    "input": 0,
    "output": 0,
    "cacheRead": 0,
    "cacheWrite5m": 0,
    "cacheWrite1h": 0
  }
}
```

本体は `usage-report.mjs` です。

```javascript
import { readdirSync, readFileSync, statSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";

const PRICE = JSON.parse(readFileSync(process.argv[3] ?? "prices.json", "utf8"));
const JPY_PER_USD = Number(process.env.JPY_PER_USD ?? 150);
const root = join(homedir(), ".claude", "projects", process.argv[2] ?? "");

function* walk(dir) {
  for (const name of readdirSync(dir)) {
    const p = join(dir, name);
    if (statSync(p).isDirectory()) yield* walk(p);
    else if (name.endsWith(".jsonl")) yield p;
  }
}

const seen = new Set();
const byDay = new Map();
const unpriced = new Map();

for (const file of walk(root)) {
  for (const line of readFileSync(file, "utf8").split("\n")) {
    if (!line) continue;
    let row;
    try { row = JSON.parse(line); } catch { continue; }
    const m = row.message;
    if (row.type !== "assistant" || !m?.usage || m.model === "<synthetic>") continue;
    // 1 レスポンスが content ブロックごとに複数行で書かれるので message.id で一意化
    if (seen.has(m.id)) continue;
    seen.add(m.id);

    const price = PRICE[m.model];
    if (!price) { unpriced.set(m.model, (unpriced.get(m.model) ?? 0) + 1); continue; }
    const u = m.usage;
    const c = u.cache_creation ?? {};
    const w5 = c.ephemeral_5m_input_tokens ?? u.cache_creation_input_tokens ?? 0;
    const w1h = c.ephemeral_1h_input_tokens ?? 0;
    const usd =
      ((u.input_tokens ?? 0) * price.input +
        (u.output_tokens ?? 0) * price.output +
        (u.cache_read_input_tokens ?? 0) * price.cacheRead +
        w5 * price.cacheWrite5m +
        w1h * price.cacheWrite1h) / 1e6;

    const day = new Date(row.timestamp).toLocaleDateString("sv-SE", { timeZone: "Asia/Tokyo" });
    const d = byDay.get(day) ?? { usd: 0, calls: 0, read: 0, input: 0 };
    d.usd += usd; d.calls += 1;
    d.read += u.cache_read_input_tokens ?? 0;
    d.input += (u.input_tokens ?? 0) + (u.cache_read_input_tokens ?? 0) + w5 + w1h;
    byDay.set(day, d);
  }
}

console.log("day         calls   USD     JPY   cache_hit");
for (const [day, d] of [...byDay].sort()) {
  const jpy = Math.round(d.usd * JPY_PER_USD);
  const hit = ((d.read / d.input) * 100).toFixed(0);
  console.log(`${day}  ${String(d.calls).padStart(5)}  ${d.usd.toFixed(2).padStart(6)}  ${String(jpy).padStart(6)}  ${hit}%`);
}
for (const [model, n] of unpriced) console.error(`単価未設定: ${model} (${n} 回)`);
```

実行はプロジェクトのディレクトリ名と単価ファイルを渡すだけです。

```bash
JPY_PER_USD=150 node usage-report.mjs -Users-yourname-Developer-your-repo prices.json
```

日付は `Asia/Tokyo` で切っています。UTC のままだと、深夜 0 時〜朝 9 時の作業が前日に入って日別の数字がずれます。

**防止策:** 単価未設定のモデルは 0 円として黙って流さず、必ず stderr に出します。新しいモデルを使い始めた日に「安く見える」事故を防げます。

## Step 4 読み方

出力の右端 `cache_hit` は、入力側トークンのうちキャッシュ読み出しが占める割合です。日別に見ると、長いセッションの日は高く、短い質問を何度も投げる日は低くなります。

低い日は、5 分キャッシュが切れる間隔でセッションを再開していないかを疑ってください。キャッシュ書き込みは読み出しより高いので、同じ文脈を何度も書き直す日が一番高くつきます。

## まとめ

- transcript は `usage` を持つ assistant 行を `message.id` で一意化して足す。34 行が 20 件になる例があった
- `<synthetic>` 行を除外し、キャッシュは 5 分 / 1 時間の書き込みを分けて計算する
- 単価は外出しにして、再計算できる形にしておく

まずは自分のプロジェクトで `wc -l` と `sort -u | wc -l` を比べてみてください。差が出た分だけ、これまでの見積もりは過大でした。