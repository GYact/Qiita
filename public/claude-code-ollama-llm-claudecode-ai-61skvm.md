---
title: "Claude Code × Ollama | ローカルLLMへの接続から品質低下点の切り分けまで"
tags:
  - ClaudeCode
  - Ollama
  - ローカルLLM
  - LLM
  - AIエージェント
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「ローカルLLMに繋げば Claude Code はタダで使えるはずなのに、なぜか全然言うことを聞かない」

——この感覚は正しいです。接続自体は環境変数 3 つで終わります。つまずくのはその後で、ツール呼び出しの失敗、コンテキスト溢れ、指示の無視のどれが原因なのか見分けがつかなくなります。

この記事では、Ollama 経由でローカルモデルに繋ぐところから、**どこで品質が落ちているかを数字で切り分ける**ところまでを、手を動かす順に書きます。

なお、ここに載せるモデル名・環境変数・制約は Ollama の公式ドキュメントで確認した内容です。品質や速度の実測値は環境(GPU・メモリ・モデル)で大きく変わるため載せていません。測るスクリプトを用意したので、手元で回してください。

## Step 1 | Ollama 側の準備とコンテキスト長

Ollama は v0.14.0 以降で Anthropic Messages API (`/v1/messages`) に対応しています。ストリーミング、システムプロンプト、ツール呼び出し、thinking ブロックが使えます。

最初の落とし穴はコンテキスト長です。ドキュメントでは、リポジトリを扱うなら**64k 以上**を推奨しています(別ページでは最低 32k)。Ollama の既定値は小さいため、そのまま繋ぐと長いファイルを読んだ時点で先頭が黙って切り捨てられます。Modelfile で明示します。

```bash
# モデルを取得(公式の例に出ている qwen3.5 を使う)
ollama pull qwen3.5

# コンテキスト長を 64k に固定した派生モデルを作る
cat > Modelfile <<'EOF'
FROM qwen3.5
PARAMETER num_ctx 65536
EOF
ollama create qwen3.5-64k -f Modelfile
```

**防止策:** 繋いだ後に「途中から指示を忘れる」症状が出たら、まず `num_ctx` を疑います。モデルの性能ではなく設定の問題であることが多いためです。

## Step 2 | Claude Code を向ける

公式の手順は次の 3 つです。

```bash
export ANTHROPIC_AUTH_TOKEN=ollama        # 必須だが値は無視される
export ANTHROPIC_API_KEY=""               # 空文字にする
export ANTHROPIC_BASE_URL=http://localhost:11434   # /v1 は付けない

claude --model qwen3.5-64k
```

Claude Code が `/v1/messages` を自分で付け足すため、`ANTHROPIC_BASE_URL` に `/v1` を付けると二重になります。

毎回 export したくない場合は、プロジェクトの `.claude/settings.local.json` に `env` として置けます。ただし公式ドキュメントに「Claude の設定はシェルの環境変数より優先される」とあるので、既存の settings に別のベース URL が入っていないか確認してください。

```json
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_API_KEY": "",
    "ANTHROPIC_BASE_URL": "http://localhost:11434"
  }
}
```

起動後に `/status` で接続先が `localhost:11434` になっているかを確認します。**ここを見ずに使い始めると、実は本家 API に繋がっていて比較が無意味になる**ことがあります。逆に、キー未設定で本家に繋がらず落ちるだけの場合もあります。

## Step 3 | Claude Code を介さず疎通だけ切り出す

挙動がおかしいとき、Claude Code とサーバーのどちらの問題かを分けるため、`curl` で直接叩きます。

```bash
curl -s http://localhost:11434/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: ollama" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "qwen3.5-64k",
    "max_tokens": 256,
    "messages": [{"role": "user", "content": "1+1は?数字だけ答えて"}]
  }' | jq '.content[0].text, .usage'
```

これが返らないなら Ollama 側(モデル名の綴り、起動状態、メモリ不足)の問題です。返るのに Claude Code で動かないなら、次のツール呼び出しの段階を疑います。

