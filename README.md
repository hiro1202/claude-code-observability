# claude-code-observability

Claude Code の OpenTelemetry export を**ローカルで簡単に試す**ための docker-compose スタック。

`docker compose up -d` 一発で OTel Collector + Prometheus + Loki + Tempo + Grafana が立ち上がり、Claude Code (`CLAUDE_CODE_ENABLE_TELEMETRY=1`) を指せばメトリクス・ログ・トレースがローカルで蓄積され Grafana で観察できる。

学習・検証用。本番運用は想定していない。

---

## TL;DR

```bash
# 1. スタックを起動
docker compose up -d

# 2. Claude Code に環境変数をセットして起動
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_SERVICE_NAME=claude-code
claude  # 何か叩く

# 3. Collector ログで生 OTLP データが流れるのを観察(最初の whoa)
docker compose logs -f otel-collector

# 4. Grafana を開く(anonymous viewer 有効、ログイン不要)
open http://localhost:3000
```

---

## Architecture

```
[Claude Code (host)]
      │ OTLP/gRPC :4317  or  OTLP/HTTP :4318
      ▼
[otel-collector-contrib]  ←── 単一 OTLP entry。シグナル別に exporter で振り分け
      ├─ metrics → [prometheus]   otlphttp → /api/v1/otlp/v1/metrics
      ├─ logs    → [loki]         otlphttp → /otlp/v1/logs
      └─ traces  → [tempo]        otlp/grpc → tempo:4317
                          │
                          ▼
                     [grafana]    ← 3つの datasource を provisioning 済み
```

| Container | Image | 役割 | 公開ポート |
|---|---|---|---|
| `cco-otel-collector` | `otel/opentelemetry-collector-contrib:0.152.1` | OTLP 受け口・ルーター | 4317 (gRPC), 4318 (HTTP), 13133 (health) |
| `cco-prometheus` | `prom/prometheus:v3.11.3` | メトリクス保管 | 9090 |
| `cco-loki` | `grafana/loki:3.2.0` | ログ保管 | 3100 |
| `cco-tempo` | `grafana/tempo:2.6.0` | トレース保管 | 3200 |
| `cco-grafana` | `grafana/grafana:11.3.0` | 可視化 UI | 3000 |

---

## Configuration

### 環境変数(Claude Code 側)

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_SERVICE_NAME=claude-code
```

ログ・プロンプト本文の OTel 送信については、現行 Claude Code の docs を確認のうえ追加。`OTEL_LOG_USER_PROMPTS` 等の env var や `settings.json` 側の設定が関わる可能性がある。

### Grafana

- URL: http://localhost:3000
- Anonymous viewer 有効(ログイン不要で閲覧可能)
- 編集が必要な場合: `admin` / `admin` でログイン

### データソース(自動登録済み)

- **Prometheus** (`http://prometheus:9090`) — default
- **Loki** (`http://loki:3100`)
- **Tempo** (`http://tempo:3200`) — trace → log 連携設定済み

---

## 動作確認

### コンテナの状態確認

```bash
docker compose ps
```

`cco-prometheus` と `cco-grafana` は `healthy` 表示になる。
`cco-loki` / `cco-tempo` / `cco-otel-collector` は distroless image で in-container healthcheck が書けないため、host から ready を確認する:

```bash
# Prometheus (健康)
curl http://localhost:9090/-/ready

# Grafana (健康)
curl http://localhost:3000/api/health

# Loki (host から確認)
curl http://localhost:3100/ready

# Tempo (host から確認)
curl http://localhost:3200/ready

# OTel Collector (health_check extension)
curl http://localhost:13133/
```

### 生 OTLP データの観察(Step 6 = 最初の whoa)

```bash
docker compose logs -f otel-collector
```

Claude Code を起動して何か叩くと、Collector のログに `debug exporter` が以下のような出力を吐く:

```
ResourceMetrics #0
...
Metric #0
  Name: claude_code.token.usage
  ...
```

ここで実際に流れるメトリクス名・属性 label を**メモする**。次のステップで Grafana ダッシュボードを作るときに使う。

### Loki / Prometheus / Tempo を Grafana で覗く

http://localhost:3000/explore を開いて:

- **Loki** に切り替え → `{service_name="claude-code"}` で events を見る
- **Prometheus** に切り替え → `{__name__=~"claude.+"}` でメトリクス名一覧
- **Tempo** に切り替え → 検索でトレースを探す

---

## Troubleshooting

### Grafana にデータが見えない

1. `docker compose ps` で全コンテナが running か確認
2. `docker compose logs otel-collector` で OTLP が届いているか確認
3. Claude Code 側の env var が正しくセットされているか:
   ```bash
   env | grep -E 'CLAUDE_CODE|OTEL_'
   ```
4. ファイアウォール・VPN が 4317/4318 を塞いでないか

### Claude Code をコンテナ内から動かす場合

`localhost` ではなく `host.docker.internal` を使う:

```bash
# Docker Desktop (macOS/Windows)
export OTEL_EXPORTER_OTLP_ENDPOINT=http://host.docker.internal:4317

# Linux ホスト(docker run 時)
docker run --add-host=host.docker.internal:host-gateway ...
```

### Tempo が起動に失敗する (permission denied)

`tempo-data` ボリュームの所有権問題。`docker-compose.yml` で `user: "10001:10001"` を指定済みだが、既存ボリュームが root 所有で残っている場合は一度削除:

```bash
docker compose down -v
docker compose up -d
```

### Loki が起動に失敗する

ログを確認:

```bash
docker compose logs loki
```

`tsdb_shipper` 関連のエラーが出る場合は `loki-data` ボリュームをリセット:

```bash
docker compose down
docker volume rm claude-code-observability_loki-data
docker compose up -d
```

---

## Stopping / Cleanup

```bash
# 停止のみ(データは残る)
docker compose down

# 停止 + データも完全削除(クリーンに戻す)
docker compose down -v
```

---

## What's next

このリポジトリは **Phase 1**(生 OTLP を観察)を確実に動かす最小構成。次の段階で Step 7 として:

- 観察した実際のメトリクス名で Grafana 初期ダッシュボードを作る
- `config/grafana/provisioning/dashboards/dashboards.yml` と `config/grafana/dashboards/claude-code.json` を追加
- README に観察ログ(実際に流れたメトリクス名・サンプル値)を追記

を行う。

設計ドキュメント: `~/.gstack/projects/hiro1202-claude-code-observability/funakihirokazu-main-design-20260524-190811.md`

---

## License

MIT — see `LICENSE`.
