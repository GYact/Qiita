---
title: "Claude Code CLI × GitHub Actions | headlessモードでPRレビューBotの実装からコスト制御まで"
tags:
  - ClaudeCode
  - CLI
  - GitHubActions
  - CICD
  - 自動化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## 導入

「Claude Code をCIに組み込みたいけど、対話型の許可プロンプトでランナーが固まりそう」「何も制御しないとAPIコストが青天井になりそう」——この感覚は正しいです。

Claude Code CLI には `-p`(`--print`)を付けるだけで対話ループを1回のバッチ実行に変える headless モードがあります。ただし素の `-p` は、既定の権限モードが「Manual」のままなので、最初にBashやファイル書き込みが必要な場面で許可待ちに入り、CIには答える人間がいないためそのまま止まります。さらにリポジトリ内の `.claude/settings.json` や `.mcp.json` を信頼確認なしに読み込んでしまうため、fork元のPRを無防備に流すと素性の知れないhookやMCPサーバーが動く余地も残ります。

この記事では、Claude Code CLI を使って GitHub Actions 上に PR レビューBot を実装しながら、①権限の固定 ②実行環境の再現性確保 ③構造化出力の取得 ④コストの可視化、までを手を動かして組み立てます。

## Step 1: 最小構成で動かす

まずは許可ツールを明示してローカルで動かします。

```bash
# インストール(npm経由は非推奨化済み。ネイティブインストーラを使う)
curl -fsSL https://claude.ai/install.sh | bash
export PATH="$HOME/.local/bin:$PATH"

claude -p "auth.py のバグを見つけて直して" --allowedTools "Read,Edit"
```

`--allowedTools` に列挙したツールだけが確認なしで実行されます。ここを空にしたまま流すと、初回のBash実行やファイル編集で許可待ちに入り、CIでは永久にハングします。

**防止策:** CIで使わせたいツールは必ず `--allowedTools` で名指しする。ワイルドカードで絞る場合は `Bash(git diff *)` のように末尾にスペース+`*`を入れる(`Bash(git diff*)` だと `git diff-index` まで拾ってしまう)。

## Step 2: `--bare` でリポジトリの中身を信用しない

CIランナーは使い捨てなので一見リスクが低く見えますが、`-p` は既定でそのリポジトリの `.claude/settings.json` の hooks や `.mcp.json` の MCP サーバーを、ワークスペース信頼ダイアログなしにそのまま読み込みます。fork からの PR を扱うワークフローではこれが攻撃面になります。

```bash
claude --bare -p "この差分をレビューして" --allowedTools "Read"
```

`--bare` は hooks・skills・カスタムコマンド・サブエージェント・プラグイン・MCPサーバー・自動記憶・CLAUDE.md の自動読み込みをすべてスキップし、Bash・ファイル読み書きだけが使える最小構成で起動します。bareモードは購読ログインを使わないため、`ANTHROPIC_API_KEY` を環境変数で渡す必要があります。

**防止策:** fork PR を扱うジョブでは `--bare` を必須にする。どうしても任意の hook や MCP を使いたい場合だけ `--mcp-config` / `--settings` で明示的に渡す。

## Step 3: 構造化出力を取り出す

CI側でパースしやすい形が欲しいので `--output-format json` と `--json-schema` を組み合わせます。

```bash
SCHEMA='{"type":"object","properties":{"summary":{"type":"string"},"issues":{"type":"array","items":{"type":"string"}}},"required":["summary","issues"]}'

gh pr diff 123 | claude --bare -p \
  "この差分をレビューし、バグ・型崩れだけをissuesに列挙して" \
  --allowedTools "" \
  --output-format json \
  --json-schema "$SCHEMA" | jq '.structured_output'
```

指摘内容は `.structured_output` に型付きで入り、コストなどのメタ情報は同じJSONの `total_cost_usd` に入ります。渡したスキーマが不正なJSON Schemaだと `Error: --json-schema is not a valid JSON Schema` で即座に落ちます(Claude Code v2.1.205以降の挙動)。

## Step 4: 権限モードを固定して暴走を止める

`-p` は権限モードを何も指定しないと Manual のまま起動するため、CI向けには明示的にモードを選びます。

