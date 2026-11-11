# Live Exercise: 10 minutes

**Rules:** no AI, no internet, no phone. **Python or JavaScript/TypeScript**, your choice. Exact pseudocode is acceptable. Do Part 1 first. Think out loud.

---

## Part 1: Code (about 5 min)

Write `fill_rate_by_trader(orders)` that returns one entry per trader:

- `trader`
- `ordered`: total ordered quantity
- `filled`: total filled quantity
- `fill_rate`: `filled / ordered` as a percentage, 1 decimal place

**Ignore orders with status `"REJECTED"`.** Sort by `fill_rate`, highest first; on a tie, by trader name.

```python
orders = [
    {"id": 1, "trader": "AMY", "symbol": "700.HK",  "qty": 1000, "filled_qty": 1000, "status": "FILLED"},
    {"id": 2, "trader": "BEN", "symbol": "9988.HK", "qty": 500,  "filled_qty": 200,  "status": "PARTIALLY_FILLED"},
    {"id": 3, "trader": "AMY", "symbol": "0005.HK", "qty": 400,  "filled_qty": 0,    "status": "REJECTED"},
    {"id": 4, "trader": "BEN", "symbol": "700.HK",  "qty": 500,  "filled_qty": 500,  "status": "FILLED"},
    {"id": 5, "trader": "AMY", "symbol": "7203.T",  "qty": 1000, "filled_qty": 600,  "status": "PARTIALLY_FILLED"},
]

fill_rate_by_trader(orders)
# -> ?   (write down the expected output)
```

---

## Part 2: Code review, Python API (about 3 min)

This endpoint creates an order. **List every problem you see** and how you'd fix it.

```python
@app.route("/orders", methods=["POST"])
def create_order():
    data = request.json
    qty = data["qty"]
    price = float(data["price"])
    db.execute(
        "INSERT INTO orders (symbol, side, qty, price) VALUES ('%s', '%s', %s, %s)"
        % (data["symbol"], data["side"], qty, price)
    )
    return {"status": "ok"}, 200
```

---

## Part 3: Bonus, React (if time remains)

What's wrong with this component?

```jsx
function OrdersTable({ trader }) {
  const [orders, setOrders] = useState();

  useEffect(() => {
    fetch(`/api/orders?trader=${trader}`)
      .then(r => r.json())
      .then(data => setOrders(data));
  }, []);

  return (
    <table>
      {orders.map(o => <tr><td>{o.symbol}</td><td>{o.qty}</td></tr>)}
    </table>
  );
}
```
