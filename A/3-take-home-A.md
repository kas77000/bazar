# Take-Home Assignment: "Order Blotter & Fill-Rate Dashboard"

**Time budget:** about 6 hours. Please don't exceed it.
**Deadline:** 5 days after receiving this.
**Deliverable:** a Git repository link with full commit history.

> **No AI tools (Copilot, ChatGPT, Claude, etc.) for this assignment.** After you submit, there is a **20-minute call** where you explain your code and change it **live, with no AI and no internet**. Only submit code you fully understand.

## Context
Traders want a single screen to watch their orders get filled and see how well each trader's orders are being executed.

## Required stack
**Python** backend (FastAPI or Flask) · **SQL** database (SQLite or PostgreSQL) with a schema file · **React + TypeScript** frontend

## Must-have

### Backend
1. `POST /orders`: create an order `{ trader, symbol, side: BUY|SELL, qty, limit_price }`. Validate all fields and return clear `400` errors.
2. **Fill simulator:** a background task that partially fills open orders every 1–2 seconds. Filled quantity must **never exceed** the order quantity. Status moves `NEW → PARTIALLY_FILLED → FILLED`. Symbols: `700.HK, 9988.HK, 0005.HK, 7203.T, BHP.AX`.
3. `PATCH /orders/{id}/cancel`: cancel an open order (`409` if it's already filled).
4. `GET /orders`: filters (`status`, `symbol`, `trader`), sorting, pagination.
5. `GET /stats/traders`: per trader, ordered qty, filled qty, fill rate %, number of orders. **Aggregate in SQL.**

### Frontend (the core of this assignment)
1. **Order blotter:** sortable columns, filters, a progress indicator for filled/qty, status colours, a Cancel button. **Updates live** without a page refresh.
2. **Trader fill-rate panel** with a **chart** (any charting library).
3. **New order form** with client-side validation and display of server errors.
4. Loading, empty and error states. The layout must work on a laptop and on a large trading monitor.

### Quality
- Tests for the fill-rate calculation, order validation, and "fill never exceeds qty".
- README: how to run it in 1–2 commands, design decisions, trade-offs, and what you'd do next.
- Commit as you go.

## Decisions left to you (explain them in the README)
Polling vs SSE vs WebSockets · how you store prices (float, Decimal, integers?) · time zones (HK, Tokyo, Sydney) · project structure.

## Not required
Authentication, real market data, deployment.

## How we evaluate
Correct numbers, API and SQL design, a usable and responsive UI, clean code, tests, and how well you explain and change your own code in the follow-up call.
