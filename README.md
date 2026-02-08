# Timeless

Embedded time series database for Elixir. Combines [Gorilla compression](https://github.com/awksedgreep/gorilla_stream) with SQLite for fast, compact metric storage with automatic rollups and configurable retention.

Run it as a library inside your Elixir app or as a standalone container.

## Features

- **High throughput** — 400K+ points/sec single-writer, 2.5M+ points/sec over HTTP (parallel clients)
- **Compact storage** — Gorilla + zstd compression, ~0.67 bytes/point for real-world data
- **Sharded writes** — parallel buffer/builder shards across CPU cores
- **Automatic rollups** — configurable tiers (hourly, daily, monthly) with retention policies
- **VictoriaMetrics compatible** — JSON line import, works with Vector, Grafana, and existing VM tooling
- **Prometheus compatible** — text exposition import and PromQL-compatible query endpoint for Grafana
- **SVG charts** — pure Elixir chart rendering, embeddable via `<img>` tags with light/dark/auto themes
- **Built-in dashboard** — zero-dependency HTML overview with auto-refresh
- **Annotations** — event markers (deploys, incidents) that overlay on charts
- **Alerts** — threshold-based rules with webhook notifications
- **Metric metadata** — type, unit, and description registration
- **Zero external dependencies** — SQLite + pure Elixir, no Redis/Kafka/Postgres required

## Quick Start

### As a library

Add to your `mix.exs`:

```elixir
{:timeless, github: "awksedgreep/timeless"}
```

Add to your supervision tree:

```elixir
children = [
  {Timeless, name: :metrics, data_dir: "/var/lib/metrics"},
  {Timeless.HTTP, store: :metrics, port: 8428}
]
```

Write and query:

```elixir
Timeless.write(:metrics, "cpu_usage", %{"host" => "web-1"}, 73.2)

{:ok, points} = Timeless.query(:metrics, "cpu_usage", %{"host" => "web-1"},
  from: System.os_time(:second) - 3600)
```

### As a container

```bash
podman build -f Containerfile -t timeless:latest .
podman run -d -p 8428:8428 -v timeless_data:/data:Z localhost/timeless:latest
```

Ingest data:

```bash
curl -X POST http://localhost:8428/api/v1/import -d '
{"metric":{"__name__":"cpu_usage","host":"web-1"},"values":[73.2],"timestamps":[1700000000]}'
```

Query:

```bash
curl 'http://localhost:8428/api/v1/query_range?metric=cpu_usage&from=-1h&step=60'
```

Embed a chart:

```html
<img src="http://localhost:8428/chart?metric=cpu_usage&from=-6h&theme=auto" />
```

View the dashboard at `http://localhost:8428/`.

## HTTP Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/import` | VictoriaMetrics JSON line ingest |
| `POST` | `/api/v1/import/prometheus` | Prometheus text format ingest |
| `GET` | `/api/v1/export` | Export raw points (VM JSON line format) |
| `GET` | `/api/v1/query` | Latest value for matching series |
| `GET` | `/api/v1/query_range` | Range query with bucketed aggregation |
| `GET` | `/prometheus/api/v1/query_range` | Grafana-compatible Prometheus endpoint |
| `GET` | `/api/v1/label/__name__/values` | List all metric names |
| `GET` | `/api/v1/label/:name/values` | List values for a label key |
| `GET` | `/api/v1/series` | List series for a metric |
| `POST` | `/api/v1/metadata` | Register metric metadata |
| `GET` | `/api/v1/metadata` | Get metric metadata |
| `POST` | `/api/v1/annotations` | Create an annotation |
| `GET` | `/api/v1/annotations` | Query annotations |
| `DELETE` | `/api/v1/annotations/:id` | Delete an annotation |
| `POST` | `/api/v1/alerts` | Create an alert rule |
| `GET` | `/api/v1/alerts` | List alert rules |
| `DELETE` | `/api/v1/alerts/:id` | Delete an alert rule |
| `GET` | `/chart` | SVG line chart |
| `GET` | `/health` | Health check with stats |
| `GET` | `/` | HTML dashboard |

See [docs/API.md](docs/API.md) for full request/response documentation with examples.

## Container Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `TIMELESS_DATA_DIR` | `/data` | SQLite storage directory |
| `TIMELESS_PORT` | `8428` | HTTP listen port |
| `TIMELESS_SHARDS` | CPU count | Write buffer shard count |
| `TIMELESS_SEGMENT_DURATION` | `14400` | Raw segment duration (seconds) |

## Podman Quadlet

Copy `timeless.container` to `~/.config/containers/systemd/`:

```bash
cp timeless.container ~/.config/containers/systemd/
systemctl --user daemon-reload
systemctl --user start timeless
```

## Architecture

```
Writes ──> Buffer Shards (ETS) ──> Segment Builders ──> SQLite Shard DBs
                                                              │
                                              Rollup Engine ──┤── hourly tier
                                                              ├── daily tier
                                                              └── monthly tier

Main DB: series registry, metadata, annotations, alerts
Shard DBs: raw_segments, tier tables, watermarks
```

- **Buffer shards** — lock-free ETS tables, one per CPU core, flushed every 5s or 10K points
- **Segment builders** — Gorilla + zstd compression, one per shard for parallel writes
- **Rollup engine** — parallel per-shard aggregation into configurable tiers
- **Retention enforcer** — periodic cleanup of expired raw data and tier rows
- **SQLite** — WAL mode, mmap, WITHOUT ROWID for clustered B-trees, 16KB pages

## Custom Rollup Schema

```elixir
defmodule MyApp.MetricsSchema do
  use Timeless.Schema

  raw_retention {7, :days}

  tier :hourly,
    resolution: :hour,
    aggregates: [:avg, :min, :max, :count, :sum, :last],
    retention: {30, :days}

  tier :daily,
    resolution: :day,
    aggregates: [:avg, :min, :max, :count, :sum, :last],
    retention: {365, :days}

  tier :monthly,
    resolution: {30, :days},
    aggregates: [:avg, :min, :max, :count, :sum, :last],
    retention: :forever
end
```

```elixir
{Timeless, name: :metrics, data_dir: "/data", schema: MyApp.MetricsSchema}
```

## License

MIT
