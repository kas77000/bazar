# Interview Script: Nicholas Shu Fan Au
**Role:** Full Stack Application Developer, APAC Equities Electronic Trading (HK) · **Duration:** 30 min
**His materials:** live exercise `2-live-exercise-B.md` · take-home `3-take-home-B.md`
**Interviewer copy. Do not share.**

---

## Before the interview (5 min)

**Who he is:** Data Solution Engineer at Bank of East Asia (Jul 2025 – Jul 2026). BSc CS (Toronto), Maths/Stats minors. AWS Solutions Architect Associate. Previously: TypeScript Shopify payment plugin (Storm Media), Django + React Native part-time.

| ✅ Strengths vs the job | ⚠️ Gaps vs the job |
|---|---|
| **Best match on paper:** CS degree (DS&A, databases, web programming) | React **Native**, not React web |
| Shipped **TypeScript** (Stripe, Adyen integrations) | No dashboard or visualization work shown |
| Full-stack: Django + React Native | Latest role is GenAI (Gemini) pipelines, not UI |
| Python + BigQuery in a bank; 1M+ row analysis | Big claims ("headcount 7 → 2", "80% cost cut"): his personal share? Short Storm Media stint (2 months) |

**The one question this interview must answer:** *Is his depth real, or AI-assisted output with big CV claims?* He should be able to go deepest, so push him.

Have ready: `2-live-exercise-B.md` (hidden), the scoring table at the bottom, a timer.

---

## 0:00 – 0:02 · Introduction (2 min)

**Do:**
- Introduce yourself and the team in one sentence.
- Say: *"30 minutes: some quick questions, a few about your CV, then a 10-minute coding exercise. I'll interrupt to keep time; that's normal."*
- State the rules: **no AI, no internet, no phone during the exercise.**

---

## 0:02 – 0:06 · Knockout questions (4 min, ~1 min each)

Missing 2 or more of these 4 is a strong **no**. If so, skip 0:12–0:17, still run the exercise, then close politely.

**K1 (SQL).** *"Table `fills(symbol, qty, price, ts)`. Average fill price per symbol for today, only symbols with at least 3 fills."*
- ✅ `SELECT symbol, AVG(price) FROM fills WHERE ts >= <today> GROUP BY symbol HAVING COUNT(*) >= 3;`
- 🌟 Bonus: he points out that the *volume-weighted* average is `SUM(qty*price)/SUM(qty)`, not `AVG(price)`.

**K2 (Python).** *"`a = {"700.HK": 100}; b = a; b["700.HK"] = 0`. What is `a` now? How do you copy properly? And `is` vs `==`?"*
- ✅ `a` is also changed (same object). Copy with `dict(a)`, `a.copy()`, or `copy.deepcopy` for nested data. `is` = same object, `==` = equal value.

**K3 (React).** *"`useEffect` with no dependency array vs `[]` vs `[x]`: the difference? What's the cleanup function for?"*
- ✅ Runs after every render / once on mount / when `x` changes. Cleanup runs on unmount or before the next run: clear timers, close sockets, abort fetches.

**K4 (REST).** *"What does idempotent mean? PUT vs PATCH? A client retries `POST /orders` after a timeout. What can go wrong?"*
- ✅ Idempotent = repeating the call gives the same result. PUT replaces, PATCH partially updates. A retried POST can create a **duplicate order**; fix with an idempotency key.

---

## 0:06 – 0:12 · CV deep-dive (6 min)

Goal: **(a) verify the claims are his, (b) probe React web.** After each answer, ask *"Why?"* or *"What would you do differently?"* once.

**Q1 (verify, 2 min).** *"With Stripe, how did you handle webhooks: duplicate events, retries, signature verification?"*
- ✅ Verify the signature with the webhook secret; store processed event IDs (idempotency); return 2xx quickly and process asynchronously; Stripe retries on failure.
- 🚩 Unsure what a webhook is. Then he likely didn't build the integration.

**Q2 (verify, 1.5 min).** *"The 1M+ transaction logs: what tool, what was the slowest query, and how did you speed it up?"*
- ✅ Specific: BigQuery partitioning or clustering, selecting fewer columns, pre-aggregation.

**Q3 (gap, 1.5 min).** *"React Native vs React for the web: the differences? How do you render a 10,000-row live table in a browser?"*
- ✅ DOM vs native views, CSS vs StyleSheet, routing, browser APIs. Virtualization, memoization, batched updates.

**Q4 (ownership, 1 min).** *"'Headcount 7 → 2': which part did you build yourself, and which did others build?"*
- ✅ A clear, honest split. 🚩 Can't separate his work from the team's.

---

## 0:12 – 0:17 · Depth questions (5 min)

He should handle these. Push with follow-ups.

**I3 (big tables).** *"A blotter with 50,000 rows updating every second freezes the browser. Fix it."*
- ✅ Virtualization, throttle/batch updates (e.g., flush every 250 ms), immutable keyed updates, memoized rows, server-side aggregation.

**I4 (slow query).** *"`SELECT * FROM fills WHERE symbol='700.HK' AND ts > ...` takes 8 seconds. What do you do?"*
- ✅ `EXPLAIN` → composite index `(symbol, ts)` (in that order: equality column first), select only needed columns, partition by date.

**A2 (time zones).** *"Fills come from Tokyo, Hong Kong and Sydney. How do you store and display time?"*
- ✅ Store UTC; display in exchange-local time; trading-day boundaries differ by market; HK lunch break; Sydney has daylight saving.

