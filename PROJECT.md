# Project Samwise Gardener: AI-First Stock Assistant

A loyal, headless, and highly customizable stock monitoring and asset management system. Designed to support personalized trading logic and AI collaboration as first-class concerns.

---

## 1. Core Objectives

- **Loyalty & Automation:** A background service that monitors the market continuously without manual intervention.
- **Complex Logic:** Multi-condition strategy triggers (e.g., Technical + Volume + Price) that go beyond the constraints of standard retail platforms.
- **AI-Driven Lifecycle:** Strategies are discussed with AI, authored by AI, and automatically deployed to the cloud through a seamless CI/CD loop.

---

## 2. Technical Architecture

| Concern | Choice |
|---|---|
| Runtime | Docker (containerized, consistent across local and cloud) |
| Base Currency | TWD — all valuations and P/L normalized via real-time USD/TWD rates |
| TW Market Data | FinMind or Fugle API |
| US Market Data | `yfinance` |
| Data Frequency | 15-minute delayed (standard); polling interval configurable per ticker |

---

## 3. Data & Asset Layers (The Three-Tier Map)

### Layer 1 — Portfolio (The Ledger)
Tracks actual holdings: ticker, market, cost basis, and quantity. Provides real-time valuation and unrealized P/L calculation.

### Layer 2 — Watchlist (The Sentry)
Tracks potential entries with specific monitoring triggers.
> Example: *"Alert if RSI < 30 and price touches 20-day MA."*

### Layer 3 — Strategy Lab (The Brain)
- **Plug-and-play architecture:** Each strategy is an independent Python module.
- **Hot-reloading:** New strategy files are detected and loaded dynamically — no full system restart required.

---

## 4. AI & DevOps Workflow (The Closed Loop)

Strategies flow from idea to production without manual steps:

```
1. Discuss & generate strategy with Claude / Claude Code
2. Push Python module to GitHub
3. Cloud server pulls the update
4. Docker container reloads the new strategy automatically
```

**MCP Integration:** A Model Context Protocol (MCP) server exposes portfolio and watchlist data so AI agents can query system state directly through natural language.

---

## 5. Interface & Interaction

### Primary Console: Slack
- **Active Alerts:** Rich notifications with charts, data points, and interactive buttons (e.g., *Acknowledge*, *Ignore*, *Liquidate*).
- **Scheduled Reports:** Daily pre-market and post-market summaries of asset distribution and performance.

### LUI (Language User Interface)
Direct, ad-hoc queries via AI agents.
> Example: *"Sam, what is my current exposure to US tech stocks?"*
