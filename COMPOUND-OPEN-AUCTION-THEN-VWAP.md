# Compound: Open Auction at 20%, then VWAP

## Proposals

Both proposals do the same thing. They differ only in the unit used to set the buffer after the open.

**Proposal A: buffer in minutes (1 minute)**
```
9001=COMPOUND;9012=139=3;9060=M,C,-60_1=LINE^217=1^218=20^283=2^LOR=0.05,1_1=VWAP^6=10
```

**Proposal B: buffer in seconds (30 seconds)**
```
9001=COMPOUND;9012=139=3;9060=S,C,-3600_1=LINE^217=1^218=20^283=2^LOR=0.05,30_1=VWAP^6=10
```

| | Proposal A (minutes) | Proposal B (seconds) |
|---|---|---|
| Unit header | `M` | `S` |
| Leg 1 starts | `-60` = 60 minutes before the open | `-3600` = 3600 seconds (1 hour) before the open |
| Switch to VWAP | `1` = 1 minute after the open | `30` = 30 seconds after the open |
| Finest buffer possible | 1 minute | 1 second (any value, e.g. `15`, `45`, `90`) |
| Best for | Markets with a random end or frequent delayed opens | Markets with a clean, on-time open |

## What it does

1. **Before the open:** sends a **limit** auction order for **20% of the auction's matched volume**, capped at the order size. The limit is the theoretical open price ± 5% (previous close ± 5% if no theoretical price is available yet).
2. **After the buffer (1 minute in A, 30 seconds in B):** closes the auction leg and starts **VWAP** on the remaining quantity (order size minus filled), with a **10% max participation**.

Whatever the auction doesn't fill goes to the VWAP leg.

## What each element does

| Part of request | Element (A / B) | Meaning | Code reference |
|---|---|---|---|
| Top-level tag | `9001=COMPOUND` | Uses the relay / compound engine | `RelayScheduler.java:1334` |
| Inside `9012` | `139=3` | Compound mode 3 = time-based tracker. Unlike mode 2, it does **not** drop the first leg when the order arrives before the open | `Parameters.java:2588`, `FlexOrderStream.java:14431` |
| Top-level tag | `9060=...` | Holds the compound config: headers, then one entry per leg, all separated by **commas** | `Parameters.java:11343`, `Parameters.java:12451` |
| `9060` header 1 | `M` / `S` | Unit of the thresholds: **minutes** (A) or **seconds** (B). `H` = hours, `T` = clock time | `CompoundTimeTracker.java:285` |
| `9060` header 2 | `C` | Thresholds are measured from the **start of continuous trading** (the open) | `CompoundTimeTracker.java:146` |
| Leg 1 start | `-60_` / `-3600_` | Leg 1 is active from 1 hour before the open. Anything earlier also uses leg 1 | `CompoundTimeTracker.calcNewMode` |
| Leg 1 param | `1=LINE` | Leg 1 algorithm: LINE (the inline algo; `INLINE` is **not** a valid name) | `Algorithm.java` |
| Leg 1 param | `217=1` | Open auction (MOO) rate type 1 = size based on the auction's indicated matched volume | `Parameters.java:3196`, `ConcertStrategy.java:5904` |
| Leg 1 param | `218=20` | MOO rate = 20% (of the auction volume, per `217=1`) | `Parameters.java:3226` |
| Leg 1 param | `283=2` | Base price for the auction limit = **theoretical open price**. Falls back to previous close if not available | `Parameters.java:3751`, `OpenWorker.java:214` |
| Leg 1 param | `LOR=0.05` (tag 1527) | Auction limit = base price ± 5%. Setting it also stops the server rule that forces the band to 0 | `Parameters.java:6450`, `config/ai.fixmsg.properties:138` |
| Leg 2 start | `1_` / `30_` | **The buffer:** switches to leg 2 at 1 minute (A) or 30 seconds (B) after the open | `CompoundTimeTracker.calcNewMode` |
| Leg 2 param | `1=VWAP` | Leg 2 algorithm: VWAP, on the remaining quantity | `RelayScheduler.java:627` |
| Leg 2 param | `6=10` | Max participation rate 10% | `Parameters.java:1502` |

## Why we have a buffer

The buffer is the short wait between the scheduled open and the switch to VWAP. It protects the auction fill.

**1. The scheduled open is not the real open.**
`C` is the time the exchange is *scheduled* to start continuous trading. The stock's real auction (the uncross) can happen later:
- **Random end:** some markets end the auction at a random moment within a short window (often up to 30 seconds).
- **Delayed open:** a stock can open late because of a large imbalance or a price-limit / volatility check.
- **Auction extension:** the exchange can extend the auction for a stock.

