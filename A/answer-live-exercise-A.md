# Answers: Live Exercise A (Susan Liu)
**Interviewer copy. Do not share.** Handout: `2-live-exercise-A.md`

---

## Part 1: `fill_rate_by_trader(orders)`

### Expected output
```python
[
  {"trader": "AMY", "ordered": 2000, "filled": 1600, "fill_rate": 80.0},
  {"trader": "BEN", "ordered": 1000, "filled": 700,  "fill_rate": 70.0},
]
```
**How it's computed:**
- **AMY:** orders 1 + 5 → ordered 1000 + 1000 = 2000, filled 1000 + 600 = 1600 → **80.0%**. Order 3 is REJECTED and excluded.
- **BEN:** orders 2 + 4 → ordered 500 + 500 = 1000, filled 200 + 500 = 700 → **70.0%**.

### Common mistakes
| Mistake | Wrong result |
|---|---|
| Doesn't exclude the REJECTED order | AMY = 1600 / 2400 = **66.7%** (and BEN ranks first) |
| Averages each order's rate instead of totals | AMY = (100% + 60%) / 2 = 80% (right by luck here); BEN = (40% + 100%) / 2 = 70% (also lucky). Ask: *"Is that the same thing in general?"* No: it isn't weighted by quantity. |
| Integer division (old habits) or no rounding | 80 / 70, or long decimals. Minor. |
| Forgets the tie-break by name | Minor; ask what happens on a tie. |

### Solution: Python
```python
from collections import defaultdict

def fill_rate_by_trader(orders):
    ordered = defaultdict(int)
    filled = defaultdict(int)
    for o in orders:
        if o["status"] == "REJECTED":
            continue
        ordered[o["trader"]] += o["qty"]
        filled[o["trader"]] += o["filled_qty"]

    result = [
        {
            "trader": t,
            "ordered": ordered[t],
            "filled": filled[t],
            "fill_rate": round(100 * filled[t] / ordered[t], 1),
        }
        for t in ordered
    ]
    return sorted(result, key=lambda r: (-r["fill_rate"], r["trader"]))
```

### Solution: JavaScript
```js
function fillRateByTrader(orders) {
  const totals = {};
  for (const o of orders) {
    if (o.status === 'REJECTED') continue;
    totals[o.trader] ??= { trader: o.trader, ordered: 0, filled: 0 };
    totals[o.trader].ordered += o.qty;
    totals[o.trader].filled += o.filled_qty;
  }
  return Object.values(totals)
    .map(t => ({ ...t, fill_rate: Math.round((1000 * t.filled) / t.ordered) / 10 }))
    .sort((a, b) => b.fill_rate - a.fill_rate || a.trader.localeCompare(b.trader));
}
```

**Complexity:** O(n) to accumulate + O(k log k) to sort (k = number of traders).
**Edge cases to ask about:** empty list → `[]`; a trader with `ordered = 0` → division by zero (guard it); missing keys.

### Grading Part 1
- **3:** correct output, REJECTED excluded, tie-break, mentions an edge case unprompted
- **2:** correct logic, small slip (rounding, tie-break)
- **1:** right idea but includes REJECTED, or doesn't aggregate per trader correctly
- **0:** no working approach

---

## Part 2: Flask `POST /orders`, bugs

| # | Problem | Why it matters | Fix |
|---|---|---|---|
| 1 | **SQL injection**: values inserted with `%` string formatting | `symbol = "x'); DROP TABLE orders;--"` | Parameterized query (`?` placeholders) |
| 2 | **No validation**: qty can be negative, 0, text; side can be anything | Bad orders in the database | Validate type, range and allowed values |
| 3 | `data["qty"]` on a missing field → `KeyError` | Returns **500** instead of **400** | `data.get(...)` and return 400 with a message |
| 4 | `request.json` can be `None` (wrong content type, empty body) | `TypeError` crash → 500 | `request.get_json(silent=True)` and check it |
| 5 | `float(price)` | Floating-point errors on money; `float("abc")` crashes | `Decimal`, stored as `NUMERIC` or integer ticks |
| 6 | Returns **200** with no order id | The client can't reference the order | **201 Created** + the new `id` |
| 7 | No commit or transaction | The insert may never be persisted | `with db:` (commit/rollback) |
| 8 | No trader or authentication | Anyone can place orders for anyone | Take the trader from the authenticated session |

