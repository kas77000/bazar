# Interview Script: Susan Liu
**Role:** Full Stack Application Developer, APAC Equities Electronic Trading (HK) · **Duration:** 30 min
**Her materials:** live exercise `2-live-exercise-A.md` · take-home `3-take-home-A.md`
**Interviewer copy. Do not share.**

---

## Before the interview (5 min)

**Who she is:** Automation Engineer at Santander CIB (Oct 2024 – Oct 2026). BSc Business Computing & Data Analytics (HKBU). Part-time MBA (HKU). Trading-research intern (crypto backtesting).

| ✅ Strengths vs the job | ⚠️ Gaps vs the job |
|---|---|
| Closest to the business: Global Markets reconciliation, trading-message mapping (FX, swaps, bonds) | **No React, TypeScript, HTML/CSS or web app on her CV** |
| Python + SQL in a bank | Not a CS degree |
| Stakeholder coordination; most professional experience | Heavy AI/agentic tooling (Devin, MCP), so hand-coding is unverified |
| Markets knowledge (Sharpe, Calmar, backtesting) | "20+ workflows every 3 weeks" may be low-code/RPA |

**The one question this interview must answer:** *Can she actually code, especially front-end, without AI?*

Have ready: `2-live-exercise-A.md` (hidden), the scoring table at the bottom, a timer.

---

## 0:00 – 0:02 · Introduction (2 min)

**Do:**
- Introduce yourself and the team in one sentence.
- Say: *"30 minutes: some quick questions, a few about your CV, then a 10-minute coding exercise. I'll interrupt to keep time; that's normal."*
- State the rules: **no AI, no internet, no phone during the exercise.**

---

## 0:02 – 0:06 · Knockout questions (4 min, ~1 min each)

Missing 2 or more of these 4 is a strong **no**. If so, skip 0:12–0:17, still run the exercise, then close politely.

**K1 (SQL).** *"Table `orders(trader, symbol, qty)`. Give me, per trader, the number of orders and total quantity, only for traders with more than 5 orders."*
- ✅ `SELECT trader, COUNT(*), SUM(qty) FROM orders GROUP BY trader HAVING COUNT(*) > 5;`
- 🚩 Uses `WHERE COUNT(*) > 5`, or no `GROUP BY`.

**K2 (Python).** *"What's wrong with this?"*
```python
def add_fill(f, fills=[]):
    fills.append(f)
    return fills
```
- ✅ The default list is created once and shared between calls. Fix: `fills=None`, then `if fills is None: fills = []`.

**K3 (React).** *"State vs props? When does a component re-render?"*
- ✅ Props come from the parent and are read-only; state is owned by the component. A re-render happens when state changes (`setState`) or the parent re-renders.
- 🚩 Can't explain either one. That's serious, given her CV gap.

**K4 (REST).** *"When do you return 400, 401, 403, 404, 500?"*
- ✅ 400 bad input · 401 not authenticated · 403 authenticated but not allowed · 404 not found · 500 server error.

---

## 0:06 – 0:12 · CV deep-dive (6 min)

Goal: **(a) verify a claim is hers, (b) probe the front-end gap.** After each answer, ask *"Why?"* or *"What went wrong?"* once.

**Q1 (verify, 2 min).** *"You mapped trading messages across FX, swaps and bonds. What format were they: FIX, XML, CSV? Describe the Python you wrote for one field."*
- ✅ Concrete: e.g., a FIX tag → internal field via a dict or pandas, handling missing tags, validation, tests.
- 🚩 Vague, or "the tool/agent generated it".

**Q2 (AI reliance, 1.5 min).** *"In your Devin/agentic workflow, what did you personally write? Tell me about the last bug you fixed by hand."*
- ✅ A specific bug with the cause and the fix.
- 🚩 Only describes prompting.

**Q3 (front-end gap, 2.5 min).** *"Build me a React page with an orders table that refreshes every 5 seconds. Which hooks? What do you need to clean up?"*
- ✅ `useState` for data, `useEffect` with `setInterval` → fetch, `return () => clearInterval(id)`, plus loading/error state and `key` on rows.
- 🚩 Blank. A big red flag for a UI-heavy role.

*Bonus if time:* *"Sharpe vs Calmar in one sentence each."* ✅ Sharpe = excess return / volatility; Calmar = return / maximum drawdown.

---

## 0:12 – 0:17 · Depth questions (5 min)

Ask in order and stop when time is up. Go harder while she keeps answering well.

**I2 (live data).** *"A trader blotter must show live order updates. Polling, Server-Sent Events, or WebSockets? Trade-offs?"*
- ✅ Polling is simple but laggy and wasteful. SSE is one-way server→client and simple. WebSocket is two-way and needs reconnect logic. Bonus: out-of-order messages, heartbeats.

**I3 (big tables).** *"The blotter has 50,000 rows updating every second and the browser freezes. What do you do?"*
- ✅ Virtualized rows (react-window, AG Grid), batch or throttle updates, memoize rows, aggregate server-side, paginate.

**A1 (money).** *"What's `0.1 + 0.2` in Python? How should we store prices?"*
- ✅ `0.30000000000000004`. Use `Decimal`, integer ticks/cents, or SQL `NUMERIC`.

