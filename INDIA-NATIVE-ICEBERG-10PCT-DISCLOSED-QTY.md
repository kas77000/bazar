# India: Native Iceberg with a 10% Disclosed Quantity

## The goal

For Indian orders (NSE `IS`, BSE `IB`), send a **native exchange iceberg** where the disclosed (shown) quantity is **10% of each child order**.

Example: a child order of 100 shares goes to the exchange with a disclosed qty of 10. The exchange then refills 10 at a time until the 100 is done.

Today India uses **synthetic icebergs**: the algo itself sends small visible orders one after another. The exchange never sees one large order with a hidden part.

## Can we just send tag 73?

**Not for a percentage.** Sub-tag `73` (inside tag `9012`) is the client's iceberg/display size. It accepts only:

| Value | Meaning |
|---|---|
| `73=10` | Show a fixed 10 shares |
| `73=AVG3` | Show 3 × the stock's average trade size |
| `73=AQS0.5` | Show 0.5 × the stock's average quote size |
| `73=10,<mode>,<randomize>` | Fixed size, plus optional iceberg mode and randomization |

There's no percent form. `73=10%` is invalid.

A fixed `73=10` is also **not** 10% of each child. It's a ceiling applied to every child: a child of 50 still shows 10 (20%), and a child of 1,000 shows 10 (1%).

So tag 73 only works if the client works out 10% of the full order and sends that number. Even then it's a fixed size, not 10% of each child.

## The proposal

Do it in config, per market, so that every Indian child order gets a 10% disclosed qty automatically. Three changes:

### 1. Set the minimum display to 10% of the order for India

```
min_display: NO NMS CI 0.1ORD MM 0.05ORD IS 0.1ORD IB 0.1ORD
```

- `0.1ORD` means "display at least 10% of the order size".
- The existing entries (`NO NMS CI 0.1ORD MM 0.05ORD`) are today's defaults. They must be restated, because setting this key replaces the whole default list.
- The display is rounded **up** to the lot size. Indian cash equities trade in lots of 1, so 10% of 100 = 10, and 10% of 105 = 11.

### 2. Switch India from synthetic to native iceberg

Remove `IS` and `IB` from the synthetic iceberg market list:

```
synthetic_iceberg_markets: HK AO AT SP JT JF JX JQ JN JO JS IJ MK NZ NV PM TB KP KQ TT CG CS BD FS PK SL VH VM
```

While India is on this list, the algo keeps slicing the order itself and never sends a disclosed qty to the exchange. Step 1 has no effect until this is done.

*Alternative:* leave the list alone and switch synthetic off per order (synthetic iceberg mode = 0) for flows that want native. That's safer for a pilot, but every flow that wants native has to opt in.

### 3. Turn off size randomization for India (optional)

The display size is multiplied by `1 + random × iceberg_size_randomize_factor` so that it varies from order to order. For an **exact** 10%, set this factor to 0 for India. Keep it above 0 if a slightly varying display (for example 10–12%) is acceptable or wanted.

## What each tag and setting does

| Tag / setting | Role |
|---|---|
| `207=IS` / `207=IB` | Indian venue (NSE / BSE). Picks the India-specific settings. |
| `9012` | Algo parameter string. Holds the sub-tags below. |
| `9012` → `73` | Client's desired display size (fixed, `AVGn`, or `AQSn`). If sent, it's used together with the 10% rule (see caveats). |
| `min_display` (`0.1ORD`) | Per-market minimum display: 10% of each child order. |
| `synthetic_iceberg_markets` | Markets where the algo simulates icebergs itself. India must be removed for native. |
| `iceberg_size_randomize_factor` | Random uplift on the display size. Set to 0 for an exact 10%. |
| Outbound display tag (FIX `111` MaxFloor, or the venue's DisclosedQty tag) | Carries the disclosed qty to the exchange. It's mapped in the FIX gateway, so confirm it for NSE/BSE. |

## Worked example

Parent: buy 1,000 shares `207=IS`, VWAP. Randomization is 0 and the client sends no tag 73.

| Child qty | Disclosed qty sent |
|---|---|
| 100 | 10 |
| 250 | 25 |
| 37 | 4 (10% = 3.7, rounded up) |

## Pros

- **Regulatory fit.** SEBI/NSE require the disclosed qty to be **at least 10%** of the order qty. This rule enforces that floor on every child order.
- **Better queue priority.** A native iceberg keeps its place at the exchange and refills there. A synthetic iceberg has to send a new order for each slice, and each new order goes to the back of the queue.
- **Fewer messages.** One child order replaces many small slices. That means less order traffic and lower order-to-trade ratio pressure at the exchange.
- **Scales with order size.** The display is always proportional, unlike a fixed tag 73 value.
- **Config only, per market.** No client change, and the change can be reverted by restoring the two settings.

## Cons

- **Less control than synthetic.** Once a child is at the exchange, the exchange decides when to refill, not the algo. The algo can't pause, re-price, or vary each slice.
- **More information leaked to the market.** Each refill appears at the same price with the same size, and some participants watch for that. Synthetic slices can vary price, size, and timing.
- **A higher qty at risk.** The whole child (100) sits at the exchange, not just the visible slice (10). If the market moves fast, more can fill at a stale price before the algo cancels.
- **Changes behavior for all Indian flow** if India is removed from the synthetic list globally (step 2), not just for the client who asked.

## Caveats

1. **10% is a floor, not an exact value.** If the client sends no tag 73, the system still works out a default display from the stock's average trade size and uses whichever is **larger**. On a liquid stock, the disclosed qty can be well above 10%. That still meets SEBI, but it isn't exactly 10%. An exact 10% needs a small logic change: skip the average-trade-size default when a percentage rule applies.
2. **Tag 73 also acts as a floor.** If the client sends `73=50` on a child of 100, the disclosed qty is 50 (the larger of 50 and 10).
3. **Small orders.** If the display ends up equal to the whole child qty (for example a child of 1, or a large tag 73 on a small child), the order goes out fully visible **without** an iceberg.
4. **The venue must accept reserve orders.** The Indian exchange definitions must allow reserve/iceberg orders, or the disclosed qty is forced to 0 and the order goes out fully visible.
5. **The FIX mapping must be confirmed.** The gateway must send the display on the tag NSE/BSE expect for disclosed qty, and the exchange must accept it for the order type (for example, not on IOC or market orders).
6. **Order types.** Disclosed qty applies to **day limit** orders only. IOC, auction, and market orders are sent without a disclosed qty.
7. **Exchange minimums.** Check current NSE/BSE rules for any minimum order value or disclosed-qty value for icebergs, and for derivatives, where the rules differ from cash equities.
8. **Testing.** Validate in UAT and the exchange simulator first: the disclosed qty on the wire, refill behavior on partial fills, and amend/cancel of a partly disclosed order.

## Recommendation

1. Start with the **per-order opt-out** of synthetic (step 2 alternative) plus the India `min_display` entry, as a pilot on one client flow.
2. Set randomization to 0 for India if the client wants exactly 10%. Otherwise keep it, for less predictable refills.
3. After UAT and a successful pilot, remove `IS`/`IB` from the synthetic market list to roll out to all Indian flow.
4. If an **exact** 10% (not "at least 10%") is a hard requirement, raise the small logic change from caveat 1 as a follow-up.