**Bonus points:** idempotency key against duplicate submissions, logging, checking the symbol against a reference list.

### Corrected version
```python
from decimal import Decimal, InvalidOperation
from flask import request

VALID_SIDES = {"BUY", "SELL"}
VALID_SYMBOLS = {"700.HK", "9988.HK", "0005.HK", "7203.T", "BHP.AX"}

def parse_price(value):
    try:
        price = Decimal(str(value))
    except InvalidOperation:
        return None
    return price if price.is_finite() and price > 0 else None

@app.route("/orders", methods=["POST"])
@login_required                                   # 8. authentication
def create_order():
    data = request.get_json(silent=True)           # 4. no crash on empty/invalid body
    if not isinstance(data, dict):
        return {"error": "JSON body required"}, 400

    symbol, side, qty = data.get("symbol"), data.get("side"), data.get("qty")
    price = parse_price(data.get("price"))         # 5. Decimal, not float

    errors = {}                                    # 2 + 3. validation → 400
    if symbol not in VALID_SYMBOLS:
        errors["symbol"] = "unknown symbol"
    if side not in VALID_SIDES:
        errors["side"] = "must be BUY or SELL"
    if not isinstance(qty, int) or isinstance(qty, bool) or qty <= 0:
        errors["qty"] = "must be a positive integer"
    if price is None:
        errors["price"] = "must be a positive number"
    if errors:
        return {"errors": errors}, 400

    with db:                                       # 7. transaction (commit/rollback)
        cur = db.execute(                          # 1. parameterized query
            "INSERT INTO orders (trader, symbol, side, qty, price) VALUES (?, ?, ?, ?, ?)",
            (current_user.id, symbol, side, qty, str(price)),
        )
    return {"id": cur.lastrowid, "symbol": symbol, "side": side,
            "qty": qty, "price": str(price)}, 201  # 6. 201 + id
```

**Grading Part 2:** 6+ bugs = 3 · 4–5 = 2 · 3 = 1 (minimum pass) · 0–2 = 0

---

## Part 3: React `OrdersTable`, bugs

| # | Problem | Fix |
|---|---|---|
| 1 | `useState()` → `orders` is `undefined` on first render → `orders.map` **crashes** | `useState([])` |
| 2 | Effect deps `[]` but it uses `trader` → doesn't refetch when `trader` changes | `[trader]` |
| 3 | No `key` on `<tr>` | `key={o.id}` |
| 4 | No loading, error or empty state; no `r.ok` check | Add states; throw on `!r.ok` |
| 5 | Race condition: switching trader quickly can show the *old* trader's data | `AbortController` + cleanup |
| 6 | `<tr>` directly in `<table>` (no `<tbody>`); `trader` not URL-encoded | `<thead>/<tbody>`, `encodeURIComponent` |

### Corrected version
```jsx
function OrdersTable({ trader }) {
  const [orders, setOrders] = useState([]);          // 1
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();         // 5
    setLoading(true);
    setError(null);

    fetch(`/api/orders?trader=${encodeURIComponent(trader)}`, { signal: controller.signal })
      .then(r => {
        if (!r.ok) throw new Error(`HTTP ${r.status}`); // 4
        return r.json();
      })
      .then(setOrders)
      .catch(err => {
        if (err.name !== 'AbortError') setError(err.message);
      })
      .finally(() => {
        if (!controller.signal.aborted) setLoading(false);
      });

    return () => controller.abort();                  // 5 cleanup
  }, [trader]);                                       // 2

  if (loading) return <p>Loading…</p>;
  if (error) return <p role="alert">Could not load orders: {error}</p>;
  if (orders.length === 0) return <p>No orders for {trader}.</p>;

  return (
    <table>
      <thead>
        <tr><th>Symbol</th><th>Qty</th></tr>
      </thead>
      <tbody>                                         {/* 6 */}
        {orders.map(o => (
          <tr key={o.id}>                             {/* 3 */}
            <td>{o.symbol}</td>
            <td>{o.qty}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Grading Part 3:** 4+ bugs = 3 · 3 = 2 · 1–2 = 1 · 0 = 0. **For Susan this part matters more** (front-end gap on her CV): finding bugs 1 and 2 is the minimum you want to see.
