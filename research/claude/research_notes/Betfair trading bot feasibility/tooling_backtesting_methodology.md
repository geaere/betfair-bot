# Python tooling, system architecture and research methodology for a Betfair Exchange bot (as of 15 Nov 2026)

Researcher notes on provenance. Three kinds of evidence are used and labelled throughout:
- **[fetched]** = page or API read directly in this session (PyPI JSON API, GitHub API, raw GitHub files, Betfair developer docs via the Confluence REST API, flumine docs, Betfair Data Scientists hub, Nautilus docs, Quantpedia).
- **[search summary]** = taken from a web-search result summary; the underlying page was not opened. Treat as lower confidence.
- **[prior knowledge]** = standard literature or practice cited from the researcher's background knowledge; the link is the canonical location but the page was NOT fetched this session. The report writer should treat these as correct-in-substance but unverified for exact wording/numbers.

---

## 1. Main Python libraries and their maintenance status in 2026

### Takeaway
The betcode-org stack (betfairlightweight + flumine) is alive and very actively released in 2026 (flumine 3.4.1 on 14 Nov 2026; betfairlightweight 2.24.0 on 4 Sep 2026) and remains the default recommendation for a Python hobbyist. Nautilus Trader's Rust-backed Betfair adapter is the serious alternative; `betfair_data` (Rust file reader) is unmaintained since 2022, while `betfair_parser` (limx0) is active but requires Python >= 3.12.

### Cited Findings

