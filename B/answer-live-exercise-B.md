# Answers: Live Exercise B (Nicholas Au)
**Interviewer copy. Do not share.** Handout: `2-live-exercise-B.md`

---

## Part 1: `slippage_by_symbol(fills)`

### Expected output
```python
[
  {"symbol": "9988.HK", "qty": 300, "slippage_bps": 16.67},
  {"symbol": "700.HK",  "qty": 400, "slippage_bps": 8.33},
]
```
**How it's computed:**

| Fill | Side | Formula | bps | × qty |
|---|---|---|---|---|
| 700.HK 100 @ 301 (arr. 300) | BUY | (301 − 300) / 300 × 10000 | +33.33 | 3333.3 |
| 700.HK 300 @ 300 (arr. 300) | BUY | (300 − 300) / 300 × 10000 | 0 | 0 |
| 9988.HK 200 @ 79.6 (arr. 80) | SELL | (80 − 79.6) / 80 × 10000 | +50 | 10000 |
| 9988.HK 100 @ 80.4 (arr. 80) | SELL | (80 − 80.4) / 80 × 10000 | −50 | −5000 |
| 0005.HK (REJECTED) | | **excluded** | | |

- **700.HK** = 3333.3 / 400 = **8.33 bps**
- **9988.HK** = (10000 − 5000) / 300 = **16.67 bps** → worst, so it comes first

### Common mistakes
| Mistake | Wrong result |
|---|---|
| Ignores the SELL sign (uses `price − arrival` for everything) | 9988.HK = **−16.67**, sorted **last** |
| Simple average instead of quantity-weighted | 700.HK = (33.33 + 0) / 2 = **16.67**; 9988.HK = (50 − 50) / 2 = **0** → order flips |
| Includes the REJECTED fill | An extra **0005.HK = 100 bps** row at the top |
| Forgets × 10000 | Values like 0.0017 (a units mistake; ask what a basis point is) |

### Solution: Python
```python
from collections import defaultdict

def slippage_by_symbol(fills):
    qty = defaultdict(int)
    weighted = defaultdict(float)
    for f in fills:
        if f["status"] != "FILLED":
            continue
        diff = f["price"] - f["arrival_price"]
        if f["side"] == "SELL":
            diff = -diff
        bps = diff / f["arrival_price"] * 10_000
        qty[f["symbol"]] += f["qty"]
        weighted[f["symbol"]] += bps * f["qty"]

    result = [
        {"symbol": s, "qty": qty[s], "slippage_bps": round(weighted[s] / qty[s], 2)}
        for s in qty
    ]
    return sorted(result, key=lambda r: r["slippage_bps"], reverse=True)
```

### Solution: TypeScript
```ts
type Fill = {
  symbol: string; side: 'BUY' | 'SELL'; qty: number;
  price: number; arrival_price: number; status: string;
};
type SymbolSlippage = { symbol: string; qty: number; slippage_bps: number };

function slippageBySymbol(fills: Fill[]): SymbolSlippage[] {
  const acc: Record<string, { qty: number; weighted: number }> = {};
  for (const f of fills) {
    if (f.status !== 'FILLED') continue;
    const sign = f.side === 'SELL' ? -1 : 1;
    const bps = (sign * (f.price - f.arrival_price)) / f.arrival_price * 10_000;
    acc[f.symbol] ??= { qty: 0, weighted: 0 };
    acc[f.symbol].qty += f.qty;
    acc[f.symbol].weighted += bps * f.qty;
  }
  return Object.entries(acc)
    .map(([symbol, { qty, weighted }]) => ({
      symbol, qty, slippage_bps: Math.round((100 * weighted) / qty) / 100,
    }))
    .sort((a, b) => b.slippage_bps - a.slippage_bps);
}
```

**Complexity:** O(n) + O(k log k).
**Edge cases to ask about:** `arrival_price = 0` (division by zero), unknown side, empty list, float precision on money.

### Grading Part 1
- **3:** correct output, SELL sign + weighting + REJECTED all handled, mentions an edge case
- **2:** correct logic, small slip
- **1:** one of the three traps missed
- **0:** two or more traps missed, or no working approach

---

## Part 2: FastAPI `/stats/{symbol}`, bugs

