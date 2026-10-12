---
title: "Claude Code × カスタムスラッシュコマンド | `!`コマンド`` でgit差分を埋め込み、PR前レビューを1行で呼ぶまで"
tags:
  - ClaudeCode
  - 自動化
  - Git
  - 生産性向上
  - AIエージェント
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「PRを出す前に毎回、同じ長文プロンプトを貼っている。`git diff` の結果もコピペしている」

——この感覚は正しいです。毎回貼っているなら、それはプロンプトではなくコマンドにすべき処理です。

Claude Code のカスタムスラッシュコマンドは、Markdown ファイル1枚で作れます。さらに `` !`コマンド` `` 構文を使うと、**シェルの実行結果を Claude に渡す前に本文へ埋め込めます**。Claude がツールを呼ぶ往復を挟まないので、差分取得の手間がなくなります。

この記事では、`/pre-pr` コマンドを手を動かして作ります。公式ドキュメント(code.claude.com の slash-commands ページ)で仕様を確認したうえで、実際にハマりやすい点まで書きます。

## 背景: commands と skills の関係

先に前提を整理します。

- `.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` は、どちらも `/deploy` を作る
- 同名なら skill が優先される
- 既存の `commands/` ファイルはそのまま動く
- `commands/` のサブディレクトリは `/frontend:component` のようにコロン区切りの名前空間になる

補助ファイルが要らない単機能のコマンドなら、`commands/` に1ファイル置くのが最短です。本記事はこちらで進めます。

## Step 1 | 最小のコマンドを置く

プロジェクト直下に作ります。チーム共有したいなら Git に含め、個人用なら `~/.claude/commands/` に置きます。

```bash
mkdir -p .claude/commands
```

```markdown
---
description: PR を出す前に、現在のブランチの差分をレビューする
argument-hint: [重点的に見たい観点]
---

現在のブランチの差分をレビューしてください。重点観点: $ARGUMENTS
```

保存したら `/pre-pr 型安全性` と打つだけです。`$ARGUMENTS` が `型安全性` に置き換わります。

引数まわりの仕様は次のとおりです。

| 書き方 | 意味 |
|---|---|
| `$ARGUMENTS` | 渡された引数の全体 |
| `$0`, `$1` | 0始まりの位置引数。`"hello world"` のようにクォートすると1つとして扱う |
| `arguments` で宣言した `$name` | 名前付き引数 |

本文にプレースホルダが無いのに引数を渡すと、末尾に `ARGUMENTS: <値>` が自動で付きます。**引数が黙って捨てられることはありません**。

## Step 2 | `` !`コマンド` `` で差分を先に埋め込む

Step 1 のままでは、Claude が自分で `git diff` を呼ぶ必要があります。差分を本文へ直接埋め込みます。

````markdown
---
description: PR を出す前に、現在のブランチの差分をレビューする
argument-hint: [重点的に見たい観点]
allowed-tools: Bash(git diff *) Bash(git log *) Bash(git status *)
---

## 現在の状態

```!
git status --short
git log --oneline origin/main..HEAD
```

## 差分 (main との比較)

```!
git diff origin/main...HEAD --stat
git diff origin/main...HEAD
```

## 依頼

上の差分をレビューしてください。重点観点: $ARGUMENTS

- バグになりうる箇所を、ファイル名と行番号つきで先に挙げる
- 差分に無いファイルの話はしない
- 問題が無ければ「指摘なし」とだけ書く
````

ポイントは3つです。

- 1行なら `` !`git status --short` ``、複数行なら ` ```! ` で始まるフェンスブロックを使う
- 実行は Claude に内容を渡す**前**。Claude に届くのはコマンドではなく結果
- 実行は通常の権限ルールで判定される。事前承認したいコマンドは `allowed-tools` に書く

`allowed-tools` は、そのターンだけ有効です。次のメッセージを送ると解除されるので、`settings.json` の allowlist を汚さずに済みます。

## Step 3 | ハマりどころを先に潰す

仕様を読んで気づいた、実運用で踏みやすい点を3つ挙げます。

### 落とし穴1: 非ゼロ終了でコマンド全体が中止される

埋め込んだコマンドが失敗すると、**スキルの呼び出し全体が中止されます**。たとえば `origin/main` が存在しない環境では、`git diff` が失敗してコマンド自体が動きません。検索・比較系コマンドの終了コード1は正常扱いですが、それ以外は `|| true` で回避します。

```bash
git diff origin/main...HEAD || git diff HEAD~1
```

**防止策: 取得系のコマンドには必ずフォールバックを付ける。**

### 落とし穴2: `KEY=!`cmd`` は実行されない

`!` の前に文字があると、実行されずリテラルのまま残ります。`!` は行頭か空白の直後に置きます。

```markdown
<!-- 実行されない -->
BRANCH=!`git branch --show-current`

<!-- 実行される -->
現在のブランチ: !`git branch --show-current`
```

**防止策: 埋め込みは必ず行頭か空白の後ろに置き、初回は出力を目で確認する。**

### 落とし穴3: 巨大な差分がコンテキストを食う

差分を丸ごと埋め込むので、大きな PR ではそのままトークンを消費します。本文中の `git diff` に上限を付けます。

```bash
git diff origin/main...HEAD --stat
git diff origin/main...HEAD -- . ':(exclude)pnpm-lock.yaml' ':(exclude)*.snap' | head -n 1500
```

ロックファイルとスナップショットを除外し、行数に上限を設けています。**防止策: `--stat` を先に出し、本体の差分は除外パターンと `head` で切る。**

## Step 4 | 副作用のあるコマンドは自動起動させない

`/commit` のように副作用のあるコマンドは、Claude が文脈から勝手に呼ぶと困ります。frontmatter で止められます。

```markdown
---
description: 変更をステージしてコミットする
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---

!`git status --short`

上記の変更を、既存のコミットメッセージの書式に合わせてコミットしてください。
```

`disable-model-invocation: true` にすると、手動の `/commit` だけで動きます。レビュー専用のコマンドは read-only のまま `allowed-tools` を `git diff` 系に限定し、書き込み系とは分けておくと安全です。

組織のポリシーで埋め込みシェル自体を止めたい場合は、設定に `"disableSkillShellExecution": true` を入れます。ユーザー・プロジェクト・プラグイン由来のコマンドが `[shell command execution disabled by policy]` に置き換わります。共有リポジトリのコマンドを無条件に実行したくないチームは検討の価値があります。

## まとめ

- カスタムコマンドは `.claude/commands/<name>.md` の1ファイルで作れる
- `` !`cmd` `` は Claude に渡す前に実行され、結果が本文へ入る
- 埋め込みコマンドの失敗は呼び出し全体を止めるので、フォールバックを付ける
- `allowed-tools` はそのターン限りの許可で、`settings.json` を汚さない
- 副作用のあるコマンドは `disable-model-invocation: true` で手動専用にする

私の環境では `/pre-pr` を最初に作り、そこから `/commit` と `/release-notes` に広げました。毎回貼っていた定型プロンプトを見つけたら、まず1ファイルに切り出してみてください。

なお、本記事の仕様は 2026年10月時点の公式ドキュメントに基づきます。frontmatter のキーは更新が続いているため、導入前に最新版を確認してください。