**A3 (Elasticsearch).** *"When would you use Elasticsearch instead of SQL?"*
- ✅ Full-text or log search and fast aggregations over huge append-only event data (with Kibana). SQL for relational and transactional data with constraints and joins.

---

## 0:17 – 0:27 · Live exercise (10 min)

**Do:**
1. Share `2-live-exercise-B.md`. Read Part 1 aloud, including the slippage formulas.
2. Say: *"Python or JavaScript/TypeScript, your choice. Think out loud. Do Part 1 first."*
3. Start the timer. **Stay quiet.** If he's stuck for 2 minutes, give one hint (*"What do you need to accumulate to compute a weighted average?"*) and note it.
4. Watch: does he notice the SELL sign? Filter `REJECTED`? Weight by quantity rather than a simple average?

### Answer key, Part 1: `slippage_by_symbol(fills)`
**Expected output:**
```
[{symbol: 9988.HK, qty: 300, slippage_bps: 16.67},
 {symbol: 700.HK,  qty: 400, slippage_bps: 8.33}]
```
⚠️ Traps:
- **SELL sign:** forgetting it gives 9988.HK = −16.67 and the order flips.
- **REJECTED:** the 0005.HK fill must not appear.
- **Weighting:** a simple average gives 700.HK = 16.67 instead of 8.33.
```python
from collections import defaultdict
def slippage_by_symbol(fills):
    qty, weighted = defaultdict(int), defaultdict(float)
    for f in fills:
        if f["status"] != "FILLED":
            continue
        diff = f["price"] - f["arrival_price"]
        if f["side"] == "SELL":
            diff = -diff
        bps = diff / f["arrival_price"] * 10_000
        qty[f["symbol"]] += f["qty"]
        weighted[f["symbol"]] += bps * f["qty"]
    res = [{"symbol": s, "qty": qty[s], "slippage_bps": round(weighted[s] / qty[s], 2)} for s in qty]
    return sorted(res, key=lambda r: r["slippage_bps"], reverse=True)
```

### Answer key, Part 2: FastAPI `/stats/{symbol}` (count the bugs found)
1. Loads the **whole table**, filters in Python → `WHERE symbol = ?`
2. Aggregates in Python → `SUM(qty*price)/SUM(qty)` in SQL
3. `ZeroDivisionError` when no fills → return 404 or `null`
4. Blocking sync DB call inside `async def` → blocks the event loop
5. Global `cache` is never invalidated, grows without bound, and is never read
6. No date filter (all-time, not today)
7. Float money
8. `SELECT *`

### Answer key, Part 3: React + TS `LiveFills`
1. Mutates state (`fills.push`) and sets the same reference → **no re-render**
2. Stale closure: always the first render's `fills` → `setFills(prev => [...prev, x])`
3. No cleanup → `return () => ws.close()`
4. List grows forever → cap it (e.g., the last 500)
5. `JSON.parse` result isn't validated, so the `Fill` type is a lie
6. Index used as `key`
7. No `onerror` or reconnect

**Pass bar:** Part 1 logic correct **and** ≥3 bugs in Part 2. From Nicholas, expect ≥5 in Part 2 and some of Part 3.

---

## 0:27 – 0:29 · Exercise review (2 min)

Ask:
- *"Walk me through your solution."*
- *"What input would break it?"* ✅ `arrival_price` = 0, empty list, unknown side, missing fields.
- *"Would you compute this in SQL or Python in production? Why?"* ✅ SQL or the DB for large data; Python is fine for small in-memory sets.

---

## 0:29 – 0:30 · Close (1 min)

**If he passed:** *"You'll receive a take-home today. Around 6 hours, 5 days, no AI. Then a 20-minute call where you walk me through it and change it live."* → send `3-take-home-B.md`.
**If he didn't pass:** thank him; *"We'll get back to you by [date]."*
Either way: *"Please don't share the exercise. Each candidate has a different one anyway."*

---

## Later: take-home `3-take-home-B.md`, "Execution Quality Monitor"
Python + SQL + React/TS. **Real-time and robustness focus:** push live fills (WebSocket/SSE), slippage by symbol and algo in SQL, alerts, shared typed API contracts, reconnect handling.

## Later: follow-up call (20 min, screen shared, AI and internet off)

| Time | Do |
|---|---|
| 0:00–0:05 | *"Walk me through the architecture. WebSocket or SSE, and why? How did you keep the TS types in sync with the backend? How does the simulator avoid races with API writes?"* |
| 0:05–0:15 | **Live change:** *"Make the alert threshold configurable per symbol from the UI."* (DB/API + UI) |
| 0:15–0:20 | Point at random lines: *"What does this do? What if the socket drops here?"* |

**Fail** if he can't find his way around his own code or can't make the change. Failing the call cancels the take-home score.

---

## Scoring: Nicholas Au (0 = nothing · 1 = partial · 2 = solid · 3 = strong)

| Area | Weight | Score | Notes |
|---|---|---|---|
| K1–K4 knockouts (passed /4) | must pass ≥3 | | |
| CV claims verified (Q1, Q2, Q4) | must pass | | |
| React web gap (Q3) | ×2 | | |
| Depth: I3, I4 | ×2 | | |
| Depth: A2, A3 | ×1 | | |
| Live Part 1 | ×3 | | |
| Live Part 2 (bugs /8) | ×2 | | |
| Live Part 3 React (bugs /7) | ×1 | | |
| Hints needed (count) | info | | |
| Communication | ×2 | | |
| Domain interest | ×1 | | |
| **Interview total** | | | |
| Take-home /100 | | | |
| Follow-up call | pass/fail | | |

**Decision:** ☐ Advance to take-home ☐ Reject ☐ Hire ☐ No hire