**A4 (production issue).** *"A trader says the dashboard has been frozen since 10:00. Walk me through it."*
- ✅ Reproduce → browser console and network → WebSocket/polling status → backend logs → is the upstream feed alive? → recent deploys → keep the trader informed.
- 🚩 Starts changing code before diagnosing.

---

## 0:17 – 0:27 · Live exercise (10 min)

**Do:**
1. Share `2-live-exercise-A.md`. Read Part 1 aloud.
2. Say: *"Python or JavaScript, your choice. Think out loud. Do Part 1 first."*
3. Start the timer. **Stay quiet.** If she's stuck for 2 minutes, give one hint (*"What do you need to accumulate per trader?"*) and note it.
4. Watch: does she read the whole problem? Notice the `REJECTED` filter? Check the output by hand? Name things clearly?

### Answer key, Part 1: `fill_rate_by_trader(orders)`
**Expected output:**
```
[{trader: AMY, ordered: 2000, filled: 1600, fill_rate: 80.0},
 {trader: BEN, ordered: 1000, filled: 700,  fill_rate: 70.0}]
```
⚠️ Trap: order 3 is REJECTED and must be excluded. If she includes it, AMY becomes 2400 / 1600 = 66.7%.
```python
from collections import defaultdict
def fill_rate_by_trader(orders):
    ordered, filled = defaultdict(int), defaultdict(int)
    for o in orders:
        if o["status"] == "REJECTED":
            continue
        ordered[o["trader"]] += o["qty"]
        filled[o["trader"]] += o["filled_qty"]
    res = [{"trader": t, "ordered": ordered[t], "filled": filled[t],
            "fill_rate": round(100 * filled[t] / ordered[t], 1)} for t in ordered]
    return sorted(res, key=lambda r: (-r["fill_rate"], r["trader"]))
```

### Answer key, Part 2: Flask `POST /orders` (count the bugs found)
1. SQL injection (`%` string formatting) → parameterized query
2. No validation: qty can be negative or text, side can be anything
3. Missing field → `KeyError` → 500 instead of 400
4. `request.json` can be `None` → crash
5. `float` for price → `Decimal`
6. Returns 200 without an order id → should be **201** + id
7. No commit or transaction
8. No trader or authentication

### Answer key, Part 3: React `OrdersTable` (very important for her)
1. `orders` starts as `undefined` → `.map` crashes
2. `trader` missing from the `useEffect` deps
3. No `key` on rows
4. No loading or error state
5. Race condition when `trader` changes (no abort)
6. `<tr>` directly in `<table>` without `<tbody>` (minor)

**Pass bar:** Part 1 logic correct **and** ≥3 bugs in Part 2. For Susan, Part 3 matters more than for the others.

---

## 0:27 – 0:29 · Exercise review (2 min)

Ask:
- *"Walk me through your solution."*
- *"What input would break it?"* ✅ An empty list, a trader whose orders are all rejected (division by zero), missing keys.
- *"How fast is it?"* ✅ One pass, O(n), plus sorting.

---

## 0:29 – 0:30 · Close (1 min)

**If she passed:** *"You'll receive a take-home today. Around 6 hours of work, 5 days to submit, no AI. We'll then do a 20-minute call where you walk me through it and change it live."* → send `3-take-home-A.md`.
**If she didn't pass:** thank her; *"We'll get back to you by [date]."*
Either way: *"Please don't share the exercise. Each candidate has a different one anyway."*

---

## Later: take-home `3-take-home-A.md`, "Order Blotter & Fill-Rate Dashboard"
Python + SQL + React/TS. **Front-end heavy on purpose** (her gap): sortable live grid, chart, order form, cancel button.

## Later: follow-up call (20 min, screen shared, AI and internet off)

| Time | Do |
|---|---|
| 0:00–0:05 | *"Walk me through your architecture. Why did you choose polling, SSE or WebSocket? How do you store prices?"* |
| 0:05–0:15 | **Live change:** *"Add a BUY/SELL side filter to the grid, end to end (API + UI)."* This tests her React gap directly. |
| 0:15–0:20 | Point at random lines in her code: *"What does this do? What if it's empty or null?"* |

**Fail** if she can't find her way around her own code or can't make the change. Failing the call cancels the take-home score.

---

## Scoring: Susan Liu (0 = nothing · 1 = partial · 2 = solid · 3 = strong)

| Area | Weight | Score | Notes |
|---|---|---|---|
| K1–K4 knockouts (passed /4) | must pass ≥3 | | |
| CV claims verified (Q1, Q2) | must pass | | |
| Front-end gap (Q3) | ×2 | | |
| Depth: I2, I3 | ×2 | | |
| Depth: A1, A4 | ×1 | | |
| Live Part 1 | ×3 | | |
| Live Part 2 (bugs /8) | ×2 | | |
| Live Part 3 React (bugs /6) | ×2 | | |
| Hints needed (count) | info | | |
| Communication | ×2 | | |
| Domain interest | ×1 | | |
| **Interview total** | | | |
| Take-home /100 | | | |
| Follow-up call | pass/fail | | |

**Decision:** ☐ Advance to take-home ☐ Reject ☐ Hire ☐ No hire
