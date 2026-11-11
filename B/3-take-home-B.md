# Take-Home Assignment: "Execution Quality Monitor"

**Time budget:** about 6 hours. Please don't exceed it.
**Deadline:** 5 days after receiving this.
**Deliverable:** a Git repository link with full commit history.

> **No AI tools (Copilot, ChatGPT, Claude, etc.) for this assignment.** After you submit, there is a **20-minute call** where you explain your code and change it **live, with no AI and no internet**. Only submit code you fully understand.

## Context
Traders and quants want to see, in real time, how well our algorithms execute compared to the market price when each order arrived (**slippage**).

Slippage in basis points:
- BUY: `(fill_price - arrival_price) / arrival_price * 10000`
- SELL: `(arrival_price - fill_price) / arrival_price * 10000`

## Required stack
**Python** backend (FastAPI or Flask) · **SQL** database (SQLite or PostgreSQL) with a schema file · **React + TypeScript** frontend

## Must-have

### Backend
1. **Market + fill simulator:** a background task that, every 1–2 seconds:
   - moves a mid price for each symbol (`700.HK, 9988.HK, 0005.HK, 7203.T, BHP.AX`), and
   - generates fills for open orders around that price.
2. `POST /orders`: create an order `{ symbol, side, qty, algo: "VWAP"|"TWAP"|"POV" }`. Record the **arrival price** at creation. Validate input and return clear `400` errors.
3. **Push live fills to the browser** (WebSocket or SSE, your choice).
4. `GET /stats/slippage`: quantity-weighted slippage in bps, grouped by symbol **and** by algo, with an optional time window. **Aggregate in SQL.**
5. **Alerts:** when an order's running slippage exceeds a threshold (default 20 bps), store an alert and push it.
6. `GET /alerts`.

### Frontend
1. **Live fills feed** (bounded; the list must not grow forever).
2. **Slippage table** by symbol and algo, plus a chart.
3. **Alerts panel** that highlights new alerts.
4. **Order form** with validation.
5. **Shared, typed API contracts:** the TS types must match the backend schemas. Explain how you kept them in sync.
6. Handle connection loss. Show a disconnected state and reconnect automatically.

### Quality
- Tests for the slippage calculation (including SELL sign and zero quantity), validation, and the alert trigger.
- README: how to run it in 1–2 commands, design decisions, trade-offs, and what you'd do next.
- Commit as you go.

## Decisions left to you (explain them in the README)
WebSocket vs SSE · how you store prices · time zones · how the simulator avoids race conditions with API writes.

## Not required
Authentication, real market data, deployment.

## How we evaluate
Correct numbers, API and SQL design, robust real-time handling, clean code, tests, and how well you explain and change your own code in the follow-up call.
