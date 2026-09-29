---
title: Google AX × Kubernetes | kindクラスタ構築からAgentのTask実行・SSHデバッグまで
tags:
  - kubernetes
  - GoogleCloud
  - AIエージェント
  - devops
  - AX
private: false
updated_at: '2026-09-29T17:21:52+09:00'
id: 4450f33aba56765c8acf
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## 「エージェントを100個並列で動かしたら、どこかで必ず1つが暴走する」

この感覚は正しいです。ローカルで1〜2体のAIエージェントを動かしている分には気づきませんが、並列数が増えると「どのエージェントがどのファイルを触ったか」「暴走したプロセスをどう隔離するか」「クラッシュ後に途中から再開できるか」が運用課題になります。個別にDockerコンテナを立てて頑張ってきたチームは多いはずです。

GoogleがこれをKubernetesのレイヤーで解決しようとするOSS「AX」(`google/ax`)を公開しました。本記事ではREADMEの手順を追い、**ローカルのkindクラスタにAX(と土台のAgent Substrate)を構築し、最初のTaskを動かしてSSHで中を覗くところまで**をまとめます。

## 背景: なぜ「Kubernetes style」なのか

AXはApache 2.0の宣言型エージェントオーケストレーターです。サンドボックス実行は [Agent Substrate](https://github.com/agent-substrate/substrate) に任せ、AX自身は `ax.io/v1alpha1` のマニフェストでエージェントを宣言します。READMEが挙げるプリミティブは次の3つです。

| リソース | 役割 |
|---|---|
| `Task` | 隔離された最小実行単位。イメージ、CPU/メモリの requests/limits、環境変数、Workspaceへの参照を持つ |
| `Workspace` | Gitリポジトリ、MCPサーバー、スキルを事前に用意する環境定義 |
| `Model` | LLMプロバイダーとモデル名。認証情報はKubernetes Secretから参照する |

操作は `kubectl` に似せた `ax` CLI(`apply` / `get` / `describe` / `watch` / `delete`)で行い、エージェント固有の動詞として `suspend` / `resume` / `ssh` があります。

注意点が2つあります。まず、公式リポジトリはREADME冒頭で「コアコンセプトとプロトコルは開発中で、安定版までに大きな破壊的変更が入る可能性が高い」と警告しています。次に、一部の解説記事にはアウトバウンド通信を制限する `Gateway` を4つ目のプリミティブとして紹介するものがありますが、私が確認したREADMEとconcepts.mdには記載がありませんでした。本記事では扱いません。検証目的で読んでください。

## Step 1: 前提条件を揃える

READMEの前提は次のとおりです。

- Agent Substrateが入ったKubernetesクラスタ
- Go と `kubectl`
- [`ko`](https://ko.build/) と、クラスタから pull できるコンテナレジストリ

Redisは前提ではありません。後述の `make deploy` がAXのコントロールプレーンと一緒にデプロイします。

## Step 2: kindクラスタにAgent Substrateを入れる

AXのREADMEはSubstrateのインストールを「Substrate側のREADMEに従う」としています。そのSubstrateのDevelopment Quickstartが、kindを使う手順です。`kind` 自体はGoの依存として自動管理されるため、手元には Go、`kubectl`、`docker` があれば足ります。

```bash
git clone https://github.com/agent-substrate/substrate.git
cd substrate

# クラスタとローカルレジストリを作成
hack/create-kind-cluster.sh

# Substrate(ate-system)、PostgreSQL、rustfs を導入
hack/install-ate-kind.sh --deploy-ate-system
```

AXは Substrate の Control API を `api.ate-system.svc.cluster.local:443` で探します。先に起動を確認します。

```bash
kubectl get svc api -n ate-system
```

## Step 3: AXのCLIとコントロールプレーンを入れる

```bash
go install github.com/google/ax/cmd/ax@latest
export PATH="$(go env GOPATH)/bin:$PATH"

git clone https://github.com/google/ax.git
cd ax
make deploy AX_IMAGE_REPO=<クラスタから pull できるレジストリ>
```

`make deploy` は `deploy/redis.yaml` を適用し、`ko apply` でコントロールプレーンのイメージをビルドして `ax-system` 名前空間に配置します。`AX_IMAGE_REPO` には、Step 2で作られたローカルレジストリのアドレスを指定します。値はスクリプトの出力で確認してください。私は具体的なアドレスを確認できていないため、ここでは書きません。

## Step 4: 最初のTaskを動かす

`examples/task.yaml` は Task、Workspace、Model を1ファイルにまとめた例です(抜粋)。

```yaml
apiVersion: ax.io/v1alpha1
kind: Task
metadata:
  name: task123
  atespace: default
spec:
  env:
    - name: ENVIRONMENT
      value: "test"
  image: "gcr.io/ax-substrate/ate-images/ax-task-runner@sha256:..."
  resources:
    requests: { cpu: "500m", memory: "1Gi" }
    limits:   { cpu: "2",    memory: "4Gi" }
  workspaces:
    - name: default-workspace
      path: "/workspace"
  debug: true # `ax ssh` を使うために必要。既定は off
---
apiVersion: ax.io/v1alpha1
kind: Workspace
metadata:
  name: default-workspace
spec:
  git:
    - name: origin
      repo: "https://github.com/chalk/chalk.git"
      branch: "main"
---
apiVersion: ax.io/v1alpha1
kind: Model
metadata:
  name: default-model
spec:
  provider: google
  model: gemini-3.8-flash
  secretKey:
    name: gemini-api-secret
    key: GEMINI_API_KEY
```

`Model` は `gemini-api-secret` というSecretを参照するので、事前に作っておきます。キーは環境変数に入れておき、コマンドラインに直接書かないようにします。Secretを置く名前空間はmanifests.mdで確認してください。

```bash
kubectl create secret generic gemini-api-secret \
  --from-literal=GEMINI_API_KEY="$GEMINI_API_KEY"
```

適用して状態を見ます。

```bash
ax apply -f examples/task.yaml
ax get tasks
ax watch task task123
```

`ax watch` は phase(`Running` / `Suspended` / `Failed` / `Terminating` など)とconditionの変化を流します。待つべきは `Ready` です。これは「タスクが動いていて、かつ全Workspaceの `WorkspaceReady` が True」のときに真になります。例のイメージは `gcr.io` 上のものなので、kind環境から pull できない場合は `make build-task-runner` / `make push-task-runner` で自前ビルドして `spec.image` を差し替えます。

## Step 5: SSHで中を覗く

`spec.debug: true` のTaskには `ax ssh` が使えます。

```bash
# 単発コマンド
ax ssh task123 -- ls -la /workspace

# 対話シェル
ax ssh task123
```

エージェントが実際にリポジトリをcloneできているか、`goal` 付きのWorkspaceでツールチェーンが入ったか、といった確認に使えます。本番相当の環境では `debug` を外すのが前提でしょう。

## Step 6: suspend / resume と後片付け

```bash
ax suspend task task123   # 状態をチェックポイントして停止
ax resume task task123    # 続きから再開
ax delete task task123    # サンドボックスを破棄(完了まで待つ)
```

suspendすると `Ready` は False(理由 `TaskSuspended`)になり、resumeで戻ります。「クラッシュや暴走のあとに途中から再開する」という冒頭の課題に、コマンド1つで触れられる部分です。Substrate側は、サブ秒での再開と、RAMとファイルシステムの状態保存を売りにしています。ただし、手元のkindで同じ速度が出るかは私は測っていません。

複数クラスタを使う場合、`ax` は `kubectx` の現在のコンテキストに従います。`ax --context=dev-cluster get tasks` のように指定もできます。

## まとめ

kindでの構築は「Substrateを入れる → `make deploy` → `ax apply`」の3段です。関門はAX本体よりも、Substrateの導入とレジストリ・イメージの取り回しでした。使ってみて、`Task` を「Podに近いが、止めて再開できる単位」と捉えると理解が早いと感じました。

一方で、両プロジェクトとも pre-1.0 で、APIは変わり得ます。まずは `ax ssh` で中を見ながら、1つのTaskの挙動を確かめるところから始めるのが安全です。

## 参考

- [google/ax](https://github.com/google/ax)(README、examples/task.yaml、docs/concepts.md、Makefile)
- [agent-substrate/substrate](https://github.com/agent-substrate/substrate)(Quickstart)
