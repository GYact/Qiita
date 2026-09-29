---
title: Claude Agent SDK × TypeScript | 独自ツールを持つ自律エージェントの実装からアクセス制御まで
tags:
  - ClaudeAgentSDK
  - TypeScript
  - AIエージェント
  - MCP
  - Anthropic
private: false
updated_at: '2026-09-29T17:39:15+09:00'
id: 1f382fdd1df2b2cb2cc8
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

「Claude Codeは便利だけど、結局あのCLIの中でしか使えないんでしょう?」——そう思っていませんか。この感覚は半分正しく、半分もう古いです。Anthropicは2025年後半、Claude Codeを動かしているのと同じエージェントループを `@anthropic-ai/claude-agent-sdk` としてTypeScript/Pythonに公開しました。自分のNode.jsアプリの中に「独自ツールを呼べる自律エージェント」を、外部プロセスもHTTPサーバーも立てずに数十行で組み込めます。

自社プロダクトのAIエージェント基盤でも、社内API専用のツールだけをエージェントに持たせて動かす構成を使っています。この記事では公式ドキュメントで示されているAPIをベースに、セットアップからカスタムツール実装、権限制御までを手を動かして追います。

## Step 1: セットアップ

```bash
npm install @anthropic-ai/claude-agent-sdk zod
```

ツールの入力スキーマはTypeScriptでは常にZodで書くので、`zod` も一緒に入れます。以降のコードは `npx tsx agent.ts` で実行する想定です。

題材は「注文照会と返金を行う社内サポート用エージェント」にします。読み取りだけのツールと、副作用のあるツールを分けておくと、後半のアクセス制御の話が見えやすくなります。

## Step 2: カスタムツールを定義する

ツールは「名前・説明・入力スキーマ・ハンドラ」の4点で定義し、`tool()` で作ります。それを `createSdkMcpServer` に包むと、アプリと同じプロセス内で動くMCPサーバーになります。

```typescript
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";

const API = process.env.INTERNAL_API_BASE ?? "http://localhost:8080";

const getOrder = tool(
  "get_order",
  "注文IDから注文の状態と金額を取得する",
  { orderId: z.string().describe("注文ID") },
  async (args) => {
    try {
      const res = await fetch(`${API}/orders/${encodeURIComponent(args.orderId)}`);
      if (!res.ok) {
        return {
          content: [{ type: "text", text: `注文API error: ${res.status} ${res.statusText}` }],
          isError: true
        };
      }
      return { content: [{ type: "text", text: JSON.stringify(await res.json()) }] };
    } catch (e) {
      return {
        content: [{ type: "text", text: `注文APIに接続できません: ${e instanceof Error ? e.message : String(e)}` }],
        isError: true
      };
    }
  },
  { annotations: { readOnlyHint: true } }
);

const issueRefund = tool(
  "issue_refund",
  "注文を返金する。副作用あり",
  {
    orderId: z.string(),
    amount: z.number().int().positive().describe("返金額(円)")
  },
  async (args) => {
    const res = await fetch(`${API}/orders/${encodeURIComponent(args.orderId)}/refund`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ amount: args.amount })
    });
    return {
      content: [{ type: "text", text: res.ok ? "返金を受け付けました" : `返金失敗: ${res.status}` }],
      isError: !res.ok
    };
  }
);

export const internalServer = createSdkMcpServer({
  name: "internal",
  version: "1.0.0",
  tools: [getOrder, issueRefund]
});
```

ポイントは3つです。

- ハンドラは `content` 配列を返す。失敗時は `isError: true` を付けて、Claudeが読むメッセージを自分で組み立てる。例外を投げてもエージェントループは止まらないが、Claudeにはそのままの例外文字列が渡る
- `readOnlyHint: true` は副作用のないツールの目印で、他の読み取り専用ツールと並列に呼べるようになる。ただしあくまでメタデータで、強制力はない
- 副作用のある `issue_refund` には付けない。付けるかどうかが、そのまま次のアクセス制御の分け方になる

## Step 3: エージェントを動かす

サーバーを `mcpServers` に渡します。ここで使ったキー(`internal`)が、ツールの完全修飾名 `mcp__internal__<ツール名>` の一部になります。

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";
import { internalServer } from "./tools";

