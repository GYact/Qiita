---
title: "Claude Code × settings.json | 許可プロンプトの集計からallowlistの設計・自動承認の線引きまで"
tags:
  - ClaudeCode
  - 生成AI
  - セキュリティ
  - 自動化
  - settings.json
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「`git status` にまで毎回許可を聞かれる。いっそ全部スキップしてしまおうか」

——この感覚は正しいです。許可プロンプトが多すぎると、人は中身を読まずに Enter を押すようになります。確認が形骸化した時点で、確認は安全装置として働いていません。

ただし、解決策は「確認を全部切る」ではありません。**実際に聞かれている操作を集計し、読み取り専用のものだけを allowlist に落とし、危険なものは deny で固定する**、という順で進めます。この記事では、その手順をコマンドとともに書きます。

## 前提：許可ルールの評価順

`settings.json` の `permissions` には `allow` / `ask` / `deny` の3つがあります。評価は **deny → ask → allow** の順で、deny に当たれば allow に書いてあっても通りません。この性質があるので、「広めに allow し、危険な例外を deny で潰す」という設計ができます。

設定ファイルは次の場所に置けます。

| 置き場所 | 用途 |
|---|---|
| `~/.claude/settings.json` | 全プロジェクト共通の個人設定 |
| `.claude/settings.json` | チームで共有（Git 管理） |
| `.claude/settings.local.json` | 自分だけの上書き（Git 除外） |

## Step 1 まず「何を聞かれているか」を数える

勘で allowlist を書くと、効果の薄いルールばかりになります。先にセッションのログから、実行された Bash コマンドの先頭2語を集計します。

```bash
# 過去セッションの Bash 呼び出しを、先頭2語で集計して上位30件を出す
cat ~/.claude/projects/*/*.jsonl \
  | jq -r 'select(.type=="assistant")
           | .message.content[]?
           | select(.type=="tool_use" and .name=="Bash")
           | .input.command' 2>/dev/null \
  | awk '{print $1" "$2}' \
  | sort | uniq -c | sort -rn | head -30
```

ログの形式は将来変わりうるので、`jq` が何も出さなければ `head -1` で1行読んで構造を確認してください。

上位に並ぶのは、たいてい `git status` `git diff` `ls` `pnpm test` のような操作です。ここが allow の候補になります。

**防止策：集計結果を見る前に allow を書かない。** 聞かれていない操作を許可しても、プロンプトは減りません。

## Step 2 読み取り専用だけを allow に入れる

候補のうち、副作用のないものだけを残します。判断基準は「実行して取り消す必要が生じないか」です。

```json
{
  "permissions": {
    "allow": [
      "Bash(git status:*)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Bash(git branch --show-current)",
      "Bash(ls:*)",
      "Bash(pnpm test:*)",
      "Bash(pnpm typecheck)",
      "Bash(pnpm exec tsc --noEmit)"
    ]
  }
}
```

注意点が2つあります。

- `Bash(git branch:*)` のように広く書くと `git branch -D` まで通ります。副作用のある引数を持つコマンドは、引数まで固定して書きます。
- `pnpm test` はテストが書き込みや外部通信をする場合があります。そのリポジトリで中身を確認してから入れてください。

組み込みの `fewer-permission-prompts` スキルは、トランスクリプトから読み取り専用の呼び出しを拾って allowlist 案を出してくれます。手で集計する代わりに使えますが、出てきた案は Step 2 の基準で必ず目で確認します。

## Step 3 deny で「絶対に通さないもの」を固定する

allow を足す前に、deny を書きます。シークレットと不可逆な操作が対象です。

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(~/.ssh/**)",
      "Bash(rm -rf:*)",
      "Bash(git push --force:*)",
      "Bash(curl:*)"
    ]
  }
}
```

deny は allow より先に評価されるので、後から誰かが `Bash(*)` を足しても `.env` の読み取りは止まります。

**防止策：deny は共有の `.claude/settings.json` に置いて Git 管理する。** 個人の `settings.local.json` にだけ書くと、別の端末や別のメンバーでは効きません。

## Step 4 編集は「モード」で緩め、Bash は緩めない

ファイル編集の確認が多いなら、個別の allow ではなく `defaultMode` で扱います。

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

`acceptEdits` はファイル編集の承認を省くモードで、Bash の承認は残ります。差分は `git diff` で後から確認でき、元に戻せます。一方、Bash は外部への送信や削除を含むので、緩めるのは Step 2 で選んだ読み取り系までにします。

`bypassPermissions`（全スキップ）は、使い捨てのコンテナなど、壊れても困らない環境に限定します。手元の開発マシンでは使いません。

## Step 5 効いているかを確認する

設定を書いたら、実際に確認します。

```bash
# 現在有効なルールを対話画面で確認（allow / ask / deny の一覧）
claude
# セッション内で /permissions を実行
```

確認項目は3つです。

1. `git status` が聞かれなくなったか
2. `cat .env` が拒否されるか
3. `git status && curl example.com` のように許可済みコマンドへ別コマンドを連結しても、後半が許可されないか

3つ目は特に大事です。前方一致のルールは連結コマンドの全体を通す仕組みではないので、実際に試して挙動を見てください。

## まとめ

進め方は次の順です。

1. ログから聞かれている操作を集計する
2. deny でシークレットと不可逆操作を先に固定する
3. 読み取り専用の操作だけを allow に入れる
4. 編集は `acceptEdits` で緩め、Bash は緩めない
5. `/permissions` と実際のコマンドで効き方を確かめる

プロンプトを減らす目的は、押す回数を減らすことです。聞かれる操作が「本当に判断が要るもの」だけになれば、残った確認はちゃんと読まれます。

次の一歩としては、Step 1 のコマンドを手元で実行し、上位10件のうち読み取り専用がいくつあるかを数えるところから始めるのが手軽です。