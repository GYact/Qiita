---
title: Claude Code の MCP サーバー活用術：外部ツール連携で開発効率を最大化する
tags:
  - AI
  - MCP
  - 開発効率化
  - Anthropic
  - ClaudeCode
private: false
updated_at: '2026-04-12T13:45:32+09:00'
id: d0a99d49b399529290f9
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

Claude Code を単なる「コード補完ツール」として使っているなら、その真価の半分も引き出せていない。  
MCP（Model Context Protocol）サーバーを組み合わせることで、Claude Code は「コードを書くだけのAI」から「外部サービスを横断して動く自律エージェント」へと変貌する。

本記事では、MCP サーバーの仕組みから実践的な設定・活用パターンまでを、現場目線でまとめる。

---

## MCP とは何か

MCP（Model Context Protocol）は Anthropic が策定したオープンプロトコルで、**LLM が外部ツール・データソースと安全に対話するための標準インターフェース**を定義する。

従来の「function calling」と何が違うのか？

| 比較軸 | Function Calling | MCP |
|---|---|---|
| 定義場所 | プロンプト内 / APIリクエスト | サーバーとして独立 |
| 再利用性 | アプリ固有 | 任意のMCP対応クライアントで使い回せる |
| スコープ | 1つのリクエスト | セッションを超えた永続的な接続 |
| エコシステム | ベンダー依存 | オープン標準（多数のOSS実装あり）|

MCP サーバーは **stdio** または **SSE（Server-Sent Events）** 経由で通信し、`tools`・`resources`・`prompts` の3種のプリミティブを提供する。

---

## Claude Code への MCP サーバー追加方法

設定は `~/.claude/claude_code_config.json`（グローバル）またはプロジェクトルートの `.claude/claude_code_config.json`（ローカル）で管理する。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server-supabase@latest",
               "--project-ref", "your_project_ref"],
      "env": {
        "SUPABASE_ACCESS_TOKEN": "${SUPABASE_ACCESS_TOKEN}"
      }
    }
  }
}
```

```bash
# CLI での追加も可能（グローバル）
claude mcp add github -e GITHUB_PERSONAL_ACCESS_TOKEN=your_token \
  -- npx -y @modelcontextprotocol/server-github

# プロジェクトスコープで追加
claude mcp add --scope project supabase \
  -- npx -y @supabase/mcp-server-supabase --project-ref your_ref
```

起動確認は `claude mcp list` で行う。

---

## 実践的な活用パターン

### 1. GitHub MCP × Claude Code：PR レビューをAIに任せる

```
# Claude Code での指示例
gh pr 一覧から未レビューのPRを取得して、
変更ファイルを確認したうえでコードレビューコメントを投稿してください。
セキュリティリスクと N+1 クエリに特に注目してください。
```

GitHub MCP が持つツール群（`list_pull_requests`・`get_pull_request_files`・`create_pull_request_review`）を Claude が自律的に呼び出し、人間がやっていたコンテキスト収集→コメント投稿の一連の作業を代行する。

---

### 2. Supabase MCP：自然言語でスキーマ変更・SQL実行

```
# Claude Code での指示例
users テーブルに last_login_at カラム（timestamptz, nullable）を追加して、
既存のセッションデータから値をバックフィルするマイグレーションを実行してください。
```

`execute_sql`・`apply_migration`・`list_tables` を組み合わせ、スキーマ確認→マイグレーション生成→適用の一連のフローを自動化できる。

> ⚠️ **注意**: DDL 操作は `--no-verify-jwt` フラグの有無によって挙動が変わる。本番環境での実行前に必ず適用内容を確認すること。

---

### 3. Filesystem + Memory MCP：長期記憶を持つ開発アシスタント

標準の `@modelcontextprotocol/server-filesystem` と Memory サーバーを組み合わせると、セッションをまたいだ文脈保持が実現できる。

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem",
               "/Users/you/Developer/my-project"]
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    }
  }
}
```

```
# 使い方例
前回の設計議論の内容をメモリから取得して、
今日追加する機能がその設計方針と矛盾しないか確認してください。
```

Memory サーバーはナレッジグラフとして `entities` と `relations` を管理するため、「あのとき決めたこと」を Claude が自力で思い出せるようになる。

---

### 4. カスタム MCP サーバーを自作する

既存サーバーにない社内ツール（Jira・社内API等）は、SDK を使って数十行で作れる。

```typescript
// my-jira-mcp/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "jira-mcp", version: "1.0.0" });

server.tool(
  "get_my_issues",
  "自分にアサインされた未完了チケットを取得する",
  { project: z.string().optional() },
  async ({ project }) => {
    const issues = await fetchJiraIssues({ assignee: "currentUser()", project });
    return {
      content: [{ type: "text", text: JSON.stringify(issues, null, 2) }],
    };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

`stdio` 経由で通信するだけなので、既存の認証情報（環境変数）をそのまま使い回せる点が実用上の大きなメリットだ。

---

## 設計上の注意点

### 権限スコープを最小化する

MCP サーバーに渡すトークンは**必要最小限の権限**に絞ること。GitHub であれば `repo:read` のみにし、`delete` 権限は与えない。Claude が誤ってリポジトリを削除するリスクを構造的に排除する。

### ローカルスコープ vs グローバルスコープ

個人のワークフロー用ツール（GitHub, Memory など）はグローバル、プロジェクト固有のもの（Supabase, 社内API）はプロジェクトスコープで管理するのがベストプラクティスだ。チームリポジトリで `.claude/claude_code_config.json` を管理することで、チーム全体でMCPを共有できる。

### CORS・認証エラーのデバッグ

MCP サーバーが応答しない場合は `claude mcp list` でステータスを確認し、`stderr` のログを確認する。Supabase Edge Functions を MCP バックエンドとして使う場合は、`Authorization`・`apikey`・`content-type` の3つが `Access-Control-Allow-Headers` に含まれているかチェックすること（CORS ヘッダー不備は最も多いミスの一つ）。

---

## まとめ

| 活用パターン | 得られる効果 |
|---|---|
| GitHub MCP | PR レビュー・Issue 管理の自動化 |
| Supabase MCP | 自然言語でのDB操作・マイグレーション |
| Memory MCP | セッション間の文脈保持 |
| カスタム MCP | 社内ツールとのシームレスな統合 |

MCP の真髄は「AIに道具を持たせる」ことではなく、**「AIが必要な情報に自律的にアクセスできる環境を整えること」**にある。  
適切なサーバーを揃えれば、Claude Code は「指示を待つツール」から「タスクを完遂するエージェント」に変わる。まずは GitHub MCP の1台から始めてみてほしい。

---

## 参考リンク

- [MCP 公式仕様](https://spec.modelcontextprotocol.io/)
- [MCP サーバー一覧（公式）](https://github.com/modelcontextprotocol/servers)
- [Claude Code ドキュメント](https://docs.anthropic.com/ja/docs/claude-code)