**2. The switch cancels the auction order.**
When the time passes the threshold, the engine closes the auction leg (`RelayScheduler.java:672`). If the auction hasn't happened yet, the auction order is cancelled and **the auction fill is lost**. That auction trade, the whole point of leg 1, never happens.

**3. The fill needs time to come back.**
The VWAP leg is sized as *order size minus filled*. The buffer lets the auction fill report arrive first, so VWAP starts on the right remaining quantity.

**Example (scheduled open 09:00:00):**

| Time | No buffer (switch at 09:00:00) | With 30s buffer (switch at 09:00:30) |
|---|---|---|
| 09:00:00 | Scheduled open → auction order cancelled, VWAP starts on 100% of the order | Auction order still live |
| 09:00:12 | Stock actually uncrosses (random end) → **we are not in it** | Stock uncrosses → **auction fill received** |
| 09:00:30 | — | Switch → VWAP starts on the remaining quantity |

**What the buffer costs:** VWAP starts 30 to 60 seconds later. Over a full trading day this barely moves the result.

**Picking the buffer size:**
- **30 seconds (Proposal B):** fine where the open is clean and on time.
- **60 seconds or more (Proposal A, or B with `60_`/`90_`):** where the open has a random end or delayed opens are common.
- A buffer won't fully protect a stock that opens very late (several minutes). In that case the auction leg can still be cancelled before the uncross.

## Issues found during testing

| Error | Cause | Fix |
|---|---|---|
| `SUBSYSORDER_ERROR` | Leg 1 used `1=INLINE`. `INLINE` isn't in the `Algorithm` enum or the shorthand map, so the leg resolved to `UNKNOWN` and failed validation (`FlexOrderStream.java:13339`). The parent is then rejected (`FlexOrderStream.java:14767`) | Use `1=LINE` |
| `Invalid Order: Price is 0 but price mode is open` | The rule at `config/ai.fixmsg.properties:138` injects `164=0^LOR=0` when the order doesn't set `LOR`. With both bands at 0, no auction limit is computed (`OpenWorker.java:206`), so the auction child is sent as a market order with price 0 and rejected downstream | Add `283=2^LOR=0.05` to leg 1 so the engine computes a limit from the theoretical price |

## Changes from the original request

Original: `9001=COMPOUND^9012=139=2^9060=M^0_1=INLINE^217=1^218=20^220=1^1_1=VWAP^6=10`

| Change | Why |
|---|---|
| `139=2` → `139=3` | Mode 2 deletes the first leg if the stock hasn't opened yet, so a pre-open order would run as pure VWAP with no auction leg |
| Added `C` header | Without it, the thresholds count from the order's start time, not the open (e.g. an order sent at 08:30 would switch at 08:31, before the auction) |
| `0_` / `1_` → negative start + buffer | These numbers are start offsets (time from the reference), **not** leg numbers |
| `^` → `,` between header and legs inside `9060` | The parser splits legs on commas; with `^` it won't parse |
| Removed `220=1` | "Stop after MOO" is skipped for sub-orders, so it does nothing inside a leg (`CandidateProcessor.java:225`). On the parent it could reject the order after the open |
| `INLINE` → `LINE` | `INLINE` is not a valid algorithm name (caused `SUBSYSORDER_ERROR`) |
| Added `283=2^LOR=0.05` | Gives the auction order a limit price based on the theoretical open (fixes the price-0 reject) |

## Variants

| Goal | Change |
|---|---|
| 20% of the **order** at the open, instead of 20% of auction volume | `217=1` → `217=100` (`ConcertStrategy.java:5856`: `moosize = 0.20 × order size`) |
| Tighter or wider auction limit | Change `LOR=0.05` (e.g. `0.02` = ±2%, `0.10` = ±10%) |

## To verify on a test order

- [ ] Log line `MOO limit delta:` for the auction leg shows `_theoprice` and a final `limit` **above 0**
- [ ] Log line `compoundTimeTracker thresholds:` shows the switch at open + 1 minute (A) or open + 30s (B)
- [ ] The smart-open log shows the auction size ≈ 20% of the indicated matched volume
- [ ] After the switch, the VWAP leg size = order size − auction fill

Checking the legs in kdb (OMS), where `p` is the compound's `id_target`. Each leg is a `target` row whose `id_source` is the parent id (`FlexOrderStream.java:2877`):

```q
d:.z.d; p:123456;
select id_server,id_target,algo,size,make,avg_fill_price,t_start,t_end,fixmsg from target where date=d, id_source=p
select from compoundalgo where date=d, id_target=p      / switch history
subs:exec id_target from target where date=d, id_source=p;
select from execution where date=d, id_target in subs   / fills per leg
```
Add `id_server=<server>` to the filters if ids repeat across servers.
