---
title: "Google AX × Kubernetes | kindクラスタ構築からAgentのTask実行・SSHデバッグまで"
tags:
  - Kubernetes
  - GoogleCloud
  - AIエージェント
  - DevOps
  - AX
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## 「エージェントを100個並列で動かしたら、どこかで必ず1つが暴走する」

——この感覚は正しいです。ローカルで1〜2体のAIエージェントを動かしている分には気づきませんが、並列数が増えた瞬間に「どのエージェントがどのファイルを触ったか」「暴走したプロセスをどう隔離するか」「クラッシュ後に途中から再開できるか」が一気に運用課題になります。個別にDockerコンテナを立てて頑張ってきたチームは多いはずです。

2026年9月18日、GoogleがこれをKubernetesのレイヤーで解決しようとする実験的OSS「AX(Agentic Orchestrator)」を公開しました。GitHub Star数は公開から1週間で1万を超え、9月23日には1日で2,300超のStarが付くほどの勢いです。本記事では実際にリポジトリの手順を追いながら、**ローカルのkindクラスタにAXを構築し、最初のTaskを動かしてSSHで中を覗くところまで**を実装ベースで解説します。

## 背景: なぜ「Kubernetes style」なのか

AXは`google/ax`としてApache 2.0ライセンスで公開されている、宣言型のエージェントオーケストレーターです。前提として「Agent Substrate」というサンドボックス実行ランタイムの上で動作し、以下の4つのプリミティブでエージェントのライフサイクルを表現します。

| プリミティブ | 役割 |
|---|---|
| `Task` | 実行ライフサイクル・CPU/メモリ制約・サンドボックス隔離を定義する最小単位 |
| `Workspace` | Gitリポジトリ・MCPサーバー・スキルパッケージを事前マウントする環境定義 |
| `Model` | LLMプロバイダーとその認証情報をKubernetes Secretとして管理 |
| `Gateway` | アウトバウンド通信を許可リストに限定し、認証情報を注入するネットワーク境界 |

つまり「Podの代わりにTask」「ConfigMap/Secretの代わりにModel」「NetworkPolicyの代わりにGateway」というマッピングで、既存のKubernetesオペレーターに近い語彙のままエージェント基盤を組めるのが売りです。ただし現時点(2026年9月28日)ではまだpre-1.0で、READMEにも「コアコンセプトとプロトコルを積極的に見直し中で、安定版までに破壊的変更がある」と明記されています。本番投入ではなく検証目的で読んでください。

## Step 1: 前提条件を揃える

必要なのは以下の4つです。

- Kubernetesクラスタ(ローカル検証なら`kind`)
- コンテナイメージをpushできるレジストリ(`ko`経由でビルドするため)
- `kubectl` / `go` / `ko`
- Redis(AXコントロールプレーンが状態管理に使用)