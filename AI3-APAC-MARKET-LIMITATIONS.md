# ai3 — APAC Market Limitations

_Last updated: 2026-11-12_

This is the list of known ai3 limitations per market. It was gathered from past prod-issue investigations, then checked against the code. It covers things ai3 can't do, does wrongly, or only does for some markets.

**Status values:**
- **Confirmed**: the code shows the limitation.
- **Workaround**: a tag or config change gets around it.
- **Proposed**: a fix was suggested but not applied or tested.
- **Open**: the cause is still unknown.

**Caveats:**
- Several core classes are **not in the ai3 repo**: `Statics` (`uptick_limit`, `enforcePriceLimits`), `SysOrder`, `Util`, `SecurityMaster`, `TickSizeRuleSet`, `PriceLimitRuleSet`. Root causes that depend on them could not be fully confirmed.
- Config line numbers refer to the local `config/ai.fixmsg.properties`. Production / APIB copies may differ.

**Market codes** (Bloomberg-style): JP / JT = Japan / Tokyo, KS = Korea, IN / IS / IB = India / NSE / BSE, TT = Taiwan, HK = Hong Kong, AU = Australia, CH / C1 / C2 = China, MK = Malaysia, TB = Thailand. Note that `CN` = **Canada** in this code base.

---

## Japan (JP / JT)

| # | Limitation | Status | Where |
|---|---|---|---|
| JP-1 | The short-sell restriction flag (`regSHOState`) can arrive from market data **after** the first order on a symbol. Rules in `ai.fixmsg.properties` run only once, when the order arrives, so the order trades as unrestricted until a cancel/replace. | Confirmed. Workaround: `enforce_regsho: US 1 JP 2`, or preload restricted names in `ai.stockevent.properties` | `Parameters.java:15866`, `FlexOrderStream.java:1898` |
| JP-2 | Restricted short-sell price enforcement is **off by default for JP**: `enforce_regsho` defaults to `"US 1"`. When enabled, it prices with the **US rule** (bid + 1 tick), not the Japanese rule (ask / ask + 1 tick depending on trend). | Confirmed. Options: `post_regsho_price_level`, tag 302 + `AUAA`, or a Java change in `ABSWorker` | `AlgoConfig.java:1859`, `PostWorker.java:724` |
| JP-3 | **Bug:** the third value of tag 302 (apply uptick limit) is read from `a[1]` instead of `a[2]`. | Confirmed bug | `Parameters.java:3928` |
| JP-4 | The close price on a restricted short sell ignores tag 331 (uptick only). The close is priced by LCR. The uptick override only fires when `regSHOState>0 && enforceRegSHO>0 && exchange=JT`, or when `RSHO=1`. | Confirmed. Proposed: `RSHO=1`, or add `uptick_only` to the condition | `FlexOrderStream.java:10903-10924`, `oeo/AuctionWorker.java:275-287` |
| JP-5 | The default close floor `LCR=0.0999,2` makes the close slice unfillable once the stock is already more than about 10% down (e.g. 6976.JP: floor 6850, market 6580). | Workaround: send `LCR=0` or a wider value. A code fix is proposed and needs compliance sign-off | `ai.fixmsg.properties` rules 162-163, `FlexOrderStream.java:11343` |
| JP-6 | At limit up/down, the algo correctly blocks the take. Its fallback `tryMustMakePost` then still posts a market order **at the limit price** (e.g. 5344.JP at 54000). The `halted` flag is only logged. | Confirmed, unresolved (desired behaviour not decided) | `oeo/DMAStrategy.java:493/668/735`, `oeo/ABSStrategy.java:1398-1405` |
| JP-7 | At limit down, LCR did not move a buy order's close price off the limit (6981.JP). In the CPL mode, a buy only checks limit up, and `if (Util.ZEROPRICE > 0)` skips LCR. | Open. Cause not confirmed because `enforcePriceLimits` is not in the repo | `FlexOrderStream.java:11404-11557` |
| JP-8 | Close child orders always use the parent's limit price, and no tag can price below the client limit (9984.JP). | Confirmed by design | `createAndScheduleMOCOrder` |
| JP-9 | If a stock **never opens**, `recompute()` returns early, so there is no slicing and no close (6787.JP). The open override `320=60^162=10` deliberately excludes JP. | Workaround: `9012=320=21600^162=10` for JP | `oeo/ConcertStrategy.java:6880`, `ai.fixmsg.properties:404` |
| JP-10 | "Send residual to close" is forced off on the Tokyo exchange (`CSLE=-1`). | Confirmed (config) | `ai.fixmsg.properties:561` |
| JP-11 | **Config typo:** `echangeCode!=JT`, so the JT exclusion in the `232=0` rule never applies. | Confirmed bug | `ai.fixmsg.properties:246` |
| JP-12 | The reject limit (REJECTTOOMANY, 3) applies to the whole instance. There is no per-market or short-sell-only override. | Workaround: `avoid_after_events REJECTED 105`, or `NOAREJ=1` for JP short sells | `AlgoConfig.java:3252`, `AlertMonitor.java:612` |
| JP-13 | Post-churn rejects on JP short sells: in the layered-post branch, posts are cancelled on half-tick moves without waiting a minimum duration, and QMD defaults to 0. | Workaround: send `QMD`, or set a venue minimum time | `PostWorker.java:440-465` |
| JP-14 | Restricted short sells get different tag 302 values (`600` vs `15,1`). The local config cannot produce `600`, so prod config probably differs. Tag `331=1` was also missing on APIH orders but present on APIC orders. | Open | `ai.fixmsg.properties:551-552` |

