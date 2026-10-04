---
title: "Claude Code × OpenTelemetry | メトリクス収集からコスト可視化・アラートまで"
tags:
  - ClaudeCode
  - OpenTelemetry
  - Prometheus
  - Docker
  - 可観測性
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

「今月のClaude Codeの使用量、結局どのリポジトリで、どのモデルが、いくら使ったのか分からない」
「`/cost` は自分のセッションしか見えないのでは?」

——この感覚は正しいです。`/cost` は手元のセッションの数字で、複数人・複数リポジトリ・複数日を横断して見る用途には向きません。Claude Code は OpenTelemetry (OTel) でメトリクスとイベントを出せるので、これを自前のコレクターに流せば横断集計できます。

この記事では、Docker Compose で OTel Collector と Prometheus を立て、トークン・コスト・キャッシュ利用を見て、しきい値アラートを書くところまでを順に進めます。メトリクス名・環境変数は公式ドキュメント（monitoring-usage、2026-10-04 時点）を確認して書いています。

## 全体像

```
Claude Code ──OTLP/gRPC──▶ OTel Collector ──▶ Prometheus(:8889をscrape)
                                    └──▶ debug(標準出力でイベント確認)
```

Claude Code が出すのは次の2系統です。

| 系統 | 例 | 用途 |
|---|---|---|
| メトリクス | `claude_code.cost.usage`, `claude_code.token.usage` | 集計・アラート |
| イベント(ログ) | `claude_code.api_request`, `claude_code.tool_result` | 個別の呼び出しの調査 |

## Step 1 コレクターと Prometheus を起動する

`latest` は避けてタグを固定します。

```yaml
# docker-compose.yml
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.111.0
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "127.0.0.1:4317:4317"   # OTLP gRPC（ローカルだけに公開）
  prometheus:
    image: prom/prometheus:v2.55.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./alerts.yml:/etc/prometheus/alerts.yml:ro
    ports:
      - "127.0.0.1:9090:9090"
```

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889
  debug:
    verbosity: basic
service:
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      exporters: [debug]
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
rule_files:
  - /etc/prometheus/alerts.yml
scrape_configs:
  - job_name: claude-code
    static_configs:
      - targets: ["otel-collector:8889"]
```

```bash
docker compose up -d
```

**防止策**: ポートは `127.0.0.1` に束縛します。認証のないコレクターをそのまま公開すると、誰でもメトリクスを書き込めます。

## Step 2 Claude Code 側で有効化する

最小構成は環境変数だけです。まず手元で試します。

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
# 既定は60秒。動作確認中だけ短くする
export OTEL_METRIC_EXPORT_INTERVAL=10000
claude
```

常用するなら `settings.json` の `env` に書きます。チームに配るときは managed settings に置くと、個人が外せません。

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
    "OTEL_RESOURCE_ATTRIBUTES": "team.id=platform,department=engineering"
  }
}
```

`OTEL_RESOURCE_ATTRIBUTES` に入れた値は全メトリクスに付くので、後でチーム別に `sum by` できます。

数プロンプト投げたら、コレクターの出力を確認します。

```bash
curl -s localhost:8889/metrics | grep claude_code
docker compose logs otel-collector | grep -i api_request
```

**防止策**: Prometheus 上の名前は変換されます。`claude_code.cost.usage`（単位 USD）は、手元の環境では `claude_code_cost_usage_USD_total` のような形になりました。クエリを書く前に必ず `/metrics` で実名を確認してください。以降のクエリは、この名前で書きます。

## Step 3 コストとキャッシュを見るクエリ

`cost.usage` と `token.usage` には `type`（`input` / `output` / `cacheRead` / `cacheCreation`）属性が付きます。

```promql
# 直近24時間のコスト合計(USD)
sum(increase(claude_code_cost_usage_USD_total[24h]))

# モデル別
sum by (model) (increase(claude_code_cost_usage_USD_total[24h]))

# キャッシュ読み出しの比率(入力系トークンのうち何割がキャッシュ読みか)
sum(increase(claude_code_token_usage_tokens_total{type="cacheRead"}[24h]))
/
sum(increase(claude_code_token_usage_tokens_total{type=~"input|cacheRead|cacheCreation"}[24h]))

# 1時間あたりの編集の受け入れ率
sum(increase(claude_code_code_edit_tool_decision_total{decision="accept"}[1h]))
/
sum(increase(claude_code_code_edit_tool_decision_total[1h]))
```

キャッシュ比率が極端に低い場合は、CLAUDE.md や MCP ツール定義が頻繁に変わってプレフィックスが毎回崩れていないかを疑う手がかりになります。

**防止策**: `session.id` や `user.account_uuid` は既定で属性に入ります。人数やセッション数が増えるとシリーズ数が膨らむので、不要なら `OTEL_METRICS_INCLUDE_SESSION_ID=false` で落とします。

## Step 4 しきい値アラートを書く

暴走ループ（無限リトライやサブエージェントの増殖）は、コストの傾きに最初に現れます。

```yaml
# alerts.yml
groups:
  - name: claude-code
    rules:
      - alert: ClaudeCodeCostSpike
        expr: sum(increase(claude_code_cost_usage_USD_total[1h])) > 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "直近1時間のClaude Codeコストが20USDを超えました"
      - alert: ClaudeCodeEditRejectHigh
        expr: |
          sum(increase(claude_code_code_edit_tool_decision_total{decision="reject"}[1h]))
          /
          sum(increase(claude_code_code_edit_tool_decision_total[1h])) > 0.5
        for: 30m
        labels:
          severity: info
```

20USD は例です。まず1週間そのまま観測し、自分たちの通常時の p95 を見てから決めてください。

**防止策**: アラートは「止める」ものではなく「気づく」ものです。強制停止が必要なら、API 側の支出上限と組み合わせます。

## Step 5 イベントで原因を掘る(プライバシーに注意)

メトリクスで異常に気づいたら、イベントで個別の `api_request` を見ます。ここで1つ落とし穴があります。

```bash
# 付けるとプロンプト本文がログに載る。既定ではオフ
export OTEL_LOG_USER_PROMPTS=1
export OTEL_LOG_TOOL_DETAILS=1
```

`OTEL_LOG_USER_PROMPTS` や `OTEL_LOG_TOOL_DETAILS` を有効にすると、プロンプトやツール引数（コマンド文字列など）が送信先に残ります。ここにシークレットが入る可能性があります。社内収集であっても、既定のオフのまま始め、調査が必要な期間だけ限定的にオンにするのが安全です。

**防止策**: コレクターの `debug` エクスポーターは標準出力に出すだけです。本番ではログ送信先のアクセス制御と保持期間を決めてから、プロンプト系のフラグを開けてください。

## まとめ

- 有効化に必要なのは `CLAUDE_CODE_ENABLE_TELEMETRY=1` と OTLP のエクスポーター指定だけ。
- メトリクスはコスト・トークン・編集の受け入れを、イベントは個別 API 呼び出しを見る。
- Prometheus 側のメトリクス名は変換されるため、クエリを書く前に `/metrics` で実名を確認する。
- プロンプト・ツール詳細のログは既定オフ。開けるなら期間と送信先を限定する。

次の一手としては、Grafana でチーム別ダッシュボードを作る、`OTEL_RESOURCE_ATTRIBUTES` にリポジトリ名を入れて案件別に原価を出す、といったところが手堅いです。