for await (const message of query({
  prompt: "注文 A-1042 の状態を教えて",
  options: {
    mcpServers: { internal: internalServer },
    tools: [], // 組み込みツール(Bash/Read/Write等)を全部外す
    allowedTools: ["mcp__internal__get_order"],
    maxTurns: 5
  }
})) {
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log(message.result);
      console.log(`cost: $${message.total_cost_usd}`);
    } else {
      console.error(`失敗: ${message.subtype}`); // error_max_turns など
    }
  }
}
```

`tools: []` にすると、組み込みツールがClaudeのコンテキストから消え、使えるのは自分のMCPツールだけになります。社内API専用エージェントなら、まずここから始めるのが安全です。`maxTurns` で往復回数に上限を付けておけば、暴走ループも抑えられます。結果メッセージの `subtype` は `success` のほか `error_max_turns` などがあるので、失敗系も分岐しておきましょう。

## Step 4: アクセス制御を重ねる

ここからが本題です。公式ドキュメントによれば、ツール呼び出しは次の順で評価されます。

1. Hooks(`PreToolUse` など)
2. deny ルール(`disallowedTools`)
3. ask ルール
4. 権限モード
5. allow ルール(`allowedTools`)
6. `canUseTool` コールバック

読み取りは `allowedTools` で自動承認済みなので、返金だけ2段構えにします。

### 金額の上限は Hook で「必ず」止める

```typescript
import type { HookCallback, PreToolUseHookInput } from "@anthropic-ai/claude-agent-sdk";

const refundCap: HookCallback = async (input) => {
  const pre = input as PreToolUseHookInput;
  const { amount } = pre.tool_input as { amount?: number };
  if ((amount ?? 0) > 50_000) {
    return {
      hookSpecificOutput: {
        hookEventName: pre.hook_event_name,
        permissionDecision: "deny",
        permissionDecisionReason: "5万円超の返金はエージェントでは実行できません"
      }
    };
  }
  return {};
};
```

### 人の承認は canUseTool で挟む

`issue_refund` は `allowedTools` に入れていないので、上のどのステップにも引っかからず、最後の `canUseTool` に回ってきます。

```typescript
for await (const message of query({
  prompt: "注文 A-1042 を全額返金して",
  options: {
    mcpServers: { internal: internalServer },
    tools: [],
    allowedTools: ["mcp__internal__get_order"],
    hooks: {
      PreToolUse: [{ matcher: "mcp__internal__issue_refund", hooks: [refundCap] }]
    },
    canUseTool: async (toolName, input) => {
      const ok = await askOperator(`${toolName} ${JSON.stringify(input)} を実行しますか?`);
      return ok
        ? { behavior: "allow", updatedInput: input }
        : { behavior: "deny", message: "担当者が却下しました" };
    }
  }
})) {
  if (message.type === "result" && message.subtype === "success") console.log(message.result);
}
```

`askOperator` はSlackでもWeb UIでも、承認を取る自前の処理に差し替えてください。`deny` の `message` はClaudeに渡るので、却下理由を書いておくと別の手段を提案してくれます。

## ハマりどころ

公式ドキュメントに明記されている、事故につながりやすい点です。

- **`allowedTools` に入れたツールは `canUseTool` を通らない。** ツール名をそのまま書くと全呼び出しが自動承認される。毎回必ず検査したいものは Hook に置く
- **`allowedTools` は `bypassPermissions` の制限にならない。** `allowedTools: ["Read"]` と `permissionMode: "bypassPermissions"` を併用しても、`Bash` を含む全ツールが承認される。止めたいものは `disallowedTools` に書く
- **`disallowedTools` は書き方で効き方が変わる。** `"Bash"` のような素の名前はツールごとコンテキストから消え、`"Bash(rm *)"` のようなスコープ付きは呼び出しだけを拒否する
- **承認を出さずに拒否だけしたいなら `permissionMode: "dontAsk"`。** ヘッドレス運用で `allowedTools` と組み合わせると、リストにない呼び出しは `canUseTool` を呼ばずに拒否される

## まとめ

`tool()` と `createSdkMcpServer` で社内APIをツール化し、`tools: []` で組み込みツールを外し、読み取りは `allowedTools`、書き込みは Hook と `canUseTool` で段階的に締める、という構成を追いました。全呼び出しに効かせたい制約は Hook、人の判断が要るものは `canUseTool`、と役割を分けておけば、あとからツールを足しても制御の穴が増えにくくなります。

なお、本記事のAPI名・オプション名は公式ドキュメント(code.claude.com の Agent SDK 各ページ)の記載に基づいています。SDKは更新が速いので、導入時にはバージョンとあわせてリファレンスも確認してください。
