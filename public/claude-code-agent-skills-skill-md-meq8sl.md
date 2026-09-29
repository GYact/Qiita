---
title: Claude Code × Agent Skills | SKILL.mdの設計から自動発火・権限制御まで
tags:
  - ClaudeCode
  - Anthropic
  - AIエージェント
  - 自動化
  - CLI
private: false
updated_at: '2026-09-29T17:39:15+09:00'
id: 96ba2c23fd6e27871362
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

「また同じ手順書をチャットに貼り付けている」――この感覚は正しいです。

デプロイ手順、コミット規約、社内API設計ルール。同じコンテキストを毎回プロンプトに貼っていると、そのうち「毎回貼る」こと自体が仕事になります。Claude Code の Agent Skills は、この「毎回貼るコンテキスト」を `SKILL.md` という1ファイルに固定し、必要な時だけ Claude 自身に読み込ませる仕組みです。カスタムスラッシュコマンドと違い、**ユーザーが呼ばなくても Claude が description を見て自発的に発火できる**のが最大の違いで、ここが設計を誤ると「発火しない」「余計な時に発火する」の両方の事故につながります。

この記事では、最小構成から権限制御・自動発火制御・サブエージェント実行まで、実際に手を動かして検証できる手順で追っていきます。手元のプロジェクトでも project スキルを何本も運用していて、書き方を間違えると「発火してほしい時に発火しない」問題に必ず一度は当たるので、そこも踏まえて書きます。

なお、フィールド名や挙動は公式ドキュメント（code.claude.com の Skills ページ）で確認できたものだけを書いています。

## Step 1: 最小構成のSKILL.mdを作る

Agent Skills はディレクトリ単位です。プロジェクト直下に置くと、そのリポジトリの全セッションで有効になります。

| 種類 | 置き場所 | 有効範囲 |
|---|---|---|
| Personal | `~/.claude/skills/<skill-name>/SKILL.md` | この端末の全プロジェクト |
| Project | `.claude/skills/<skill-name>/SKILL.md` | そのリポジトリ（コミットしてチームで共有） |

同名スキルが複数ある場合は enterprise > personal > project の順で優先されます。個人の `deploy` がプロジェクトの `deploy` を黙って上書きするので、名前の衝突には注意してください。

まずは公式の例に沿って、差分を要約するスキルを作ります。

```bash
mkdir -p .claude/skills/summarize-changes
```

```yaml
---
description: Summarizes uncommitted changes and flags anything risky. Use when the user asks what changed, wants a commit message, or asks to review their diff.
---

## Current changes

!`git diff HEAD`

## Instructions

Summarize the changes above in two or three bullet points, then list any risks you notice such as missing error handling, hardcoded values, or tests that need updating. If the diff is empty, say there are no uncommitted changes.
```

これを `.claude/skills/summarize-changes/SKILL.md` に保存します。ポイントは3つです。

- `name` を省略するとディレクトリ名がコマンド名になる（ここでは `/summarize-changes`）
- frontmatter は**ファイルの1行目が `---` のときだけ**解釈される。先頭に空行やコメントがあると全体が本文扱いになる
- `` !`git diff HEAD` `` は動的コンテキスト注入。Claude に渡る前にコマンドが実行され、出力で置き換わる

起動して確認します。

```text
> What did I change?
```

description に合致する質問をすると Claude が自分でスキルを読み込みます。`/summarize-changes` と直接打っても動きます。スキルの追加・編集・削除は実行中のセッションに反映されるので、再起動しながら試す必要はありません。

## Step 2: 自動発火する description の書き方

自動発火の判定材料は `description` と、任意の `when_to_use` です。公式ドキュメントによると、この2つを合わせた文字列はスキル一覧で **1,536 文字で切り詰められます**。だから「何をするか」より先に「どんな依頼で使うか」を前に置きます。

```yaml
---
name: api-conventions
description: 社内REST APIの命名・エラー形式・ページネーション規約。APIエンドポイントの追加・レビュー・OpenAPI定義の編集を頼まれたときに使う。
when_to_use: 「エンドポイントを追加して」「レスポンス形式を揃えて」「APIレビューして」
paths: src/api/**, openapi/**
---

## 規約

- パスは複数形の名詞（`/users/{id}`）。動詞は使わない
- エラーは `{ "error": { "code": string, "message": string } }` で返す
- 一覧は cursor 方式。`limit` の上限は 100
```

ここで使っている `paths` は、glob に一致するファイルを扱っているときだけ自動で読み込ませる絞り込みです。「余計な時に発火する」問題の対策になります。

発火しないときの確認手順は公式のトラブルシュートに沿うのが早いです。

1. description にユーザーが普通に言う言葉が入っているか
2. `What skills are available?` と聞いて一覧に出るか
3. 依頼文を description に寄せて言い直す
4. `/skill-name` で直接呼べるか

一覧に出ない場合は frontmatter の YAML が壊れている可能性があります。壊れていると本文は読まれても metadata が空になり、`/skill-name` は動くのに Claude は description で判定できません。`--debug` でパースエラーが見えます。

```bash
claude plugin validate .claude/skills
```

`SKILL.md` の frontmatter が壊れていないか、このコマンドでまとめて検査できます（Claude Code v2.1.233 以降）。

## Step 3: 自動発火を止める・呼び出し元を絞る