## Korea (KS / KP / KQ)

| # | Limitation | Status | Where |
|---|---|---|---|
| KS-1 | Close participation is sized from **estimated** close volume (`evclose`), and the close child is never resized. On thin names the order can go over the cap (ARROWST: 34% against a 14.5% cap). | Confirmed, no tag fix | `OrderHolder.java:520-522`, `ConcertStrategy.java:2201` |
| KS-2 | Several rules set `232=0` and override the client (rule 200 ARROWST, rules 198/199 VWAP+26/27, rule 247). Rule 306 sets `232=11`, which can let the close go well over the cap. | Fix line 625 (`232=1` for KS) added locally but untested. **Not yet in APIB/prod** | `ai.fixmsg.properties:198-200, 247, 306, 625` |
| KS-3 | No tag waits for the indicative close volume before sizing the close. `MOCWIB` only covers missing data. | Partial workaround: `histv_for_zero_theov: PM 0 KS 0` (side effects). A proper fix needs code (e.g. `MOCWIV`) | `RelayScheduler.java:833` |
| KS-4 | Residual-to-close with `27=10` ignores MaxPart, and 302 does not stop it. Some Korea short-sell 302 rules can never fire. | Confirmed | `ConcertStrategy.java:5122`, `KOREA-SHORTSELL-302-RESIDUAL-TO-CLOSE.md` |
| KS-5 | Close-only mode (`ks_close_mode=1`) is hardcoded for KS and blocks all continuous-session candidates. | Confirmed (by design) | `CandidateProcessor.java:286-290` |

## India (IN / IS / IB)

