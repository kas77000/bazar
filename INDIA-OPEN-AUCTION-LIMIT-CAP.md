# India: Cap the Open Auction Child Instead of Using the Parent Limit

## The problem

An Indian buy order (`9001=VWAP`, `40=2` limit, `44=345.8`, `207=IS`) was rejected at the open.

- The VWAP open slice (the opening auction child) was sent at the **parent limit, 345.8**.
- Previous close was **305.15** and the stock opened at **306.2**.
- 345.8 is **13.3% above the previous close** and about 13% above the open. That breaks the client's bespoke price check in tag 58: *"5% away warning 10% Halt & warning and MAX_RATE_APPLIED"*.

The open child should have been capped near the market, not sent at the parent limit.

## Why the child went at the parent limit

The open child's price is the **more conservative** of two prices:

1. The **auction limit**, computed from a base price and a band (`OpenWorker.computeAuctionLimitPrice`).
2. The **parent limit** (tag 44) (`AuctionWorker.computePrice`).

For Indian **limit** orders, no band is set:

- The only India rule that sets the band (`LOR=0.1`) applies to **market orders only** (`price<0.00001`).
- The next rule (`9012=164=0^LOR=0 :: compositeExchangeCode != NZ^PM^IJ^MK^C1^C2^CH^HK, ...`) then sets **both bands to 0** for Indian limit orders.
- With both bands at 0, no auction limit is computed, so the child takes the parent limit (345.8).

## The proposal

Add one rule to `config/ai.fixmsg.properties`:

```
#IN limit orders: cap the open auction child at prev close ±2.9% instead of the parent limit.
9012=LOR=0.029 :: compositeExchangeCode=IN, tag9012(LOR)=null, price>0, tag9012(155)!=SG
```

**Result on the rejected order:** buy cap = 305.15 × 1.029 ≈ **314.0**. The child would have gone at 314.0 instead of 345.8. That's about 2.5% from the 306.2 open: under both the 5% warning and the 10% halt, and it would have filled in the auction.

For a sell, the cap is mirrored: previous close × (1 − 0.029).

## Where to put it

Put the new rule **directly above** this existing line:

```
9012=164=0^LOR=0 :: compositeExchangeCode != NZ^PM^IJ^MK^C1^C2^CH^HK, tag9012(LOR)=null
```

The section should read:

```
9012=LOR=0.1 :: compositeExchangeCode=IN, tag9012(LOR)=null^0, price<0.00001, tag9012(155)!=SG
#IN limit orders: cap the open auction child at prev close ±2.9% instead of the parent limit.
9012=LOR=0.029 :: compositeExchangeCode=IN, tag9012(LOR)=null, price>0, tag9012(155)!=SG
9012=164=0^LOR=0 :: compositeExchangeCode != NZ^PM^IJ^MK^C1^C2^CH^HK, tag9012(LOR)=null
```

**Why the position matters:** every rule is checked top to bottom. The `164=0^LOR=0` rule only fires when `LOR` is not yet set (`tag9012(LOR)=null`). If the new rule sits **below** it, `LOR` is already 0 and the new rule's own `tag9012(LOR)=null` check blocks it. Placed **above**, the new rule sets `LOR=0.029` first, and the `164=0^LOR=0` rule skips the order.

**Other environments:** `config/APIB.ai.fixmsg.properties` has the same two lines. Add the rule there too, in the same position, if APIB should behave the same.

## What each tag does

### In the new rule

| Part | Meaning |
|---|---|
| `9012=LOR=0.029` | Sets the **open auction band** (tag 1527, `moo_limit_fraction`) to 2.9%. The open child's limit becomes base price × (1 ± 0.029): `+` for buys, `−` for sells. |
| `compositeExchangeCode=IN` | Indian stocks only. |
| `tag9012(LOR)=null` | Only applies if `LOR` is not already set, by the client or by an earlier rule. A client-sent `LOR` always wins. |
| `price>0` | **Limit orders only.** Market orders keep the existing `LOR=0.1` rule. |
| `tag9012(155)!=SG` | Skips orders whose sub-parameter 155 is `SG`, the same exclusion as the existing India market-order rule. |

