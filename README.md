# 📈 Equity-Traders

**Equity-Traders** is a modular trading sandbox that wires up:
- **MCP (Model Context Protocol) servers** for market data, accounts, and push updates
- **Trader agents** that reason over this data using the **OpenAI Python SDK**
- A simple **SQLite** backing store for accounts/positions
- Light orchestration to simulate a “trading floor”

The goal: a clean, hackable foundation for experimenting with LLM-assisted equity trading flows, not a production broker or financial advice engine.

> ⚠️ This project is **for research and education**. It does **not** place live trades and offers **no investment advice**.

---

## 🧱 Architecture

```

```
       ┌───────────────────┐
       │   OpenAI SDK      │  (LLM calls: analysis, plans, orders)
       └─────────┬─────────┘
                 │
          ┌──────▼──────┐
          │  Traders    │  traders.py
          │ (agents)    │
          └───┬─────┬───┘
              │     │
    ┌─────────▼─┐ ┌─▼──────────┐
    │ Market    │ │ Accounts    │
    │ MCP       │ │ MCP         │
    │ Server    │ │ Server      │
    │ (prices)  │ │ (ledger)    │
    └──────┬────┘ └────┬───────┘
           │            │
     ┌─────▼────┐  ┌───▼───────┐
     │ market.py │  │ accounts.py│
     └──────────┘  └────────────┘

             ┌───────────────────────────┐
             │ Push MCP Server           │  push_server.py
             │ (stream updates/events)   │
             └───────────────────────────┘
```

Orchestration glue, util fns, DB helpers:

* trading\_floor.py, app.py, util.py, database.py, mcp\_params.py, tracers.py, templates.py

````

---

## 📂 Project Structure (mapped to actual files)

- **`app.py`** – CLI/entry utility to bring up components or run demos.
- **`trading_floor.py`** – Orchestrates the system: wires traders to MCP servers; runs scenarios/loops.
- **`traders.py`** – Defines trader agent(s): strategy scaffolds, decision loop, how LLM calls are used.
- **`market_server.py`** – **MCP server** exposing market-data tools (quotes, historical fetch, symbols).
- **`accounts_server.py`** – **MCP server** for accounts/positions/portfolio operations.
- **`push_server.py`** – **MCP server** for streaming/push events (price ticks, fills, alerts).
- **`market.py`** – Market adapters/utilities (price retrieval, symbol validation, transforms).
- **`accounts.py`** – Account/position models and business logic (balances, P&L, order checks).
- **`accounts_client.py`** – Thin client to talk to the Accounts MCP server from agents.
- **`database.py`** – SQLite helpers (schema, CRUD). Uses `accounts.db` by default.
- **`mcp_params.py`** – MCP server registration, tool schemas, and parameter/type definitions.
- **`templates.py`** – Prompt/LLM templates for analysis, orders, risk checks, etc.
- **`tracers.py`** – Logging/trace utilities for runs (LLM prompts, decisions, tool calls).
- **`util.py`** – Misc helpers (time, config, formatting).
- **`reset.py`** – Resets/initializes the SQLite database to a known state.
- **`accounts.db`** – SQLite database file.
- **`pyproject.toml`, `uv.lock`** – Python project + **uv** lock for reproducible installs.
- **`.python-version`, `.gitignore`** – Tooling hygiene.

> The above list reflects the actual filenames present in the repo root.

---

## ✨ Features

- **MCP-native** data access: market/portfolio as first-class tools the LLM can call.
- **Agent loop** in Python using **OpenAI SDK** (≥ `openai` v1): structured prompts → tool calls → decisions.
- **SQLite** baseline state for accounts/positions – easy to reset and inspect.
- **Composable**: swap strategies, add tools, or point to a different market adapter.

---

## 🚀 Quickstart

### 1) Clone & install
```bash
git clone https://github.com/ceodaniyal/Equity-Traders.git
cd Equity-Traders

# Using uv (recommended)
uv install
````

### 2) Environment

Create `.env` (or export) with at least:

```bash
export OPENAI_API_KEY=sk-...
# Optional:
# export OPENAI_BASE_URL=...   # if using an Azure/OpenAI-compatible endpoint
# export MODEL=gpt-4.1-mini    # or your preferred model
```

> If you use dotenv, ensure `python-dotenv` or equivalent is loaded in `app.py`/`trading_floor.py`.

### 3) Initialize database

```bash
uv run python reset.py
```

### 4) Start the MCP servers (each in its own shell)

```bash
# Market data
uv run python market_server.py

# Accounts & portfolio
uv run python accounts_server.py

# Push / streaming events
uv run python push_server.py
```

### 5) Run the trading floor (agents + OpenAI SDK)

```bash
uv run python trading_floor.py
```

Or run any demo entry:

```bash
uv run python app.py
```

---

## 🧩 MCP Integration

This project exposes three MCP servers. If you want to register them with an MCP-aware client (e.g., Claude Desktop or other MCP hosts), use a config similar to:

```json
{
  "mcpServers": {
    "equity-traders-market": {
      "command": "uv",
      "args": ["run", "python", "market_server.py"],
      "env": {}
    },
    "equity-traders-accounts": {
      "command": "uv",
      "args": ["run", "python", "accounts_server.py"],
      "env": {}
    },
    "equity-traders-push": {
      "command": "uv",
      "args": ["run", "python", "push_server.py"],
      "env": {}
    }
  }
}
```

Each server defines tools (see `mcp_params.py`) with JSON-serializable inputs/outputs so LLMs can call them deterministically.

---

## 🤖 OpenAI SDK Usage

Trader agents use the **OpenAI Python SDK** to:

* analyze symbols/market context,
* decide actions (e.g., “propose order”, “rebalance”, “do nothing”),
* optionally call MCP tools (market/accounts) to get authoritative data before acting.

Typical call pattern (schematic):

```python
from openai import OpenAI
client = OpenAI()

resp = client.chat.completions.create(
    model=os.getenv("MODEL", "gpt-4.1-mini"),
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_or_loop_message},
    ],
    tools=[...],  # MCP tool schemas or function-calling bridge
)
```

(Exact code lives in `traders.py` / `templates.py`.)

---

## 🔧 Configuration

* **Models**: set `MODEL` env var or adjust inside `traders.py`.
* **Market adapter**: extend `market.py` to point at your preferred data source (cached, CSV, API).
* **Risk/limits**: enforce in `accounts.py` (order checks) and/or `traders.py` (policy prompt + tool checks).
* **Prompts**: tweak in `templates.py`.
* **Tracing**: enable detail in `tracers.py` (e.g., file logs or structured JSON).

---

## 🧪 Example Workflows

* **Morning scan**: agent pulls pre-market snapshots (market MCP) → ranks candidates → proposes paper orders (accounts MCP).
* **Reactive trading**: push MCP streams an event (e.g., price break) → trader requests fresh quote → validates risk caps → submits simulated order.
* **Portfolio cleanup**: agent iterates holdings from accounts MCP → evaluates stale positions → suggests exits.

---

## 🛠️ Development

```bash
# Lint/format/testing (suggested)
uv run ruff check .
uv run ruff format .
uv run pytest -q
```

> If these tools aren’t in `pyproject.toml` yet, add them; `uv` keeps it fast and reproducible.

---

## 🧾 Troubleshooting

* **No data?** Ensure `market_server.py` is running and that `market.py` can resolve your symbols.
* **DB errors?** Delete `accounts.db` and run `uv run python reset.py` again.
* **LLM errors/limits?** Verify `OPENAI_API_KEY` and chosen `MODEL`. If self-hosted, set `OPENAI_BASE_URL`.
* **MCP client can’t discover tools?** Confirm the MCP config paths and that servers start without exceptions.

---

## 🗺️ Roadmap Ideas

* Live broker abstractions (paper-trade only)
* Backtesting harness against historical CSVs
* Risk dashboards & richer tracing/telemetry
* Strategy registry (momentum, mean-reversion, pairs, L/S)

---
