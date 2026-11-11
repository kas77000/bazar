# Interview Script: Pranav Agarwal
**Role:** Full Stack Application Developer, APAC Equities Electronic Trading (HK) · **Duration:** 30 min
**His materials:** live exercise `2-live-exercise-C.md` · take-home `3-take-home-C.md`
**Interviewer copy. Do not share.**

---

## Before the interview (5 min)

**Who he is:** fresh graduate, BEng Computer Engineering (HKUST, Jun 2026, final-year project A+). In FDM Skills Lab training **Oct 2026 – Jan 2027**. Internships: data analyst (GREEN Hospitality), XR developer (DReality).

| ✅ Strengths vs the job | ⚠️ Gaps vs the job |
|---|---|
| Performance mindset: voice latency 17–20 s → 3.3 s; fixed CPU contention | **Least professional experience** (internships only) |
| Python REST automation (6 APIs → one dataset) | React/HTML/CSS only from FDM training |
| Docker, Linux, Nginx | **SQL barely shown on the CV** |
| **MQTT pub/sub** fits real-time trading feeds | No finance exposure |
| Built a Looker Studio dashboard; public GitHub | **Availability:** training until January 2027 |

**Prep:** spend 3 minutes on **github.com/pagarwalad**. Pick one file or function to ask about (*"Why did you write it this way?"*).

**The one question this interview must answer:** *Are his web and SQL fundamentals solid enough, and when can he start?*

Have ready: `2-live-exercise-C.md` (hidden), the scoring table at the bottom, a timer.

---

## 0:00 – 0:02 · Introduction (2 min)

**Do:**
- Introduce yourself and the team in one sentence.
- Say: *"30 minutes: some quick questions, a few about your CV, then a 10-minute coding exercise. I'll interrupt to keep time; that's normal."*
- State the rules: **no AI, no internet, no phone during the exercise.**

---

## 0:02 – 0:06 · Knockout questions (4 min, ~1 min each)

Missing 2 or more of these 4 is a strong **no**. If so, skip 0:12–0:17, still run the exercise, then close politely.

**K1 (SQL).** *"Table `fills(symbol, qty, price, ts)`. Symbols with more than 10 fills today, sorted by total quantity, biggest first."*
- ✅ `SELECT symbol, SUM(qty) FROM fills WHERE ts >= <today> GROUP BY symbol HAVING COUNT(*) > 10 ORDER BY SUM(qty) DESC;`
- 🚩 Can't combine `GROUP BY` with `HAVING`. Watch closely; SQL is his gap.

**K2 (Python).** *"`x = [1, 2, 3]; y = x; y += [4]; print(x)`. Output and why? When do you use a list vs tuple vs set?"*
- ✅ `[1, 2, 3, 4]`: `y` is the same object and `+=` modifies the list in place. Tuple = immutable; set = unique values, fast membership, unordered.

**K3 (React).** *"Why does React need a `key` in lists? Controlled vs uncontrolled input?"*
- ✅ Keys let React match items between renders (avoid the index if the list reorders). Controlled = value lives in state with `onChange`; uncontrolled = the DOM holds the value (refs).

**K4 (HTTP).** *"What's inside an HTTP request? GET vs POST? What is CORS and why does it exist?"*
- ✅ Method, URL, headers, body. GET reads (no body, cacheable); POST sends data or creates. CORS: the browser blocks cross-origin requests unless the server allows them via headers. It protects users.

---

## 0:06 – 0:12 · CV deep-dive (6 min)

Goal: **(a) verify a claim is his, (b) probe SQL and real-time.** After each answer, ask *"Why?"* once.

**Q1 (verify, 1.5 min).** *"You cut voice latency from 17–20 s to 3.3 s. How did you measure it, and what was the slowest part before?"*
- ✅ Timestamps per stage, profiling, repeated runs, a clear before/after method.
- 🚩 Can't explain how he measured it.

**Q2 (real-time, 1.5 min).** *"MQTT vs WebSockets? How would you get live fills from a Python backend into a React page?"*
- ✅ MQTT = broker-based pub/sub with topics and QoS, used for devices. Browsers use WebSockets or SSE (FastAPI endpoint) → React state update. MQTT-over-WebSocket is also possible.

**Q3 (SQL gap, 2 min).** *"Tables `orders(id, symbol)` and `fills(order_id, qty)`. Find the orders that have no fills at all."*
- ✅ `SELECT o.* FROM orders o LEFT JOIN fills f ON f.order_id = o.id WHERE f.order_id IS NULL;` (or `NOT EXISTS`)
- 🚩 Can't write a join. Serious.

**Q4 (GitHub, 0.5 min).** Ask about the file you picked: *"Why did you write it this way?"*

**Q5 (logistics, 0.5 min).** *"Your Skills Lab runs until January. When could you realistically start?"*

---

## 0:12 – 0:17 · Depth questions (5 min)

**I1 (event loop).** *"What does this print?"*
```js
console.log('A');
setTimeout(() => console.log('B'), 0);
Promise.resolve().then(() => console.log('C'));
console.log('D');
```
- ✅ `A D C B`: synchronous code first, then microtasks (promises), then timers.

**I4 (slow query, SQL focus).** *"`SELECT * FROM fills WHERE symbol='700.HK' AND ts > ...` takes 8 seconds. What do you do?"*
- ✅ `EXPLAIN` → composite index `(symbol, ts)` → select only needed columns → partition by date.

**A1 (money).** *"What's `0.1 + 0.2` in Python? How should we store prices?"*
- ✅ `0.30000000000000004`. Use `Decimal`, integer ticks/cents, or SQL `NUMERIC`.