| # | Limitation | Status | Where |
|---|---|---|---|
| IN-1 | The open-auction child uses the parent limit with **no cap at the exchange price band** (the rule `164=0^LOR=0`), so orders were rejected at limit down. The band is stored but never read. | Proposed `LOR=0.029` rule (2% bands are still at risk). A real fix needs a code change in `OpenWorker` | `OpenWorker.java:192/200/243`, `ai.fixmsg.properties:137-138`, `INDIA-OPEN-AUCTION-LIMIT-CAP.md` |
| IN-2 | Open-auction tags: `LCT=4` only works at the close; `LORT` counts from the previous close, not the indicative auction price; `getTheoPrice` keeps the first indicative price and never refreshes it. | Confirmed. Closest workaround: `283=2^LOR=0.0007` | `OpenWorker.java` |
| IN-3 | `feed_ticksize: 2` replaces India's tick table from `ai.ticksize.txt`. | Proposed: `feed_ticksize: ALL 2 IN 0` | `ABSQuoteSource.java:2993/3002`, `FEED-TICKSIZE-MODES.md` |
| IN-4 | Tag 155 picks venues but not the tick size; children use the home exchange's tick. `TECT` has no effect. | Workaround: `mixed_ticksize: IN 1` | `Stock.java:2124-2172`, `INDIA-TICKSIZE-AND-ROUTING.md` |
| IN-5 | No native iceberg: IS/IB are in `synthetic_iceberg_markets`, and the display tag has no percentage form. | Proposed config change | `ai.server.properties:253`, `INDIA-NATIVE-ICEBERG-10PCT-DISCLOSED-QTY.md` |
| IN-6 | Every IIS order without 26/27 gets `26=25^27=0`, so the close is always 25% of expected close volume. | Confirmed (config) | `ai.fixmsg.properties:265, 268` |
| IN-7 | The India synthetic close relay covers only **IIS and MOC**, not VWAP. Close window defaults to 17:30-17:58. | Confirmed | `Parameters.java:7281/11086`, `RelayScheduler.java:1127` |
| IN-8 | SDMA "very aggressive" is still capped by MaxPart (e.g. 10%), and orders routed to BSE cannot hit NSE offers (DAEH.IN). | Likely cause, not confirmed | — |
| IN-9 | SEBI CAS work is not implemented. Cashfall/Portfall reject rule is pending the algorithm codes. | Open | `SEBI-CAS-Implementation-Master.md` |
| IN-10 | A compound leg does not inherit `155=SCBIN` from the parent, so the leg has no venue (`NO_VENUE`). | Workaround: `155` on each leg, or `237=155` | `ConcertStrategy.java:1044` |

## Hong Kong (HK)

| # | Limitation | Status | Where |
|---|---|---|---|
| HK-1 | **EOD venue switching only works in North America.** `enterEOD()` returns false outside NA, so in APAC the EOD mode shows the residual but never changes venue. `192` also needs `EODBX`, and `23=0` can silently block the switch. | Confirmed | `oeo/PostWorker.java:304`, `EOD-MODE-HK-SETUP.md` |
| HK-2 | A short sell cannot cross the spread; posting is the only tool. | Confirmed | `HK-SHORTSELL-PQS-EOD-BRIEFING.md` |
| HK-3 | Non-VWAP short sells get `15=0`, so there is no close participation unless the client sends `15=1`. | Workaround: send `15=1` | `ai.fixmsg.properties:135-136` |
| HK-4 | An open-auction short sell is forced to a limit at the previous close. | Confirmed (hardcoded) | `FlexOrderStream.java:9759-9764` |
| HK-5 | Rule 306 injects `232=11` on HK VWAP orders that don't send 232, which silently blocks the close. | Workaround: send `232=0` | `ai.fixmsg.properties:306` |
| HK-6 | The open price band is hardcoded to about 7 minutes after `hk_snap_quote_cutoff` (09:15). Likely bug: `ABSQuoteSource.java:459/482` compares against `".HK"` with a leading dot. | Confirmed / likely bug | `FlexOrderStream.java:9782-9858` |
| HK-7 | ISDARK with a limit far from the market loops (lit child at price 0, dark child re-sent): 21k work orders and a quote-stuffing alert (6869.HK). | Confirmed, back-off fix not implemented | — |
| HK-8 | A compound LINE auction leg can take the remainder just after the open, so VWAP never starts. No tag can end a leg relative to the open. | Workaround: SMOO auction leg | `CandidateRegulator.java:339` |
| HK-9 | A market-order compound auction leg goes out at price 0 because of `164=0^LOR=0` and is rejected. | Workaround: `LOR=0.05` (+`283=2`) on the leg | `ai.fixmsg.properties:138-139`, `OpenWorker.java:192` |

