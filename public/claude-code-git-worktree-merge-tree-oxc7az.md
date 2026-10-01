---
title: "Claude Code × git worktree | 並列セッションの隔離からmerge-tree検査・後片付けまで"
tags:
  - ClaudeCode
  - Git
  - git-worktree
  - AIエージェント
  - 開発効率化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「同じリポジトリで Claude Code を2本走らせたら、片方の作業中にブランチが切り替わっていた」「`git restore` を打ったら、別セッションの未コミット実装が消えた」

——この感覚は正しいです。1つの作業ツリーを複数のエージェントで共有すると、HEAD・index・未コミット差分が全員の共有状態になります。私たちの運用でも、ブランチを別セッションに切り替えられる事故や、未コミットの実装を巻き込んで消した事故を実際に起こしました。

この記事では、`git worktree` で Claude Code のセッションごとに作業ツリーを分け、`git merge-tree` で衝突を事前に検査してから取り込み、最後に片付けるところまでを、コピペで動く形で実装します。

## なぜ clone ではなく worktree なのか

| 方式 | ディスク | `.git` の共有 | 同一ブランチの二重チェックアウト |
|---|---|---|---|
| 同一ディレクトリで並行 | 0 | 共有(HEAD/indexも共有) | 起きる |
| `git clone` を複数 | 重い | 別々 | 起きない |
| `git worktree` | 軽い | オブジェクトDBは共有、HEAD/indexは別 | git が拒否する |

worktree は履歴を共有しつつ、HEAD と index をツリーごとに持ちます。さらに、同じブランチを2つのツリーでチェックアウトしようとすると git が拒否します。二重作業をツールの側で止められるのが利点です。

## Step 1 セッション用の worktree を切るスクリプト

まず、作業名を渡すと `origin/main` から専用ブランチと専用ディレクトリを作るスクリプトを用意します。

```bash
#!/usr/bin/env bash
# wt-new: セッション専用の worktree を origin/main から作る
set -euo pipefail

name="${1:?usage: wt-new <name>}"
repo_root="$(git rev-parse --show-toplevel)"
dir="$(dirname "$repo_root")/$(basename "$repo_root")-wt/${name}"

git fetch origin main
# --no-track: 作業ブランチが origin/main を upstream にして誤 push されるのを防ぐ
git worktree add --no-track -b "wt/${name}" "$dir" origin/main

# pnpm はストアを共有するので、2本目以降の install は速い
(cd "$dir" && pnpm install --frozen-lockfile)

echo "$dir"
```

`.env` はコピーしません。シークレットを worktree ごとに複製すると、消し忘れた平文が増えます。SOPS の `.env.enc` を各ツリーで復号するか、必要なキーだけ環境変数で渡してください。

## Step 2 Claude Code をその worktree で起動する

```bash
dir="$(./wt-new fix-login-redirect)"
cd "$dir" && claude
```

ここで守るルールは1つです。**編集は worktree の絶対パスで行う**。調査のために元のチェックアウトへ `cd` し、そのまま編集まで進んでしまう事故が実際にありました。CLAUDE.md に次の1行を置いておくと、セッションが自分の居場所を取り違えにくくなります。

```markdown
## 作業場所
- このセッションの作業ツリーは `git rev-parse --show-toplevel` の値のみ。
  別ディレクトリのファイルを編集する前に、必ずユーザーに確認する。
```

**防止策**: 各 worktree の先頭で `git branch --show-current` が `wt/` で始まらなければ、コミット前に止める pre-commit を入れます。

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit の一部(worktree 用ブランチ以外での commit を止める)
branch="$(git branch --show-current)"
if [[ "$branch" == "main" && "${ALLOW_MAIN_COMMIT:-0}" != "1" ]]; then
  echo "main への直接 commit は止めています。wt/ ブランチで作業してください" >&2
  exit 1
fi
```

## Step 3 取り込み前に merge-tree で衝突を調べる

並列で走らせると、2本が同じファイルを触ります。マージしてから衝突に気づくのでは遅いので、作業ツリーを汚さずに結果だけ見られる `git merge-tree`(git 2.38 以降の `--write-tree`)を使います。

```bash
# 衝突の有無とファイル名だけ確認する(作業ツリー・index は触らない)
if out="$(git merge-tree --write-tree --name-only main wt/fix-login-redirect)"; then
  echo "衝突なし: tree=$(echo "$out" | head -1)"
else
  echo "衝突あり:"
  echo "$out" | tail -n +2
  exit 1
fi
```

終了コードが 0 なら衝突なし、1 なら衝突ありです。溜まったブランチをまとめて仕分けるときも、この方法が一番信頼できました。私たちの環境では、46本のブランチのうち42本は取り込み不要か衝突なしの偽陽性で、実際に人が見るべきものは5本でした。

**注意**: 衝突がなくても意味が壊れることはあります。片方が関数名を変え、もう片方がその旧名を使う新規コードを足した場合、textual には衝突しません。2本以上が触ったファイルは、取り込み後に旧名を `grep` で確認してください。

## Step 4 取り込みは cherry-pick、判断に git cherry を使わない

```bash
git switch main && git pull --ff-only
git cherry-pick main..wt/fix-login-redirect
pnpm check   # 取り込み後に検証してから push
```

「すでに main に入っているか」を `git cherry` の `+` で判断してはいけません。`+` は patch-id が一致しないという意味でしかなく、内容が既に main にあっても、見送った実験ブランチでも `+` になります。実際に、この判断で古いブランチを merge して画面の UI が巻き戻ったことがあります。取り込む前に `git diff main...wt/xxx --stat` で、そのブランチが本当に何を足し、何を消すのかを読みます。

## Step 5 後片付けまで自動化する

```bash
#!/usr/bin/env bash
# wt-done: 取り込み済みの worktree とブランチを片付ける
set -euo pipefail

name="${1:?usage: wt-done <name>}"
repo_root="$(git rev-parse --show-toplevel)"
dir="$(dirname "$repo_root")/$(basename "$repo_root")-wt/${name}"

# 未コミットの変更があると git が拒否する(--force は付けない)
git worktree remove "$dir"
# -d は main にマージ済みのときだけ消える。未マージなら失敗して知らせてくれる
git branch -d "wt/${name}"
git worktree prune
```

`--force` と `-D` を付けないのがポイントです。未コミットや未マージがあれば git が止めてくれるので、他セッションの成果を巻き込んで消す事故を防げます。

## まとめ

- 並列で動かすエージェントには、セッションごとに `git worktree` を1つ割り当てる
- 作業場所の取り違えは、絶対パスと pre-commit で機械的に止める
- 取り込み前は `git merge-tree --write-tree` で衝突を確認し、`git cherry` は判断に使わない
- 片付けは `--force` なしの `worktree remove` と `branch -d` にして、git に安全確認をさせる

まずは `wt-new` を1本作って、次の並列タスクから使ってみてください。