**A4 (production issue).** *"A trader says the dashboard has been frozen since 10:00. Walk me through it."*
- ✅ Reproduce → browser console and network → WebSocket status → backend logs → upstream feed → recent deploys → keep the trader informed. (His debugging background should show here.)

---

## 0:17 – 0:27 · Live exercise (10 min)

**Do:**
1. Share `2-live-exercise-C.md`. Read Part 1 aloud.
2. Say: *"Python or JavaScript, your choice. Think out loud. Do Part 1 first. In Part 2, try to rewrite it as one SQL query."*
3. Start the timer. **Stay quiet.** If he's stuck for 2 minutes, give one hint (*"How does a SELL change the position?"*) and note it.
4. Watch: SELL negative? `REJECTED` filtered? Sorted by **absolute** size?

### Answer key, Part 1: `positions(fills, limit=250)`
**Expected output:**
```
[{symbol: 9988.HK, net_qty: -500, breach: True},
 {symbol: 700.HK,  net_qty: 200,  breach: False},
 {symbol: 0005.HK, net_qty: 150,  breach: False}]
```
⚠️ Traps:
- **Sort by absolute value:** −500 must come first.
- **SELL subtracts:** without it, 700.HK = 400 → breach.
- **REJECTED:** the 9988.HK BUY must be ignored.
```python
from collections import defaultdict
def positions(fills, limit):
    pos = defaultdict(int)
    for f in fills:
        if f["status"] != "FILLED":
            continue
        pos[f["symbol"]] += f["qty"] if f["side"] == "BUY" else -f["qty"]
    res = [{"symbol": s, "net_qty": q, "breach": abs(q) > limit} for s, q in pos.items()]
    return sorted(res, key=lambda r: abs(r["net_qty"]), reverse=True)
```

### Answer key, Part 2: `get_symbol_report` (count the bugs found)
1. Loads **all orders** and filters in Python → `WHERE` in SQL
2. Date filter by string slicing ignores time zone (UTC vs the HK day)
3. **N+1 queries** (one query per order)
4. String-concatenated SQL → parameterized query
5. Aggregation in Python → `JOIN` + `GROUP BY`
6. `SELECT *`
7. Symbols without fills are missing from the report

**Rewrite as one query (this is the real test of his SQL):**
```sql
SELECT o.symbol, SUM(f.qty) AS filled_qty
FROM orders o
JOIN fills f ON f.order_id = o.id
WHERE o.created_at >= ? AND o.created_at < ?   -- HK day boundaries converted to UTC
GROUP BY o.symbol;
```

### Answer key, Part 3: React `AlertBanner`
1. `onClick={setVisible(false)}` is **called during render** → "Too many re-renders". Fix: `onClick={() => setVisible(false)}`.
2. State initialised from props only once, so it doesn't update when new alerts arrive.
3. Doesn't list the alerts (and would need a `key` if it did).
4. Once dismissed, the banner never comes back, even for new alerts.

**Pass bar:** Part 1 logic correct **and** ≥3 bugs in Part 2. For Pranav, the SQL rewrite is a strong signal either way.

---

## 0:27 – 0:29 · Exercise review (2 min)

Ask:
- *"Walk me through your solution."*
- *"What input would break it?"* ✅ Unknown side, negative qty, missing fields, empty list.
- *"How would you do Part 1 in SQL instead?"* ✅ `SUM(CASE WHEN side='BUY' THEN qty ELSE -qty END) ... GROUP BY symbol ORDER BY ABS(...) DESC`

---

## 0:29 – 0:30 · Close (1 min)

**If he passed:** *"You'll receive a take-home today. Around 6 hours, 5 days, no AI. Then a 20-minute call where you walk me through it and change it live."* → send `3-take-home-C.md`.
**If he didn't pass:** thank him; *"We'll get back to you by [date]."*
Either way: *"Please don't share the exercise. Each candidate has a different one anyway."*

---

## Later: take-home `3-take-home-C.md`, "Position & Risk-Limit Monitor"
Python + SQL + React/TS. **SQL heavy on purpose** (his gap): positions computed in one SQL query, limits table, HK trading-day history (time zones), live colour-coded grid, breach alerts.

## Later: follow-up call (20 min, screen shared, AI and internet off)

| Time | Do |
|---|---|
| 0:00–0:05 | *"Walk me through the architecture. Show me the positions query. How did you define the HK trading day in UTC? Which indexes did you add?"* |
| 0:05–0:15 | **Live change:** *"Add a gross exposure column (`SUM(ABS(qty * price))`), computed in SQL, shown in the grid."* |
| 0:15–0:20 | Point at random lines: *"What does this do? What if there are no fills?"* |

**Fail** if he can't find his way around his own code or can't make the change. Failing the call cancels the take-home score.

---

## Scoring: Pranav Agarwal (0 = nothing · 1 = partial · 2 = solid · 3 = strong)

| Area | Weight | Score | Notes |
|---|---|---|---|
| K1–K4 knockouts (passed /4) | must pass ≥3 | | |
| CV claims verified (Q1, Q4) | must pass | | |
| SQL gap (Q3 + Part 2 rewrite) | ×2 | | |
| Real-time (Q2) | ×1 | | |
| Depth: I1, I4 | ×2 | | |
| Depth: A1, A4 | ×1 | | |
| Live Part 1 | ×3 | | |
| Live Part 2 (bugs /7) | ×2 | | |
| Live Part 3 React (bugs /4) | ×1 | | |
| Hints needed (count) | info | | |
| Communication | ×2 | | |
| Domain interest | ×1 | | |
| **Interview total** | | | |
| Take-home /100 | | | |
| Follow-up call | pass/fail | | |
| **Start date confirmed** | info | | |

**Decision:** ☐ Advance to take-home ☐ Reject ☐ Hire ☐ No hire
