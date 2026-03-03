# Specification: Project Samwise Gardener

> **Status: Draft** — Subject to revision as implementation begins.

An AI-first, headless stock market assistant designed for cross-market monitoring (US & TW), portfolio tracking, and strategy execution via AI collaboration.

---

## 1. Project Vision & Philosophy

- **The "Samwise" Principle:** A loyal, low-profile assistant that handles the heavy lifting (data fetching, monitoring) and alerts the user only when specific criteria are met.
- **Headless & AI-First:** No traditional GUI. Interaction happens via **Slack** (for alerts) and **AI Agents/MCP** (for queries and strategy development).
- **Monitoring Only:** Samwise surfaces signals and alerts — it does **not** place real trades. Order execution is out of scope.
- **Custom Orchestration:** Instead of heavy frameworks (Freqtrade, vn.py), Samwise uses a lightweight, modular Python orchestrator to ensure maximum flexibility for AI-generated strategy code.

---

## 2. System Architecture & Design Patterns

### 2.1 The Context-Driven ("Briefing") Pattern

The engine uses a context-driven approach inspired by *Zipline* and *Clean Architecture*, keeping strategy code simple and AI-friendly:

- **Engine's Role:** Fetches raw market data, calculates technical indicators (MA, RSI, Bollinger Bands, etc.), and fetches macro signals.
- **The Context Object:** All processed data is bundled into a single `context` object passed to each strategy.
- **Strategy's Role:** A simple function or class that receives the `context`, runs its logic, and returns a trigger/alert status.

```
┌─────────────────────────────────────────────┐
│                   Engine                    │
│  fetch data → compute indicators → build    │
│  context → dispatch to strategy plugins     │
└─────────────────────────────────────────────┘
               │
               ▼
      context = { price, rsi, ma, macro, ... }
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
strategy_a  strategy_b  strategy_c
(returns alert or None)
```

### 2.2 Autonomous Strategy Plugins

- **Independence:** Each strategy is a standalone `.py` file in the `strategies/` folder. Strategies do not share state.
- **Hard-coded Parameters:** Strategy-specific constants (e.g., `RSI_THRESHOLD = 30`) are written directly in the strategy file. This eliminates side effects between strategies and provides a clean Git history for versioning.
- **Hot-Reloading:** The engine monitors the `strategies/` directory and loads new strategy files dynamically. New strategies authored and deployed via a server restart are immediately active without manual intervention.

---

## 3. Data Management (The Three-Tier Map)

### 3.1 SQLite Database — The Persistent Layer

The database remains lightweight (target: < 10 MB) and stores only long-lived facts:

| Table | Contents |
|---|---|
| `transactions` | Trade history, cost basis per position (stored in original currency) |
| `holdings` | Current portfolio positions |
| `watchlist` | Tickers under active monitoring and user-defined metadata |
| `macro_history` | Daily/monthly snapshots: Fear & Greed Index, TW Business Cycle Signals, etc. |

**Currency note:** All monetary values are stored in their **original currency** (USD for US-listed securities, TWD for TW-listed securities). Conversion to TWD for display and reporting happens at query time using the latest cached FX rate.

> `data/samwise.db` must be listed in `.gitignore` — it contains financial transaction data and must never be committed.

### 3.2 Memory Cache — The Transient Layer

- **Technical Indicators:** K-lines, moving averages, RSI, volatility, and other computed values live in memory only.
- **Lifecycle:** `Fetched → Computed → Bundled into context → Passed to strategy → Discarded.`
- This design prevents database bloat from high-frequency indicator data.

---

## 4. Technical Stack

| Category | Choice |
|---|---|
| Language | Python 3.11+ |
| Containerization | Docker & Docker Compose |
| US Market Data | `yfinance` |
| TW Market Data | `FinMind` (historical/fundamental), Fugle API (real-time tick) |
| Technical Analysis | `pandas-ta` |
| Scheduling | `APScheduler` (in-process, AsyncIO) |
| Plugin / Hot-Reload | `pluggy` + `watchdog` + `importlib.reload` |
| Primary Interface | Slack SDK — `slack-bolt` with Socket Mode, Block Kit for interactive alerts |
| AI Integration | MCP server (`mcp` SDK + `fastmcp`) — SSE transport |
| FX Rates | `httpx` → ExchangeRate-API or CurrencyBeacon (cached in memory, refreshed hourly) |
| Monetary Arithmetic | `decimal.Decimal` (never `float`) |
| Package Config | `requirements.txt` (may migrate to `pyproject.toml` as the project matures) |

---

## 5. Deployment & AI Workflow (The Closed-Loop)

```
User ──discuss──▶ Claude ──generate──▶ strategy file
                                            │
                                     git push to main
                                            │
                              server restart (Docker pull & redeploy)
                                            │
                              Engine loads new strategy on startup
                                            │
                              Strategy runs against live context ──▶ Slack alert
```

### Step-by-step

1. **Discuss** — User describes a new monitoring idea to Claude.
2. **Generate** — Claude writes a new strategy file (e.g., `strategies/rsi_oversold_tw.py`).
3. **Deploy** — Claude Code commits and pushes the file to the `main` branch on GitHub.
4. **Redeploy** — The cloud server pulls the updated image / latest code and restarts the Docker container.
5. **Execute** — On startup, the Samwise engine discovers and loads all files in `strategies/`, including the new one.

> **No polling or webhook is required** for strategy loading. The engine loads all strategies at startup; the deployment restart is the synchronisation mechanism.

---

## 6. Directory Structure

```text
sam-wise-trade/               # Repository root
├── main.py                   # Core loop & orchestrator entry point
├── Dockerfile                # Container definition
├── docker-compose.yml        # Service orchestration
├── requirements.txt          # Python dependencies
├── .env.example              # Environment variable template (never commit .env)
├── data/
│   └── samwise.db            # SQLite database (git-ignored)
├── core/
│   ├── data_fetcher.py       # Market & macro API wrappers
│   ├── engine.py             # Strategy loader & context builder
│   └── notifier.py           # Slack interaction logic
├── strategies/               # AI-generated strategy modules (*.py)
├── tests/                    # Mirrors core/ and strategies/ structure
│   ├── core/
│   │   ├── test_data_fetcher.py
│   │   ├── test_engine.py
│   │   └── test_notifier.py
│   └── strategies/
└── docs/                     # Project documentation
```

---

## 7. Key Constraints & Conventions

- **No trade execution.** Samwise is read-only with respect to brokerage accounts. It monitors and alerts; it never places orders.
- **Decimal arithmetic.** All monetary values in code use `Decimal`, not `float`. See CLAUDE.md.
- **Secrets via environment variables.** API keys (Fugle, FinMind, Slack, FX provider) live in `.env` only — never committed.
- **Strategy isolation.** Strategies must not import from other strategy files or write to shared state. Each file is self-contained.
- **`importlib.reload()` constraint.** Hot-reload works for single-file strategy modules only. Multi-file strategy packages are not supported and will not reload correctly.