| # | Problem | Why it matters | Fix |
|---|---|---|---|
| 1 | `SELECT * FROM fills` loads **the entire table** | Millions of rows per request | `WHERE symbol = ?` |
| 2 | Filtering and aggregating in Python | Slow, memory-heavy | `SUM(qty*price) / SUM(qty)` in SQL |
| 3 | `ZeroDivisionError` when the symbol has no fills | 500 error | Return 404 (or `null` VWAP) |
| 4 | **Blocking** DB call inside `async def` | Freezes the whole event loop, so every request stalls | Plain `def` (FastAPI runs it in a thread pool) or an async DB driver |
| 5 | Global `cache` written but never read, never invalidated, grows forever | Memory leak; stale data if someone starts reading it | Remove it, or use a TTL cache invalidated on new fills |
| 6 | No date filter | All-time VWAP, not today's | `AND ts >= ? AND ts < ?` (today's bounds in UTC) |
| 7 | Float money | Precision errors | `NUMERIC` / integer ticks / `Decimal` |
| 8 | `SELECT *` | Fetches unneeded columns, fragile | Name the columns |

**Bonus points:** an index on `(symbol, ts)`; validating `symbol`; a response model (Pydantic).

### Corrected version
```python
from fastapi import HTTPException

@app.get("/stats/{symbol}")
def stats(symbol: str):                            # 4. sync def → runs in a thread pool
    start, end = today_bounds_utc()                # 6. today only
    row = db.execute(                              # 1, 2, 8. filter + aggregate in SQL
        """
        SELECT SUM(qty * price) AS notional, SUM(qty) AS qty
        FROM fills
        WHERE symbol = ? AND ts >= ? AND ts < ?
        """,
        (symbol, start, end),
    ).fetchone()

    if row is None or not row["qty"]:             # 3. no division by zero
        raise HTTPException(status_code=404, detail=f"No fills for {symbol} today")

    return {"symbol": symbol, "qty": row["qty"],
            "vwap": row["notional"] / row["qty"]}  # 7. prices stored as NUMERIC/ticks
# 5. global cache removed
# Index: CREATE INDEX idx_fills_symbol_ts ON fills(symbol, ts);
```

**Grading Part 2:** 6+ bugs = 3 · 4–5 = 2 · 3 = 1 (minimum pass) · 0–2 = 0. **From Nicholas, expect 5 or more**, including the `async` blocking issue.

---

## Part 3: React + TS `LiveFills`, bugs

| # | Problem | Fix |
|---|---|---|
| 1 | `fills.push(...)` **mutates state**, and `setFills(fills)` passes the same reference → React sees no change → **no re-render** | Create a new array |
| 2 | **Stale closure:** `onmessage` always sees the first render's `fills` | Functional update `setFills(prev => ...)` |
| 3 | No cleanup → socket stays open after unmount, duplicates in StrictMode | `return () => ws.close()` |
| 4 | The list grows forever → memory and rendering slow down over a trading day | Keep the last N (e.g., 500) |
| 5 | `JSON.parse` result isn't validated, so the `Fill` type is a lie | Type guard (or zod) |
| 6 | Index as `key` while prepending items → wrong DOM reuse | Stable id from the server |
| 7 | No `onerror`, no reconnect, no "disconnected" indicator | Reconnect with a delay, show status |

### Corrected version
```tsx
type Fill = { id: string; symbol: string; qty: number; price: number };
const MAX_FILLS = 500;

function isFill(x: unknown): x is Fill {
  if (typeof x !== 'object' || x === null) return false;
  const f = x as Record<string, unknown>;
  return typeof f.id === 'string' && typeof f.symbol === 'string'
      && typeof f.qty === 'number' && typeof f.price === 'number';
}

function safeParse(text: string): unknown {
  try { return JSON.parse(text); } catch { return null; }
}

function LiveFills() {
  const [fills, setFills] = useState<Fill[]>([]);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    let ws: WebSocket;
    let retryTimer: ReturnType<typeof setTimeout> | undefined;
    let stopped = false;

    const connect = () => {
      ws = new WebSocket('wss://example.internal/fills');
      ws.onopen = () => setConnected(true);
      ws.onmessage = (e) => {
        const data = safeParse(e.data);
        if (!isFill(data)) return;                                  // 5
        setFills(prev => [data, ...prev].slice(0, MAX_FILLS));     // 1, 2, 4
      };
      ws.onerror = () => ws.close();                                // 7
      ws.onclose = () => {
        setConnected(false);
        if (!stopped) retryTimer = setTimeout(connect, 2000);      // 7 reconnect
      };
    };

    connect();
    return () => {                                                  // 3 cleanup
      stopped = true;
      clearTimeout(retryTimer);
      ws.close();
    };
  }, []);

  return (
    <>
      {!connected && <p role="status">Disconnected, reconnecting…</p>}
      <ul>
        {fills.map(f => (
          <li key={f.id}>{f.symbol} {f.qty} @ {f.price}</li>       // 6
        ))}
      </ul>
    </>
  );
}
```

**Grading Part 3:** 5+ bugs = 3 · 3–4 = 2 · 1–2 = 1 · 0 = 0. Bugs 1 and 2 (mutation and stale closure) are the important ones for someone who claims TypeScript and React experience.
