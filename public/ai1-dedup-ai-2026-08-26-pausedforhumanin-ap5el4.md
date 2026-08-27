---
title: "Claude Code × Docker | ローカルMCPサーバーをHTTP化してコンテナ配布するまで"
tags:
  - MCP
  - ClaudeCode
  - Docker
  - TypeScript
  - NodeJS
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「stdioで動くMCPサーバーは作れたけど、チームメンバーの端末やCIからも同じサーバーを叩きたい」——この感覚は正しいです。stdioトランスポートはプロセスをローカルに1個ずつ起動する前提なので、共有したい瞬間に詰まります。

この記事では、既存のstdio実装をStreamable HTTPトランスポートに載せ替え、Dockerでコンテナ化し、Claude Code CLIから`--transport http`で登録するところまでを、実際に動くコードで追っていきます。

## なぜstdioのままだと共有できないのか

stdioトランスポートは「Claude CodeがサブプロセスとしてMCPサーバーを起動し、標準入出力でJSON-RPCをやり取りする」方式です。1人の開発機で完結する分には手軽ですが、次のような要求には応えられません。

- 別マシン(CI、他のメンバーの端末)から同じツールを呼びたい
- 認証を挟んでアクセス制御したい
- サーバー側の状態(RAGインデックス、DBコネクションプール等)をプロセス間で共有したい

2025年3月のMCP仕様改定で、リモート向けの標準トランスポートとしてStreamable HTTPが追加されました。Claude Code CLI側もこれに対応していて、`claude mcp add --transport http <name> <url>`で登録できます。ここではこの経路を実際に組み立てます。

## Step 1: stdio実装をStreamable HTTPへ載せ替える

`@modelcontextprotocol/sdk`の`McpServer`自体はトランスポートに依存しません。stdio版で使っていた`server.tool(...)`の登録コードはそのままに、接続部分だけ`StreamableHTTPServerTransport`に差し替えます。

```typescript
// src/server.ts
import express from "express";
import { randomUUID } from "node:crypto";
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { z } from "zod";

function buildServer() {
  const server = new McpServer({ name: "hub-seo", version: "1.0.0" });

  // stdio版から移植したツール定義。ここは変更不要
  server.tool(
    "expand_keywords",
    "Googleサジェストからキーワード候補を展開する",
    { seed: z.string() },
    async ({ seed }) => ({
      content: [{ type: "text", text: `expanded: ${seed} ...` }],
    }),
  );

  return server;
}

const app = express();
app.use(express.json());

// セッションごとにtransportを保持する(状態を持つツールがある場合に必要)
const transports = new Map<string, StreamableHTTPServerTransport>();

app.post("/mcp", async (req, res) => {
  const sessionId = req.header("mcp-session-id");
  let transport = sessionId ? transports.get(sessionId) : undefined;

  if (!transport) {
    transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => randomUUID(),
      onsessioninitialized: (id) => transports.set(id, transport!),
    });
    const server = buildServer();
    await server.connect(transport);
  }

  await transport.handleRequest(req, res, req.body);
});

app.listen(3800, () => console.log("MCP server listening on :3800"));
```

`sessionIdGenerator`を`undefined`にすればステートレス(リクエストごとに使い捨て)にもできます。ツールが外部APIを叩くだけで内部状態を持たないなら、まずはステートレスから始めるのが安全です。セッション管理のバグを増やさずに済みます。

## Step 2: Dockerでコンテナ化する

stdioのままだとホストのファイルシステムやnpx実行環境に依存しがちですが、HTTP化するとコンテナに閉じ込めやすくなります。

```dockerfile
FROM node:24-slim AS build
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:24-slim
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
EXPOSE 3800
CMD ["node", "dist/server.js"]
```

```bash
docker build -t hub-mcp-seo:1.0.0 .
docker run -d --name hub-mcp-seo -p 127.0.0.1:3800:3800 hub-mcp-seo:1.0.0
```

ここで意図的に`-p 127.0.0.1:3800:3800`とホスト側バインドを絞っています。`-p 3800:3800`とだけ書くと全インターフェースに公開され、同一LAN上の別端末から素通しでツールが叩けてしまいます。実運用では後述のトークン認証とセットで、まず公開範囲を絞るのが先です。

## Step 3: Claude Code CLIへHTTPサーバーとして登録する

```bash
claude mcp add --transport http hub-seo http://127.0.0.1:3800/mcp
```

登録後は`claude mcp list`で`hub-seo`が`connected`になっているか確認します。stdio版と違い、Claude Code側でプロセスを起動し直す必要がないので、サーバーを再デプロイしても`claude mcp add`をやり直す必要はありません。

疎通確認だけならcurlでも叩けます。

```bash
curl -i -X POST http://127.0.0.1:3800/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"curl-test","version":"0"}}}'
```

レスポンスヘッダの`mcp-session-id`が発行されていれば、以降のリクエストにこのIDを`Mcp-Session-Id`ヘッダとして付けることで同一セッションを継続できます。

## Step 4: 認証ヘッダで無認証運用を避ける

HTTP化した時点で「誰でも叩ける」状態になりがちです。`claude mcp add`は`--header`でリクエストヘッダを追加できるので、Bearerトークンを1枚挟みます。

```bash
claude mcp add --transport http hub-seo http://127.0.0.1:3800/mcp \
  --header "Authorization: Bearer ${HUB_MCP_TOKEN}"
```

サーバー側は素直にミドルウェアで検証します。

```typescript
app.use("/mcp", (req, res, next) => {
  const auth = req.header("authorization");
  if (auth !== `Bearer ${process.env.HUB_MCP_TOKEN}`) {
    res.status(401).json({
      jsonrpc: "2.0",
      error: { code: -32001, message: "unauthorized" },
      id: null,
    });
    return;
  }
  next();
});
```

トークンは`.env`に置き、リポジトリにはコミットしません。`docker run`に渡す場合も`--env-file .env`経由にして、`docker inspect`でホスト側の誰かがコマンド履歴から値を見られる状態を避けます。認証を後回しにしたままDockerで公開ポートを開けると、`-p 3800:3800`の一行だけで社内ネットワークの誰からでもツールが呼べる、という事故につながります。

## まとめ

stdio実装をStreamable HTTPに載せ替える作業自体は、`McpServer`のツール定義を変えずに接続部分だけ差し替えるので大きくありません。実際に手を動かすと詰まるのはむしろ運用面で、「セッションをどう持つか」「公開範囲をどう絞るか」「認証をどこで挟むか」の3点です。

まずステートレスで動かして、状態が必要になったタイミングでセッション管理を足す。公開ポートは`127.0.0.1`縛りから始めて、必要な範囲だけ開ける。この順番を守るだけで、共有可能なMCPサーバーへの移行はそれほど怖くありません。