デプロイやコミットのように副作用のある手順は、Claude に「コードが仕上がったようなのでデプロイします」と判断させたくありません。`disable-model-invocation: true` で手動専用にします。

```yaml
---
name: deploy
description: Deploy the application to production
disable-model-invocation: true
---

Deploy $ARGUMENTS to production:

1. Run the test suite
2. Build the application
3. Push to the deployment target
```

`/deploy staging` と打つと `$ARGUMENTS` が `staging` に置換されます。位置指定なら `$0`、`$1`（`$ARGUMENTS[N]` の短縮形）も使えます。引数を受ける placeholder が本文に無い場合は、末尾に `ARGUMENTS: <入力>` が自動で付きます。

逆に「背景知識だけ持たせたい」場合は `user-invocable: false` です。

| 設定 | ユーザーが `/` で呼ぶ | Claude が自動で呼ぶ | description のコンテキスト常駐 |
|---|---|---|---|
| 既定 | 可 | 可 | あり |
| `disable-model-invocation: true` | 可 | 不可 | なし |
| `user-invocable: false` | 不可 | 可 | あり |

注意点として、`user-invocable: false` は「Claude に呼ばせない」設定ではありません。Skill ツール経由の呼び出しまで止めたいなら `disable-model-invocation: true` が必要です。

SKILL.md を書き換えたくない共有リポジトリのスキルは、settings 側の `skillOverrides` で制御できます。

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "old-deploy": "off"
  }
}
```

`name-only` は名前だけ一覧に出して description を落とす設定で、スキルが増えて一覧の予算を圧迫するときに使えます。

## Step 4: 権限制御（allowed-tools と Skill ルール）

権限まわりは2層あって、混同しやすいところです。

**1. スキル実行中に承認なしで使えるツール: `allowed-tools`**

```yaml
---
name: commit
description: Stage and commit the current changes
disable-model-invocation: true
allowed-tools: Bash(git add *) Bash(git commit *) Bash(git status *)
---

変更内容を確認し、規約に沿ったメッセージで commit する。
```

ドキュメントを読んで一番大事だと感じたのは次の点です。

- `allowed-tools` は**そのスキルを呼んだターンだけ**の事前承認で、次のメッセージを送ると失効する
- **使えるツールを制限するものではない**。列挙していないツールも呼べて、その承認は通常の permission 設定に従う
- 作業フォルダを信頼していなくても、プロジェクトスキルの `allowed-tools` は適用される

つまり、他人のリポジトリを clone して `claude` を動かす前に `.claude/skills/*/SKILL.md` の `allowed-tools` を見ておく必要があります。

使えるツールを実際に減らしたい場合は `disallowed-tools` です。スキルが有効な間だけ、列挙したツールが Claude の利用可能なツールから外れます。こちらも次のメッセージで解除されます。

```yaml
disallowed-tools: AskUserQuestion
```

**2. Claude がどのスキルを呼べるか: permission ルール**

`settings.json` の permissions（または `/permissions`）に `Skill(...)` の形で書きます。

```text
# 全スキルを無効化（deny に追加）
Skill

# allow: 指定スキルだけ
Skill(commit)
Skill(review-pr *)

# deny: 指定スキルを禁止
Skill(deploy *)
```

`Skill(name)` は完全一致、`Skill(name *)` は引数付きを含む前方一致です。

## Step 5: サブエージェントで隔離実行する

調査系のスキルは、本文をメイン会話に流し込むとコンテキストを食います。`context: fork` を付けると、スキルの本文をプロンプトとして新しいサブエージェントが実行します。

```yaml
---
name: pr-summary
description: Summarize changes in a pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## Pull request context
- PR diff: !`gh pr diff`
- PR comments: !`gh pr view --comments`
- Changed files: !`gh pr diff --name-only`

## Your task
Summarize this pull request...
```

`agent` には `Explore` / `Plan` / `general-purpose` か、`.claude/agents/` の自作サブエージェントを指定でき、省略すると `general-purpose` です。

落とし穴が3つあります。

- サブエージェントは**会話履歴を見ない**。本文だけで完結する「タスク」を書く。「この API 規約に従え」のような指針だけのスキルに `context: fork` を付けると、実行すべきものが無く、意味のある出力が返ってきません
- 既定ではバックグラウンド実行（v2.1.218 以降）。結果を同じターンで待つなら `background: false`。`-p` や Agent SDK では常に待ちます
- バックグラウンドの fork が加えた編集は `/rewind` で戻せない。戻すのは git です

## まとめ

- 置き場所は `.claude/skills/<name>/SKILL.md`。frontmatter は1行目の `---` から始める
- 自動発火の精度は description の書き出しで決まる。`when_to_use` と `paths` で範囲を絞る
- 副作用のある手順は `disable-model-invocation: true`。`user-invocable: false` は Claude を止めない
- `allowed-tools` は「そのターンだけの事前承認」であって制限ではない。制限は `disallowed-tools` と `Skill(...)` の permission ルール
- 調べ物は `context: fork` + `agent` で隔離する。本文は履歴に依存しない書き方にする

まず1本、毎回貼っている手順書を `SKILL.md` に移して、`What skills are available?` で一覧に出るところから確認してみてください。発火しない場合は、ほとんどが description か frontmatter の書き方が原因です。
