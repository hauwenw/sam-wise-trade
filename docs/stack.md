# Technology Stack

> **Early Draft** — This document reflects initial research and is subject to change as the project matures. Library choices have not yet been validated through implementation.

---

## Application Spine

**FastAPI + asyncio + Uvicorn** serves as the single-process backbone. It shares one event loop with the scheduler, hosts the MCP SSE endpoint, receives Slack webhook callbacks, and exposes a `/health` endpoint for Docker. Pydantic v2 (bundled with FastAPI) handles all data model validation.

```
FastAPI process (single asyncio event loop)
├── APScheduler AsyncIOScheduler  → per-ticker polling + daily reports
├── watchdog Observer thread       → detects new/changed strategy files
├── pluggy PluginManager           → dispatches price events to strategies
├── Slack Bolt AsyncApp            → interactive button callbacks
├── MCP FastMCP server             → SSE endpoint for Claude/AI agents
└── httpx AsyncClient              → yfinance / FinMind / Fugle / FX APIs
```

---

## Library Choices by Category

### Scheduler — `APScheduler 3.x`
Runs in-process with no external broker. Supports `IntervalTrigger` for per-ticker polling, `CronTrigger` for daily reports, and persistent job stores so jobs survive restarts. Jobs can be added and removed dynamically at runtime, which pairs naturally with the plugin system.

> Note: APScheduler 4.x has a cleaner async-first design (AnyIO-based) but is still in alpha — not production-ready yet.

### Plugin / Hot-Reload — `pluggy` + `watchdog` + `importlib.reload`
- **`pluggy`** provides hook specifications (`@hookspec`) and implementations (`@hookimpl`). Strategies implement hooks like `on_price_update` and are registered/unregistered dynamically via a `PluginManager`.
- **`watchdog`** monitors the `strategies/` directory for file changes using native OS APIs (`inotify` on Linux), triggering a reload without a full restart.
- **`importlib.reload()`** reloads the changed module in-place.

> Constraint: strategies must be single-file modules, not packages — `reload()` does not recurse into sub-modules.

### Slack — `slack-bolt` (`AsyncApp` + Socket Mode)
The official Slack-recommended framework. Handles interactive button callbacks, Block Kit message construction, and scheduled message delivery. Socket Mode opens a persistent WebSocket to Slack — no public inbound URL required, which is ideal for Docker containers behind NAT.

> Note: Legacy Slack bots (classic tokens, old `slackclient` library) were retired in March 2025. Tutorials referencing these are obsolete.

### MCP Server — `mcp` SDK + `fastmcp`
The official Anthropic MCP Python SDK provides the protocol layer. `fastmcp` adds a high-level decorator interface that auto-generates tool schemas from type hints and docstrings, minimizing boilerplate.

> Note: When using stdio transport, never write to stdout — it corrupts the JSON-RPC stream. Use stderr or file-based logging only. For Docker/remote deployment, use SSE transport instead.

### US Market Data — `yfinance`
De-facto standard for free US market data. Returns OHLCV, dividends, splits, and options chains as pandas DataFrames. No API key required for basic use.

> Limitation: unofficial Yahoo Finance API — endpoints have changed without notice in the past. No documented rate-limit SLA.

### Taiwan Market Data — `FinMind` + Fugle API
- **FinMind** — 50+ Taiwan market datasets (daily OHLCV, institutional flows, margin trading, fundamentals). Best for historical and fundamental data. Free tier: 300 requests/hour (600/hour with a registered token).
- **Fugle API** — official provider with real-time tick data and, via Fugle Masterlink, programmatic order placement and account/position access.

> Decision point: FinMind alone is sufficient for monitoring-only use. Fugle is required for live tick data or trade execution.

### USD/TWD FX Rates — `ExchangeRate-API` or `CurrencyBeacon`
Fetched via `httpx` and cached in-memory. Refreshed on a scheduled job — hourly updates are more than sufficient given the 15-minute data polling interval.

### Technical Indicators — `pandas-ta`
150+ indicators as a Pandas DataFrame extension (RSI, SMA, EMA, MACD, Bollinger Bands, candlestick patterns). Installs cleanly via pip with no C compilation step, keeping Docker builds simple.

> Alternative: TA-Lib is 2–4x faster due to its C core but requires compiling a C library at Docker build time. Worth the complexity only if throughput across many tickers becomes a bottleneck.

### Monetary Arithmetic — `decimal.Decimal`
All cost basis, valuations, P&L, and currency conversions must use `Decimal`, not `float`. IEEE 754 floating-point rounding errors accumulate in ways that are difficult to debug in financial contexts.

```python
from decimal import Decimal, ROUND_HALF_UP

cost_basis = Decimal("150.25")
current_price = Decimal("172.80")
unrealized_pnl = (current_price - cost_basis).quantize(
    Decimal("0.01"), rounding=ROUND_HALF_UP
)
```

---

## Summary Table

| Category | Primary Choice | Alternative | Key Trade-off |
|---|---|---|---|
| App framework | `FastAPI` + `asyncio` + `Uvicorn` | Bare asyncio (no HTTP) | Single-process; not horizontally scalable by default |
| Scheduler | `APScheduler 3.x` | `Celery Beat` (distributed) | APScheduler 4.x alpha — not production-ready yet |
| Plugin system | `pluggy` + `watchdog` + `importlib.reload` | Directory scan on each tick | `reload()` does not recurse into sub-modules |
| Slack | `slack-bolt` + Socket Mode | `slack-sdk` WebClient (outbound-only) | Legacy bots dead as of March 2025 |
| MCP server | `mcp` SDK + `fastmcp` | Raw JSON-RPC server | stdio transport must not write to stdout |
| US market data | `yfinance` | Polygon.io (paid, more reliable) | Unofficial API; no rate-limit SLA |
| TW market data | `FinMind` + Fugle | TWSE direct API | FinMind free tier: 300 req/hour |
| FX rates | `ExchangeRate-API` / `CurrencyBeacon` | Taiwan CBC published rates | Free tiers update hourly or daily |
| Technical indicators | `pandas-ta` | `TA-Lib` (faster, harder to install) | TA-Lib requires C library compilation in Docker |
| Monetary arithmetic | `decimal.Decimal` | — (never use `float`) | Must be enforced consistently across all modules |
