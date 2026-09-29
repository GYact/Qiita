---
title: "Claude Code × Hooks | PreToolUseで危険コマンドとシークレット読み取りを止めるまで"
tags:
  - ClaudeCode
  - Hooks
  - セキュリティ
  - シェルスクリプト
  - 自動化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「CLAUDE.md に『rm -rf は禁止』と書いたのに、本当に守られている保証はあるのか」

——この感覚は正しいです。CLAUDE.md の禁止文はモデルへの「お願い」であり、長いセッションや複雑なタスクの途中で読み落とされる可能性があります。守らせたいルールは、モデルの外側、つまり実行の直前に決定的なコードで検査するのが確実です。それを担うのが Claude Code の **Hooks** です。

この記事では `PreToolUse` フックを使い、次の3つを実装します。

1. 破壊的な Bash コマンドを拒否する
2. `.env` などシークレットファイルの読み取りを拒否する
3. 迷うものは拒否せず確認ダイアログに回す(`ask`)

私は複数のプロジェクトで、Claude Code に自律的にコードを書かせています。そこでは「頼む」ではなく「通れない」形でガードを置く運用にしています。以下は、その考え方を最小構成で動かせるようにしたものです。

## Step 0 フックの入出力を押さえる

`PreToolUse` フックは、ツール実行の直前に指定コマンドを呼びます。押さえておく点は次の3つです。

- ツール呼び出しの内容が **stdin に JSON** で渡される(`tool_name`、`tool_input` など)
- **終了コード 2** でツール実行をブロックでき、stderr の内容が Claude にフィードバックされる
- 終了コード 0 で JSON を stdout に返すと、`permissionDecision` に `allow` / `deny` / `ask` を指定できる

Bash ツールに渡される JSON はおおよそ次の形です。

```json
{
  "session_id": "abc123",
  "cwd": "/Users/me/project",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf ./dist" }
}
```

JSON のパースには `jq` を使います。フック内で python を使うとシステム Python 依存になりやすいので、`jq` が手軽です。

## Step 1 settings.json にフックを登録する

プロジェクト共通で使うなら `.claude/settings.json`、個人用なら `~/.claude/settings.json` に書きます。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/guard-bash.sh" }
        ]
      },
      {
        "matcher": "Read|Edit|Write",
        "hooks": [
          { "type": "command", "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/guard-files.sh" }
        ]
      }
    ]
  }
}
```

`matcher` はツール名に対する正規表現です。スクリプトには実行権限を付けておきます。

```bash
chmod +x .claude/hooks/guard-bash.sh .claude/hooks/guard-files.sh
```

**防止策:** フックのパスは `$CLAUDE_PROJECT_DIR` 起点で書きます。相対パスにすると、Claude が `cd` した先でスクリプトが見つからず、フックが黙って失敗する原因になります。

## Step 2 破壊的な Bash コマンドを拒否する

`guard-bash.sh` では、コマンド文字列を取り出して危険なパターンに照合します。

```bash
#!/usr/bin/env bash
# 危険コマンドを実行前に止める。ブロック時は exit 2 + stderr で理由を Claude に返す
set -euo pipefail

cmd=$(jq -r '.tool_input.command // empty')

deny() {
  echo "ブロック: $1" >&2
  exit 2
}

# 再帰削除(ルート・ホーム・カレント直下の全消し)
if grep -Eq '(^|[;&|[:space:]])rm[[:space:]]+(-[a-zA-Z]*r[a-zA-Z]*f|-[a-zA-Z]*f[a-zA-Z]*r)[[:space:]]+(/|~|\$HOME|\.|\*)([[:space:]]|$)' <<<"$cmd"; then
  deny "rm -rf の対象が広すぎます。削除対象を具体的なパスで指定してください"
fi

# force push
if grep -Eq 'git[[:space:]]+push.*(--force|-f)([[:space:]]|$)' <<<"$cmd"; then
  deny "force push は禁止です"
fi

# パイプで直接シェルに流し込む
if grep -Eq '(curl|wget)[^|]*\|[[:space:]]*(ba|z)?sh' <<<"$cmd"; then
  deny "ダウンロードしたスクリプトの直接実行は禁止です"
fi