## Step 4 | 同じタスクを複数モデルで回して品質低下点を測る

「なんとなく悪い」を「この種類のタスクで落ちる」に変えます。タスクを小さく 3 種類に分け、`claude -p`(非対話)で各モデルに投げ、所要時間・ターン数・成否を JSON で集めます。

```bash
#!/usr/bin/env bash
# bench.sh <model> — 同一リポジトリで 3 種のタスクを回して結果を JSONL に追記する
set -euo pipefail
MODEL="$1"
OUT="bench-${MODEL//[:\/]/_}.jsonl"

run() {  # run <label> <prompt> <verify command>
  local label="$1" prompt="$2" verify="$3"
  git stash -u -q 2>/dev/null || true
  local res
  res=$(claude -p "$prompt" --model "$MODEL" --output-format json \
        --permission-mode acceptEdits 2>/dev/null || echo '{"is_error":true}')
  local ok=false
  if bash -c "$verify" >/dev/null 2>&1; then ok=true; fi
  echo "$res" | jq -c --arg l "$label" --arg m "$MODEL" --argjson ok "$ok" \
    '{label:$l, model:$m, ok:$ok, is_error:(.is_error // false),
      turns:.num_turns, ms:.duration_ms}' >> "$OUT"
  git checkout -q -- . && git clean -fdq
}

# 1) 読むだけ: 長いファイルから事実を拾えるか(コンテキスト長の試験)
run read "src/ 配下で最も行数の多いファイルのパスだけを答えて" "true"
# 2) 1 ファイル編集: ツール呼び出し(Edit)が壊れないか
run edit "README の見出し 'Usage' を 'Quick start' に変更して" \
  "grep -q 'Quick start' README.md"
# 3) 複数ファイル+検査: 計画と自己修正ができるか
run multi "関数 foo を bar に改名し、全呼び出しを直して tsc を通して" \
  "pnpm exec tsc --noEmit"
```

```bash
# 本家と比較(本家は ANTHROPIC_BASE_URL を外した別シェルで)
./bench.sh qwen3.5-64k
jq -s 'group_by(.label)[] | {label: .[0].label, ok: map(.ok), ms: map(.ms)}' bench-*.jsonl
```

**注意:** `git stash -u` と `git clean -fd` を使うので、使い捨てのクローンで実行してください。作業ツリーの未コミット変更が消えます。

## Step 5 | 結果の読み方

落ちた位置で原因が決まります。

| 症状 | 疑う箇所 | 対処 |
|---|---|---|
| `read` で間違ったファイルを答える | `num_ctx` 不足で入力が切れている | Modelfile の値を上げる |
| `edit` で `is_error` が立つ/ターンが多い | ツール呼び出しの JSON を出せていない | ツール呼び出しに強いモデルへ変える |
| `edit` は通るが `multi` だけ落ちる | 計画・自己修正の能力不足 | タスクを分割する、重い工程だけ本家へ回す |
| 全部遅い | VRAM 不足で CPU に溢れている | `ollama ps` で配置を確認 |

公式ドキュメントには、ホスト型の WebSearch や高度なツール制御は完全にはサポートされないとあります。本家で動いていた Web 検索前提の手順が、ローカルでは動かなくても故障ではありません。

## まとめ

- 接続は `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_API_KEY=""` の 3 つ。`/v1` は付けない。
- コンテキスト長は 64k 以上を Modelfile で明示する。
- 症状が出たら、curl の疎通 → `/status` → タスク別ベンチの順で切り分ける。
- 全部をローカルにせず、「読む・単純編集」はローカル、「複数ファイルの計画」は本家、と振り分ける設計が現実的です。

Sources:
- [Ollama: Claude Code integration](https://docs.ollama.com/integrations/claude-code)
- [Ollama: Anthropic compatibility](https://docs.ollama.com/api/anthropic-compatibility)