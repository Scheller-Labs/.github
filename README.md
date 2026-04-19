# Market Data Service (MDS)

[![CI](https://github.com/Scheller-Labs/market-data-service/actions/workflows/ci.yml/badge.svg)](https://github.com/Scheller-Labs/market-data-service/actions/workflows/ci.yml)
[![Coverage](https://img.shields.io/codecov/c/github/Scheller-Labs/market-data-service)](https://codecov.io/gh/Scheller-Labs/market-data-service)
[![Security](https://github.com/Scheller-Labs/market-data-service/actions/workflows/security.yml/badge.svg)](https://github.com/Scheller-Labs/market-data-service/actions/workflows/security.yml)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)

**Production-grade market data infrastructure for algorithmic trading systems.**

MDS is a unified market data ingestion, caching, and storage service that normalizes data from multiple providers behind a single API. Built to serve as shared infrastructure for multi-agent trading architectures.

## Key Features

- **Multi-provider ingestion** — Alpha Vantage, Finnhub, Databento, TastyTrade with provider-agnostic adapter pattern
- **Three-tier data architecture** — Redis hot cache → TimescaleDB hypertable storage → MinIO/Parquet cold archive
- **Token bucket rate limiting** — per-provider rate limit enforcement with burst handling
- **Gap detection & repair** — SQLite coverage manifest tracks data completeness, auto-triggers backfill
- **Typer CLI** — full command-line interface for data operations, diagnostics, and administration
- **Real-time & historical** — WebSocket streaming and REST-based historical data through unified interfaces

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Consumers                          │
│         (Trading Agents, Strategy Backtester)            │
├─────────────────────────────────────────────────────────┤
│                     MDS API Layer                        │
│              FastAPI / gRPC (planned)                    │
├──────────┬──────────────────────┬────────────────────────┤
│  Redis   │    TimescaleDB       │   MinIO / Parquet      │
│  (L1)    │    (L2)              │   (L3 Cold Archive)    │
│  Hot     │    Hypertable        │   Historical           │
│  Cache   │    Storage           │   Bulk Storage         │
├──────────┴──────────────────────┴────────────────────────┤
│               Provider Adapter Layer                     │
│  ┌────────────┐ ┌────────┐ ┌──────────┐ ┌────────────┐  │
│  │AlphaVantage│ │Finnhub │ │Databento │ │TastyTrade  │  │
│  └────────────┘ └────────┘ └──────────┘ └────────────┘  │
│           Token Bucket Rate Limiter (per provider)       │
├─────────────────────────────────────────────────────────┤
│             Coverage Manifest (SQLite)                   │
│          Gap Detection ─► Auto Backfill                  │
└─────────────────────────────────────────────────────────┘
```

## Quick Start

### Prerequisites

- Python 3.11+
- Docker & Docker Compose
- API keys for at least one data provider

### One-Command Setup

```bash
# Clone and start all infrastructure
git clone https://github.com/Scheller-Labs/market-data-service.git
cd market-data-service
cp .env.example .env  # Add your API keys
make dev              # Starts Redis, TimescaleDB, MinIO + MDS
```

### Verify Installation

```bash
# Health check
make health

# Ingest sample data
mds ingest --symbol SPY --interval 1min --provider alphavantage

# Check coverage
mds coverage --symbol SPY
```

### Docker Only

```bash
docker compose up -d
docker compose exec mds mds health
```

## Usage

### CLI Examples

```bash
# Ingest historical data
mds ingest --symbol AAPL --interval 1d --start 2024-01-01 --end 2024-12-31

# Stream real-time data
mds stream --symbols SPY,QQQ,IWM --provider finnhub

# Check for data gaps
mds coverage --symbol SPY --interval 1min --start 2024-01-01

# Repair gaps automatically
mds repair --symbol SPY --interval 1min --auto

# Export to Parquet
mds export --symbol AAPL --format parquet --output ./data/
```

### Python API

```python
from mds import MarketDataService

mds = MarketDataService()

# Get historical bars
bars = await mds.get_bars("SPY", interval="1min", start="2024-01-01", end="2024-12-31")

# Subscribe to real-time data
async for bar in mds.stream("SPY", interval="1min"):
    print(f"{bar.timestamp} {bar.close} vol={bar.volume}")

# Check coverage
gaps = await mds.check_coverage("SPY", interval="1min")
```

## Configuration

MDS uses a layered configuration system: defaults → config file → environment variables → CLI flags.

See [docs/CONFIGURATION.md](docs/CONFIGURATION.md) for the full reference.

```bash
# Core settings via environment
MDS_REDIS_URL=redis://localhost:6379/0
MDS_TIMESCALE_URL=postgresql://mds:mds@localhost:5432/mds
MDS_MINIO_ENDPOINT=localhost:9000

# Provider API keys
MDS_ALPHAVANTAGE_API_KEY=your_key_here
MDS_FINNHUB_API_KEY=your_key_here
MDS_DATABENTO_API_KEY=your_key_here
```

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/ARCHITECTURE.md) | System design, data flow, and design rationale |
| [Configuration](docs/CONFIGURATION.md) | All configuration options and environment variables |
| [Deployment](docs/DEPLOYMENT.md) | Production deployment guide |
| [API Reference](docs/api/README.md) | REST API endpoints and schemas |
| [Contributing](CONTRIBUTING.md) | Development setup, coding standards, PR process |
| [Changelog](CHANGELOG.md) | Version history and release notes |
| [Security](SECURITY.md) | Security policy and vulnerability reporting |

### Architecture Decision Records

| ADR | Decision |
|-----|----------|
| [001](docs/decisions/001-three-tier-storage.md) | Three-tier storage architecture (Redis/TimescaleDB/MinIO) |
| [002](docs/decisions/002-provider-adapter-pattern.md) | Provider adapter pattern with token bucket rate limiting |
| [003](docs/decisions/003-timescaledb-over-questdb.md) | TimescaleDB over QuestDB for time-series storage |
| [004](docs/decisions/004-sqlite-coverage-manifest.md) | SQLite for coverage manifest over PostgreSQL |
| [005](docs/decisions/005-parquet-cold-storage.md) | Parquet format for cold storage archival |

### Runbooks

| Runbook | Scenario |
|---------|----------|
| [Data Provider Outage](docs/runbooks/provider-outage.md) | Handling provider downtime and failover |
| [Gap Detection & Repair](docs/runbooks/gap-repair.md) | Diagnosing and repairing data gaps |
| [Adding a New Provider](docs/runbooks/new-provider.md) | Step-by-step guide for new data source integration |
| [TimescaleDB Maintenance](docs/runbooks/timescaledb-maintenance.md) | Partition management, vacuuming, retention policies |

## Testing

```bash
make test          # Run all tests
make test-unit     # Unit tests only
make test-int      # Integration tests (requires Docker)
make test-e2e      # End-to-end scenarios
make test-perf     # Performance/load tests
make coverage      # Generate coverage report
```

See [tests/README.md](tests/README.md) for testing philosophy and conventions.

## Performance

Benchmarked on a single-node deployment (RTX 4070 Ti Super, 32GB RAM):

| Operation | Throughput | Latency (p99) |
|-----------|-----------|---------------|
| Cache read (Redis) | 45,000 req/s | < 1ms |
| Historical query (TimescaleDB) | 2,500 req/s | < 15ms |
| Ingest (single provider) | 500 bars/s | — |
| Gap detection scan | — | < 200ms / symbol |

## Roadmap

- [ ] gRPC API for low-latency agent communication
- [ ] Options chain data support
- [ ] Level 2 / depth-of-book integration
- [ ] Kafka/NATS streaming alternative
- [ ] Multi-node deployment with service mesh

## License

Licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

## Related Projects

This service is part of the **Agent Trading Firm** ecosystem:

- [agent-trading-firm](https://github.com/Scheller-Labs/agent-trading-firm) — Multi-agent autonomous trading system
- [multi-agent-architectures](https://github.com/Scheller-Labs/multi-agent-architectures) — AutoGen AG2 + LangGraph orchestration patterns
- [autoresearch-loop](https://github.com/Scheller-Labs/autoresearch-loop) — Autonomous research agent pipelines
- [llm-fine-tuning](https://github.com/Scheller-Labs/llm-fine-tuning) — Model fine-tuning and experimentation