exit 0
```

exit 2 のとき stderr が Claude に渡るので、モデルは「なぜ止まったか」を読んで別の手段を選びます。理由を具体的に書くほど、無駄な再試行が減ります。

**防止策:** 正規表現の denylist は完全ではありません。`rm` を `find -delete` や別の言い回しで書かれる可能性は残ります。denylist はあくまで事故防止の網と考え、本当に守りたい領域はサンドボックスやファイル権限と併用します。

## Step 3 シークレットファイルの読み取りを拒否する

`.env` の中身を会話に載せると、その内容は以後のコンテキストに残ります。`guard-files.sh` で Read / Edit / Write の対象パスを検査します。

```bash
#!/usr/bin/env bash
# シークレットになりうるファイルへのアクセスを止める
set -euo pipefail

path=$(jq -r '.tool_input.file_path // empty')
[ -z "$path" ] && exit 0

case "$path" in
  *.env|*.env.*|*.pem|*/id_rsa|*/id_ed25519|*/.aws/credentials|*/.netrc)
    # .env.example は雛形なので通す
    case "$path" in
      *.env.example) exit 0 ;;
    esac
    echo "ブロック: シークレットの可能性があるファイルです ($path)。値は会話に出さず、変数名だけ確認してください" >&2
    exit 2
    ;;
esac

exit 0
```

**防止策:** Read だけを塞いでも、Bash から `cat .env` されると素通りします。Step 2 のスクリプトに `cat|less|head|tail` と `.env` の組み合わせも追記しておきます。

```bash
if grep -Eq '(cat|less|more|head|tail|grep|sed|awk)[^;&|]*\.env' <<<"$cmd" \
   && ! grep -Eq '\.env\.example' <<<"$cmd"; then
  deny ".env を直接読む操作は禁止です"
fi
```

## Step 4 迷うものは deny でなく ask に回す

すべてを拒否にすると作業が止まります。「多くは正当だが、たまに危ない」コマンドは、確認ダイアログに回すのが現実的です。この場合は exit 0 で JSON を返します。

```bash
# 本番向けの操作は人間に確認させる
if grep -Eq 'supabase[[:space:]]+(db[[:space:]]+push|functions[[:space:]]+deploy)|vercel[[:space:]].*--prod' <<<"$cmd"; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "ask",
      permissionDecisionReason: "本番環境に影響するコマンドです。内容を確認してください"
    }
  }'
  exit 0
fi
```

`permissionDecision` は `allow`(確認なしで通す)、`deny`(拒否)、`ask`(ユーザー確認)から選べます。私の環境では「外部に通知が届く操作」や「本番への反映」を `ask` に寄せ、破壊的なものだけを `deny` にしています。

## Step 5 フックを単体でテストする

Claude を介さなくても、JSON を流し込めばフックは検証できます。

```bash
# 拒否されるはず (exit code 2)
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf ~"}}' | .claude/hooks/guard-bash.sh
echo "exit=$?"

# 通るはず (exit code 0)
echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf ./dist"}}' | .claude/hooks/guard-bash.sh
echo "exit=$?"

# .env は拒否、.env.example は許可
echo '{"tool_name":"Read","tool_input":{"file_path":"/app/.env"}}' | .claude/hooks/guard-files.sh; echo "exit=$?"
echo '{"tool_name":"Read","tool_input":{"file_path":"/app/.env.example"}}' | .claude/hooks/guard-files.sh; echo "exit=$?"
```

**防止策:** ガードを入れたら、必ず「止まるはずの入力」と「通るはずの入力」の両方をテストします。片方だけだと、常に exit 0 を返す壊れたフックでも緑に見えます。設定後は Claude Code 上で `/hooks` を開き、登録が読み込まれているかも確認してください。

## まとめ

| 目的 | 手段 | 判定 |
|---|---|---|
| 破壊的コマンドの阻止 | `guard-bash.sh` で正規表現照合 | exit 2(deny) |
| シークレットの読み取り阻止 | `guard-files.sh` でパス照合 | exit 2(deny) |
| 本番反映など迷うもの | JSON で `permissionDecision: ask` | 人間が確認 |

Hooks は「モデルの善意に頼らない」ための層です。まずは `rm -rf` と `.env` の2つだけでも入れておくと、事故の入口をかなり塞げます。ルールは事故や危ない場面に遭遇するたびに1件ずつ足していく形が、過剰な拒否を避けられて長続きします。

なお、この記事のパターンや JSON の形式は執筆時点の Claude Code の仕様に基づいています。フックの入出力仕様は更新されることがあるため、導入前に公式ドキュメントの Hooks の項を確認してください。