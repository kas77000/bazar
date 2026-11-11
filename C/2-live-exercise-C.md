# Live Exercise: 10 minutes

**Rules:** no AI, no internet, no phone. **Python or JavaScript/TypeScript**, your choice. Exact pseudocode is acceptable. Do Part 1 first. Think out loud.

---

## Part 1: Code (about 5 min)

A trader's **net position** in a stock = shares bought − shares sold.

Write `positions(fills, limit)` that returns, **for fills with status `"FILLED"` only**, one entry per symbol:

- `symbol`
- `net_qty`: BUY adds, SELL subtracts
- `breach`: `True` if the absolute net position is greater than `limit`

Sort by the **size** of the position (absolute value), largest first.

```python
fills = [
    {"symbol": "700.HK",  "side": "BUY",  "qty": 300, "status": "FILLED"},
    {"symbol": "700.HK",  "side": "SELL", "qty": 100, "status": "FILLED"},
    {"symbol": "9988.HK", "side": "SELL", "qty": 500, "status": "FILLED"},
    {"symbol": "9988.HK", "side": "BUY",  "qty": 200, "status": "REJECTED"},
    {"symbol": "0005.HK", "side": "BUY",  "qty": 150, "status": "FILLED"},
]

positions(fills, limit=250)
# -> ?   (write down the expected output)
```

---

## Part 2: Code review, Python + SQL (about 3 min)

This function returns the total filled quantity per symbol for one day (`day` is like `"2026-11-12"`).
**List every problem you see.** Then, if you can, **rewrite it as a single SQL query.**

```python
def get_symbol_report(conn, day):
    orders = conn.execute("SELECT * FROM orders").fetchall()
    report = {}
    for o in orders:
        if str(o["created_at"])[:10] == day:
            fills = conn.execute(
                "SELECT * FROM fills WHERE order_id = " + str(o["id"])
            ).fetchall()
            for f in fills:
                report[o["symbol"]] = report.get(o["symbol"], 0) + f["qty"]
    return report
```

---

## Part 3: Bonus, React (if time remains)

What's wrong with this component?

```jsx
function AlertBanner({ alerts }) {
  const [visible, setVisible] = useState(alerts.length > 0);

  return visible && (
    <div className="banner">
      {alerts.length} alerts
      <button onClick={setVisible(false)}>Dismiss</button>
    </div>
  );
}
```
