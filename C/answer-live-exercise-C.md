# Answers: Live Exercise C (Pranav Agarwal)
**Interviewer copy. Do not share.** Handout: `2-live-exercise-C.md`

---

## Part 1: `positions(fills, limit=250)`

### Expected output
```python
[
  {"symbol": "9988.HK", "net_qty": -500, "breach": True},
  {"symbol": "700.HK",  "net_qty": 200,  "breach": False},
  {"symbol": "0005.HK", "net_qty": 150,  "breach": False},
]
```
**How it's computed:**
- **700.HK:** BUY 300 − SELL 100 = **+200** → |200| ≤ 250 → no breach
- **9988.HK:** SELL 500 = **−500** (the BUY 200 is REJECTED and ignored) → |−500| > 250 → **breach**
- **0005.HK:** BUY 150 = **+150** → no breach
- **Sorted by absolute size:** 500, 200, 150

### Common mistakes
| Mistake | Wrong result |
|---|---|
| SELL not subtracted (all quantities added) | 700.HK = **400 → breach True**; 9988.HK = +500 |
| Sorted by `net_qty` instead of its absolute value | 9988.HK (−500) comes **last** |
| Includes the REJECTED fill | 9988.HK = **−300** (still a breach, but the wrong number) |
| `breach` checks `net_qty > limit` (no `abs`) | 9988.HK **breach False**: a short position over the limit goes unseen. This is the most dangerous mistake in real life. |

### Solution: Python
```python
from collections import defaultdict

def positions(fills, limit):
    pos = defaultdict(int)
    for f in fills:
        if f["status"] != "FILLED":
            continue
        pos[f["symbol"]] += f["qty"] if f["side"] == "BUY" else -f["qty"]

    result = [
        {"symbol": s, "net_qty": q, "breach": abs(q) > limit}
        for s, q in pos.items()
    ]
    return sorted(result, key=lambda r: abs(r["net_qty"]), reverse=True)
```

### Solution: JavaScript
```js
function positions(fills, limit) {
  const pos = {};
  for (const f of fills) {
    if (f.status !== 'FILLED') continue;
    pos[f.symbol] = (pos[f.symbol] ?? 0) + (f.side === 'BUY' ? f.qty : -f.qty);
  }
  return Object.entries(pos)
    .map(([symbol, net_qty]) => ({ symbol, net_qty, breach: Math.abs(net_qty) > limit }))
    .sort((a, b) => Math.abs(b.net_qty) - Math.abs(a.net_qty));
}
```

### Same thing in SQL (follow-up question in the review)
```sql
SELECT symbol,
       SUM(CASE WHEN side = 'BUY' THEN qty ELSE -qty END)              AS net_qty,
       ABS(SUM(CASE WHEN side = 'BUY' THEN qty ELSE -qty END)) > :limit AS breach
FROM fills
WHERE status = 'FILLED'
GROUP BY symbol
ORDER BY ABS(SUM(CASE WHEN side = 'BUY' THEN qty ELSE -qty END)) DESC;
```

**Complexity:** O(n) + O(k log k).
**Edge cases to ask about:** a side that is neither BUY nor SELL (the solution treats it as SELL; should it raise?), negative qty, a limit exactly equal to the position (`>` vs `>=`, which is a business question worth asking).

### Grading Part 1
- **3:** correct output, `abs` in both breach and sort, REJECTED excluded, asks about `>` vs `>=` or bad input
- **2:** correct logic, small slip
- **1:** one trap missed
- **0:** two or more traps missed, or no working approach

---

## Part 2: `get_symbol_report`, bugs

| # | Problem | Why it matters | Fix |
|---|---|---|---|
| 1 | `SELECT * FROM orders` loads **every order ever**, then filters in Python | Slow; memory-heavy | Filter by date in SQL (`WHERE`) |
| 2 | Date check via `str(created_at)[:10] == day` | Breaks on format changes; **ignores time zones**: a UTC timestamp's date ≠ the Hong Kong trading day (HK = UTC+8) | Convert the HK day to UTC `[start, end)` bounds |
| 3 | **N+1 queries**: one fills query per order | 10,000 orders → 10,001 queries | One `JOIN` |
| 4 | String-concatenated SQL (`"... = " + str(id)`) | Injection pattern; no plan caching | Parameters (`?`) |
| 5 | Summing in Python loops | Should be done by the database | `SUM(...) GROUP BY symbol` |
| 6 | `SELECT *` | Fetches unneeded columns | Name the columns |
| 7 | Symbols whose orders had no fills are **missing** from the report | Report looks incomplete | `LEFT JOIN` + `COALESCE(SUM(...), 0)` |