## Taiwan (TT)

| # | Limitation | Status | Where |
|---|---|---|---|
| TT-1 | No taking at the price limit. A single round-lot TMX order can be blocked all day by `block_last_worder_avoid_early_finish`. With `27=8`, the close reserve is fixed and never absorbs the leftover (3406.TT). | Root cause confirmed. Proposed: `SEB=0`, `CSLE=1` | `ABSStrategy.java:1394-1405`, `FlexOrderStream.java:8601-8615` |
| TT-2 | TMX orders with 26/27 need `SEB=0`. | Added to local config (line ~667). **Not in APIB** | `ai.fixmsg.properties` |

## Australia (AU) / Korea (HANWHA)

| # | Limitation | Status | Where |
|---|---|---|---|
| AU-1 | No `ALGO_EOD` destination is defined. The `192` residual falls back to a POST day order, which is cancelled at the end of continuous trading (`DERB` early expiry), so the residual never reaches the close. | Workaround: `192=3,1000^DERB=0` (not tested in UAT) | `venuefixtags.xml`, `WorkOrder.java:1396` |

## China (CH / C1 / C2)

| # | Limitation | Status | Where |
|---|---|---|---|
| CH-1 | Each close-auction reject cuts the order size by 10%, hardcoded. | Confirmed | `FlexOrderStream.java:12622` |
| CH-2 | The abnormal-trading-rule size check can warn or reject orders (`enable_abnormal_rule_check>2`). | Confirmed | `FlexOrderStream.java:11916-11930` |
| CH-3 | The STAR board `688*` regex in config is wrong. | Config bug | `AI-FIXMSG-CONFLICTS.md` C-18 |

## Malaysia (MK) / Thailand (TB)

| # | Limitation | Status | Where |
|---|---|---|---|
| MK-1 | Several MK short-sell rules (154/302/SEB) can never fire. | Config bug | `AI-FIXMSG-CONFLICTS.md` C-06 |
| TB-1 | The Thailand FMRPOP rule is missing (mislabelled as MK). | Config bug | `AI-FIXMSG-CONFLICTS.md` C-01 |

## All APAC — features gated to North America (`regionna()`)

These are hardcoded off in APAC:

- EOD venue switching / NINJA close: `oeo/PostWorker.java:304`
- IPO pre-open handling: `oeo/ConcertStrategy.java:222`
- NINJA passive/blind size (0 outside NA): `oeo/ConcertStrategy.java:3922`
- "Invite liquidity" iceberg display sizing: `oeo/ConcertStrategy.java:7539, 7571`
- Odd-lot venue routing: `ExchangeMaster.java:694`, `FlexOrderStream.java:8224`
- Open-order splitting: `ExecutionCustomizer.java:203`
- Aggressive finish near the close for market orders: `FlexOrderStream.java:10566`
- WXM mode 3 falls back to mode 1: `FlexOrderStream.java:9169-9175`

## General limitations (all markets)

- **Config rules run once.** `ai.fixmsg.properties` rules are checked only when the order arrives and cannot react to live prices or flags. Only the compound-trigger JS re-runs, and only for COMPOUND orders.
- **Close reserve is fixed.** The 26/27 close reserve is fixed once (`OrderHolder.java:440`), so the leftover from continuous trading is not added to the close. Workaround: `RMRAC` / `CSLE`.
- **`26=100^27=1` on IIS** silently turns the order into a MOC.
- **Smart close (`351`):** the continuous leg's leftover never reaches the auction, and `CSLE` doesn't apply.
- **Compound legs** don't inherit parent tags (`218`, `LOR`, `6`, `155`) unless they are listed in `237`.
- **The reject threshold is set per instance only** (it lives in an external library).
- **Rule order matters.** Many market rules only win because of where they sit in the file (tags 232, 15, 154). See `config/AI-FIXMSG-CONFLICTS.md`.
