---
title: "Claude Code × MCP | 自作サーバーの実装から登録・接続失敗の切り分けまで"
tags:
  - ClaudeCode
  - MCP
  - TypeScript
  - AI
  - 開発効率化
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「MCPサーバーは既製品を入れるものだと思っていた。自分のリポジトリ専用のツールを生やすのは、大げさでは？」

——この感覚は正しいです。ただ、**数十行の TypeScript で足りる**ところまで小さくすれば、話は変わります。この記事では、TODO/FIXME を拾うだけの MCP サーバーを書き、Claude Code に登録し、繋がらないときの切り分けまでを順に進めます。

## なぜ自作するのか

Claude Code は `grep` も読めますが、プロジェクト固有の「見方」は毎回プロンプトで説明することになります。

- 「TODO は `node_modules` と `dist` を除いて、ファイル名と行番号付きで」
- 「結果は 50 件で切って」

これをツールとして固定すると、呼び出しのたびに説明する必要がなくなり、出力の形も安定します。標準入出力(stdio)で動くローカルサーバーなら、ネットワークも認証も要りません。

## Step 1 プロジェクトを作る

```bash
mkdir todo-finder && cd todo-finder
pnpm init
pnpm add @modelcontextprotocol/sdk zod
pnpm add -D typescript @types/node
```

`package.json` に `"type": "module"` を足し、`tsconfig.json` は次の最小構成にします。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "strict": true
  },
  "include": ["src"]
}
```

## Step 2 ツールを実装する

`src/index.ts` です。ポイントは 2 つあります。

1. **`console.log` を使わない。** stdio では標準出力がプロトコルそのものなので、ログを書くとメッセージが壊れて接続が落ちます。ログは `console.error` に出します。
2. **読む範囲を作業ディレクトリ配下に閉じる。** `dir` はモデルが組み立てる引数なので、`../../` を渡されても外へ出ないようにします。

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { readdir, readFile } from "node:fs/promises";
import path from "node:path";
import { z } from "zod";

const ROOT = process.cwd();
const SKIP = new Set(["node_modules", ".git", "dist"]);
const PATTERN = /\b(TODO|FIXME)\b[:\s]*(.*)/;

async function* walk(dir: string): AsyncGenerator<string> {
  for (const entry of await readdir(dir, { withFileTypes: true })) {
    if (SKIP.has(entry.name)) continue;
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) yield* walk(full);
    else if (/\.(ts|tsx|js|py|md)$/.test(entry.name)) yield full;
  }
}

const server = new McpServer({ name: "todo-finder", version: "0.1.0" });

server.registerTool(
  "find_todos",
  {
    description: "指定ディレクトリ配下の TODO / FIXME を file:line 付きで返す",
    inputSchema: {
      dir: z.string().default("."),
      limit: z.number().int().min(1).max(200).default(50),
    },
  },
  async ({ dir, limit }) => {
    const target = path.resolve(ROOT, dir);
    // 作業ディレクトリの外は読まない
    if (target !== ROOT && !target.startsWith(ROOT + path.sep)) {
      return {
        isError: true,
        content: [{ type: "text", text: `許可されていないパス: ${dir}` }],
      };
    }

    const hits: string[] = [];
    for await (const file of walk(target)) {
      const lines = (await readFile(file, "utf8")).split("\n");
      lines.forEach((line, i) => {
        const m = PATTERN.exec(line);
        if (m && hits.length < limit) {
          hits.push(`${path.relative(ROOT, file)}:${i + 1} ${m[1]} ${m[2].trim()}`);
        }
      });
      if (hits.length >= limit) break;
    }

    console.error(`find_todos: ${hits.length} 件`);
    return {
      content: [{ type: "text", text: hits.join("\n") || "該当なし" }],
    };
  },
);

await server.connect(new StdioServerTransport());
```

SDK のバージョンによっては `registerTool` ではなく `server.tool(name, schema, handler)` の形です。手元の型定義に合わせてください。

ビルドします。

```bash
pnpm exec tsc
```

## Step 3 単体で動かしてから登録する

Claude Code に繋ぐ前に、MCP Inspector でサーバー単体の動作を確かめます。ここで動かないものは、Claude Code に繋いでも動きません。

```bash
pnpm dlx @modelcontextprotocol/inspector node dist/index.js
```

ブラウザで `find_todos` を実行し、`file:line` の一覧が返れば成功です。確認できたら、利用するプロジェクト側で登録します。

```bash
cd ~/Developer/my-project
claude mcp add --scope project todo-finder -- node /絶対パス/todo-finder/dist/index.js
claude mcp list
```

`--scope project` を付けると、プロジェクト直下の `.mcp.json` に書かれます。チームで共有したいならこちらが向いています。個人用なら既定のスコープのままで構いません。起動後は `/mcp` で接続状態を見られます。

## 繋がらないときの切り分け

実際に踏みやすい順に並べます。

| 症状 | 原因 | 確認 |
|---|---|---|
| `failed` と出る | 標準出力に余計な文字列 | コード内の `console.log` を `grep` |
| 起動直後に落ちる | 相対パスで登録した | `claude mcp list` の command が絶対パスか |
| ツールが見えない | `dist` が古い | `tsc` を再実行して再接続 |
| 想定外の場所を読む | cwd が違う | 返ってきた `file:line` の基準を見る |

最初の「標準出力」は一番ハマります。デバッグ用に仕込んだ `console.log` を消し忘れると、Inspector では動いたのに本番で落ちる、という順で気づくことになります。

## まとめ

- 自作 MCP サーバーは、`McpServer` + `StdioServerTransport` + Zod の 3 点で成立する
- stdio では `console.log` 禁止。ログは `console.error`
- モデルが組み立てる引数は信用せず、パスは作業ディレクトリ配下に閉じる
- 登録前に Inspector で単体確認し、`claude mcp list` と `/mcp` で接続状態を見る

次の一歩としては、`find_todos` に `git blame` を足して担当者別に返す、といった「そのリポジトリでしか意味を持たない」ツールが向いています。