### Related tags (not changed, but involved)

| Tag | Meaning | Value on this order |
|---|---|---|
| `LOR` (1527) | Open auction band as a fraction of the base price | `0` (set by the `164=0^LOR=0` rule) |
| `164` (`LOF`) | Open auction band based on **volatility** (factor × stock volatility / 15). If both 164 and `LOR` are set, the tighter one is used | `0` |
| `283` | **Base price** for the band. Not set / `0` = previous close. `2` = indicative equilibrium price (IEP), falling back to previous close. `3` = the more aggressive of previous close and IEP | not set → previous close |
| `LORT` (1624) | Alternative band in **ticks** from the previous close | not set |
| `206` | Open participation mode (`206=2` for VWAP by default) | `2` |
| `44` | Parent limit; the child can never be less conservative than this | `345.8` |

## Pros and cons

### Pros
- **Fixes the reject.** The open child stays within 2.9% of the previous close, so it passes the client's 5% warning and 10% halt checks.
- **Only touches config.** No code change and no release. One rule, easy to remove.
- **Uses a base price that's always there.** The previous close is always available. The IEP can be missing.
- **Matches existing India settings.** 2.9% is the same band already used for `MILR` / `MCLR` in India, and it stays inside SEBI's ±3% "passive limit" band.
- **Client override still works.** A client who sends their own `LOR` is not affected.
- **No impact on market orders or other markets.**

### Cons
- **Gap opens.** If the stock opens more than 2.9% away from the previous close in the order's direction (gap up for a buy, gap down for a sell), the open child won't fill in the auction. The VWAP still works the remaining quantity in continuous trading, but the auction fill is lost.
- **Applies to every Indian limit order,** not just this client. Orders whose parent limit was already inside the band are unaffected. Orders with a wide limit get a tighter open child than before.
- **Doesn't follow the auction.** The band is measured from yesterday's close, not today's IEP. On news days the IEP can move far from the previous close.

## Alternatives

| Option | Rule | When to use it |
|---|---|---|
| **A. Recommended** | `9012=LOR=0.029 :: ...` | Default. Stable, always has a base price. |
| B. Wider band | `9012=LOR=0.049 :: ...` | Get more auction fills on gap days while staying under the 5% warning. |
| C. Band around the IEP | `9012=283=2^LOR=0.029 :: ... , tag9012(283)=null` | Follow the auction price instead of the previous close. If the IEP is missing, it falls back to the previous close anyway. |
| D. Client-specific | Add a client condition (e.g. `fixTag(109)=...` or `basket~...`) to rule A | Limit the change to the client with the bespoke 5%/10% check. |

## Before rolling out

1. **Check the open price.** The stock was described as limit down, but the 306.2 open is *above* the 305.15 previous close. Confirm the open price and price band in market data. The fix works either way.
2. **Test in UAT.** Send one Indian buy and one sell limit VWAP order with a wide limit. In the engine log, the `MOO limit delta:` line should show `moo_limit_fraction = 0.029`, and the child limit should be previous close × (1 ± 0.029), not the parent limit.
3. **Check a client-sent `LOR`.** Send an order with its own `LOR` and confirm the rule does not override it.
4. **Check a market order.** Confirm it still gets `LOR=0.1`.

## Code references

| What | Where |
|---|---|
| `LOR` parsing (tag 1527) | `src/com/kas/ai/Parameters.java` — `__p_1527_LOR` |
| `164` / `LOF` parsing | `src/com/kas/ai/Parameters.java` — `__p_164_LOF` |
| `283` parsing (base price source) | `src/com/kas/ai/Parameters.java` — `__p_283` |
| Open auction limit calculation | `src/com/kas/ai/oeo/OpenWorker.java` — `computeAuctionLimitPrice` |
| Child price = more conservative of auction limit and parent limit | `src/com/kas/ai/oeo/AuctionWorker.java` — `computePrice` |
| Rules evaluated top to bottom | `config/AI-FIXMSG-CONFLICTS.md` |
