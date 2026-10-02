---
title: "Claude Code × Sandbox | Bashの書き込み・通信・認証情報をOSレベルで縛るまで"
tags:
  - ClaudeCode
  - Sandbox
  - セキュリティ
  - AIエージェント
  - 設定
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「Hooksで危険コマンドは止めているけれど、`curl` の先や `~/.aws` までは見ていない。本当に守れているのか」

——この感覚は正しいです。コマンド文字列をパターンで見張る方式は、書き方を変えられると抜けます。Claude Code にはこれとは別の層として、OS が強制するBashサンドボックスがあります。この記事では、有効化から検証、認証情報の保護、抜け道を塞ぐところまでを順番に設定します。

筆者の環境（macOS）では、このセッション自体がサンドボックス下で動いています。`.env` の読み取りや許可外ホストへの通信は、コマンドの書き方に関係なく OS に拒否されます。以下はその設定を、公式ドキュメントで確認できた項目だけで組み直したものです。

## 前提：何を守り、何を守らないか

公式ドキュメントによると、サンドボックスは **Bash などシェルコマンドと、そこから起動するプロセス** を囲います。macOS・Linux・WSL2 で動き、ネイティブ Windows では非サンドボックスで実行されます。実装は OSS の `@anthropic-ai/sandbox-runtime` です。

| 対象 | 既定の挙動 |
|---|---|
| 書き込み | 作業ディレクトリ、ユーザー別の一時領域、追加したディレクトリだけ |
| 読み取り | マシンのほぼ全域（`~/.ssh` や `~/.aws/credentials` も含む） |
| 通信 | プロキシ経由のみ。許可ドメインは空から始まる |

**読み取りが既定で広い**点に注意してください。書き込みが閉じていても、認証情報は読めてしまいます。また、Read / Edit / WebFetch などの組み込みツール、MCP サーバー、Hooks はサンドボックスの外で動きます。これらは権限ルール側で制御します。

## Step 1 有効化する

サンドボックスは既定でオフです。セッションで `/sandbox` を実行するとパネルが開き、モードを選べます。選んだ内容はプロジェクトの `.claude/settings.local.json` に保存されます。全プロジェクトで有効にするなら `~/.claude/settings.json` に書きます。

```json
{
  "sandbox": {
    "enabled": true
  }
}
```

Linux と WSL2 では `bubblewrap` と `socat` が必要です。足りない場合、`/sandbox` には Dependencies タブだけが表示されます。入れたら Claude Code を再起動してください。

**防止策:** 依存が欠けていると、既定では **サンドボックスなしでコマンドが走ります**。守りたいなら `failIfUnavailable` を立て、起動できないときは終了させます。

```json
{
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true
  }
}
```

## Step 2 本当に効いているか検証する

「緑に見える」だけでは測れていません。Claude に次の2行を実行させてください。自分で `!` プロンプトに打つとサンドボックス外で動くことが多く、検証になりません。

```bash
# ホーム直下への書き込み → 拒否されるはず
touch ~/sandbox-probe

# プロキシ迂回での通信 → 経路が無いので失敗するはず
curl --noproxy '*' https://example.com
```

期待する結果は、`touch` が macOS では `Operation not permitted`、Linux/WSL2 では `Read-only file system` で失敗すること、`curl` が `Could not resolve host` で失敗することです。`touch` が成功した場合は、`~/sandbox-probe` を消してから `/sandbox` の状態を確認します。

**防止策:** Claude が「サンドボックス外でリトライしますか」と聞いてきたら、この検証中は必ず断ってください。

## Step 3 書き込み・読み取りの境界を決める

