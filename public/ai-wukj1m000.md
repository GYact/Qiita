---
title: 2026年のAIエージェント実装：MCPとマルチエージェントアーキテクチャの実践知見
tags:
  - TypeScript
  - AI
  - MCP
  - Agent
  - LLM
private: false
updated_at: '2026-04-13T16:36:10+09:00'
id: dff02cf5271048629857
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

2026年現在、AI活用の主戦場は「どのモデルを使うか」から「どうエージェントを設計するか」に完全に移行した。GPT-4o、Claude 3系、Gemini 2.xが乱立する中、現場エンジニアに求められるのはモデル選定眼より**エージェントアーキテクチャの設計力**だ。

本記事では、実際のプロダクション運用で得た知見をもとに、MCPとマルチエージェント構成における実装パターンと落とし穴を共有する。

---

## 背景：なぜ「エージェント設計」が差別化になったのか

2025年後半から主要LLMのコンテキストウィンドウが100万トークンを超え、単純なRAGやone-shot promptingの差は縮小した。一方で：

- **Tool-use（関数呼び出し）の精度**がモデル間で依然大きく異なる
- **長タスクの信頼性**（途中でハルシネーション・ループ・タイムアウト）はアーキテクチャ依存
- **コスト構造**が「呼び出し回数 × トークン数」から「アクティブCPU時間」モデルに変化

つまり、「賢いモデルを1回叩く」より「適切な粒度のエージェントを協調させる」設計が重要になった。

---

## 実践1：MCP（Model Context Protocol）でツール統合を標準化する

MCPはAnthropicが策定したオープンなツール統合プロトコルで、2025年に急速に普及した。LLMとツール（DB、API、ファイルシステム等）の間のJSON-RPC 2.0ベースのインターフェースを定義する。

### MCPサーバーの最小実装（TypeScript）

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "my-tool-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// ツール一覧の定義
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "search_database",
      description: "社内DBからレコードを検索する",
      inputSchema: {
        type: "object",
        properties: {
          query: { type: "string", description: "検索クエリ" },
          limit: { type: "number", default: 10 },
        },
        required: ["query"],
      },
    },
  ],
}));

// ツール実行ハンドラー
server.setRequestHandler(CallToolRequestSchema, async (req) => {
  if (req.params.name === "search_database") {
    const { query, limit = 10 } = req.params.arguments as {
      query: string;
      limit?: number;
    };
    // 実際のDB検索ロジック
    const results = await db.search(query, limit);
    return {
      content: [{ type: "text", text: JSON.stringify(results) }],
    };
  }
  throw new Error(`Unknown tool: ${req.params.name}`);
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

### MCPを使う際の重要な落とし穴

**ツール数の肥大化に注意**：ツールが20個を超えると、モデルが適切なツールを選択できなくなる。**関心ごとにMCPサーバーを分割**し、タスクに応じてサーバーをマウントする設計が有効。

```typescript
// ❌ 悪い例：全ツールを1サーバーに詰め込む
const tools = [searchDB, writeDB, sendEmail, readSlack, postSlack, ...20個以上];

// ✅ 良い例：ドメインごとに分割
const dbServer = createMCPServer("database-tools", [searchDB, writeDB]);
const commServer = createMCPServer("comms-tools", [sendEmail, postSlack]);
// エージェントのタスクに応じてマウントするサーバーを切り替える
```

---

## 実践2：マルチエージェント構成のパターンと選択基準

エージェントを複数協調させるパターンは主に3つある。

### パターンA：Orchestrator-Worker（最も汎用）

```
User → Orchestrator Agent
              ├→ Worker A（データ収集）
              ├→ Worker B（分析）
              └→ Worker C（レポート生成）
```

Orchestratorが全体計画を立て、Workerに並列/逐次でサブタスクを委譲する。**各WorkerのコンテキストをOrchestrator本体に持たせない**のがポイント。Workerの出力は要約してOrchestrator側に戻す。

```typescript
// Orchestratorのタスク分解ロジック例
async function orchestrate(userRequest: string) {
  const plan = await llm.generate({
    system: "タスクを独立したサブタスクに分解してJSONで返せ",
    prompt: userRequest,
  });

  // Worker並列実行
  const results = await Promise.allSettled(
    plan.tasks.map((task) => spawnWorker(task))
  );

  // 結果を要約してからOrchestrator文脈に追加
  const summary = results.map((r) =>
    r.status === "fulfilled" ? r.value.summary : `ERROR: ${r.reason}`
  );

  return await llm.generate({
    system: "各Workerの結果を統合して最終回答を生成せよ",
    prompt: summary.join("\n"),
  });
}
```

### パターンB：Pipeline（決定論的フロー）

入力 → エージェントA → エージェントB → 出力、と処理が一方向に流れる。ステージごとにバリデーションを挟むことで品質を保証しやすい。**コンテンツ生成 → レビュー → 整形**のような定型フローに向く。

### パターンC：Swarm（動的委譲）

エージェント同士が互いに仕事を渡し合う。柔軟性が高い反面、ループや無限委譲が発生しやすい。**最大ステップ数の上限**と**訪問済みエージェントの記録**は必須。

```typescript
const MAX_HOPS = 10;

async function swarmStep(
  agent: Agent,
  message: string,
  visited: Set<string>,
  hop: number
): Promise<string> {
  if (hop >= MAX_HOPS) throw new Error("Max hops exceeded");
  if (visited.has(agent.id)) throw new Error(`Loop detected: ${agent.id}`);

  visited.add(agent.id);
  const result = await agent.run(message);

  if (result.nextAgent) {
    const next = registry.get(result.nextAgent);
    return swarmStep(next, result.handoff, visited, hop + 1);
  }
  return result.output;
}
```

---

## 実践3：エージェントの信頼性を上げる3つの施策

### 1. 構造化出力の強制

自由テキストの代わりにJSONスキーマを強制する。

```typescript
import { z } from "zod";

const ActionSchema = z.object({
  action: z.enum(["search", "write", "delegate", "done"]),
  tool: z.string().optional(),
  args: z.record(z.unknown()).optional(),
  reasoning: z.string(), // 必ずreasoningを含めさせる（可視化・デバッグ用）
});
```

### 2. チェックポイントと再開

長時間タスクは途中状態をDBに永続化し、失敗時に再開できるようにする。

```typescript
async function runWithCheckpoint(taskId: string, steps: Step[]) {
  const checkpoint = await db.getCheckpoint(taskId);
  const startIdx = checkpoint?.stepIndex ?? 0;

  for (let i = startIdx; i < steps.length; i++) {
    const result = await steps[i].execute();
    await db.saveCheckpoint(taskId, { stepIndex: i + 1, state: result });
  }
}
```

### 3. 観測可能性（Observability）の確保

エージェントの各ステップにスパンを張り、LLMの入出力・ツール呼び出し・レイテンシを記録する。OpenTelemetryベースのトレーシングとの統合が2026年の標準になりつつある。

---

## まとめ

| 観点 | 推奨アプローチ |
|------|---------------|
| ツール統合 | MCPでドメイン別サーバーに分割 |
| エージェント構成 | タスク特性に合わせてOrchestratorかPipelineを選択 |
| 信頼性 | 構造化出力 + チェックポイント + トレーシング |
| コスト管理 | Workerの出力は要約してから上位に渡す |

AIエージェントの設計は、分散システム設計と同じ問題を多く含む。信頼性・観測可能性・障害設計という古典的なバックエンドエンジニアリングの知見が、そのまま活きる領域だ。モデルに振り回されず、アーキテクチャの原則で設計することが、2026年に現場エンジニアに求められるスキルセットといえる。
