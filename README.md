# Sistema Centinela — Architectural Case Study

> **Proprietary System** · Source code is maintained in a private repository.  
> This repository is a public architectural showcase for portfolio and case-study purposes.

[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![Docker](https://img.shields.io/badge/docker-compose-blue.svg)](https://www.docker.com/)
[![GCP](https://img.shields.io/badge/GCP-Tokyo%20Region-orange.svg)](https://cloud.google.com/)
[![License](https://img.shields.io/badge/license-Proprietary-red.svg)]()
[![Strategy](https://img.shields.io/badge/strategy-TF--01_Golden_Cross_Dual_70/30-gold.svg)]()
[![Version](https://img.shields.io/badge/version-v6.3-blue.svg)]()

---

## Overview

**Sistema Centinela** is a production-grade algorithmic trading system built on a Service-Oriented Architecture (SOA) with 7 Docker microservices deployed on GCP (Tokyo Region). It implements the **TF-01 Golden Cross Dual 70/30** strategy on Bybit Exchange, embedding **native fiscal compliance** for Mexican tax law (SAT / LISR / UIF) at the execution layer — not as an afterthought.

**Core philosophy:** _"Risk-First Execution"_ — no order reaches the market without passing through the `RiskGatekeeper` (`svc-risk`).

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GCP — Tokyo Region                          │
│                                                                     │
│  ┌──────────┐   OHLCV 5m/4H   ┌──────────────┐                    │
│  │ svc-feed │ ──────────────▶ │  TimescaleDB │                    │
│  └──────────┘                 └──────┬───────┘                    │
│                                      │                             │
│  ┌───────────┐  EMA 21/150 4H        │                            │
│  │ svc-brain │ ◀────────────────────┘   ──▶ BUY/SELL signal       │
│  └─────┬─────┘                                                     │
│        │ Señal                                                      │
│        ▼                                                            │
│  ┌──────────┐  Validates: drawdown, Kelly 0.3×,                    │
│  │ svc-risk │  FIFO cost basis, UIF $50K MXN,                      │
│  └─────┬────┘  Kill-Switch flag in DB                               │
│        │ Signed order + risk_signature                              │
│        ▼                                                            │
│  ┌──────────┐   Marketable LIMIT                                   │
│  │ svc-exec │ ─────────────────────────────▶  Bybit API V5         │
│  └──────────┘   (reprice T+30s, cancel T+120s)                     │
│                                                                     │
│  ┌─────────────┐  Bybit → Blockchain → Bitso → SPEI                │
│  │ svc-watcher │  (repatriation pipeline with tx_hash)             │
│  └─────────────┘                                                    │
│                                                                     │
│  ┌─────────────┐  NSGA-III multi-objective optimization,           │
│  │   svc-lab   │  CPCV backtesting (read-only DB access)           │
│  └─────────────┘                                                    │
│                                                                     │
│  ┌───────────────┐  Health checks, Prometheus alerts, n8n          │
│  │  svc-monitor  │                                                  │
│  └───────────────┘                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Microservices

| Service       | Port | Responsibility                                                                                   |
| ------------- | ---- | ------------------------------------------------------------------------------------------------ |
| `svc-feed`    | 8001 | Ingests OHLCV from Bybit WebSocket V5 → TimescaleDB + Parquet                                    |
| `svc-brain`   | 8002 | TF-01 engine: computes EMA 21/150, emits Golden/Death Cross signals, manages dual trailing stops |
| `svc-risk`    | 8003 | Risk firewall: drawdown limits, Kelly 0.3×, UIF $50K MXN, FIFO pre-calc, Kill Switch             |
| `svc-exec`    | 8004 | Sends signed LIMIT orders to Bybit (reprice T+30s, cancel T+120s)                                |
| `svc-lab`     | 8005 | Isolated backtesting (read-only DB), NSGA-III, CPCV, Monte Carlo                                 |
| `svc-monitor` | 8006 | Health checks, Prometheus metrics, n8n alerting workflows                                        |
| `svc-watcher` | 8007 | Repatriation monitoring: Bybit → Blockchain → Bitso → SPEI                                       |

### Docker Network Isolation (3-Tier)

| Network        | CIDR          | Access                                    |
| -------------- | ------------- | ----------------------------------------- |
| `net_public`   | 172.20.0.0/24 | Inbound webhooks only                     |
| `net_internal` | 172.20.1.0/24 | Inter-service communication (no internet) |
| `net_data`     | 172.20.2.0/24 | DB access only                            |

API keys never travel on `net_public`. Services that touch the exchange (`svc-exec`) are isolated from data-layer services.

---

## Trading Strategy: TF-01 Golden Cross Dual 70/30

| Signal   | Condition                                             | Timeframe |
| -------- | ----------------------------------------------------- | --------- |
| **BUY**  | EMA_21 crosses ↑ EMA_150 (Golden Cross)               | 4H        |
| **SELL** | EMA_21 crosses ↓ EMA_150 (Death Cross)                | 4H        |
| **STOP** | Price ≤ `entry − 3×ATR_14` (70% CONSERVATIVE tranche) | 4H        |
| **STOP** | Price ≤ `entry − 5×ATR_14` (30% AGGRESSIVE tranche)   | 4H        |

- Trailing stops **only move up** — updated at each 4H candle close.
- Immunity windows ±2–5 min around candle close to avoid wicks triggering false exits.
- Split position sizing (70/30) reduces drawdown while keeping asymmetric upside.

### Intelligence Layer

- **Hidden Markov Model (HMM)** — runs in shadow mode classifying market regime (BULL / BEAR / LATERAL) to dynamically adjust Kelly sizing and trailing stop width.
- **NSGA-III Optimizer** (`svc-lab`) — continuous multi-objective optimization across 28 assets (70% global equities + 30% crypto), validated with CPCV and Monte Carlo simulation.

### Validated Backtest Results (4 years, 3 pairs, 138 configurations)

| Pair              | Best Config      |  Net Return   | Sharpe | Max DD |
| ----------------- | ---------------- | :-----------: | :----: | :----: |
| SOLUSDT / BTCUSDT | TF-01 Dual 70/30 | **+134–313%** |  ~1.8  |  -15%  |
| ETHUSDT (control) | Buy & Hold       |    -1.29%     |   —    |   —    |
| ETHUSDT (HMM)     | GaussHMM         |  Sharpe 0.87  |   —    |   —    |

---

## Risk Management Constitution (v5.1.1)

| Rule               | Value                               | Action                                              |
| ------------------ | ----------------------------------- | --------------------------------------------------- |
| Daily Drawdown     | -4%                                 | Automatic 24h pause                                 |
| Weekly Drawdown    | -7%                                 | Hard stop — manual intervention required            |
| Kelly Fraction     | 0.3× (capped at 25% equity)         | Prevents over-leverage                              |
| **Kill Switch**    | `FORCE_EXIT` flag persisted in DB   | Cancels all open orders + liquidates                |
| Order Type         | Marketable LIMIT (single)           | MARKET orders prohibited except forced liquidation  |
| UIF Limit          | $50,000 MXN / operation             | Anti-money laundering (auto-enforced at risk layer) |
| Jitter             | Mandatory on all timers             | Prevents exchange rate-limiting                     |
| Paper Trading Gate | 4 weeks minimum before real capital | Non-negotiable pre-deployment requirement           |

The Kill Switch is DB-persisted (not in-memory) so it survives service restarts. Any operator or monitoring service can trigger it atomically.

---

## Native Fiscal Compliance (Mexico)

This is the system's defining differentiator. Tax compliance is embedded at the execution layer, not added later.

| Framework          | Implementation                                                                          |
| ------------------ | --------------------------------------------------------------------------------------- |
| **LISR Art. 124**  | Strict FIFO costing with `fifo_ref` and `lotes_afectados` per transaction               |
| **Exchange Rate**  | DOF/Banxico daily rate (Serie SF43718), `NOT NULL` constraint on all non-MXN operations |
| **INPC**           | Acquisition month required per lot for inflation-adjusted cost basis                    |
| **UIF Reporting**  | Automatic $50K MXN / operation and $50K MXN / month cumulative limit                    |
| **Legal Entity**   | `FISICA` — immutable by DB schema constraint                                            |
| **Repatriation**   | Bybit → Blockchain → Bitso → SPEI with `tx_hash` traceability at each hop               |
| **SAT Audit View** | `vista_isr_anual` in TimescaleDB for annual tax return reconciliation                   |

---

## Key Architecture Decisions (ADR Summary)

| ADR     | Decision                                 | Rationale                                                        |
| ------- | ---------------------------------------- | ---------------------------------------------------------------- |
| ADR-001 | Entity type `FISICA` immutable           | Legal; prevents accidental regime switch                         |
| ADR-002 | Repatriation flow Bybit→Bitso→SPEI       | Only regulated on-ramp for MXN                                   |
| ADR-003 | Pivot Mean Reversion → TF-01             | Backtesting showed MR failed on volatile crypto pairs            |
| ADR-008 | TimescaleDB over pure PostgreSQL         | Native time-series compression, continuous aggregates            |
| ADR-009 | Dual Trailing Stop 70/30                 | Reduces max drawdown while retaining upside capture              |
| ADR-010 | EMA 21/150 on 4H (not 50/200 on 1H)      | 4H filters micro-volatility; 21/150 cross confirmed on 4yr data  |
| ADR-011 | Kelly Fraction 0.3× capped at 25% equity | Prevents ruin under correlated losses                            |
| ADR-012 | Marketable LIMIT only                    | Guarantees fill price; avoids slippage from pure MARKET          |
| ADR-015 | Strict FIFO — LISR Art. 124              | Legal requirement; prevents tax basis manipulation               |
| ADR-016 | Immunity windows ±2–5 min on 4H close    | Eliminates false signals from candle-close wicks                 |
| ADR-017 | Mandatory jitter on all timers           | Exchange rate-limiting prevention                                |
| ADR-018 | Paper trading 4-week gate                | Risk control; validates live feed vs backtest parity             |
| ADR-019 | NSGA-III as optimization engine          | Multi-objective (Sharpe, max DD, Calmar) outperforms grid search |
| ADR-020 | Kill Switch persisted in DB              | Survives service crash; single source of truth                   |

---

## Technology Stack

| Layer              | Technologies                                                                   |
| ------------------ | ------------------------------------------------------------------------------ |
| **Runtime**        | Python 3.11, AsyncIO, FastAPI                                                  |
| **Data**           | TimescaleDB (PostgreSQL extension), Parquet                                    |
| **Exchange**       | Bybit V5 WebSocket + REST (CCXT)                                               |
| **Infra**          | Docker Compose, GCP Compute Engine (Tokyo region)                              |
| **Monitoring**     | Prometheus, n8n, custom health endpoints                                       |
| **Security**       | Docker secrets (`/run/secrets/`), 3-tier network isolation, no env-var secrets |
| **Optimization**   | NSGA-III (pymoo), Monte Carlo, CPCV                                            |
| **Statistical ML** | GaussHMM (hmmlearn)                                                            |

---

## Documentation Structure

The private repository maintains comprehensive documentation following engineering best practices:

```
docs/
├── 01_Auditoria/                    ← Immutable state snapshots (timestamped)
├── 02_Core_y_Arquitectura/          ← Living technical specifications
├── 03_Bitacora_de_Implementacion/   ← Implementation history
├── 04_Investigacion_y_Labs/         ← Immutable research documents
├── 05_Estrategias_y_Algoritmos/     ← Strategy specifications
├── 06_Operaciones_y_Despliegue/     ← Runbooks, infra, secrets management
├── adrs/                            ← 20 Architecture Decision Records
├── reportes/YYYY-MM/                ← Execution reports (backtest, NSGA)
└── plantillas/                      ← Standard templates and protocol
```

---

## Why This Project Matters

Most algorithmic trading bots ignore the legal and financial reality of operating in a specific jurisdiction. Sistema Centinela was designed from day one with the **assumption that it must be deployable for a real person in Mexico**, which means:

1. **Every profit is trackable** — the system emits the data needed for an annual SAT declaration.
2. **Every risk has a hard limit** — drawdown, UIF, and order type constraints are enforced by a dedicated service (`svc-risk`), not scattered throughout the codebase.
3. **Every decision is documented** — 20 ADRs record why specific choices were made, making the system maintainable and auditable.

This is not a side project. It is a financial-grade system built to professional standards by a single engineer using AI-augmented development — demonstrating that a small team (or solo founder) can deliver enterprise-level architecture when the right tools and disciplines are applied.

---

## Contact

Interested in discussing the architecture, the strategy mathematics, or the Mexican regulatory framework?

- **Email:** <kenethissac@gmail.com>
- **LinkedIn:** [linkedin.com/in/keneth-huerta](https://www.linkedin.com/in/keneth-isaac-huerta-moreno-8285552b1/)
- **GitHub:** [github.com/Keneth-Huerta](https://github.com/Keneth-Huerta)

> _"The goal was never just to build a trading bot. The goal was to build trust — with regulators, with auditors, and with the capital at risk."_
