# Smart Close: send the continuous residual to the close (HK)

## Scope

| Order type | What happens today | Change needed |
|---|---|---|
| **Pure MOC** (`9001=MOC`) | Never trades in continuous. The whole order goes to the auction (`ConcertStrategy.java:869`) | **None** |
| **Smart close** (tag `351`) that trades before the close | Splits into a MOC leg and a continuous leg. **The continuous leg's leftover at 16:00 is not sent to the close** | **This proposal** |

Market: **HK** (continuous trading ends 16:00, closing auction 16:00–16:10).

## The problem

Smart close (`9012=351=1`) splits the parent order into two sub-orders (`FlexOrderStream.java:14849`):

1. **MOC leg:** sized from `26`/`27`, sent to the closing auction.
2. **Continuous leg:** the rest of the order, traded during the day.

The continuous leg is created with the **close turned off**:

```java
if (subsize>0) parameters.setDoClose(0);      // FlexOrderStream.java:14915
```

And once both legs exist, smart close does nothing more (`FlexOrderStream.java:14859`: `if (sysorder.subsysorders().length>=2) return;`).

So if the continuous leg has not finished by 16:00, **its leftover is cancelled and never reaches the auction**.

## Proposals

Both proposals do the same thing. B adds EOD mode so the full residual is shown on the book in the last 5 minutes.

**Proposal A: residual to the close after 16:00**
```
9001=IIS;9012=351=1,28800^15=1^232=0^26=20^27=11^SPB=0
```

**Proposal B: same as A + full residual on the book from 15:55**
```
9001=IIS;9012=351=1,28800^15=1^232=0^26=20^27=11^SPB=0^192=1,300000^EODBX=300000
```

| | Proposal A | Proposal B |
|---|---|---|
| Continuous leg until 15:55 | Normal IIS | Normal IIS |
| 15:55–16:00 | Normal IIS, keeps posting until 16:00 | **Whole residual posted**, no size cap |
| 16:00 | Residual sent to the close | Residual sent to the close |
| Best for | Liquid names, or where showing size is a concern | Names where a last push in continuous can fill more |

## How it fixes the problem

At 16:00 the continuous leg's strategy cancels its child orders and sends the residual to the auction (`ConcertStrategy.java:5086`, `overtimeHandling`). That step runs if **either** of these is true (`ConcertStrategy.java:5111`):

- the order has the close on (`doClose > 0`), **or**
- `27` is between **10 and 19** and `26 > 0`.

Smart close switches the close **off** on the continuous leg, so the first condition is false. **`27=11` meets the second one**, and the continuous leg inherits it from the parent. Its residual then goes to the auction anyway.

With `27=1` (the usual setting) neither condition is met, and the residual is lost.

## Timeline

| Time | MOC leg | Continuous leg |
|---|---|---|
| **Order start** | Created straight away, holding 20% of the order for the close | Created straight away (`351` field 2 = 8 hours, so it starts at order start) and trades the other 80% |
| **15:55** (B only) | – | **EOD switch:** whole residual posted on the book |
| **15:59:30** | – | Keeps posting (`SPB=0`) |
| **16:00** | Close order in the auction | Child orders cancelled → **residual sent to the close** |
| **16:10** | Auction matches | Auction matches |

## What each tag does

| Tag | Value | Meaning | Code reference |
|---|---|---|---|
| `9001` | `IIS` | Parent algo. Smart close copies it to the continuous leg | – |
| `351` field 1 | `1` | Turns smart close on. Also needs `26` > 0 and `27` ≥ 0, or it doesn't trigger | `Parameters.java:5102`, `:11067` |
| `351` field 2 | `28800` | Seconds before 16:00 when the continuous leg starts. **The default is 300 (only the last 5 minutes)**, so it must be set for trading through the day. 28800 = 8 hours = from order start | `FlexOrderStream.java:14853` |
| `351` field 3 | *(not sent)* | Seconds before 16:00 when the MOC leg is created. Default = at order start | `Parameters.java:5118` |
| `15` | `1` | Close on. **Required on HK short sells**: config rule 136 sets `15=0` on non-VWAP short sells | `Parameters.java:1838` |
| `232` | `0` | Must be 0 or the 16:00 residual step exits early | `ConcertStrategy.java:5108` |
| `26` | `20` | Size of the MOC leg: 20%. Change to suit | `Parameters.java:2252` |
| `27` | `11` | % of order, "after close" type. **This is the fix**: lets the continuous leg send its residual at 16:00 even with the close off | `ConcertStrategy.java:5111` |
| `SPB` | `0` | Keeps posting until 16:00. The server default pulls posts **30 seconds early** on must-complete orders, and IIS is must-complete by default | `ConcertStrategy.java:5319`, `ai.server.properties:236` |
| `192` (B) | `1,300000` | EOD mode, 5 minutes before the close: whole residual posted, no cap, no price layering. Never crosses the spread. **Keep at 1** (2 and 3 change venue and pricing) | `ConcertStrategy.java:3896` |
| `EODBX` (B) | `300000` | Wakes up the EOD check. Must be ≥ the `192` lead time, so keep them equal | `Parameters.java:8097` |

## Don't use

| Setting | Why |
|---|---|
| `27=1` | Standard type: the continuous leg's residual is lost at 16:00 |
| `26=100^27=1` on IIS | Silently converts the order to a MOC algo, so it stops trading in continuous (`FlexOrderStream.java:13972`) |
| `CSLE` / `CSLEE` | Only run when the order has the close on (`FlexOrderStream.java:17012`). The continuous leg has it off, so they do nothing |
| `192=2` or `3` | Change venue and pricing behaviour |
| `23=0` in the template | Can silently block the EOD switch (B) |

## To verify on a test order

- [ ] `setSmartCloseRelay:` shows `SMART_CLOSE` with `moc_rate_type` 11 and `moc_rate` 20
- [ ] `relaySmartCloseOrder: MOC` shows the MOC leg size (≈ 20% of the order)
- [ ] `relaySmartCloseOrder: residual` appears at order start, not at 15:55
- [ ] `<reset_IIS_to_MOC>` does **not** appear
- [ ] (B) `flip eod` on the continuous leg at 15:55
- [ ] At 16:00: the continuous leg's child orders are cancelled, then a close child order for the residual
- [ ] Not present: `overtimeHandling: not putting balance into auction`
