# Quote stuffing alert: limit far from market — draft reply

Draft reply to a compliance escalation (SMARTS "Possible Quote Stuffing") on a dark-seeking algo order whose client limit was far from the market.
All client, security, date and order details are replaced by placeholders.

---

**Subject:** RE: SMARTS | Escalation | Possible Quote Stuffing | `<SECURITY>` | `<TICKET>`

Hi `<COMPLIANCE>`,

Thanks for the details. Please find our review of the ISDARK behaviour on `<SECURITY>` (`<DATE>`) below.

**Client instruction**
The client sent two sell limit orders on `<SECURITY>` on the ISDARK strategy, with 10%–20% participation, valid for the day:
- one with a limit of `<LIMIT_1>`
- one with a limit of `<LIMIT_2>`

**1. Why the algo kept quoting at the limit prices while the market fell**
These were the client's limit prices. The algo never sells below the client's limit. When the stock fell well below them, both limits were simply out of reach, so the algo stayed at the limits and waited. It did not decide to quote away from the market on its own.

**2. Why the sell side looked so passive**
ISDARK looks for liquidity in dark pools first and posts on the lit market as a second step, always at or better than the client's limit. With the limits about 20% above the market, every resting order was far from the best price. This came from the client's price instructions, not from a decision by the strategy.

**3. Parameters, controls and market conditions**
- Participation set to 10%–20%, dark venues enabled.
- Price controls worked as designed: our internal price check rejected the lit child orders as off-market, and one dark pool disabled the orders because its price deviation check failed (desk alert: "PRICE VIOLATES DEVIATION THRESHOLD. Please call Desk").
- However, after each rejection the algo kept re-creating child orders (about every second) and re-sending its dark orders. Because the limits could never be reached, this repeated all day and produced the high message count flagged by SMARTS. This was not intended to place and cancel quotes; it was the algo retrying orders whose prices were far from the market. The order-to-trade ratio stayed low and there was no price impact.

**Next steps**
We are changing the algo so that when the client's limit is far from the market, or a price control rejects an order, it stops retrying, slows down its re-sends and alerts the desk. We will confirm the release date.

Please let us know if you need the order and child-order logs.

Regards,
`<NAME>`

---

## Internal notes (remove before sending)

- Symptom in the workorder table: tens of thousands of workorders for one parent; lit children at price 0 closed with `close_bad_price`, dark children at the limit cancelled with `cxl:goal_change_reduce` and re-sent.
- Reconcile the lit messages the exchange saw with what the workorder table shows:

```q
select n:count i, minT:min time, maxT:max time by id_target, venue, price, state from workorder where date=<DATE>, id_target in <TARGET_IDS>
```

- Confirm the meaning of `close_bad_price` and `bps_candidate = -1000000` in the code.
- Agree the wording of point 3 (admits the retry loop) with the manager / desk head before sending.