**betfairlightweight (bflw)** — API-NG wrapper + streaming
- Latest version 2.24.0, uploaded 2026-09-04; previous 2.23.2 on 2026-03-16; requires Python >= 3.9; summary "Lightweight python wrapper for Betfair API-NG" [fetched] — [PyPI](https://pypi.org/project/betfairlightweight/)
- GitHub repo betcode-org/betfair: last push 2026-11-13, 515 stars, 3 open issues, not archived [fetched] — [GitHub](https://github.com/betcode-org/betfair)
- Can parse/output historical data the same way it processes streaming data; optional C and Rust libraries speed things up [search summary] — [PyPI](https://pypi.org/project/betfairlightweight)
- Installing `betfairlightweight[speed]` "will have a big impact on processing speed due to the inclusion of C and Rust libraries for datetime and json decoding" [fetched] — [flumine performance docs](https://github.com/betcode-org/flumine/blob/master/docs/performance.md)

**flumine** — event-driven trading framework (live, paper, simulation)
- Latest stable 3.4.1 released 2026-11-14; 3.0.0 final released 2026-06-29 after betas b0-b12 (5-22 Jun 2026); 3.1.0 on 24 Jul; 3.2.x through Sep-Oct; 3.3.0/3.3.1 (16 Oct) were yanked for a regression; 2.12.0/2.12.1 (26 Feb 2026) yanked for a bug. Last 2.x release was 2.13.11 on 2026-05-05. Requires Python >= 3.10, tested to 3.14 [fetched] — [PyPI](https://pypi.org/project/flumine/)
- GitHub betcode-org/flumine: last push 2026-11-14, 252 stars, 20 open issues [fetched] — [GitHub](https://github.com/betcode-org/flumine)
- Feature list: event-based strategy execution, custom strategies, risk-management controls, modular components, simulation and paper trading, market/order/custom streams, multi-venue concurrent operation, multiple strategies and clients, middleware and background workers (Scores, RaceCard, InPlayService) [fetched] — [PyPI](https://pypi.org/project/flumine/)
- Venues in 3.x: Betfair fully supported; Betconnect account + execution; Tote in development; Betdaq "no longer available" [fetched] — [PyPI](https://pypi.org/project/flumine/)
- Dependencies: betfairlightweight, betconnect, tenacity, python-json-logger, requests, smart-open (stream files from S3) [fetched] — [PyPI](https://pypi.org/project/flumine/)

**betfairutil** (mberk) — utility functions
- Latest 0.7.1 uploaded 2025-05-07 (0.7.0 on 2025-01-09; before that 0.6.3 in Aug 2023); requires Python >= 3.9; "Utility functions for working with Betfair data"; repo last pushed 2025-05-07, 37 stars [fetched] — [PyPI](https://pypi.org/project/betfairutil/), [GitHub](https://github.com/mberk/betfairutil)

**betfair_data** (tarb) — Rust-backed historical file reader
- Latest 0.3.4 uploaded 2022-04-08; requires Python >= 3.7; repo last pushed 2022-05-05, 47 stars, 15 open issues — effectively unmaintained [fetched] — [PyPI](https://pypi.org/project/betfair-data), [GitHub](https://github.com/tarb/betfair_data)
- Described as a "very fast, Rust based" parser supporting official historic data and self-recorded stream files; reads bz2, gzip or plain JSON and .tar/.zip archives of them [search summary] — [PyPI](https://pypi.org/project/betfair-data)

**betfair_parser** (limx0) — typed parser for Betfair API/stream messages
- Latest 0.21.2 uploaded 2026-11-10 (0.20.0 on 16 Sep, 0.21.0 on 29 Sep, 0.21.1 on 9 Oct 2026); requires Python >= 3.12; repo last pushed 2026-11-10, 7 stars, 0 open issues [fetched] — [PyPI](https://pypi.org/project/betfair-parser/), [GitHub](https://github.com/limx0/betfair_parser)

**Nautilus Trader Betfair adapter**
- nautilus-trader latest stable 1.231.0, with 2.0.0 release candidates rc6-rc9 published 5 Oct - 10 Nov 2026; requires Python >= 3.12, < 3.15; repo pushed 2026-11-15, ~31.5k stars [fetched] — [PyPI](https://pypi.org/project/nautilus-trader/), [GitHub](https://github.com/nautechsystems/nautilus_trader)
- Adapter is implemented in Rust and exposed to Python, shipped in the main package; covers instrument discovery, live market data, account state, order management and execution via Betting, Accounts and Exchange Streaming APIs; optional race (RCM) and cricket (CCM) streams [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- `BetfairDataLoader` converts recorded stream files (NDJSON, optionally .gz/.bz2) into instruments, order-book deltas, trade ticks, status and close events; trade ticks are derived from cumulative volumes so loader state persists across lines (call `reset` between files) [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- Limitations: no stop/trailing/bracket/iceberg orders, no post_only/reduce_only, no position querying, no cleared-order history, size increases unsupported, `use_market_version` currently has no effect (no price protection), MARKET orders only as AT_THE_CLOSE (BSP) [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- Adapter rate buckets: general HTTP 5/s, order endpoints 20/s; it cites Betfair limits of 1,000 instructions/s for order ops and a recommended 5 req/s per market for listMarketBook; warns that a delayed app key conflates the stream to multi-minute updates that exceed its recovery limits (i.e. Nautilus live needs a live key) [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- A worked "book imbalance" Betfair backtest tutorial exists [search summary] — [Nautilus tutorial](https://nautilustrader.io/docs/latest/tutorials/backtest_book_imbalance_betfair)

**Other**
- `betfair-scraper` 1.0 (2020) is a stale website scraper, not relevant [fetched] — [PyPI](https://pypi.org/project/betfair-scraper/)
- Non-Python: a .NET client `BetfairDotNet` exists [search summary] — [NuGet](https://feed.nuget.org/packages/BetfairDotNet)

### Inferences
- Recommended default for this reader: **flumine 3.x + betfairlightweight[speed]** for recording, simulation, paper and live; **betfairutil** for turning stream files into DataFrames for research; consider **Nautilus Trader** only if the reader wants a stricter, Rust-speed, order-book-delta engine and accepts Python >= 3.12 and a steeper learning curve.
- flumine 3.0 (Jun 2026) was a major version with a rapid patch cadence afterwards (two yanked releases in Oct 2026). Pin exact versions, read HISTORY/changelog before upgrading, and re-run a fixed regression backtest after each upgrade. Many tutorials online (Betfair Data Scientists hub, blog posts) were written against flumine 1.x/2.x and may need import/API adjustments.
- `betfair_data` still works for bulk file parsing but should be treated as frozen; do not build a pipeline that depends on new wheels for future Python versions.
- "betcode" is the GitHub organisation (betcode-org) that hosts betfairlightweight and flumine, not a separate fork.

### Gaps
- I could not confirm what changed in flumine 3.0 vs 2.x (changelog not read); breaking changes are likely but unverified.
- The betfair_parser GitHub repo has no description; whether Nautilus 2.0 still depends on it or has fully moved to its own Rust loader was not confirmed.
- No evidence was found of a genuinely new (2025-2026) Python Betfair framework beyond these; absence of evidence only.
- Status of the betcode Slack community (still open, invite link) was not verified this session.

---

## 2. How flumine's simulation/backtest works, its limitations, and differences from live

### Takeaway
flumine replays recorded/purchased stream files through the same strategy code used live, swapping in a simulated client and `SimulatedExecution` that adds latency, bet delay, liquidity-based queue position, lapsing on market-version change and non-runner reductions. It is passive-replay: your orders never change the book, cross-matching is absent, cancellations ahead of you in the queue are not modelled, and liquidity can be double counted — so it is optimistic for passive/queue strategies and for anything larger than small stakes.

### Cited Findings
- Setup: `FlumineSimulation(client=clients.SimulatedClient())` plus `BetfairHistoricalStream(file_path=...)`; strategy code is added exactly as for live [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- Simulation "tries to accurately simulate matching" with: estimated place/cancel/replace latency (configurable), Betfair betDelay added based on market state, queue positioning based on liquidity available, order lapse on market version change, order lapse and reduction on runner removal; BSP is also simulated [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- Default simulated latencies in `flumine/config.py`: place 0.120 s, cancel 0.170 s, update 0.150 s, replace 0.280 s [fetched] — [flumine config.py](https://github.com/betcode-org/flumine/blob/master/flumine/config.py)
- Documented limitations (quickstart, referencing issue #192): queue cancellations, double counting of liquidity, currency fluctuations [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- Issue #192 ("Simulation improvements", closed June 2020) shows two items struck through as not done: "Update PIQ based on 'potential' cancellations" and "Prevent double counting of liquidity i.e. if you take £x sitting at y should x still be available on the next update"; implemented items include validating market open/runner active, marketVersion validation, lapsing non-PERSIST limit orders on market version updates, and non-runner adjustment of matched and resting orders [fetched] — [flumine issue #192](https://github.com/betcode-org/flumine/issues/192)
- Config knobs: `simulated_strategy_isolation` (default True: orders matched per strategy; False prevents double counting of passive liquidity across all orders regardless of strategy); `simulation_available_prices` (default False; when True simulates matches against available prices after initial execution — "note this will double count liquidity") [fetched] — [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md)
- By default the backtest streams each market in turn, whereas live receives updates for all markets in time order; `event_processing=True` merges an event's markets (e.g. win/place, all football markets) and delivers them in original order; `event_groups` allow grouped parallel simulation [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies), [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- The simulator patches the system clock (`utcnow`) to event time; background workers do not work well in backtests, so middleware should be used instead [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies), [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md)
- Market catalogue (runner names/metadata) is not in purchased stream files; it is available in backtests only if recorded at the time (usually with the marketrecorder example) and loaded via middleware [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies)
- Betfair's own tutorial lists simulation limitations: no market catalogue data; cross matching (virtual bets) not included (said to matter little for racing); simulated bets do not affect other participants, which matters for large stakes, high odds, or trading models; advice is to keep stakes realistic [fetched] — [Betfair Data Scientists: flumine simulations](https://betfair-datascientists.github.io/tutorials/flumineSimulations)
- That tutorial uses PRO files only (not tested on BASIC/ADVANCED) and does not discuss commission [fetched] — [Betfair Data Scientists: flumine simulations](https://betfair-datascientists.github.io/tutorials/flumineSimulations)
- Performance: simulation is CPU-bound; use multiprocessing (one process per core, ~8 markets per process chunk "optimal", prevents memory leaks); `listener_kwargs` such as `{"seconds_to_start": 600, "inplay": False}` or `{"inplay": True}` cut the updates processed; keep files local (S3 via smart_open is supported but slower); disable logging; remove simulation middleware when only collecting data [fetched] — [flumine performance docs](https://github.com/betcode-org/flumine/blob/master/docs/performance.md)
- Other `listener_kwargs`: `max_inplay_seconds`, `event_type_ids`, `calculate_market_tv`, `cumulative_runner_tv` [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- Paper trading: `clients.BetfairClient(trading, paper_trade=True)` — "Market data will be recieved as per live but any orders will use Simulated execution and Simulated order polling to replicate live trading"; the Betfair website can be used to validate the market visually [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)

### Inferences
- Expected direction of sim-vs-live divergence: (a) passive (resting) orders fill more often in simulation than live because queue-ahead cancellations are not modelled the way they benefit/hurt you and because your own displayed size does not deter or attract others; (b) aggressive orders in thin books look better than live because you do not move the price and the same liquidity can be consumed repeatedly; (c) strategies relying on cross-matched (virtual) prices in football/tennis-style markets are mis-modelled; (d) fixed 120 ms place latency is optimistic or pessimistic depending on hosting — set it from measured live latencies.
- Commission is a per-market net-winnings charge, so it should be applied in post-processing of per-market P&L rather than per bet; because the Betfair tutorial does not cover it, the reader must add it explicitly (see section 4).
- A sound practice is to treat simulation as a hypothesis filter, then calibrate it: run the same strategy in paper mode and at minimum stakes live, and compare fill rate, average matched price and per-market P&L to the simulation on the same markets (self-recorded) — the gap is the simulator's bias for that strategy.

### Gaps
- No published quantitative study of flumine sim-vs-live error was found; the size of the bias is strategy-specific and must be measured.
- Whether flumine 3.x changed queue/matching logic versus the 2020 issue #192 state was not verified (the quickstart still lists the same three limitations as of this session).
- The exact way flumine computes commission in simulated cleared orders/markets in 3.x was not checked.

---

## 3. Sensible architecture (recorder, storage, features, model, engine, risk, monitoring, hosting)

### Takeaway
Record raw stream data yourself from day one (cheap, includes catalogue data, and identical in format to what the simulator replays), keep raw files as the source of truth and derive Parquet feature tables from them, and run one small always-on Linux VPS in the UK/Ireland; co-location-grade latency is neither affordable nor necessary at hobby scale, but note the live app key now costs £499 on its own.

### Cited Findings
- flumine "Data Stream" passes raw streaming data back, reducing RAM/CPU, and "allows recording of the data for future playback, see the example marketrecorder.py" [fetched] — [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md)
- Historical catalogue data is only available for backtesting if you collected it at the time, usually with the marketrecorder [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies)
- A common storage pattern per flumine docs: S3 for all market files with a local cache for frequently processed markets; flumine opens files with `smart_open` [fetched] — [flumine performance docs](https://github.com/betcode-org/flumine/blob/master/docs/performance.md)
- Betfair historic data tiers: BASIC = 1-minute intervals, last traded price only, no volume, free; ADVANCED = 1-second intervals, top-3 ladder, with volume, paid; PRO = API tick intervals (50 ms), full ladder, with volume, paid. Delivered as `data.tar` containing `.bz2` files; each period can be "purchased" only once; site downloads are slow and the Historic Data API is better for bulk [fetched] — [Betfair Data Scientists: using the historic data site](https://betfair-datascientists.github.io/data/usingHistoricDataSite)
- Betfair docs: "Detailed Historical Data is made available for testing and analysis purposes" (stated alongside the rule that read-only use of a live key is not permitted) [fetched] — [Betfair Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- Delayed app key: free; operates on the live production exchange (not a sandbox); delayed prices with variable 1-180 second snapshots; 3 price levels; no `totalMatched` by selection and no EX_ALL_OFFERS; no BSP far/near; bet placement on the live exchange IS allowed; Stream API allowed but limited to 2 connections and 200 markets (default for delayed keys created after 8 Apr 2020) [fetched] — [Betfair Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- Live app key: **£499 activation fee** debited from the Betfair account on activation; apply via Developer Support under "Exchange API > For My Personal Betting"; account must be KYC-verified; read-only use of a live key is not permitted; app keys are for personal betting only and commercial use needs Betfair approval (page last updated 2 Nov 2026) [fetched] — [Betfair Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys). Conflict: a search summary of another Betfair docs page reported "£299" for personal betting and an aggregator described £499 as the vendor fee [search summary] — [Betfair docs short link](https://betfair-developer-docs.atlassian.net/wiki/x/fyUp), [apis.io](https://apis.io/plans/betfair/betfair-plans-pricing/). The directly fetched official page says £499.
- REST polling limits: for listMarketBook/listRunnerBook/listMarketCatalogue/listMarketProfitandLoss, sum(weights) x number of marketIds must be <= 200 points per request or the API returns TOO_MUCH_DATA (e.g. EX_BEST_OFFERS = 5, EX_ALL_OFFERS = 17, EX_TRADED = 17, SP_TRADED = 7) [fetched] — [Betfair Market Data Request Limits](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687478/Market+Data+Request+Limits)
- Latency/hosting: no authoritative source for the physical location of Betfair API endpoints was found. One AWS re:Post thread reports a user with Dublin-hosted servers seeing latency to the Betfair API move from <1 ms to ~13 ms from 2025-05-01, with no explanation — a single anecdote [search summary] — [AWS re:Post](https://repost.aws/questions/QUHinIWIYVSeScKNuTZp-arg/higher-network-latency-since-1st-of-may)
- flumine live CPU load "tends to be considerably lower compared to simulating" [fetched] — [flumine performance docs](https://github.com/betcode-org/flumine/blob/master/docs/performance.md)
- flumine logging: JSON logging via python-json-logger is the documented pattern; "logging controls" can be added to the framework to capture orders/markets [fetched] — [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md)

### Inferences
Suggested component layout (researcher synthesis, not from a single source):
1. **Recorder** — a separate, boring, always-on process (flumine marketrecorder pattern) writing raw stream messages per market (gz/bz2) plus market catalogue JSON; never co-deploy with strategy code so strategy crashes cannot lose data. With a delayed key the recording is conflated/delayed and capped at 200 markets, so high-fidelity self-recording effectively needs the live key.
2. **Storage** — raw stream files are the immutable source of truth (they are what flumine/Nautilus replay). Derive columnar **Parquet** tables (per-market snapshots at fixed times-to-off, top-of-book, traded volume, BSP, result) for research in pandas/polars/DuckDB. Do not backtest execution from Parquet snapshots; use them for feature/model work and use replay for execution.
3. **Feature pipeline / model training** — offline, deterministic, versioned; features must be computable from data available at decision time only; the same feature code should be imported by the live strategy to avoid train/serve skew.
4. **Strategy engine** — flumine strategy class (`check_market_book` / `process_market_book` / `process_orders`), identical code in simulation, paper and live.
5. **Order/risk layer** — flumine trading/client controls plus custom controls (section 6).
6. **Logging/monitoring** — structured JSON logs, an order/market log for reconciliation against Betfair account statement, heartbeat + alerting (e.g. push notification on no-data, exceptions, drawdown breach).
7. **Hosting** — a small Linux VPS (1-2 vCPU, 2-4 GB) in London or Dublin is adequate for live trading since live CPU load is low; backtests run on the local workstation (CPU-bound, multiprocess). Measure ping/traceroute to the Stream and Betting endpoints from candidates rather than assuming. Typical cost for such a VPS is in the region of single-digit to low-double-digit GBP/month [prior knowledge, not verified].
- Budget consequence: £499 (live key) consumes essentially all of a £500 tooling budget before any PRO data purchase. A plausible low-cost path is: develop on the delayed key + free BASIC data + any free PRO samples; then pay for the live key and self-record full-fidelity data going forward instead of buying PRO history.

### Gaps
- Current GBP prices for ADVANCED/PRO historic data packages were not found (not listed on the hub page; historicdata.betfair.com not fetched). UK-customer access terms for PRO data unverified.
- Location of Betfair's exchange data centres (Dublin vs London) could not be confirmed from any primary source this session.
- Whether Betfair waives or discounts the £499 fee in any circumstance (e.g. for existing active customers) was not established.
- Stream API subscription limits for live keys (markets per connection, connections) not fetched.

---

## 4. Backtesting realism (queue, partial fills, impact, delay, suspensions, non-runners, cross-matching, fees, limits)

### Takeaway
The main sources of backtest optimism on Betfair are fill assumptions (queue position, repeated use of the same liquidity, no market impact), ignoring bet delay/suspensions, missing cross-matched liquidity, and under-costing fees. flumine handles some (bet delay, version-change lapses, non-runner reductions) and explicitly not others (queue cancellations, double counting, cross-matching, impact).

### Cited Findings
- flumine models: bet delay by market state, liquidity-based queue position, order lapse on market version change (i.e. suspensions/material changes lapse non-PERSIST orders), lapse and reduction on runner removal [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart), [flumine issue #192](https://github.com/betcode-org/flumine/issues/192)
- flumine does not model: queue cancellations ahead of you, prevention of double-counted liquidity (unless isolation settings changed), currency fluctuations [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- Cross-matching: Betfair "uses a 'cross matching' algorithm to display the best possible prices ... taking into account the back and lay offers across all selections"; virtual bets are returned via `virtualise: true` on listMarketBook, or the `EX_BEST_OFFERS_DISP` filter on the Stream API. Worked example in the docs: backs at 2.0 and 3.0 on two football outcomes create a virtual lay-side offer at 6.0 on the draw with stakes inversely proportional to price [fetched] — [Betfair Additional Information: Virtual Bets](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2686993/Additional+Information)
- flumine simulation does not include cross matching; simulated bets do not affect other participants [fetched] — [Betfair Data Scientists: flumine simulations](https://betfair-datascientists.github.io/tutorials/flumineSimulations)
- Minimum sizes (GBP): Min Bet Size £1; Minimum BSP Liability £10 (for LIMIT_ON_CLOSE / MARKET_ON_CLOSE); Minimum Bet Payout £10, which "relates to the ability to place orders at lower stakes via placeOrders" (page updated 9 Nov 2026) [fetched] — [Betfair Additional Information: Currency Parameters](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2686993/Additional+Information)
- Expert Fee replaced the Premium Charge from 6 January 2025: based on rolling 52-week gross profit; no fee below £25,000; 20% for £25,000-£100,000; 40% above £100,000 (reported by Racing Post, not fetched from a Betfair page) [search summary] — [Racing Post](https://www.racingpost.com/news/britain/betfair-exchange-to-introduce-new-commission-system-for-2025-as-premium-charge-is-dropped-a7wbg0v4GCAJ)
- Transaction charge: applies to both delayed and live keys per the app-key table [fetched] — [Betfair Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys). Described threshold: customers logging more than 5,000 transactions in one clock hour (placed, cancelled or modified bets, including failed transactions), with the charge offset against commission [search summary; the source page returned 404 when fetched directly] — [Betfair Data Scientists: commission](https://betfair-datascientists.github.io/wagering/commission)
- Order-instruction limit of 1,000 instructions per second is cited by the Nautilus adapter docs [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- BSP bets are sized by liability, cannot be cancelled once placed, and remain unresolved until BSP reconciliation [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- Price ladder: 1.01 to 1000 with tiered tick sizes (0.01 at the short end up to 10 above 100) [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- betcode FAQ: successful bettors may work with returns as low as 1-3% after commission — so a small error in fill/price/fee assumptions can flip the sign [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies)

### Inferences
Practical realism checklist for the backtest harness (researcher synthesis):
- **Queue position**: for passive orders assume you join the back of the queue at your price at (decision time + place latency); count a fill only when traded volume at that price after your arrival exceeds the size ahead of you. Run a pessimistic variant (e.g. require price to trade *through* your level) and report both.
- **Partial fills**: persist matched vs remaining sizes; decide a policy for remainders (cancel at T-x, convert to SP, leave to lapse) and simulate it.
- **Market impact / thin markets**: cap stake to a small fraction of visible size at the touch and of recent traded volume; never re-use liquidity you have notionally consumed (set flumine's isolation/double-counting options accordingly); stress-test with one extra tick of slippage on every aggressive fill.
- **Bet delay / in-play**: use the `betDelay` in market definition; assume orders can be voided/lapsed on suspension; pre-off orders lapse at turn-in-play unless persistence is set. Strategies that depend on reacting faster than the bet delay to public events are not viable without specialised feeds.
- **Non-runners / reduction factors** (racing): matched bets get price reductions and resting orders may be lapsed/reduced; verify the simulator's output against a handful of real settled non-runner markets.
- **Cross-matching**: if using raw (non-virtual) ladders in markets with few selections, displayed-to-users prices were better than the data shows; either record `EX_BEST_OFFERS_DISP` or avoid strategies that hinge on the touch price in such markets.
- **Fees**: apply commission on net market winnings per market at the account's actual rate; at hobby scale the Expert Fee (>£25k/yr gross profit) is irrelevant; keep transactions well below 5,000/hour (rate-limit place/cancel/replace loops).
- **Minimum stake**: £1 minimum, with sub-minimum stakes only where payout >= £10; a staking plan that outputs £0.37 bets is not executable — round and re-test.
- **Data fidelity**: PRO (50 ms) or self-recorded live-key stream is required for execution modelling; ADVANCED (1 s, top 3) is usable for slow pre-off strategies; BASIC is only good for coarse price-vs-outcome studies.

### Gaps
- Current standard commission rate(s) for UK accounts (historically a 5% Market Base Rate with a discount scheme; a reported 2025 restructure toward a flexible rate) could not be confirmed from an official Betfair page — check the account's own "My Betfair Rewards"/commission setting.
- Official Betfair wording of the Expert Fee and the transaction charge (thresholds, charge per excess transaction) was not fetched from betfair.com; figures above are from secondary sources.
- Exact in-play bet-delay values per sport were not sourced this session (commonly a few seconds, varying by sport/market) [prior knowledge, unverified].
- The exact rule for sub-£1 stakes (which order types, persistence) beyond the "Minimum Bet Payout £10" table entry was not extracted from the placeOrders page.

---

## 5. Research methodology to avoid false discoveries

### Takeaway
With edges of 1-3% and noisy outcomes, P&L alone needs thousands of bets to distinguish skill from luck; the defensible approach is time-ordered splits with a locked final holdout, explicit accounting for the number of variants tried (deflated Sharpe / PBO / reality-check style corrections), calibration measured against market prices, and "beating the closing price (BSP)" as the faster-converging primary metric.

### Cited Findings
- "Sports data is very noisy" and "it's incredibly easy to mine opportunities that aren't real"; judge results on whether the sample is large enough — spanning several months and, for racing and soccer, thousands of markets [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies)
- "Every new strategy should be extensively backtested" [fetched] — [betcode FAQ](https://betcode-org.github.io/faq/developing-and-testing-strategies)
- Buchdahl ("Monte Carlo or Bust"): in a football dataset with 952 positive-EV bets identified from Pinnacle prices, only ~65 bets were needed to demonstrate the edge via closing-line movement; a secondary article attributes to him that proving profit from results alone may take several thousand bets whereas a consistent ~5% closing-line edge may need only ~50 [search summary; second claim secondhand] — [CLV demystified by Joseph Buchdahl (secondary)](https://pod-d614f9k2t-pod-team.vercel.app/blog/closing-line-value--clv-demystified-by-expert-joseph-buchdahl)
- Generic CLV sample-size guidance from non-authoritative sites varies from 50-100 to 200-500 bets — an order-of-magnitude disagreement [search summary] — [comparenbet guide](https://comparenbet.pages.dev/guide-sample-size-significance)
- Betfair's simulation tutorial illustrates the data-snooping hazard directly: results are sliced by time-to-off, race type, price band, field size, distance, track etc. "to find angles" — many slices on one sample [fetched] — [Betfair Data Scientists: flumine simulations](https://betfair-datascientists.github.io/tutorials/flumineSimulations)
- Bailey, Borwein, López de Prado & Zhu, "Pseudo-Mathematics and Financial Charlatanism" (Notices of the AMS, 2014): with enough trials, a high in-sample Sharpe is expected from pure noise; the minimum backtest length grows with the number of configurations tried [prior knowledge] — [AMS Notices PDF](https://www.ams.org/notices/201405/rnoti-p458.pdf)
- Bailey, Borwein, López de Prado & Zhu, "The Probability of Backtest Overfitting" — combinatorially symmetric cross-validation (CSCV) to estimate PBO, the probability that the in-sample best configuration underperforms the median out of sample [prior knowledge] — [SSRN 2326253](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2326253)
- Bailey & López de Prado, "The Deflated Sharpe Ratio" — adjusts an observed Sharpe for the number of trials, their variance, sample length, skew and kurtosis [prior knowledge] — [SSRN 2460551](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2460551)
- White (2000), "A Reality Check for Data Snooping" (Econometrica) — bootstrap test of whether the best of many rules beats a benchmark after accounting for the search [prior knowledge] — [DOI](https://doi.org/10.1111/1468-0262.00152)
- Harvey, Liu & Zhu, "... and the Cross-Section of Expected Returns" (RFS 2016) — with hundreds of factors tested, a t-stat hurdle around 3 rather than 2 is appropriate; Harvey & Liu "Backtesting" proposes Sharpe haircuts for multiple testing [prior knowledge] — [DOI](https://doi.org/10.1093/rfs/hhv059), [SSRN 2345489](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2345489)
- Real-world illustration of out-of-sample decay under faithful replication: of nine published equity anomalies rebuilt by an AI agent, none survived out of sample (2023-2025), no result was significant after multiple-testing correction, and a look-ahead bug from split-adjusted prices created ~3.5 Sharpe points of illusory edge [fetched] — [Quantpedia](https://quantpedia.com/guardrails-make-the-researcher-what-an-ai-agent-got-right-and-wrong-replicating-nine-equity-anomalies/)

### Inferences
Recommended protocol (researcher synthesis applying the above to Betfair):
- **Splits by time, never random**: e.g. train = oldest 60%, validation = next 20%, **locked test = most recent 20%**, with an embargo gap (a few days, or longer than any feature look-back) between segments. Group by event so that correlated markets (win/place, same match) never straddle a split.
- **Walk-forward**: refit on an expanding or rolling window and evaluate on the next block; report the distribution of block results, not just the aggregate.
- **Look-ahead audit**: every feature timestamped <= decision time minus latency; BSP, final traded volume, result, and any "market closed" field are post-decision; verify by asserting that features computed from a truncated file equal those computed from the full file.
- **Primary metric = edge versus closing price**: for each bet compute expected value at BSP (e.g. back at price p vs BSP b: p/b - 1 before commission; mirror for lays) and the proportion of bets beating BSP. Because BSP on liquid markets is a well-calibrated probability estimate, mean CLV has far lower variance than realised P&L and converges in tens-to-hundreds of bets rather than thousands. Caveats: BSP is less efficient in illiquid markets, and a strategy that systematically trades right at the off has no meaningful "closing line".
- **Calibration**: if the strategy uses a probability model, compare log loss and Brier score against market-implied probabilities (BSP-implied, normalised for overround) on the same events; plot reliability diagrams; a model that cannot beat the market's log loss can still be useful only as a *residual* signal conditional on price — test that explicitly (e.g. regression of outcome on market logit + model logit).
- **Sample-size arithmetic**: per-bet P&L standard deviation for a back at decimal odds o is roughly sqrt(o - 1) units; to detect a 2% edge at ~2 standard errors at odds 3.0 needs on the order of (2 x 1.41 / 0.02)^2 ≈ 20,000 bets; at odds 2.0, ≈ 10,000. This is why CLV is essential. (Arithmetic by researcher.)
- **Uncertainty**: bootstrap confidence intervals on ROI and CLV, resampling by event/day (block bootstrap) rather than by bet, since bets in the same market/day are correlated.
- **Multiple testing**: log every variant ever evaluated (count N); report deflated Sharpe or apply Bonferroni/Holm/BH on p-values; prefer t > 3 for anything found by search; estimate PBO via CSCV when choosing among many parameterisations; treat any subgroup "angle" (track, price band, time-to-off) as a new hypothesis requiring fresh data.
- **Final holdout used once**: touching the locked test set more than once per hypothesis family converts it into a validation set.

### Gaps
- Buchdahl's primary articles/books (e.g. "Squares & Sharps, Suckers & Sharks", "Monte Carlo or Bust") were not opened; the specific numbers above are from a search summary of secondary pages.
- The academic papers were not fetched this session; citations are from background knowledge and should be spot-checked for exact titles/years before publication.
- No Betfair-specific published study quantifying how quickly BSP-based CLV converges, or BSP calibration by sport/liquidity, was retrieved this session.

---

## 6. Staking and risk management in code

### Takeaway
flumine provides the hooks (client controls, trading controls, per-strategy exposure parameters, order persistence/lapse options); the reader must add their own fractional-Kelly sizing, hard exposure caps, kill switch and unmatched-order policy. At hobby scale the binding constraints are the £1 minimum stake and correlation of bets within an event.

### Cited Findings
- flumine exposes `add_client_control` and `add_trading_control` for custom controls, plus logging controls; risk management tools "for monitoring and limiting exposure" are a headline feature [fetched] — [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md), [PyPI](https://pypi.org/project/flumine/)
- flumine catches errors in `check_market` / `process_market_book` and logs them by default (`config.raise_errors = True` disables this) — meaning a buggy strategy can keep running silently unless logs are monitored [fetched] — [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md)
- `config.execution_retry_attempts = 10` governs cancel attempts when the order stream is not connected; `customer_strategy_ref` scopes the order stream to the running instance [fetched] — [flumine config.py](https://github.com/betcode-org/flumine/blob/master/flumine/config.py), [flumine advanced docs](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md)
- In the Betfair tutorial strategy, orders are placed with LAPSE persistence so that unmatched remainders are cancelled when the market turns in-play [fetched] — [Betfair Data Scientists: flumine simulations](https://betfair-datascientists.github.io/tutorials/flumineSimulations)
- Minimum GBP bet £1; minimum BSP liability £10; minimum payout £10 for lower-stake orders [fetched] — [Betfair Additional Information](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2686993/Additional+Information)
- Nautilus notes for Betfair: on reconnect, new submissions are halted until mass-status reconciliation completes; cancel-all with no side sends a single market-wide request [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)

### Inferences
(Standard practice; formulas are textbook and not sourced to a fetched page.)
- **Kelly**: for a back bet at decimal odds o with estimated win probability p and commission rate c on winnings, net odds b = (o - 1)(1 - c); full-Kelly fraction f* = (p·b - (1 - p)) / b. For a lay, compute from the layer's side (risking liability (o - 1)·stake to win stake·(1 - c)). Use **fractional Kelly (e.g. 0.1-0.3x)** because p is estimated with error and full Kelly on an overestimated edge is ruinous; cap any single bet at a small percentage of bankroll regardless of Kelly output. For mutually exclusive runners in one market, size jointly (or simply cap total market exposure) rather than applying Kelly independently.
- **Exposure limits** (enforced in a trading control, before the order leaves): max liability per order, per runner, per market, per event, per day; max open unmatched orders; max orders per market; max transactions per hour (well under 5,000).
- **Kill switches**: (1) daily/weekly loss limit and max drawdown from high-water mark → stop placing, cancel resting orders; (2) data staleness (no stream update for N seconds, or latency above threshold) → cancel all; (3) reconciliation mismatch between internal position and Betfair's order stream; (4) manual flag file / remote command; (5) process supervisor that cancels all orders on crash/exit. Consider Betfair's server-side heartbeat (cancels unmatched bets if the client stops heart-beating) [prior knowledge: the Betting API has a Heartbeat operation; not verified this session].
- **Unmatched bets**: define time-in-force explicitly — lapse at in-play, cancel after T seconds or if price moves k ticks away, or take SP; make the back-tested policy and live policy identical.
- **Hedging / greening up**: hedge stake = (entry stake x entry price) / current opposing price equalises P&L across outcomes; it costs spread + is subject to the £1 minimum and to fills; back-test hedging as real orders, not as a formula applied at mid-price.
- With a small bankroll, £1 minimum stakes make fractional Kelly unimplementable for small edges (the Kelly stake is often < £1) — live small-stakes testing is therefore *flat minimum stake*, and is a test of execution and CLV, not of the staking plan.

### Gaps
- Names and default values of flumine 3.x built-in trading/client controls (e.g. order validation, strategy exposure, max transaction count) were not read from the 3.x docs this session.
- Betfair Heartbeat API specifics not verified this session.

---

## 7. Paper trading vs live: delayed key, flumine paper mode, minimum stakes, staged rollout

### Takeaway
The free delayed key can exercise code paths (and can even place real bets) but its 1-180 s delayed, 3-level, volume-less data makes it unsuitable for evaluating anything price-sensitive; meaningful paper trading (flumine `paper_trade=True`) and data recording need the £499 live key. Minimum stake is £1 in GBP, which makes a long, cheap small-stakes live phase feasible.

### Cited Findings
- Delayed key: for development and functional testing; operates on the live production exchange; 1-180 s variable delay; "must also be used in simulation/practice applications where the facility to bet into live Betfair markets is not available"; no traded volume (`totalMatched`) or EX_ALL_OFFERS; 3 price levels; bet placement allowed; streaming capped at 2 connections / 200 markets [fetched] — [Betfair Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- Live key: £499 one-off, personal betting use, KYC-verified account, testing should be completed on the delayed key first, read-only usage not permitted [fetched] — [Betfair Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- flumine paper trading: live market data, simulated execution and simulated order polling [fetched] — [flumine quickstart](https://betcode-org.github.io/flumine/quickstart)
- Nautilus: a delayed key's conflated stream breaks its recovery logic [fetched] — [Nautilus Betfair docs](https://nautilustrader.io/docs/latest/integrations/betfair/)
- GBP minimum bet size £1; minimum BSP liability £10; minimum payout £10 for lower stakes (docs page updated 9 Nov 2026) [fetched] — [Betfair Additional Information](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2686993/Additional+Information). Conflict: some third-party guides still state a £2 exchange minimum [search summary] — [Good Money Guide](https://goodmoneyguide.com/betting-exchange/); Bet Angel's help article (2023) states £1 [search summary] — [Bet Angel](https://betangel.freshdesk.com/support/solutions/articles/80001073147-what-is-the-minimum-bet-allowed). The official developer docs (fetched) say £1.

### Inferences
Staged rollout (researcher synthesis):
0. **Offline** — pipeline, simulator, pre-registered hypotheses, locked holdout; strategy passes validation and one-shot holdout with CLV > 0 and realistic fills.
1. **Delayed key smoke test** — login (certificate/non-interactive), stream subscription, order placement/cancel plumbing with tiny real bets far from the market (they are real: the delayed key bets into the live exchange), reconciliation, logging, kill switch drills.
2. **Live key + paper trade (2-4+ weeks)** — flumine `paper_trade=True` on live data while the recorder captures the same markets; then replay those recorded markets in simulation and confirm paper ≈ simulation (tests code parity and data handling, not real fills).
3. **Live at minimum stake (£1 flat), few markets** — goal is execution truth: fill rate, queue outcomes, slippage vs simulation on the same markets, CLV vs BSP, error rates. Pre-define pass/fail thresholds and a minimum bet count (hundreds for CLV significance).
4. **Scale slowly** — increase stakes/markets in steps only while live CLV and fill metrics stay within the pre-registered tolerance of simulation; expect impact to grow with stake; hard bankroll and loss limits throughout.
- Expected cost of stage 3: at £1 stakes, even a strategy with zero edge loses roughly commission + spread, i.e. pennies per bet — hundreds of bets can be bought for a few pounds of expected cost, though variance is larger at long odds or when laying.

### Gaps
- Whether a delayed key combined with flumine paper mode behaves sensibly (given conflated data) is not documented; presumed poor.
- flumine's handling of the "£1 min / £10 payout" rule in simulated order validation was not checked.
- UK regulatory/account-side constraints (affordability checks, account restrictions for bot users) are outside this scope and were not researched here.

---

## 8. Using LLM coding agents for strategy research: experience and guardrails

### Takeaway
Published 2025-2026 experience is consistent: agents are effective at building pipelines and running replications fast, but they misread specifications, introduce look-ahead/data bugs that create spectacular fake edges, and — because they can generate and test variants cheaply — multiply the data-snooping problem. The mitigations are procedural, not model-level: humans fix hypotheses, data windows, costs and pass/fail gates before results are seen; holdouts are physically locked; every trial is logged; and independent verification passes inspect the trade log, not just the code. "Trust the guardrails, not the agent."

### Cited Findings
- Quantpedia case study (agent built by Vlad Rodeski / PolyQuant; underlying model not named): an autonomous agent replicated nine published US-equity anomalies. Guardrails: a pre-backtest "dossier" of verbatim quotes from each paper checked against the agent's extracted spec; a code-fidelity review pass and a separate execution-fidelity pass over the trade log; point-in-time, survivorship-free data; liquidity screens and turnover-aware costs with break-even cost reporting; statistics recomputed independently from equity curves and promising results re-run on a second engine; papers, windows, costs and viability gates fixed by humans before results were read [fetched] — [Quantpedia](https://quantpedia.com/guardrails-make-the-researcher-what-an-ai-agent-got-right-and-wrong-replicating-nine-equity-anomalies/)
- Failures observed in that study: the agent built a momentum strategy for a contrarian paper, guessed universes from titles, used a wrong price floor; a schedule bug left the first year untraded; rebalancing generated millions of orders until gated; a halted micro-cap produced a +76% month; an apparent "survivor" with out-of-sample Sharpe 1.95 was a mis-built sort that reversed when checked against the source; look-ahead from split-adjusted prices produced ~3.5 Sharpe points of illusory edge. Trade-log review caught bugs that code review missed (e.g. zero trades despite entry logic) [fetched] — [Quantpedia](https://quantpedia.com/guardrails-make-the-researcher-what-an-ai-agent-got-right-and-wrong-replicating-nine-equity-anomalies/)
- Outcome: none of nine anomalies survived out of sample on a faithful build; nothing was significant after multiple-testing correction [fetched] — [Quantpedia](https://quantpedia.com/guardrails-make-the-researcher-what-an-ai-agent-got-right-and-wrong-replicating-nine-equity-anomalies/)
- An HKUST preprint (arXiv 2603.14288, March 2026) on agentic factor investing frames backtest overfitting and "factor zoo" proliferation as the core risks of automated search; its defences are strict out-of-sample testing and requiring an economic rationale for each signal [search summary] — [arXiv 2603.14288](https://arxiv.org/html/2603.14288v1)
- BacktestBench (arXiv 2605.17937, 2026) benchmarks LLM agents on automating the whole backtesting pipeline; its AutoBacktest system splits roles into Summarizer, Retriever (data queries and quality checks) and Coder, with self-verification at each stage [search summary] — [arXiv 2605.17937](https://arxiv.org/pdf/2605.17937)
- "AQuA" (reported by the LLMQuant newsletter, Aug 2026; paywalled, unverified) is described as using two independent research systems to stop agents producing false alpha [search summary] — [LLMQuant](https://llmquant.substack.com/p/the-ai-quant-lab-that-learns-from)
- Vendor/commentary pieces recommend fixed backtest templates, multiple-testing awareness, leakage tests and out-of-sample promotion gates, and flag temporal contamination (the LLM itself having seen post-backtest-date information) [search summary; non-peer-reviewed] — [StackAI](https://www.stackai.com/insights/how-agentic-ai-is-revolutionizing-factor-investing-and-quantitative-research-for-aqr-style-teams)

### Inferences
A responsible agent-driven "strategy discovery" loop for Betfair (researcher synthesis):
1. **Human-owned, read-only evaluation harness.** The backtest runner, cost model (commission, latency, queue rules), metric code (CLV vs BSP, ROI, bootstrap CI, deflated Sharpe) and data splits live in a module the agent is not permitted to edit (file permissions / separate repo / CI check on diff). The agent only writes strategy/feature code against a fixed interface.
2. **Physically locked holdout.** The most recent block of markets is stored where the agent's environment cannot read it (different machine/bucket/credentials). Only a human-triggered job runs a frozen strategy commit on it, once, and the result is appended to an immutable log.
3. **Pre-registration.** Before any run, the agent (or human) writes a hypothesis card: mechanism/economic rationale (who is the counterparty and why are they wrong?), market universe, entry/exit rules, parameters and allowed ranges, primary metric and pass threshold, expected sign. Commit it (git hash + timestamp) before data is touched. Results without a prior card are exploratory only.
4. **Experiment tracking with a trial counter.** Every backtest invocation is logged automatically by the harness (not by the agent's narrative): commit hash, config, data range, metrics. The cumulative number of trials N per hypothesis family feeds the deflated-Sharpe / multiple-testing adjustment. MLflow or a simple append-only SQLite/JSONL log suffices [tool choice is researcher suggestion].
5. **Budgeted search.** Cap variants per hypothesis (e.g. <= 10 parameter sets) and total hypotheses per holdout "spend"; forbid the agent from iterating on validation results indefinitely; prefer coarse, pre-declared grids over open-ended optimisation.
6. **Independent verification passes.** A second agent session/model with no access to the first's reasoning reviews (a) code for look-ahead (use of BSP, result, final volume, future timestamps; features from the full file), and (b) the *order log* for implausible fills: fills at prices that never traded, fills larger than traded volume, fills during suspension, zero-trade runs, P&L concentrated in a handful of markets. Add automated leakage tests: truncate-file invariance, shuffled-outcome test (edge must vanish when results are permuted), time-shift test (delay signals by N seconds and check graceful decay).
7. **Suspicion rule.** Any result that looks too good (e.g. ROI >> 3% after commission, Sharpe > 3, near-100% fill of passive orders) is treated as a bug until proven otherwise, mirroring the Quantpedia "survivor" that was a construction error.
8. **Promotion gates set in advance**: validation CLV > 0 with CI excluding 0 → one-shot holdout → paper → £1 live. A human approves each promotion; the agent never has live-key credentials or the ability to place real orders during research.
9. **Temporal contamination**: an LLM may "know" results of historical sporting events inside its training window; never let the model's own recall act as a feature or a label source, and prefer evaluation on events after the model's knowledge cutoff when the agent contributes priors.
- The core quantitative risk: an agent can run hundreds of variants per day. Under a pure-noise null, the expected best-of-N Sharpe grows roughly with sqrt(2 ln N), so automated search will reliably "discover" strategies unless N is counted and penalised (Bailey et al. logic, section 5).

### Gaps
- No published write-up specifically about using Claude Code/Codex-style agents to discover Betfair or sports-betting strategies was found; all evidence is from equities/quant-finance.
- The arXiv papers and the AQuA work were not opened; descriptions come from search summaries.
- No vendor guidance from Anthropic/OpenAI specific to quant-research loops was retrieved this session.

---

## 9. Learning resources

### Takeaway
The best free, current material is the flumine docs/examples and betcode FAQ (for engineering) and the Betfair Data Scientists hub (for tutorials, though Australia-centric and partly written for older flumine versions); the Betfair developer docs were updated in November 2026 and are authoritative for rules and limits.

### Cited Findings
- flumine docs: quickstart (simulation, paper trading, listener kwargs, event processing), advanced (controls, streams, config), performance (multiprocessing) [fetched] — [flumine docs](https://betcode-org.github.io/flumine/quickstart), [advanced](https://github.com/betcode-org/flumine/blob/master/docs/advanced.md), [performance](https://github.com/betcode-org/flumine/blob/master/docs/performance.md)
- betcode FAQ: "Getting started" and "Developing and testing strategies" pages (sample size, noise, sim-vs-live ordering, catalogue recording) [fetched / search summary] — [developing and testing strategies](https://betcode-org.github.io/faq/developing-and-testing-strategies), [getting started](https://betcode-org.github.io/faq/getting-started)
- betfairlightweight docs [fetched link from PyPI metadata] — [bflw docs](https://betcode-org.github.io/betfair/)
- Betfair Data Scientists hub: "Simulating Past Betfair Markets" (flumine simulation with PRO files, multiprocessing, results analysis) [fetched] — [flumineSimulations](https://betfair-datascientists.github.io/tutorials/flumineSimulations); "How to Automate" series, part 5 on backtesting/simulation [search summary] — [How to Automate 5](https://betfair-datascientists.github.io/tutorials/How_to_Automate_5); historic data site guide [fetched] — [usingHistoricDataSite](https://betfair-datascientists.github.io/data/usingHistoricDataSite); historical data sources list [search summary] — [dataSources](https://betfair-datascientists.github.io/modelling/dataSources); AU app key guide [fetched link] — [apiappkey](https://betfair-datascientists.github.io/api/apiappkey/)
- Betfair developer docs (Confluence): Application Keys (updated 2 Nov 2026), Additional Information incl. currency minimums and virtual bets (updated 9 Nov 2026), Market Data Request Limits (updated 9 Nov 2026), Best Practice Guide, Exchange Stream API, Betting Type Definitions, placeOrders [fetched or located via docs search] — [Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys), [Additional Information](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2686993/Additional+Information), [Market Data Request Limits](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687478/Market+Data+Request+Limits)
- Nautilus Trader Betfair integration guide and book-imbalance backtest tutorial [fetched / search summary] — [integration](https://nautilustrader.io/docs/latest/integrations/betfair/), [tutorial](https://nautilustrader.io/docs/latest/tutorials/backtest_book_imbalance_betfair)
- Overfitting literature: Bailey et al. (AMS Notices 2014; PBO), Bailey & López de Prado (Deflated Sharpe), White (Reality Check), Harvey-Liu-Zhu [prior knowledge] — links in section 5
- Agent-research case study — [Quantpedia](https://quantpedia.com/guardrails-make-the-researcher-what-an-ai-agent-got-right-and-wrong-replicating-nine-equity-anomalies/)

### Inferences
- Suggested reading order for an experienced engineer new to betting: Betfair developer docs (app keys, stream API, best practice) → flumine quickstart + examples (marketrecorder, simulate) → Betfair hub "How to Automate" series and simulation tutorial → betcode FAQ → Buchdahl on CLV/sample size → López de Prado ("Advances in Financial Machine Learning", chapters on backtesting and purged cross-validation) [book recommendation from prior knowledge].
- Hub tutorials target Australian/NZ racing and Betfair AU accounts (different key activation path and PRO-data contact); UK readers must adapt market filters, country codes and data-purchase steps.

### Gaps
- Betfair developer forum (forum.developer.betfair.com) and the betcode Slack were not visited; current activity level and join process unverified.
- No practitioner blog post about building a Python Betfair bot was fetched this session, so none is cited; the report writer should not imply one was reviewed.
- Buchdahl's books/articles not directly fetched.