**Bonus discussion point:** the code filters by the day the **order** was created, but "filled quantity for a day" usually means fills that **happened** that day (`f.ts`). An order created yesterday can fill today. A strong candidate asks which one is meant.
**Bonus points:** indexes on `orders(created_at)` and `fills(order_id)`.

### Corrected version
```python
from datetime import datetime, timedelta, timezone
from zoneinfo import ZoneInfo

HKT = ZoneInfo("Asia/Hong_Kong")

def hk_day_bounds_utc(day: str):
    """'2026-11-12' -> UTC [start, end) of that Hong Kong calendar day."""
    start_local = datetime.strptime(day, "%Y-%m-%d").replace(tzinfo=HKT)
    end_local = start_local + timedelta(days=1)
    return start_local.astimezone(timezone.utc), end_local.astimezone(timezone.utc)

def get_symbol_report(conn, day):
    start, end = hk_day_bounds_utc(day)                       # 2. time zones
    rows = conn.execute(
        """
        SELECT o.symbol, COALESCE(SUM(f.qty), 0) AS filled_qty  -- 5, 6, 7
        FROM orders o
        LEFT JOIN fills f ON f.order_id = o.id                   -- 3, 7
        WHERE o.created_at >= ? AND o.created_at < ?             -- 1
        GROUP BY o.symbol
        ORDER BY o.symbol
        """,
        (start.isoformat(), end.isoformat()),                    # 4. parameters
    ).fetchall()
    return {r["symbol"]: r["filled_qty"] for r in rows}
# Assumes created_at is stored as UTC ISO-8601 text (or a real timestamp column).
```

### The minimum acceptable single-query answer
```sql
SELECT o.symbol, SUM(f.qty) AS filled_qty
FROM orders o
JOIN fills f ON f.order_id = o.id
WHERE o.created_at >= ? AND o.created_at < ?
GROUP BY o.symbol;
```
If he writes this unaided, his SQL is fine for a junior. `LEFT JOIN` + `COALESCE` + time zone handling = strong.

**Grading Part 2:** 6+ bugs = 3 · 4–5 = 2 · 3 = 1 (minimum pass) · 0–2 = 0. **The SQL rewrite counts as a separate signal** in his scoring sheet ("SQL gap").

---

## Part 3: React `AlertBanner`, bugs

| # | Problem | Fix |
|---|---|---|
| 1 | `onClick={setVisible(false)}` **calls** `setVisible` during render → state update during render → *"Too many re-renders"* crash | `onClick={() => setVisible(false)}` |
| 2 | `useState(alerts.length > 0)` reads props **only on the first render**. If alerts arrive later, the banner never shows. | Derive from props instead of copying them into state |
| 3 | Once dismissed, new alerts never bring the banner back | Track which alerts were dismissed (by id), not a single boolean |
| 4 | Shows only a count, not the alerts; a list would need a `key` | Render the list with `key={a.id}` |

**Bonus:** `role="alert"` for screen readers. Note also the `{count && <X/>}` trap in general: if `count` is `0`, React renders "0". Here it's a boolean, so it's fine.

### Corrected version
```jsx
function AlertBanner({ alerts }) {
  const [dismissedIds, setDismissedIds] = useState(() => new Set());

  const active = alerts.filter(a => !dismissedIds.has(a.id));      // 2, 3
  if (active.length === 0) return null;

  const dismissAll = () =>
    setDismissedIds(prev => new Set([...prev, ...active.map(a => a.id)]));

  return (
    <div className="banner" role="alert">
      <strong>{active.length} alert{active.length > 1 ? 's' : ''}</strong>
      <ul>
        {active.map(a => <li key={a.id}>{a.message}</li>)}         {/* 4 */}
      </ul>
      <button onClick={dismissAll}>Dismiss</button>                {/* 1 */}
    </div>
  );
}
```

**Grading Part 3:** 3+ bugs = 3 · 2 = 2 · 1 = 1 · 0 = 0. Bug 1 is the key one: anyone who has really written React has hit it.
