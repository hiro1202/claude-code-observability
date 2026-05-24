# Changelog

All notable changes to this project will be documented in this file.

## [0.1.0] - 2026-05-24

### Overview

初期リリース。`docker compose up -d` 一発で立ち上がる Claude Code の OpenTelemetry
ローカル検証スタック。Prometheus + Loki + Tempo + Grafana + OTel Collector の
5サービス構成。

### What works

- **メトリクス収集**: Claude Code が `OTEL_METRICS_EXPORTER=otlp` で送る
  `claude_code_active_time_seconds_total`, `claude_code_cost_usage_USD_total`,
  `claude_code_session_count_total`, `claude_code_token_usage_tokens_total`
  が Prometheus に蓄積される。
- **イベント収集**: `OTEL_LOGS_EXPORTER=otlp` で `user_prompt`, `tool_result`,
  `api_request`, `mcp_server_connection`, `hook_*` イベントが Loki に蓄積される。
- **可視化**: provisioning 済み Grafana ダッシュボード `claude-code` で 9 パネル
  (累積コスト、累積トークン、セッション数、active time、モデル別コスト時系列、
  トークンタイプ別時系列、query_source 別バーチャート、セッション別テーブル、
  Loki 生イベント)を表示。
- **匿名 viewer 有効**: ログインなしでダッシュボード閲覧可能。
  編集には admin/admin。

### Added

- `docker-compose.yml`: 5サービス。image 全 pin、Tempo は UID 10001 で起動、
  Prometheus と Grafana は in-container healthcheck (`start_period: 30s`)。
  Loki / Tempo / OTel Collector は distroless image のため healthcheck 省略、
  README に host から curl で確認する手順記載。
- `config/otel-collector.yaml`: 単一 OTLP entry → fan-out。`deltatocumulative`
  processor で Claude Code の Delta metrics を Prometheus 互換の Cumulative
  に変換。debug exporter も並走で生 OTLP を観察可能。
- `config/prometheus.yml`: OTLP receiver は CLI フラグ `--web.enable-otlp-receiver`
  で有効化。Prometheus 3.x 系。
- `config/loki-config.yaml`: シングルバイナリ + OTLP receiver、`tsdb_shipper`、
  `compactor.working_directory`、`limits_config.allow_structured_metadata: true`、
  `analytics.reporting_enabled: false`。
- `config/tempo-config.yaml`: シングルバイナリ + OTLP receiver (gRPC/HTTP)、
  local FS storage。
- `config/grafana/provisioning/datasources/datasources.yml`: Prometheus / Loki /
  Tempo を auto-register。trace → log 連携設定済み。
- `config/grafana/provisioning/dashboards/dashboards.yml`: dashboard auto-load
  provider 設定。
- `config/grafana/dashboards/claude-code.json`: 初期ダッシュボード (9 panels)。
- `README.md`: 手順書。Quick start、Architecture、env var(基本セット + 詳細
  観察用)、host からの ready 確認、Troubleshooting、Stopping/Cleanup。
- `.gitignore`: `.claude/`(個人設定)、`docker-compose.override.yml`、`.env` 等。

### Fixed (during QA)

- Cost by Query Source / Session 別の使用量 パネルが "No data" になる問題:
  Prometheus instant query の 5min lookback 制約により Claude Code 停止後に
  サンプルが見えなくなっていた。`last_over_time(metric[1h])` で1時間ウィンドウ
  に変更。
- README の env var セットアップが不完全だった問題: `OTEL_METRICS_EXPORTER`
  と `OTEL_LOGS_EXPORTER` が必須であること、`OTEL_LOG_USER_PROMPTS=1` で
  プロンプト本文を、`OTEL_LOG_TOOL_DETAILS=1` でツール入力本体(Bash の
  実コマンド等)を Loki に記録できることを明記。LogQL 監査クエリの例も追加。

### Known scope (intentional)

- **トレースは空のまま**: Claude Code v2.x は metrics と events のみ emission
  し、distributed traces は出さない。Tempo は学習目的で同梱するが、stack に
  接続されていれば常に空。
- **Logs panel が初期ビューに見えない**: ダッシュボード上部に panel が多く、
  Loki logs panel は scroll しないと到達できない。Grafana 11 の lazy-rendering
  挙動。気になる場合は dashboard JSON で順序入れ替え可能。
- **本番運用考慮なし**: 認証なし、TLS なし、HA なし、retention 設計なし。
  学習・検証専用。
