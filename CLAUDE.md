# CLAUDE.md — AI Assistant Guide for sam-wise-trade

## Project Overview

**sam-wise-trade** is a Personal Financial Assistant built with Python. This repository is in early-stage development. This document provides guidance for AI assistants (Claude, Copilot, etc.) working with this codebase.

---

## Repository Structure

```
sam-wise-trade/
├── .gitignore         # Python-specific ignore rules
├── README.md          # Brief project description
└── CLAUDE.md          # This file — AI assistant guidance
```

As the project grows, the expected structure is:

```
sam-wise-trade/
├── src/               # Application source code
├── tests/             # Test files (mirrors src/ structure)
├── docs/              # Documentation
├── scripts/           # Utility scripts
├── .env.example       # Environment variable template (never commit .env)
├── pyproject.toml     # Project metadata and dependencies
├── CLAUDE.md
└── README.md
```

---

## Technology Stack

- **Language:** Python 3
- **Package manager:** To be determined (pip/uv/Poetry/PDM are all plausible — check for `pyproject.toml`, `requirements.txt`, or a lock file when the project matures)
- **Testing:** pytest (indicated by `.gitignore`)
- **Linting/formatting:** Ruff (indicated by `.gitignore`), mypy for type-checking
- **Notebooks:** Marimo (indicated by `.gitignore`)
- **AI automation:** Abstra (indicated by `.gitignore`)

---

## Development Workflows

### Setting Up the Environment

Since no package configuration exists yet, follow standard Python project conventions:

```bash
# Create and activate a virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

# Install dependencies (once pyproject.toml or requirements.txt exists)
pip install -e ".[dev]"
# or: uv pip install -e ".[dev]"
```

### Running Tests

```bash
# Once tests are added
pytest
pytest --cov  # with coverage
```

### Linting and Formatting

```bash
# Once configured
ruff check .         # lint
ruff format .        # format
mypy .               # type-check
```

### Git Workflow

This project uses SSH-signed commits. Commits are signed automatically via the configured git settings.

```bash
# Branch from the main branch for new features
git checkout -b feature/my-feature

# Commit with descriptive messages
git commit -m "feat: add portfolio tracking module"

# Push and open a PR
git push -u origin feature/my-feature
```

---

## Key Conventions

### Python Style

- Follow [PEP 8](https://peps.python.org/pep-0008/)
- Use type annotations on all function signatures
- Prefer `ruff` over `flake8`/`pylint` for linting
- Keep functions small and single-purpose

### Environment Variables

- **Never** commit `.env` files — they are git-ignored
- Provide an `.env.example` template with placeholder values
- Use environment variables for all secrets, API keys, and credentials (financial APIs, brokerage keys, etc.)

### Secrets & Financial Data

This is a financial assistant — treat all financial data and API credentials with extra care:
- Never log raw credentials, account numbers, or portfolio balances
- Never hard-code API keys or tokens
- Assume any financial API keys are sensitive and should only live in `.env`

### Testing

- Write tests for all business logic, especially financial calculations
- Financial calculations should be tested with edge cases (zero values, negative values, large numbers)
- Use `pytest` fixtures for shared test data

---

## Git Branch Conventions

| Branch prefix      | Purpose                          |
|--------------------|----------------------------------|
| `main` / `master`  | Stable, production-ready code    |
| `feature/`         | New features                     |
| `fix/`             | Bug fixes                        |
| `chore/`           | Non-functional changes           |
| `claude/`          | AI-assistant development branches|

---

## Areas for Future Development

Based on the project description ("Personal Financial Assistant"), expect to eventually implement:

- **Portfolio tracking** — holdings, positions, asset allocation
- **Trade execution** — buy/sell order management
- **Market data** — price feeds, historical data retrieval
- **Analytics** — performance metrics, P&L calculations
- **Alerts** — price alerts, portfolio threshold notifications

When implementing these, prefer:
- Well-tested financial libraries (e.g., `pandas`, `yfinance`, `alpaca-trade-api`)
- Decimal arithmetic over floats for monetary values (`from decimal import Decimal`)
- Async I/O for market data feeds where latency matters

---

## Common Pitfalls to Avoid

1. **Floating-point arithmetic for money** — always use `Decimal` or integer cents
2. **Unhandled API rate limits** — financial APIs often have strict rate limits; implement backoff
3. **Missing `.env` validation** — fail fast on startup if required env vars are missing
4. **Untested financial logic** — edge cases in financial math can be costly; test thoroughly
5. **Committing sensitive data** — double-check before every commit involving credentials

---

## Notes for AI Assistants

- The repository is **early-stage** — no source code exists yet beyond configuration files
- When scaffolding new code, follow Python best practices and prefer simple, readable implementations
- Always check for the existence of `pyproject.toml`, `requirements.txt`, or a virtual environment before suggesting install commands
- When asked to add features, create corresponding tests in `tests/` mirroring the `src/` structure
- Prefer `ruff` for all linting suggestions over older tools like `flake8` or `pylint`
- All monetary values in code should use `Decimal`, not `float`
