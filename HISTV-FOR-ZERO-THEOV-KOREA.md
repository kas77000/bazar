# `histv_for_zero_theov=KS 0`: what it does in Korea

## Why we're looking at it

Some ARROWST Korea orders took **more than 25% of the close**. We want the close sized **only from the
market's indicative volume (IV)**, never from our historical estimate. Adding `232=1` did not fix it.

## What the setting does

It is a server config, set per exchange. It decides what happens when the IV (`theov`) is **0 or missing**:

| Value | When IV is 0 / missing |
|---|---|
| `1` (default for every exchange not listed) | Replaces the IV with the **historical close volume** (`evclose`) and sizes from that |
| `0` | Leaves IV at 0, so there's **no historical fallback** |

- Default in code: `"PM 0"` (`AlgoConfig.java:3302`). Every other exchange falls back to `1`, so
  **Korea uses the historical fallback today** unless the live config overrides it.
- Main use, the close sizing in `tryClose` (`ConcertStrategy.java:2201`): the fallback is used only if
  IV < 1 **and** no non-zero IV has been seen yet for the order (`_max_matched_moc < 1`).

## What happens with `KS 0`

`tryClose` does nothing before the closing call starts, and from then on it re-sizes the close each time
it runs. With `KS 0`:

1. Closing call starts, no IV yet: size is about **0**, so nothing goes out.
2. IV shows up: the next run sizes from the **market IV**.

So in practice the close **waits for the IV** instead of trading on the estimate.

## Important: check first whether this was the cause

The fallback only matters when the IV was 0. Find the `moc imbmap:` log line (`ConcertStrategy.java:2182`)
for the bad orders:

- `theov` = 0 when the size was set: the fallback caused it, and `KS 0` fixes it.
- `theov` > 0 when the size was set: **this setting changes nothing**. Look at the cap mode (`232=11` vs
  `232=1`) or at the imbalance added to the IV.

## Gaps

| Gap | Detail |
|---|---|
| **Applies to all of Korea** | It's per exchange: every KS order, every client, not only ARROWST. No per-order tag overrides it. |
| **Opens are affected too** | Also used for the opening auction (`ConcertStrategy.java:5913`, `:5967`) and the PM open (`:6539`, `:6549`). |
| **The scheduler/OPE gets the change too** | `OrderHolder.java:1653` fills the order info with the historical value when IV is 0; with `0` it stays 0. |
| **No IV means no close** | Feed gap, illiquid name, or IV never published: the order sends nothing to the close and can be left unfilled. |
| **The imbalance can still size the close** | With IV = 0 but an imbalance present, the size can still come from the imbalance alone. |
| **Must-complete orders ignore it** | After the continuous session ends, a must-complete order is pushed to the full remaining quantity anyway. |
| **Some close-sizing paths ignore it** | Orders with a close rate (tag 26) sized in `RelayScheduler.java:833` use `closingVolume()`. RVWAP-to-close in `FlexOrderStream.java:2351` uses `evclose()`. Both are estimates. |
| **Not a fix if IV was there** | Over-participation from the cap mode or from the IV moving before the print is not touched. |

## Pros and cons

**Pros**
- Config only, with no code change or release.
- The close is sized from what the market shows, not from a forecast that can be too high.
- In practice it waits for the IV on the normal close path.

**Cons**
- It hits every Korea order and the opening auctions too, which is a wide blast radius for one client's issue.
- There's a risk of missing the close completely when the IV is late or missing.
- It doesn't cover every close path, or causes that have nothing to do with a missing IV.
- There's no way to scope it to ARROWST.

## Alternative: a per-order tag (needs a code change)

Add a tag (e.g. `MOCWIV=1`) checked in `tryClose`: if IV = 0 and no IV has been seen yet, log
"waiting for IV" and do nothing. It's turned on only where needed:

```
9012=MOCWIV=1 :: stripedBasketID=ARROWST, compositeExchangeCode=KS
```

Same effect, limited to ARROWST Korea, opens untouched. One open decision: if the IV never arrives, either
skip the close or fall back to the estimate in the last few seconds.

Existing tag `MOCWIB` does **not** do this: it only waits when the market data is completely missing,
and only for a few ms after the order is created (built for feed subscription delays).

## Caveats

- `SysOrder.java` (the exact `getMocSize` formula) is not in the workspace; that part comes from notes
  transcribed from photos.
- Check the **live** `histv_for_zero_theov` value; the one above is the code default.
