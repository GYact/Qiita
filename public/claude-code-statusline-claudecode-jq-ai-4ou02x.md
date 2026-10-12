---
title: "Claude Code × statusLine | コンテキスト残量・キャッシュの温度・5時間枠を常時表示するまで"
tags:
  - ClaudeCode
  - シェルスクリプト
  - jq
  - 開発効率化
  - AIエージェント
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「いつの間にかコンテキストが埋まって、compact 後に急に精度が落ちた」「5時間枠をどこで使い切ったのか分からない」——この感覚は正しいです。Claude Code は残量を自分から大きく知らせてくれないので、`/usage` や `/context` を打たない限り見えません。

公式の `statusLine` 機能を使うと、画面下部に任意のシェルスクリプトの出力を常時表示できます。この記事では、コピーして動く 1 本のスクリプトで次の 4 つを出すところまで作ります。

- コンテキスト使用率(色付きバー)
- セッションのコスト概算
- 5 時間枠のレート制限消費率
- プロンプトキャッシュの温度(warm / cold と残り秒数)

検証は 2026-10 時点の公式ドキュメントの仕様に沿っています。

## 背景: statusLine は「stdin に JSON が来る」だけの仕組み

`statusLine` は `type: "command"` で指定したコマンドを実行し、標準出力を表示します。セッション情報は stdin に JSON で渡されます。使う項目は次のとおりです。

| 項目 | 内容 |
|---|---|
| `model.display_name` | モデル表示名 |
| `workspace.current_dir` | 作業ディレクトリ |
| `context_window.used_percentage` | コンテキスト使用率(入力トークンのみで計算、序盤は null) |
| `cost.total_cost_usd` | 概算コスト(リスト価格で計算。実請求と異なる場合あり) |
| `rate_limits.five_hour.used_percentage` | 5 時間枠の消費率(Pro/Max のみ、初回応答後に出現) |
| `prompt_cache.warm` / `expires_at` / `hit_ratio` | キャッシュの状態(v2.1.251 以降) |

更新は 300ms のデバウンスで、実行中に次の更新が来ると前のスクリプトはキャンセルされます。つまり**スクリプトは軽く保つ**のが前提です。

## Step 1 まず JSON の中身を見る

推測で書くと null で落ちます。最初に stdin をファイルへ落として実物を確認します。

```json
{
  "statusLine": {
    "type": "command",
    "command": "cat > /tmp/statusline-sample.json"
  }
}
```

これを `~/.claude/settings.json` に入れ、1 往復した後に `jq . /tmp/statusline-sample.json` で構造を確認します。以降はこのファイルをスクリプトのテスト入力に使います。

**防止策: 項目は必ず `// 既定値` で受ける。** `rate_limits` は API 従量課金だと出ず、`context_window.used_percentage` も序盤は null です。

## Step 2 スクリプトを書く

`~/.claude/statusline.sh` に保存します。jq の呼び出しは 1 回にまとめ、プロセス生成のコストを抑えます。

```bash
#!/bin/bash
input=$(cat)

# 欠損は -1 / na に寄せ、後段で出し分ける
IFS=$'\t' read -r MODEL DIR PCT COST H5 WARM EXP HIT <<<"$(jq -r '[
  .model.display_name,
  .workspace.current_dir,
  ((.context_window.used_percentage // 0) | floor),
  ((.cost.total_cost_usd // 0) * 100 | round / 100),
  ((.rate_limits.five_hour.used_percentage // -1) | floor),
  (.prompt_cache.warm // "na"),
  (.prompt_cache.expires_at // 0),
  (if .prompt_cache.hit_ratio == null then -1 else (.prompt_cache.hit_ratio * 100 | floor) end)
] | @tsv' <<<"$input")"

color() { # 閾値で緑 / 黄 / 赤
  if [ "$1" -ge 80 ]; then printf '\033[31m'
  elif [ "$1" -ge 60 ]; then printf '\033[33m'
  else printf '\033[32m'; fi
}
RESET=$'\033[0m'

BAR_W=10
FILLED=$((PCT * BAR_W / 100))
BAR=$(printf '%*s' "$FILLED" '' | tr ' ' '▓')$(printf '%*s' $((BAR_W - FILLED)) '' | tr ' ' '░')

BRANCH=$(git -C "$DIR" branch --show-current 2>/dev/null)
echo "[$MODEL] ${DIR##*/}${BRANCH:+ ($BRANCH)}"

LINE="$(color "$PCT")${BAR} ${PCT}%${RESET} \$${COST}"
[ "$H5" -ge 0 ] && LINE="$LINE  5h:$(color "$H5")${H5}%${RESET}"

case "$WARM" in
  true)
    LEFT=$((EXP - $(date +%s)))
    LINE="$LINE  cache:warm(${LEFT}s)"
    [ "$HIT" -ge 0 ] && LINE="$LINE hit${HIT}%" ;;
  false) LINE="$LINE  \033[31mcache:cold${RESET}" ;;
esac
echo -e "$LINE"
```

`chmod +x ~/.claude/statusline.sh` を忘れずに。

## Step 3 settings.json に登録して単体テスト

キャッシュの残り秒数は時間で変わるので、`refreshInterval` で定期再実行させます(最小 1 秒、ここでは 30 秒)。

```json
{
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "refreshInterval": 30
  }
}
```

Claude Code を起動せずに、Step 1 で保存したサンプルで動作確認できます。

```bash
~/.claude/statusline.sh < /tmp/statusline-sample.json
```

2 行出れば成功です。`rate_limits` を消した JSON でも `5h:` が出ないだけで落ちないことを必ず試してください。

**防止策: 欠損 JSON でのテストを先に通す。** 起動直後や API 課金環境でだけ壊れるスクリプトは、気づくのが遅れます。

## Step 4 数字の読み方と落とし穴

- **コスト**はクライアント側のリスト価格計算です。請求額の代わりにはならず、目安として使います。`/clear` で 0 に戻ります。
- **コンテキスト率**は入力トークンのみで計算され、出力は含まれません。80% を超えたら手動 `/compact` か新セッションを検討します。
- **cache:cold** が頻発するなら、先頭のプロンプトが変わっている可能性があります。`prompt_cache.last_miss_cause` に `tools_changed` や `system_prompt_changed` などの原因が入るので、詳しく追うときは jq で取り出して追記します。
- スクリプトで `git status` のような重い処理は避けます。300ms ごとにキャンセルされ、表示が更新されません。`branch --show-current` 程度に留めます。
- 自作ステータスラインを入れると、フッターの `esc to interrupt` などのヒントが消えます。表示が変わるのは仕様です。

**再発防止: 項目名は公式ドキュメントで確認する。** フィールドはバージョンで増えます(`prompt_cache` は v2.1.251 以降)。古い版では項目が出ないだけなので、既定値で受けておけば壊れません。

## まとめ

- `statusLine` は「stdin の JSON を読んで echo する」だけで作れ、複数行も出せます。
- 最初に実物の JSON を保存し、欠損を既定値で受けるのが壊れない設計の要です。
- コンテキスト率・5 時間枠・キャッシュ温度を常時見ると、compact や新セッションに切り替える判断を勘に頼らず行えます。

次は `last_miss_cause` を集計して、キャッシュが切れた原因別に回数を出すところまで拡張すると、コスト改善の起点になります。