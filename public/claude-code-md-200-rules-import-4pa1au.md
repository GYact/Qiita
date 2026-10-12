---
title: "Claude Code × CLAUDE.md | 200行超の指示書を rules と import に分解し、読み込みを検証するまで"
tags:
  - ClaudeCode
  - CLAUDE.md
  - AIエージェント
  - 開発環境
  - Hooks
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「CLAUDE.md にルールを書き足すたびに、前に書いたルールが守られなくなる気がする」

——この感覚は正しいです。公式ドキュメントは CLAUDE.md を 1 ファイル 200 行未満に収めるよう勧めており、長いほどコンテキストを食い、遵守率も下がると書いています。さらに `@path` で import しても、読み込みは起動時なのでコンテキストコストは減りません。

この記事では、肥大化した CLAUDE.md を「毎回効く層」「パスで絞る層」「ディレクトリ単位の層」に分け、最後に実際に読み込まれたかを機械で確かめるところまでを手順で示します。仕様は公式ドキュメント（How Claude remembers your project）で確認した内容に基づきます。

## 先に知っておく読み込みの仕様

分解の前に、何がいつ読まれるかを押さえます。

| 置き場 | 読み込みタイミング |
|---|---|
| `~/.claude/CLAUDE.md` | 起動時（全プロジェクト共通） |
| `./CLAUDE.md` または `./.claude/CLAUDE.md` | 起動時（チーム共有） |
| `./CLAUDE.local.md` | 起動時（個人用。`.gitignore` へ） |
| 作業ディレクトリより上の階層の CLAUDE.md | 起動時（ルートから順に連結） |
| サブディレクトリの CLAUDE.md | そのディレクトリのファイルを Claude が読み書きした時 |
| `.claude/rules/*.md`（`paths` なし） | 起動時 |
| `.claude/rules/*.md`（`paths` あり） | マッチするファイルを触った時 |

ファイルは上書きではなく連結されます。矛盾する指示が 2 箇所にあると、どちらかが任意に選ばれます。

## Step 1 まず `/context` で今の姿を測る

分解する前に、いま起動時に何が載っているかを見ます。

```bash
claude
# セッション内で
/context
```

**Memory files** の一覧に、読み込まれている CLAUDE.md と CLAUDE.local.md が出ます。ここに無いファイルは Claude から見えていません。行数も手元で数えておきます。

```bash
wc -l CLAUDE.md .claude/CLAUDE.md CLAUDE.local.md 2>/dev/null
```

**再発防止:** 分解の前後で `wc -l` と `/context` の結果を記録し、増えたら理由を言えるようにしておく。

## Step 2 「特定のファイルを触る時だけ要る」ルールを rules に出す

型付けのルールや API 層の規約は、フロントエンドを触るときには要りません。`.claude/rules/` に 1 トピック 1 ファイルで切り出し、`paths` で絞ります。

```markdown
---
paths:
  - "src/api/**/*.{ts,tsx}"
  - "tests/**/*.test.ts"
---

# API 層のルール

- 入力検証は Zod で行い、未検証の値を下流へ渡さない
- エラーは共通のエラー形式で返す
```

`paths` は YAML のリストでもカンマ区切りの文字列でも書けます。frontmatter で Claude Code が読むのは `paths` だけで、他のフィールドは黙って無視されます。YAML が壊れていると、エラーは出ずに「`paths` なしの常時ロード」として扱われます。絞ったつもりが全件ロードになる原因になるので、`claude --debug` でパースエラーを確認してください。

**防止策:** `paths` のグロブは、実在するファイルに `ls` で当ててから書く。`[` を含むパスはエスケープが要る。

## Step 3 他ツールと共有する指示は import にまとめる

Codex など他のエージェントも使うなら、共通の指示を `AGENTS.md` に置き、CLAUDE.md から取り込みます。

```markdown
@AGENTS.md

## Claude Code 固有
- 変更後は `pnpm check` を通す
```

import の仕様は次のとおりです。

- 相対パスは「import を書いたファイル」基準で解決される
- 再帰 import は最大 4 階層まで
- コードスパンとコードブロックの中は import として解釈されない。パスを文章として書きたいだけなら `` `@README` `` とバッククォートで囲む
- 作業ディレクトリの外を指す import は、初回に承認ダイアログが出る。断ると以後そのプロジェクトでは無効のまま

worktree を並行で使う場合、gitignore された `CLAUDE.local.md` は作成した worktree にしか存在しません。個人用の指示は `@~/.claude/my-project-instructions.md` のようにホームの実体を import する形にすると全 worktree に効きます。

なお AGENTS.md は、CLAUDE.md が無いときに限り Claude Code が直接読みます（v2.1.277 以降）。`CLAUDE.local.md` を 1 つ足しただけで読まれなくなるため、併存させるなら上の `@AGENTS.md` を書くのが確実です。

## Step 4 モノレポでは他チームの指示を除外する

祖先ディレクトリの CLAUDE.md が自分の作業と無関係なら、`claudeMdExcludes` で外せます。

```json
{
  "claudeMdExcludes": [
    "**/other-team/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

パターンは絶対パスに対するグロブで、user / project / local / managed のどの層にも置けます。ただし managed policy の CLAUDE.md は除外できません。

## Step 5 InstructionsLoaded フックで読み込みを検証する

ここまでの分解が効いているかは、ファイルを眺めても分かりません。`InstructionsLoaded` フックは、どの CLAUDE.md・rules がいつ・なぜ読まれたかを取れます。まずは入力 JSON をそのままログへ落とします。

```json
{
  "hooks": {
    "InstructionsLoaded": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "jq -c . >> \"$HOME/.claude/instructions-loaded.jsonl\""
          }
        ]
      }
    ]
  }
}
```

この設定は `.claude/settings.local.json` か `~/.claude/settings.json` に入れます。ペイロードのフィールド名は版によって変わりうるので、最初に数行をそのまま目で確認してから、集計用の `jq` を書いてください。

```bash
# 何が読まれているかをざっと確認
tail -n 5 ~/.claude/instructions-loaded.jsonl | jq .
```

`paths` 付きルールは、対象ファイルを Read / Write / Edit したときだけログに出るはずです。起動直後のログに出ていたら、Step 2 の frontmatter が効いていません。

## 守らせたいなら hook に倒す

公式も、CLAUDE.md は「強制される設定ではなくコンテキスト」だと明記しています。コミット前に必ず実行したい、という類のルールは CLAUDE.md の文章ではなく hook に移します。ブロックしたい操作は PreToolUse フック、技術的な強制は `permissions.deny` の領分です。

## まとめ

- 起動時に載るもの（無条件の CLAUDE.md と rules）を `/context` と `wc -l` で測り、200 行未満を目標にする
- 局所的なルールは `paths` 付きの rules か、サブディレクトリの CLAUDE.md に出す
- import は整理には効くが、コンテキスト削減にはならない
- 効いているかは `InstructionsLoaded` のログで確かめる。守らせたいことは hook へ

指示書は書くより「減らして検証する」ほうが効きます。まず `wc -l` から測ってみてください。