`kubectl` や `terraform` が `~/.kube` に書く場合、コマンドごとサンドボックスから外すのではなく、パスだけを開けます。OS レベルで強制され、子プロセスにも効きます。

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"],
      "denyRead": ["~/"],
      "allowRead": ["~/projects"]
    }
  }
}
```

読み取りルールが重なった場合は、より狭いパスが勝ちます。上の例では `~/projects` だけ読め、ホームのそれ以外は読めません。逆に `allowRead: ["~/"]` と `denyRead: ["~/**/.env"]` を組み合わせると、広い許可の中でも `.env` は塞がれたままです。

**防止策:** `denyRead` はBashだけに効き、Read ツールは止められません。ファイルツール側は `permissions.deny` で別に塞ぎます。

## Step 4 通信先を許可リストにする

通信は許可ドメインを足していく方式です。

```json
{
  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org"]
    }
  }
}
```

リストにないホストは、既定では権限モードに従って確認プロンプトになります。確認を出さず拒否したいなら `strictAllowlist: true` を立てます（Claude Code v2.1.219 以降、ユーザー/管理/`--settings` で有効、リポジトリ内の設定では無効）。

**防止策:** `allowedDomains` はBashの通信にだけ効き、WebFetch は制限しません。WebFetch は `WebFetch(domain:...)` の権限ルールで管理します。

## Step 5 認証情報を見せない

組み込みの拒否リストはなく、書いたものだけが対象になります。ファイルは読み取り拒否、環境変数はコマンド実行前に unset されます。

```json
{
  "sandbox": {
    "enabled": true,
    "credentials": {
      "files": [
        { "path": "~/.aws/credentials", "mode": "deny" },
        { "path": "~/.ssh", "mode": "deny" }
      ],
      "envVars": [
        { "name": "GITHUB_TOKEN", "mode": "deny" }
      ]
    }
  }
}
```

トークンが必要な通信は `mask` にできます。コマンドにはセッション固有のダミー値だけが見え、プロキシが許可ホストへのリクエストで本物に差し替えます。TLS 終端が必要で、環境変数のマスクは v2.1.199 以降です。

```json
{
  "sandbox": {
    "enabled": true,
    "network": {
      "tlsTerminate": {},
      "allowedDomains": ["*.github.com"]
    },
    "credentials": {
      "envVars": [
        { "name": "GH_TOKEN", "mode": "mask", "injectHosts": ["api.github.com"] }
      ]
    }
  }
}
```

**防止策:** `mask` や `tlsTerminate` は、ユーザー設定・管理設定・`--settings` からしか有効になりません。クローンしたリポジトリの `.claude/settings.json` に仕込まれても無視されます。

## Step 6 抜け道を塞ぐ

失敗したコマンドを Claude が `dangerouslyDisableSandbox` でサンドボックス外にリトライできる仕組みがあります。セッション限定で潰すなら次のようにします。

```bash
claude --settings '{"sandbox": {"enabled": true, "allowUnsandboxedCommands": false}}'
```

これで `dangerouslyDisableSandbox` は無視され、`excludedCommands` に一致しない限りすべてサンドボックス内で動きます。`/sandbox` の Overrides タブでは Strict sandbox mode と表示されます。もう一つの出口が `excludedCommands` です。`docker compose *` のように、本当にサンドボックスに入らないコマンドだけを書き、増やしすぎないでください。

## まとめ

- 有効化は `sandbox.enabled`。依存欠落で素通りしないよう `failIfUnavailable` も立てる
- `touch ~/sandbox-probe` と `curl --noproxy '*'` で、効いていることを実測する
- 開けるのは `allowWrite` / `allowedDomains` で必要最小限。コマンドごと除外するのは最後の手段
- 認証情報は `deny` か `mask`。リトライの出口は `allowUnsandboxedCommands: false` で閉じる
- 守備範囲はBashのみ。Read・WebFetch・MCP・Hooks は権限ルールと別に設計する

PreToolUse の Hooks が「入口で見張る」層なら、サンドボックスは「通り抜けた後も OS が止める」層です。両方置くことで、片方が破られても被害がファイルシステムとネットワークの境界で止まります。なお、設定キーや対応バージョンは公式ドキュメント（code.claude.com の Sandboxing ページ）の 2026年10月2日時点の記載に基づいています。更新が速い領域なので、導入前に最新を確認してください。