# Live Exercise: 10 minutes

**Rules:** no AI, no internet, no phone. **Python or JavaScript/TypeScript**, your choice. Exact pseudocode is acceptable. Do Part 1 first. Think out loud.

---

## Part 1: Code (about 5 min)

**Slippage** measures how much worse than the *arrival price* (the market price when the order arrived) we traded, in basis points (1 bp = 0.01%):

- BUY: `(price - arrival_price) / arrival_price * 10000`
- SELL: `(arrival_price - price) / arrival_price * 10000`

(Positive = we did worse than arrival.)

Write `slippage_by_symbol(fills)` that returns, **for fills with status `"FILLED"` only**, one entry per symbol:

- `symbol`
- `qty`: total filled quantity
- `slippage_bps`: the **quantity-weighted** average slippage, 2 decimal places

Sort by `slippage_bps`, **worst (highest) first**.

```python
fills = [
    {"symbol": "700.HK",  "side": "BUY",  "qty": 100, "price": 301.0, "arrival_price": 300.0, "status": "FILLED"},
    {"symbol": "700.HK",  "side": "BUY",  "qty": 300, "price": 300.0, "arrival_price": 300.0, "status": "FILLED"},
    {"symbol": "9988.HK", "side": "SELL", "qty": 200, "price": 79.6,  "arrival_price": 80.0,  "status": "FILLED"},
    {"symbol": "9988.HK", "side": "SELL", "qty": 100, "price": 80.4,  "arrival_price": 80.0,  "status": "FILLED"},
    {"symbol": "0005.HK", "side": "BUY",  "qty": 100, "price": 60.6,  "arrival_price": 60.0,  "status": "REJECTED"},
]

slippage_by_symbol(fills)
# -> ?   (write down the expected output)
```

---

## Part 2: Code review, Python API (about 3 min)

This endpoint returns the VWAP of a symbol. **List every problem you see** and how you'd fix it.

```python
cache = {}

@app.get("/stats/{symbol}")
async def stats(symbol: str):
    rows = db.execute("SELECT * FROM fills").fetchall()
    fills = [r for r in rows if r["symbol"] == symbol]
    total = sum(r["qty"] * r["price"] for r in fills)
    vwap = total / sum(r["qty"] for r in fills)
    cache[symbol] = vwap
    return {"vwap": vwap}
```

---

## Part 3: Bonus, React + TypeScript (if time remains)

What's wrong with this component?

```tsx
type Fill = { symbol: string; qty: number; price: number };

function LiveFills() {
  const [fills, setFills] = useState<Fill[]>([]);

  useEffect(() => {
    const ws = new WebSocket("wss://example.internal/fills");
    ws.onmessage = (e) => {
      fills.push(JSON.parse(e.data));
      setFills(fills);
    };
  }, []);

  return (
    <ul>
      {fills.map((f, i) => <li key={i}>{f.symbol} {f.qty} @ {f.price}</li>)}
    </ul>
  );
}
```