| モード | 挙動 | CIでの扱いやすさ |
|---|---|---|
| (未指定 / Manual) | 承認が要る操作すべてでプロンプト待ち | 人間がいないと停止する |
| `acceptEdits` | ファイル編集と `mkdir`/`touch`/`mv`/`cp` などを自動承認 | shellやネットワークは別途 `--allowedTools` が要る |
| `auto` | 分類器が大半の操作を審査 | 許可されない操作は自動で拒否される |
| `dontAsk` | 許可が必要な呼び出しはすべて拒否し、`--allowedTools`/`permissions.allow` に合致する分だけ実行 | 最も予測可能。ロックダウンしたCI向け |

さらに `--permission-prompts none` を足すと、許可待ちが発生する呼び出し自体を「誰も承認できない」とClaudeに伝えたうえで打ち切ります(v2.1.259以降)。

```bash
claude --bare -p "依存パッケージの脆弱性を直してテストを通して" \
  --allowedTools "Read,Edit,Bash(pnpm install),Bash(pnpm test)" \
  --permission-mode dontAsk \
  --permission-prompts none \
  --output-format json
```

**防止策:** CIでは `dontAsk` + `--permission-prompts none` を基本形にし、必要な操作だけを `--allowedTools` で穴あけする。`acceptEdits` は書き込みが緩すぎるので、意図しないファイル改変を防ぎたい場面には向かない。

## Step 5: GitHub Actions に組み込む

ここまでの要素を1本のワークフローにまとめます。

```yaml
name: claude-pr-review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    if: github.event.pull_request.head.repo.fork == false
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - name: Install Claude Code
        run: curl -fsSL https://claude.ai/install.sh | bash

      - name: Run review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GH_TOKEN: ${{ github.token }}
        run: |
          export PATH="$HOME/.local/bin:$PATH"
          SCHEMA='{"type":"object","properties":{"summary":{"type":"string"},"issues":{"type":"array","items":{"type":"string"}}},"required":["summary","issues"]}'
          gh pr diff "${{ github.event.pull_request.number }}" | \
            claude --bare -p "この差分をレビューし、バグ・命名・型崩れだけをissuesに列挙して" \
              --allowedTools "" \
              --permission-mode dontAsk \
              --permission-prompts none \
              --output-format json \
              --json-schema "$SCHEMA" > result.json

          COST=$(jq -r '.total_cost_usd' result.json)
          echo "review cost: \$${COST}"
          awk -v c="$COST" 'BEGIN { if (c > 0.5) { print "cost threshold exceeded"; exit 1 } }'

      - name: Post comment
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          SUMMARY=$(jq -r '.structured_output.summary' result.json)
          ISSUES=$(jq -r '.structured_output.issues[] // empty' result.json)
          gh pr comment "${{ github.event.pull_request.number }}" \
            --body "$(printf '%s\n%s' "$SUMMARY" "$ISSUES")"
```

fork PR は `if` 条件で最初から除外し、Secrets へ触れさせません。コストは `total_cost_usd` が [クライアント側の見積もり](https://code.claude.com/docs/en/agent-sdk/cost-tracking) である点に注意しつつ、閾値を超えたらジョブ自体を失敗させています。`--continue` / `--resume` でセッションを跨いで会話を続けると、この値は「そのセッション全体の累計」になるので、PRごとに新規セッションで走らせるならレビュー1回分のコストとしてそのまま使えます。

**防止策:** 終了コードでCIを分岐できるように、コストチェックは `exit 1` で明示的に落とす。放置すると「レビューは動いているのにコストだけ際限なく積み上がる」状態に気づけません。

## まとめ

`-p` は付けるだけで動きますが、CIで安定させる鍵は「許可を待たせない(`--permission-mode dontAsk` + `--permission-prompts none`)」「リポジトリを信用しない(`--bare`)」「後工程がパースできる形で受け取る(`--output-format json` + `--json-schema`)」「コストを終了コードに変換する」の4点でした。まずは自分のリポジトリの小さなワークフロー1本に絞って、閾値やallowedToolsの範囲を実測しながら広げていくのがおすすめです。

## 参考

- [Run Claude Code programmatically(公式ドキュメント)](https://code.claude.com/docs/en/headless)
- [CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking)