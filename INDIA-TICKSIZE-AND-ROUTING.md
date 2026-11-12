# India: tick size and IS / IB routing

How the algo gets the tick size for Indian stocks, how to make it use the local
tick file instead of the feed / kdb, and how to force an order to NSE (IS) instead
of BSE (IB). It also covers why forcing the venue does **not** change the tick size.

Exchange codes used in the code:

| Code | Meaning |
|---|---|
| `IN` | India composite (the market as a whole) |
| `IS` | NSE |
| `IB` | BSE |

All file paths are in the `ai3` project. More detail on the two tick settings is
in [FEED-TICKSIZE-MODES.md](FEED-TICKSIZE-MODES.md).

---

## 1. Where the tick size comes from

There are two sources.

### a) The local file: `ai.ticksize.txt`

- Loaded at startup by `ExchangeMaster.java:243`:
  `TickSizeRuleSet.config(dataDirectory() + "ai.ticksize.txt", _index_memberships);`
- Lives in the server's **data directory**, next to `ai.venue.properties` and
  `ai.pricelimit.ruleset`, not in `config/`.
- Holds the tick rules for every exchange, India included.
- The rule interface is `IFCTickSizeRule.java` (tickSize, closestTick, nextUpTick, ...).

### b) The feed / kdb (MBREF)

Each market data message can carry:
1. **A single tick value**, e.g. `0.05` (`msg.tickSize()`).
2. **A tick size table ID** (`msg.tickSizeTableId()`), which points to a full table.

The table IDs come from MBREF (`stype=ticksizetableid`).
`src/com/kas/ai/kdb/load_ticksizeids.q` copies them into kdb and writes
`<qcfgdir>/ref/ticksizeids_<region>`.

---

## 2. Making the algo use the file, not kdb: `feed_ticksize`

Setting in `config/ai.server.properties:74`. Today it's:

```
feed_ticksize: 2
```

It's set **per market**, by composite exchange code (`ALL 1 KS 0` = all markets 1,
Korea 0). The default when the line is missing is `1`. A market not listed gets `0`.

| Mode | What it does |
|---|---|
| 0 | Ignores the feed's table ID. The stock keeps its table from `ai.ticksize.txt`. |
| 1 | In the code we have, same as 0. Probably means "use the feed's single tick value", but the code that would do that isn't in the project. |
| 2 | **Replaces the stock's tick table with the feed's (MBREF) table** whenever they differ (`ABSQuoteSource.java:3002`, `:3658`). |

In **every** mode, the feed's single tick value is still saved on the stock as the
"dynamic tick" (`dTickSize`) (`ABSQuoteSource.java:2992`, `:3648`).

### To make India use the local file

```
feed_ticksize: ALL 2 IN 0
```

All other markets keep mode 2, and India stops taking the table from the feed.

What this does **not** cover (the code isn't in the project, so not confirmed):
- Whether the saved single feed value is still used to price India orders.
- Whether the security master loads India's table IDs from the kdb file
  (`ticksizeids_<region>`) when the server starts. If it does, this setting won't
  stop that.

### How to check

The stock's log line has `|dticksize:<feed value>:<mode>` (`Stock.java:2955`).
After the change, India stocks should end in `:0`. Then compare a real order price
with the tick you expect from `ai.ticksize.txt`.

---

## 3. Forcing an order to IS instead of IB

### TECT: no effect

`config/ai.fixmsg.properties:630-633` (under `# Indian CAS changes`) sets
`9012=TECT=IS` or `TECT=IB` on orders with `155=SG`. No Java code in the project
reads `TECT`, and testing showed it **has no effect** on routing. Don't rely on it.

### How the algo picks venues (`ExchangeMaster.java:1445`)

1. It starts with the venues of the stock's **home exchange** (IS or IB).
2. If tag `202` is above 0 and the stock is dual-listed, it adds the other
   exchange's venues.
3. Tag `155` then filters that list for this order (`ExchangeMaster.java:1352-1366`).

### The tags

| Tag (in 9012) | What it does | Code |
|---|---|---|
| `155` | Venue filter. Lists allowed venue ids / brokers, or `EXCEPT;<ids>` to block some. Parts joined with `&`. Can be per aggression level: `<LEVEL>;<ids>`. | `Parameters.java:2674`, `:11616` |
| `202` | Dual trading. `0` = stay on the home exchange only. Above 0 = also use the other listing. Reset to 0 if the stock isn't dual-listed. | `Parameters.java:2965`, `:12552` |
| `318` | `1` or more turns on the "synthetic India router", which trades both IS and IB. Keep it `0` if you want only IS. | `Parameters.java:4259` |

### Examples

```
9012=155=<IS venue id>                # only IS venues
9012=155=EXCEPT;<IB venue id>         # everything except IB
9012=155=SG&EXCEPT;<IB venue id>      # keep the existing SG filter AND block IB
9012=202=0                            # don't use the second listing (works if home is IS)
```

Watch out: the India config already sets `155=SG` on many orders. Putting
`EXCEPT;...` alone in 155 **replaces** `SG`. Use the `SG&EXCEPT;...` form to keep both.

The real venue ids and broker names for India are in `ai.venue.properties` on the
server (not in the project). 155 matches either an exchange's id or its broker.

---

## 4. Forcing IS does not give you the IS tick

Tag 155 only chooses **where** the order goes. It doesn't change which tick is used.

- **Normal pricing** uses the **home exchange's** tick (`sorder.primaryExchange()`,
  e.g. in `CandidateProcessor` and `Statics.adjustToTick...`).
- **Final rounding** (`FlexOrderStream.java:11602`, called at `:11660`): if the
  child goes to a venue on a different exchange than the home one, it asks for that
  exchange's tick.
- That only returns the other exchange's tick if **`mixed_ticksize`** is 1-5 for
  India. It isn't in the config and the default is `0`, so it returns the home
  exchange's tick (`Stock.java:2172`).

So:
- Home exchange **IS**: you get the IS tick, because IS is the home exchange.
- Home exchange **IB**: an IS-only order still uses the **IB tick**.

Which exchange is "home" for each India dual-listed stock is set in the security
master data, which isn't in the project. Check it per stock.

---

## 5. mixed_ticksize: which listing's tick to use

Setting in `ai.server.properties` (not there today, default `0`, `AlgoConfig.java:3611`).
Per market, like `feed_ticksize`. Can be changed while running.

**The value has 3 digits** (`Stock.java:2257`):

| Digit | Meaning |
|---|---|
| hundreds | `1` = log every use (`mixedTick:` lines) |
| tens | mode for feed prices |
| ones | mode for order prices |

**The modes** (`Stock.java:2115-2178`). If the exchange asked for is the home one,
or the mode is 0, the home tick is used. Otherwise:

| Mode | Tick used |
|---|---|
| 1 | The tick of the exchange asked for |
| 2 | LCM of all listings' ticks (coarsest, valid everywhere) |
| 3 | GCD of all listings' ticks (finest common grid) |
| 4 | The smallest tick |
| 5 | The largest tick |

If the result is missing or 0, it falls back to the home tick.

For order prices, modes 2-5 still try the asked-for exchange's own tick first, so
they rarely matter there.

### To round IS child orders to the IS tick

```
mixed_ticksize: IN 1
```

This only changes the **final rounding** step. Normal pricing still uses the home
exchange's tick.

---

## 6. Summary: what to set

| Goal | Setting |
|---|---|
| India uses `ai.ticksize.txt`, not the feed's table | `feed_ticksize: ALL 2 IN 0` |
| Send an order only to IS | `9012=155=SG&EXCEPT;<IB venue id>` (or a list of IS venues), with `318=0` |
| Don't use the second listing at all | `9012=202=0` (only if the home exchange is IS) |
| IS child orders rounded to the IS tick | `mixed_ticksize: IN 1` |

## 7. Still to confirm on the server

- The India lines in `ai.ticksize.txt`.
- The India venue ids / brokers in `ai.venue.properties` (to fill in tag 155).
- The home exchange of each India dual-listed stock (security master).
- Whether mode 1 / the saved feed value changes India prices (`Util.tickSizeFeed`,
  `SecurityMaster`, not in the project).
- A test order after each change: check the log `dticksize` field and the child
  order prices and venues.
