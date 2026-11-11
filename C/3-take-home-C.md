# Take-Home Assignment: "Position & Risk-Limit Monitor"

**Time budget:** about 6 hours. Please don't exceed it.
**Deadline:** 5 days after receiving this.
**Deliverable:** a Git repository link with full commit history.

> **No AI tools (Copilot, ChatGPT, Claude, etc.) for this assignment.** After you submit, there is a **20-minute call** where you explain your code and change it **live, with no AI and no internet**. Only submit code you fully understand.

## Context
The risk team needs to see, in real time, each trader's **net position** per stock (shares bought − sold) and be alerted when a position goes over its limit.

## Required stack
**Python** backend (FastAPI or Flask) · **SQL** database (SQLite or PostgreSQL) with a schema file · **React + TypeScript** frontend

## Must-have

### Backend
1. **Fill simulator:** a background task that generates random fills every 1–2 seconds `{ trader, symbol, side, qty, price, ts }`. Traders: 4 of your choice. Symbols: `700.HK, 9988.HK, 0005.HK, 7203.T, BHP.AX`.
2. **Limits:** a `limits` table (per trader + symbol, max absolute net quantity).
   - `GET /limits`
   - `PUT /limits/{trader}/{symbol}`, with validation and clear `400` errors.
3. `GET /positions`: per trader + symbol: net qty, average cost, notional, limit, % of limit used, and breach flag. **Compute it in SQL** (one query, not Python loops).
4. `GET /positions/history?date=YYYY-MM-DD`: end-of-day positions for a given **Hong Kong trading day** (watch your time zones).
5. `GET /breaches`: list of limit breaches with the time they happened.

### Frontend
1. **Positions grid:** grouped or filterable by trader; colour by % of limit used (e.g., green < 70%, amber < 100%, red ≥ 100%). **Updates live.**
2. **Breach alerts panel** that highlights new breaches.
3. **Limit editor:** change a limit from the UI, with validation.
4. A history view: pick a date, see the positions.
5. Loading, empty and error states. The layout must work on a laptop and on a large monitor.

### Quality
- Tests for the position calculation (BUY/SELL signs, no fills, limit exactly reached), the SQL queries, and limit validation.
- README: how to run it in 1–2 commands, design decisions, trade-offs, and what you'd do next.
- Commit as you go.

## Decisions left to you (explain them in the README)
Polling vs SSE vs WebSockets · how you store prices · how you define the HK trading day in UTC · indexes you added and why.

## Not required
Authentication, real market data, deployment.

## How we evaluate
Correct numbers, SQL quality, API design, a usable live UI, clean code, tests, and how well you explain and change your own code in the follow-up call.
