# feed_ticksize — what each mode does

`feed_ticksize` in `config/ai.server.properties` decides how much the algo trusts
the **market data feed** for tick size, versus the tick rules it loads itself
(`ai.ticksize.txt`).

Current setting (`config/ai.server.properties:74`):

```
feed_ticksize: 2
#feed_ticksize: ALL 1 KS 0
```

---

## How the setting is read

| Step | Where | What happens |
|---|---|---|
| Read the property | `AlgoConfig.java:1855` | Reads `feed_ticksize`. If the line is missing, the default is `"1"`. |
| Split per market | `AlgoConfig.java:1856` | Turns the text into a map of market → mode, e.g. `ALL 1 KS 0` means every market = 1, Korea = 0. |
| Look up one market | `AlgoConfig.feed_ticksize(cex)` (source line 6838) | Returns the mode for that market. If the market isn't in the map, it returns **0**. |
| Store on the stock | `Stock.java:771` | When the stock's composite exchange code is set (e.g. `IN`), the mode is saved on the stock as `useDTickSize()`. |

So the mode is chosen **per market, by composite exchange code** (`IN` for India,
`KS` for Korea, and so on).

---

## What the feed sends

Each market data message can carry two tick-related fields:

1. **A tick size value**: one number, e.g. `0.05`. (`msg.tickSize()`)
2. **A tick size table ID**: points to a full price-band table. (`msg.tickSizeTableId()`)

The table IDs come from MBREF (`stype=ticksizetableid`), the same data that
`src/com/kas/ai/kdb/load_ticksizeids.q` copies into kdb.

---

## The modes

### Mode 0: don't take the table from the feed

- The feed's **table ID is ignored**. The stock keeps the tick table it was given
  at startup (from `ai.ticksize.txt` / security master).
- The feed's **single tick value is still saved** on the stock as the "dynamic tick"
  (`dTickSize`) and sent to the security master. This happens in every mode.
  See `ABSQuoteSource.java:2992-3000` (trades) and `ABSQuoteSource.java:3648-3656`
  (other updates).

### Mode 1: use the feed's tick value

- In the code available here, mode 1 behaves **the same as mode 0**: the table ID
  is ignored and the single value is saved.
- The name `useDTickSize` suggests mode 1 means "use the feed's dynamic tick value
  when pricing". But the code that would read it (`Util.tickSizeFeed`,
  `SecurityMaster`, the `TickSize` classes) **is not in this project**, so I can't
  confirm what changes.
- This is the default when the `feed_ticksize` line is missing.

### Mode 2: also take the tick table from the feed

- Everything mode 0/1 does, **plus**:
- If the feed sends a tick size table ID that differs from the stock's current one,
  the algo **replaces the stock's table with the feed's table**, and tells the
  security master.
  See `ABSQuoteSource.java:3002-3024` and `ABSQuoteSource.java:3658-3680`.
- It does this for the main stock and, for custom composites, for the per-exchange
  sub-stock (e.g. the IS or IB line of an India dual listing).
- This is the only mode with code here that really changes things: **the feed /
  MBREF table overrides the local file**.

### Summary

| Mode | Saves feed's single tick value | Replaces tick table with feed's | Net effect |
|---|---|---|---|
| 0 | yes | **no** | Local tick table stays in charge |
| 1 | yes | **no** | Same as 0 in visible code; probably "use feed value" in code not here |
| 2 | yes | **yes** | Feed / MBREF table overrides the local one |

---

## Examples

```
feed_ticksize: 2              # all markets take the table from the feed
feed_ticksize: ALL 2 IN 0     # all markets mode 2, except India uses local table
feed_ticksize: ALL 1 KS 0     # all markets mode 1, Korea mode 0
```

A plain number like `2` appears to apply to all markets. The line that splits it
(`setStrIntMap`) isn't in this project, so that's based on how the file is used.

---

## How to check what a stock is using

The stock's log line has a field (`Stock.java:2955`):

```
|dticksize:<feed tick value>:<mode>
```

- The last number is the mode this stock got.
- If you set India to 0, India stocks should show `:0`.

Then compare a real order price against the tick you expect from `ai.ticksize.txt`.

---

## Related settings

- `mixed_ticksize`: for dual-listed or composite stocks, picks which exchange's tick
  to use. Explained in the section below.
- `tickfilter_bid_tick_diff`, `tickfilter_ask_tick_diff`, `tickfilter_last_tick_diff`
  (`ai.server.properties:241-243`): throw away feed prices that jump too many ticks.
  They use the feed tick size (`ABSQuoteSource.java:2588-2640`, `3036-3053`).

---

## mixed_ticksize: which exchange's tick to use for a dual listing

A dual-listed stock (e.g. India on IS and IB) has a tick table for its **home
exchange** plus one per **other listing** (its "sub-stocks"). `mixed_ticksize`
decides which one to use when the algo asks for a tick for a *different* exchange
than the home one.

### How it's set

- Property: `mixed_ticksize` in `ai.server.properties`. The default is `0` (off). It's
  not in your config now (`AlgoConfig.java:3611`).
- Set per market, same format as `feed_ticksize`: `mixed_ticksize: IN 1`.
- It can be changed while running (it's read through `setField`, which reloads).
- Saved on the stock when the composite exchange code is set (`Stock.java:772`).

### The number has 3 digits (`Stock.java:2257-2262`)

| Digit | Meaning |
|---|---|
| hundreds | `1` = write a `mixedTick:` log line every time it's used |
| tens | mode for **feed** prices (`tickFeed`) |
| ones | mode for **order** prices (`tickSize`, `closestTick`, `nextUpTick`, ...) |

Examples: `1` = orders mode 1. `11` = feed and orders mode 1. `101` = orders mode 1
with logging.

### The modes (`Stock.java:2115-2178`)

First, two shortcuts that skip the modes:
- If the exchange asked for **is the home exchange**, the home tick is used.
- If the mode is `0` (or 6 and above), the home tick is used.

Otherwise:

| Mode | What tick is used |
|---|---|
| 1 | The tick of the **exchange asked for** (that listing's own table) |
| 2 | **LCM** of all listings' ticks: the coarsest grid, a price valid on every exchange |
| 3 | **GCD** of all listings' ticks: the finest common grid |
| 4 | The **smallest** tick of all listings |
| 5 | The **largest** tick of all listings |

If the result is 0 or missing, it falls back to the home tick.

### Things to be aware of

- **For order prices, modes 2-5 still try the exchange asked for first.** The code
  checks that listing's own tick before LCM/GCD/min/max (`Stock.java:2129`). So for
  orders, 2-5 only kick in if that listing has no tick table. For feed prices, only
  mode 1 does that check, and 2-5 go straight to LCM/GCD/min/max.
- **It only changes prices that ask for a specific exchange.** In the code here, the
  main one is the final rounding of a child order sent to another listing's venue
  (`FlexOrderStream.java:11602`). Most pricing calls the home exchange, so it's not
  affected.
- The feed-side caller (`tickFeed`) isn't used in this project's code. It's
  probably called from `Util.tickSizeFeed`, which isn't here.

---

## Which exchange's tick a child order uses (India dual listings)

Sending an order only to IS (e.g. `9012=155=EXCEPT;<IB venue>`) does **not** switch
it to the IS tick. Tag 155 only chooses venues.

- Normal pricing uses the **home exchange's** tick (`sorder.primaryExchange()`).
- A final rounding (`FlexOrderStream.java:11602`) asks for the venue's exchange tick
  when the child goes to a different exchange than the stock's home.
- That only returns the venue's tick if `mixed_ticksize` is 1-5 for the market
  (`Stock.java:2124`). It's not set in the config, and the default is `0`, so the
  home exchange's tick is used (`Stock.java:2172`).

To round IS child orders to the IS tick at the final step: `mixed_ticksize: IN 1`.

---

## What this project doesn't have

These would show the full effect of mode 1 and how the startup table is loaded:

- `Util.java` (`tickSizeFeed`, `setStrIntMap`, `intFromMap`)
- `SecurityMaster` implementation (`setStockTickSize`, `setStockTickSizeTableId`)
- `TickSizeRuleSet` / `TickSize` / `ExchangeImpl`
- `ai.ticksize.txt` itself
