---
title: "Claude Agent SDK × TypeScript | 独自ツールを持つ自律エージェントの実装からアクセス制御まで"
tags:
  - ClaudeAgentSDK
  - TypeScript
  - AIエージェント
  - MCP
  - Anthropic
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「Claude Codeは便利だけど、結局あのCLIの中でしか使えないんでしょう?」——そう思っていませんか。この感覚は半分正しく、半分もう古いです。Anthropicは2025年後半、Claude Codeを動かしているのと同じエージェントループを `@anthropic-ai/claude-agent-sdk` としてTypeScript/Pythonに公開しました。自分のNode.jsアプリの中に「独自ツールを呼べる自律エージェント」を、外部プロセスもHTTPサーバーも立てずに数十行で組み込めます。

自社プロダクトのAIエージェント基盤でも、社内API専用のツールだけをエージェントに持たせて動かす構成を使っています。この記事では公式ドキュメントで示されているAPIをベースに、セットアップからカスタムツール実装、権限制御までを手を動かして追います。

## Step 1: セットアップ