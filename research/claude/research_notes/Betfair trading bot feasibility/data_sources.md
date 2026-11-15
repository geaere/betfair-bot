# Data sources for developing and backtesting a Betfair Exchange strategy (UK hobbyist, ~£500 budget, as of 15 Nov 2026)

Method note: prices marked "checked 15 Nov 2026" were read directly from the vendor/primary page on that date. Anything else is from search-result summaries or secondary pages and is flagged. Several vendor pricing pages render client-side and could not be read (The Racing API, historicdata.betfair.com itself, TPD).

## 1. Betfair Historical Data service (historicdata.betfair.com): tiers, granularity, cost, format, quirks

### Takeaway
Three tiers: BASIC (free, 1-minute last-traded-price only, no volume), ADVANCED (paid, 1-second, top-3 ladder + volume), PRO (paid, ~50 ms tick-level, full ladder + volume, equivalent to recording the Stream API yourself). PRO is the only tier recommended for realistic backtesting and costs £79-£230 per sport per month, so a £500 budget buys only 2-3 sport-months of PRO for the major sports.

### Cited Findings
- Tier definitions: Basic = 1-minute intervals, last traded price only, no volume, free. Advanced = 1-second intervals, top-3 price ladder, volume, paid. Pro = API tick intervals (50 ms), full price ladder, volume, paid. Advanced and Pro have the same fields at different frequencies. — [Betfair Automation Hub: using the Historic Data site](https://betfair-datascientists.github.io/data/usingHistoricDataSite)
- Price list (Betfair Developer Program support article, shown as updated 24 Sep 2026; read directly 15 Nov 2026). Per sport, per month of data / per 12 months of one sport (bulk discount, arranged via Developer Support):

  | Sport | ADVANCED /month | ADVANCED /year | PRO /month | PRO /year |
  |---|---|---|---|---|
  | Horse Racing | £69 | £699 | £230 | £2,299 |
  | Soccer | £69 | £699 | £230 | £2,299 |
  | Tennis | £49 | £499 | £150 | £1,499 |
  | Cricket | £49 | £499 | £150 | £1,499 |
  | Greyhound Racing | £49 | £499 | £150 | £1,499 |
  | Golf | £39 | £399 | £99 | £999 |
  | Other Sports | £39 | £399 | £79 | £799 |

  — [Betfair Developer Support: Are bulk purchase discounts available?](https://support.developer.betfair.com/hc/en-us/articles/360019984158-Are-bulk-purchase-discounts-available)
- Coverage: "nearly all markets offered on the Exchange since 2016" (when API-NG streaming launched). Sports: horse racing, soccer, tennis, cricket, golf, greyhounds, "other sports". — [Automation Hub: Historic Data site](https://betfair-datascientists.github.io/data/usingHistoricDataSite); [Automation Hub: Historical Data Sources](https://betfair-datascientists.github.io/modelling/dataSources)
- Contents: prices, traded volume, winner status, BSP, market definition. Data is the recorded Stream API feed: one JSON message per line with `pt` (publish time, epoch ms) and `mc` (market changes); `marketDefinition` blocks (status, inPlay, marketTime, venue, runners with id/name/status/adjustmentFactor); `rc` runner-change deltas with `atb`/`atl` (available to back/lay ladder levels; size 0 removes a level), `trd` (traded volume per price), `ltp`, `tv`, `spb`/`spl` (BSP volume), `spn`/`spf` (projected near/far BSP). — [Automation Hub: Historic stream files](https://betfair-datascientists.github.io/data/historicStreamFiles/)
- Delivery/format: files are purchased (even free ones must be "purchased" at £0), downloaded as `data.tar` containing one `.bz2` per market (M files) or event (E files); plain-text JSON inside. Filter by sport, dates, event id, market type, country. Website is slow; a Historic Data API exists for bulk download (sample code github.com/betfair/historicdata). Each period can only be purchased once. — [Automation Hub: Historic Data site](https://betfair-datascientists.github.io/data/usingHistoricDataSite)
- Official field specification PDF (live, HTTP 200 on 15 Nov 2026) states the three plans differ in both frequency and content. — [Betfair Historical Data Feed Specification](https://historicdata.betfair.com/files/Betfair-Historical-Data-Feed-Specification.pdf)
- Not in the files: cross-matching/virtual bets, persistence types, race status or match score, race type, competition ID. Workaround is to join external CSVs on event/market id. — [Automation Hub: Historic stream files](https://betfair-datascientists.github.io/data/historicStreamFiles/)
- Quirks: traded volume of 0 can mean scratching/void/settlement; the second-last message of each market zeroes all traded volumes; traded volume shifts slightly with FX (files in GBP, some markets AUD); volume can rise at a price with no matching ladder message because cross-matched bets are absent; projected BSP can be the string `'inf'`; BSP reconciliation emits four marketDefinition updates in quick succession. — [Automation Hub: Historic stream files](https://betfair-datascientists.github.io/data/historicStreamFiles/)
- Community view (betcode/flumine authors): BASIC is "free but of little value"; ADVANCED has best prices and volume but only at 1 s; PRO "aims to match the content and granularity of streaming the data yourself". Recommendation: record yourself; if you buy, buy PRO unless the strategy tolerates 1 s data. Purchased data lacks the market catalogue. Historic BSP exists only for horse and greyhound racing and is sometimes incomplete. — [betcode-org FAQ: using data](https://betcode-org.github.io/faq/using-data)
- Betfair's own data team: PRO files are required for robust simulation/backtesting across a series of markets. — [Automation Hub: Historic stream files](https://betfair-datascientists.github.io/data/historicStreamFiles/)
- Australian/NZ customers are routed via automation@betfair.com.au for PRO (not relevant to a UK account). — [Automation Hub: Historic Data site](https://betfair-datascientists.github.io/data/usingHistoricDataSite)
- Python parsing tools: `betfairlightweight` (historical stream replay), `flumine` (backtest/simulation), `betfair_data` (fast Rust-backed parser supporting official and self-recorded files), `betfairutil`. — [betfair_data on GitHub](https://www.github.com/tarb/betfair_data); [betfairlightweight](https://github.com/liampauling/betfairlightweight); [betfairutil](https://github.com/mberk/betfairutil)

### Inferences
- With £500: e.g. 2 months of horse-racing PRO (£460), or 3 months tennis PRO (£450), or 5 months golf PRO, or ~7 months horse-racing ADVANCED (£483). None is enough for a statistically robust backtest across seasons; PRO is priced for syndicates, not hobbyists.
- ADVANCED (1 s, top 3 levels) is adequate for pre-off strategies that only look at best prices and are not latency or queue-position sensitive; it is not adequate for in-play racing or anything depending on queue position or full depth.
- No tier can show your own queue position or market impact; all backtests on this data are approximations.

### Gaps
- The historicdata.betfair.com portal is a JavaScript app and could not be read; I could not confirm on-portal prices match the support article, nor per-file sizes or the exact latest available date (support FAQ "How frequently is the historical data updated?" exists but was not read).
- Could not confirm whether discounts exist for fewer than 12 months, or whether the historic-data purchase requires a funded/active account.
- Known data gaps for specific dates/markets (outage days) are discussed in the community but I found no citable list.

## 2. Free Betfair data: BSP CSVs, Automation Hub datasets, free samples

### Takeaway
The free BSP CSV archive is the single most valuable free Betfair dataset: daily per-runner summaries for UK/IRE racing since 2007-08 and greyhounds since 2012, still updated daily. BASIC historic files are also free. Free PRO data is limited to small samples.

### Cited Findings
- BSP price files at promo.betfair.com/betfairsp/prices: free, no login. Verified live 15 Nov 2026 with files dated that day. Fields: EVENT_ID, MENU_HINT (meeting), EVENT_NAME, EVENT_DT, SELECTION_ID, SELECTION_NAME, WIN_LOSE, BSP, PPWAP (pre-off volume-weighted average price), MORNINGWAP (before 11am GMT), PPMAX/PPMIN (pre-off max/min matched price, only bets with payout > £100), IPMAX/IPMIN (in-play max/min), MORNINGTRADEDVOL, PPTRADEDVOL, IPTRADEDVOL. — [Betfair BSP price files directory](https://promo.betfair.com/betfairsp/prices)
- File families and date ranges observed in the directory listing on 15 Nov 2026 (one file per day, per country, win and place separately): UK win from 29 Apr 2007, UK place from 16 Aug 2008; IRE win from 9 Apr 2007, place from 16 Aug 2008; greyhound win/place, AUS, RSA, USA from 20 Sep 2012; FR and UAE from 22 Feb 2023. Daily files are small (UK win ~32 kB). — [Betfair BSP price files directory](https://promo.betfair.com/betfairsp/prices)
- Betfair's data team describes these as runner-level snapshots back to the start of the Exchange, "free to download, no login required". — [Automation Hub: Historical Data Sources](https://betfair-datascientists.github.io/modelling/dataSources)
- Automation Hub (Betfair Australia data scientists) free CSVs: AU/NZ racing since Jan 2023 (monthly files per racing code, with matched prices/volume, BSP, results, best prices at scheduled off), Australian sports since 2020 (AFL, NRL, BBL, A-League, NBL), plus model ratings files (Iggy greyhounds, Kash thoroughbreds). AU-centric; little direct UK value but the tutorials are the best free documentation of the stream-file format. — [Automation Hub: Historical Data Sources](https://betfair-datascientists.github.io/modelling/dataSources)
- Automation Hub tutorials cover JSON-to-CSV conversion of PRO files, tar processing, flumine simulations and backtesting. — [Automation Hub index](https://betfair-datascientists.github.io/automation/)
- Samples for each tier are linked from the historic-data help page; betfairlightweight's examples reference a `horse-racing-pro-sample`; betcode notes "the free advanced and pro data they offer changes over time". — [Automation Hub: Historic Data site](https://betfair-datascientists.github.io/data/usingHistoricDataSite); [betcode-org FAQ](https://betcode-org.github.io/faq/using-data)
- football-data.co.uk CSVs include Betfair Exchange match-odds columns (BFEH/BFED/BFEA) alongside bookmaker odds, giving free exchange pre-match prices for football. — [football-data.co.uk notes](https://www.football-data.co.uk/notes.txt)
- A Kaggle dataset "betfair-sp" (repackaged BSP files) exists. — [Kaggle: betfair-sp](https://www.kaggle.com/datasets/eonsky/betfair-sp)

### Inferences
- BSP files allow free, long-horizon backtests of "bet at BSP" selection strategies (the standard way to test a racing model without price-path data): join model picks to BSP and WIN_LOSE, subtract commission. IPMIN/IPMAX allow crude in-play studies (e.g. "did the horse trade below X in-running") but without timing or volume at price.
- The PPMAX/PPMIN £100-payout filter and absence of timestamps mean these files cannot support trading (enter/exit) backtests.

### Gaps
- Could not confirm which exact free PRO/ADVANCED samples are offered in Nov 2026 (portal unreadable).
- BSP files have no race-level metadata beyond menu hint/name (no distance, going, class) and no stable link to third-party race IDs; matching is by time/course/horse name.

## 3. Recording your own data from the Stream API

### Takeaway
Self-recording with a live key gives PRO-equivalent data for free going forward (plus the market catalogue that PRO lacks), but the live key now carries a £499 one-off fee per Betfair's current developer docs, which alone consumes the budget. The free delayed key is unsuitable for building a backtest dataset.

### Cited Findings
- Delayed key (free): prices delayed by a variable 1-180 s; no selection-level `totalMatched`, no `EX_ALL_OFFERS`, no BSP near/far; 3 ladder levels only; Stream API available (keys created after 8 Apr 2020) but limited to 2 connections and 200 markets. Live key: full data, all price levels; read-only use not permitted; account must be KYC-verified and funded; activation fee £499 debited on activation; application via "For My Personal Betting". — [Betfair developer docs: Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys) (read 15 Nov 2026)
- Conflict on the fee: search-engine snippets of other/cached Betfair doc pages and a third-party summary cite £299 for personal keys (with £499 attributed to vendor keys). The Application Keys page as fetched on 15 Nov 2026 says £499. Treat £499 as current but verify at application. — [apis.io Betfair pricing summary](https://apis.io/plans/betfair/betfair-plans-pricing/) vs [Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- On a delayed key, stream data is conflated: changes are delivered as a single message every 3 minutes. — [Betfair Developer Support: How do I get access to the Stream API?](https://support.developer.betfair.com/hc/en-us/articles/115003887871-How-do-I-get-access-to-the-Stream-API)
- Personal-use rule: app keys are for personal betting only; commercial data/API use needs approval and "will be identified & blocked". Community guidance: Betfair expects some betting activity to show data collection is for personal use. — [Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys); [betcode-org FAQ](https://betcode-org.github.io/faq/using-data)
- Open-source recorder: flumine `examples/strategies/marketrecorder.py` writes raw stream text per market, gzips on market close, optional S3 upload variant. — [flumine marketrecorder.py](https://github.com/betcode-org/flumine/blob/master/examples/strategies/marketrecorder.py)
- Storage: a Java recorder author reports ~2-4 GB/day compressed for all UK/IRE horse racing plus some football (gzip saving ~85%); a Bet Angel forum user reports ~1.5 GB/day, with one Cheltenham race producing 165 MB. — [Trinity Logic: Betfair market data recorder](https://www.trinitylogic.co.uk/blog/betfair-market-data-recorder-java/); [Bet Angel forum](https://forum.betangel.com/viewtopic.php?t=21609)
- Self-recorded vs purchased: PRO is designed to match self-streamed data; self-recording can additionally capture the market catalogue (runner metadata, competition), pre-match closing prices for sports without BSP, and anything else you poll at the time. Stream cannot be filtered by competition. — [betcode-org FAQ](https://betcode-org.github.io/faq/using-data)
- Scores/race status are not in the market stream or the historic files; they come from separate services (Race Status API `listRaceDetails`, scores `listScores`) which must be polled and recorded alongside. — [Betfair docs: Race Status API](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687662/Race+Status+API); [Automation Hub: Historic stream files](https://betfair-datascientists.github.io/data/historicStreamFiles/)
- NautilusTrader's Betfair adapter is an alternative Python/Rust framework that reads the same stream format. — [NautilusTrader Betfair integration](https://nautilustrader.io/docs/latest/integrations/betfair/)

### Inferences
- Economics: £499 live key = roughly 2 months of horse-racing PRO, but then yields unlimited PRO-equivalent data across all sports indefinitely. The cost is time: months of recording before a usable sample exists, plus needing an always-on machine (a small VPS or home server; ~1-4 GB/day means roughly 0.5-1.5 TB/year if recording everything, far less if restricted to e.g. UK win markets).
- Self-recorded data reflects your own connection (conflation, disconnect gaps, latency), which makes it more representative of what your bot would see but less complete than Betfair's server-side recording.
- A delayed key is fine for developing/parsing code and for testing order placement plumbing, not for data collection (3-level ladder, missing volumes, conflated 3-minute batches).

### Gaps
- No flumine-specific storage benchmark found; figures above are from other recorders.
- Could not find an explicit Betfair statement on whether recording stream data for personal backtesting is permitted or on minimum betting activity; only community guidance.
- Whether the £499 fee is refundable or has been waived for any customer classes was not confirmed.

## 4. Third-party sellers and archives of Betfair data

### Takeaway
There is little legitimate third-party supply of tick-level Betfair data; Betfair's own service is the canonical source. What exists are derived/summarised products (charts, SP databases) and repackaged free BSP files.

### Cited Findings
- Kaggle hosts repackaged BSP files ("betfair-sp"); no Kaggle dataset of PRO-level exchange stream data was found. — [Kaggle: betfair-sp](https://www.kaggle.com/datasets/eonsky/betfair-sp)
- racing-bet-data.com sells/derives horse racing datasets built around Betfair prices (not independently assessed). — [Racing-Bet-Data](https://www.racing-bet-data.com/data/)
- bfcharts.co.uk offers Betfair price graphs; FlatStats has a historic-prices feature. — [BetFair Graphs](https://bfcharts.co.uk/); [FlatStats historic prices](https://www.flatstats.co.uk/horseracing/index.php?cmd=article&id=175)
- Betfair's own app directory lists DataForm PRO (form database with Betfair integration). — [Betfair apps: DataForm PRO](https://apps.betfair.com/data/dataform-pro/)
- Odds-API aggregators (OddsPapi, SportsGameOdds) resell current Betfair Exchange odds via their own APIs. — [OddsPapi](https://oddspapi.io/sportsbooks/betfair-ex2); [SportsGameOdds](https://sportsgameodds.com/bookmakers/betfairexchange-odds-api)
- Fracsoft: only trace found was a student project report listing Fracsoft as a historical data source; no live vendor site surfaced. — [Scribd project report](https://www.scribd.com/document/254214602/Final-Report)
- HorseRaceBase and Proform include BSP in their racing databases (see section 5). — [HorseRaceBase FAQ](https://www.horseracebase.com/faq.php)

### Inferences
- Community "shares" of purchased PRO files would breach Betfair's personal-use terms; nothing citable suggests a legitimate free PRO archive exists.
- Fracsoft appears defunct as a retail source (pre-2016 era data predating the official service); treat as unavailable.

### Gaps
- Pricing/contents of racing-bet-data.com and aggregator historical odds not checked.
- Could not confirm Fracsoft's status from a primary source.

## 5. UK/Irish horse racing fundamental data

### Takeaway
Cheap subscription databases (HorseRaceBase ~£25/month, Proform £37.50/month, Timeform Race Passes ~£30/month) and The Racing API are the realistic hobbyist options; official feeds (Racing Post, PA, Timeform B2B) are commercial-contract only. The main free scraper, rpscrape, was unavailable at its usual GitHub location on 15 Nov 2026.

### Cited Findings
- The Racing API: REST API for racecards, results, odds; core covers UK, Irish and Hong Kong racing in full plus global group races; Australia/North America add-ons closed to new subscribers; "over 500,000 results"; today's cards/odds/results refresh every 3 minutes; rate limit 5 req/s; explicitly permits use in betting models and research, prohibits use by betting operators; monthly billing. — [The Racing API FAQ](https://www.theracingapi.com/faq)
- The Racing API pricing: official page renders prices client-side and could not be read. Third-party listings conflict: "Free / $19 / $59 / Enterprise" (apis.io) vs "from £24.99 with a free version" (Slashdot). — [apis.io listing](https://apis.io/providers/the-racing-api/); [Slashdot listing](https://slashdot.org/software/p/The-Racing-API)
- HorseRaceBase: web-based form database and system builder; FAQ quotes £25/month for a Recreational account, +£10/month for an enhanced account; free trial offered. — [HorseRaceBase FAQ](https://www.horseracebase.com/faq.php); [HorseRaceBase](https://horseracebase.com)
- Proform Racing: desktop/web form database with system builder and ratings; sign-up page lists Standard £37.50/month and Premium £97.50/month with monthly/6-monthly/yearly billing (older pages show a legacy day/week pass model). — [Proform sign-up](https://www.proformracing.com/sign-up)
- Timeform Race Passes: ratings, flags, form and analysis on timeform.com; ~£30/month recurring, with shorter passes (£10 one day, £20 seven days); At The Races sells a separate "+Timeform" add-on. Some sources say it was £70/month before a restructure. No hobbyist API; consumer website product. — [Timeform Race Passes](https://www.timeform.com/horse-racing/race-passes); [Sporting Life](https://www.sportinglife.com/amp/racing/news/ultimate-form-guide-for-only-pound30/192190); [At The Races +Timeform](https://www.attheraces.com/timeform)
- Racing Post: has a B2B API (Spotlight Sports Group) with commercial access handled by its B2B team; no public developer API. — [Spotlight Sports Group: Racing Post API](https://www.spotlightsportsgroup.com/sports-betting-news/announcing-the-racing-post-api-available-now/)
- rpscrape (Python scraper of Racing Post results/racecards, ~200 stars, last pushed Feb 2026 per ecosyste.ms): github.com/joenano/rpscrape returned HTTP 404 when I checked on 15 Nov 2026. A GitHub search shows a fork "Eatkin/rpscrape-community" pushed Sep 2026. — [ecosyste.ms record of joenano/rpscrape](https://awesome.ecosyste.ms/projects/github.com%2Fjoenano%2Frpscrape); [GitHub search API: rpscrape](https://api.github.com/search/repositories?q=rpscrape&sort=stars&per_page=5)
- Press Association is the official raceday data supplier channel (commercial). — [PA Sport](https://www.pressassociation.com/Sport/GreyhoundRacing/)
- Free baseline: Betfair BSP files give winner/placed flags and runner names for every UK/IRE race since 2007-08 (results without form detail). — [Betfair BSP price files](https://promo.betfair.com/betfairsp/prices)

### Inferences
- For a Python workflow, The Racing API (if its mid tier is ~£25-60/month with historical results) is the cleanest legal route; HorseRaceBase/Proform are better for interactive system research and include BSP, but export/automation is limited by their terms.
- Scraping Racing Post/Timeform/At The Races carries terms-of-service risk and fragility; the disappearance of rpscrape's main repo is a signal of that risk (cause unknown).
- One or two months' subscription used to bulk-export history, then cancelled, is the budget-efficient pattern, subject to each vendor's export terms.

### Gaps
- The Racing API's actual 2026 plan prices and history depth per plan (unreadable page); verify at theracingapi.com/#subscribe and /data-coverage.
- Raceform Interactive, Racing Post Members' Club pricing, At The Races data, and Timeform B2B data-licence pricing were not found.
- Why rpscrape's repo is 404 (takedown vs deletion) is unknown; quality of the community fork unassessed.
- Racing Post/Timeform website terms on scraping were not read directly.

## 6. Football data

### Takeaway
Football has excellent free results-and-odds coverage (football-data.co.uk, including Pinnacle and Betfair Exchange columns) but free advanced stats shrank in 2026: FBref lost all Opta-derived advanced data in January 2026, leaving Understat (xG, six leagues) and StatsBomb open data (selected competitions) as the main free xG sources.

### Cited Findings
- football-data.co.uk (page updated 10 Nov 2026): free CSVs; results for up to 22 European divisions back to 1993/94; match stats (shots, corners, fouls, cards, referee) since 2000/01 for main leagues and all 22 divisions since 2017/18; odds from up to 10 bookmakers back to 2000/01; market average/max since 2005/06; since 2019/20 both a post-opening snapshot and closing odds; Pinnacle closing 1X2 back to 2012/13; 16 extra worldwide leagues with results and closing odds from 2012/13; updated at least twice weekly. For private individuals only, not commercial products or bot/AI scraping. — [football-data.co.uk data page](https://www.football-data.co.uk/data.php)
- Caveat from the site: since 23 Jul 2025 Pinnacle's public odds API has been unreliable, so Pinnacle odds in the files are "systematically out of date" relative to other bookmakers, should be used with caution, and are excluded from average/max calculations. — [football-data.co.uk data page](https://www.football-data.co.uk/data.php)
- The files include Betfair Exchange home/draw/away odds columns (BFEH/BFED/BFEA) and Pinnacle 1X2, over/under 2.5 and Asian handicap columns. — [football-data.co.uk notes](https://www.football-data.co.uk/notes.txt)
- FBref: on 20 Jan 2026 Sports Reference removed all advanced stats after Opta/Stats Perform terminated its feed and required deletion; only scores and basic stats remain; xG, progressive passes etc. gone. No report of restoration found. — [Awful Announcing](https://awfulannouncing.com/soccer/sports-reference-pulls-advanced-data-agreement-violation-dispute.html); [The IX Sports](https://www.theixsports.com/the-ix-soccer/fbrefs-loss-advanced-stats-womens-soccer-data-accessibility/)
- Understat: free site with shot-level xG for EPL, La Liga, Bundesliga, Serie A, Ligue 1 and Russian Premier League, 2014/15 to 2025/26+; no official API, data embedded as JSON in pages (scraped via community libraries). Site returned HTTP 200 on 15 Nov 2026. — [Apify Understat scraper description](https://apify.com/mirthful_radish/understat-xg-football-scraper); [understat.com](https://understat.com/league/EPL)
- StatsBomb open data: free event-level data for selected competitions on GitHub (repo live 15 Nov 2026), non-commercial use with attribution; Python client `statsbombpy`. — [statsbomb/open-data](https://github.com/statsbomb/open-data); [statsbombpy](https://github.com/statsbomb/statsbombpy)
- Opta/Stats Perform: no published prices; enterprise sales-led licensing, no free/hobbyist tier. — [Stats Perform pricing FAQ](https://www.statsperform.com/faqs/stats-perform-faqs-pricing-licensing/)
- API-Football: free plan 100 requests/day (all endpoints); paid tiers 7,500/day (Pro) and 75,000/day (Ultra); RapidAPI listing shows $19/month Pro, $29 Ultra, $39 Mega. — [API-Football pricing](https://www.api-football.com/pricing); [RapidAPI listing](https://rapidapi.com/api-sports/api/api-football/pricing)
- Club Elo: Elo ratings for European clubs via a CSV API at api.clubelo.com, wrapped by the `soccerdata` Python library; the API returned HTTP 502 when I tested it on 15 Nov 2026 (may be transient). — [soccerdata ClubElo docs](https://soccerdata.readthedocs.io/en/latest/datasources/ClubElo.html)
- football-data.co.uk now points API users to a partner, TheStatsAPI. — [football-data.co.uk data page](https://www.football-data.co.uk/data.php)
- Betfair football starting prices are rolled out only to limited top matches and generally not published afterwards; team-name mapping between Betfair and third-party sources must be built manually/fuzzily. — [betcode-org FAQ](https://betcode-org.github.io/faq/using-data)

### Inferences
- Closing-line benchmarking for football is weaker than it was: the classic free Pinnacle closing series is degraded from July 2025 onward; Betfair Exchange closing columns in the same files, or self-recorded Betfair prices, are the practical substitute.
- For in-play football strategies, free historical data lacks time-stamped price paths; these need Betfair ADVANCED/PRO or self-recording plus a recorded score feed.

### Gaps
- Whether FBref restored any advanced data later in 2026 is unconfirmed.
- Exact collection time of football-data's Betfair Exchange columns and which seasons carry them not verified.
- Official 2026 API-Football direct prices not readable; Club Elo terms and uptime unconfirmed.

## 7. Tennis, cricket, greyhound and golf data

### Takeaway
Cricket (Cricsheet) and tennis results/odds (tennis-data.co.uk) are free and current; golf has a strong low-cost paid source (DataGolf, $270/year needed for archives); UK greyhound fundamentals are the weakest for hobbyists. Jeff Sackmann's main tennis repos were not reachable on GitHub on 15 Nov 2026.

### Cited Findings
- tennis-data.co.uk (updated 15 Nov 2026): free Excel files per season, ATP results from 2000 and odds from 2001, WTA from 2007; match results plus head-to-head bookmaker odds; updated weekly; private individual use only, copyright retained. — [tennis-data.co.uk](https://www.tennis-data.co.uk/data.php)
- Jeff Sackmann datasets (ATP/WTA match results and stats from 1968, Grand Slam point-by-point, Match Charting Project; CC BY-NC-SA 4.0): github.com/JeffSackmann/tennis_atp and tennis_slam_pointbypoint returned HTTP 404 on 15 Nov 2026, and the GitHub API listed only `tennis_MatchChartingProject` under his account. A third party reported the same on 2 Nov 2026. Mirrors exist (e.g. Hugging Face archive); TennisDB says Sackmann data covers 1968-2024 with later years from other sources. — [sportsapis.dev tennis API notes](https://sportsapis.dev/tennis-api); [Hugging Face mirror](https://huggingface.co/datasets/Aneeshers/tennis-sackmann-archive); [TennisDB about](https://tennis-db.com/about); [GitHub API: JeffSackmann repos](https://api.github.com/users/JeffSackmann/repos?per_page=50)
- Alternative maintained ATP database: Tennismylife TML-Database on GitHub. — [TML-Database](https://github.com/Tennismylife/TML-Database)
- Cricsheet: free ball-by-ball data in JSON (also YAML/CSV/XML) for 23,241 matches (internationals and T20 leagues), with recent-match bundles updated within days; 386 Afghanistan-related matches withheld; licensed Open Data Commons Attribution (XML under CC BY 4.0). — [Cricsheet downloads](https://cricsheet.org/downloads/); [Cricsheet licensing](https://cricsheet.org/licensing/)
- DataGolf: Scratch PLUS is $30/month or $270/year; API access requires Scratch PLUS and historical archive endpoints require the annual plan. API covers round-level scoring/strokes-gained for 22 tours (PGA Tour from 2004, majors from 1983, others 2017), outright and matchup odds archives (opening/closing at multiple books, 2019 onward), pre-tournament and live model probabilities updating every 5 minutes; limit 45 requests/minute. — [DataGolf subscribe](https://datagolf.com/subscribe); [DataGolf API access](https://datagolf.com/api-access)
- UK greyhounds: GBGB holds the racing database (an academic study used 1,267,119 race entries Jan 2022-Mar 2026 from it); official distribution is via PA Betting Services (meeting IDs, off times, placings, SPs, race times, traps, comments, going allowances), which is commercial. Third-party scrapers/APIs exist (Apify actor for GBGB results with SPs and sectionals; GreyhoundAPI project). — [Frontiers in Animal Science study](https://www.frontiersin.org/journals/animal-science/articles/10.3389/fanim.2026.1868726/full); [SBC News: PA/GBGB](https://sbcnews.co.uk/retail/2024/08/13/pabs-greyhound-board/); [Apify UK greyhound results](https://apify.com/mrdoe/uk-greyhound-race-results); [GreyhoundAPI](https://github.com/GreyhoundAPI/greyhound-ai)
- Free Betfair greyhound BSP win/place files exist from 20 Sep 2012, updated daily. — [Betfair BSP price files](https://promo.betfair.com/betfairsp/prices)
- Greyhound quirk on Betfair: trap number is the first characters of the runner name; reserves renamed "(res)". — [betcode-org FAQ](https://betcode-org.github.io/faq/using-data)

### Inferences
- Tennis: tennis-data.co.uk suffices for pre-match models; in-play tennis trading (where most Betfair tennis liquidity is) needs point-level timing, which no free archive provides in sync with prices.
- Cricket: Cricsheet ball-by-ball is good enough to build in-play win-probability models, but aligning it to Betfair price timestamps requires Betfair data with timestamps (Cricsheet has no wall-clock time per ball as far as I could verify; unconfirmed).
- Golf: DataGolf annual ($270, roughly £200-215) plus Betfair golf ADVANCED months is one of the few sport combinations where both model inputs and market data fit under £500; golf outright markets on Betfair are thinner than racing/football.

### Gaps
- Cause and permanence of the Sackmann repo removal unknown; licence status of mirrors unverified.
- No citable hobbyist price for GBGB/PA greyhound data; Timeform greyhound and Racing Post greyhound data terms not checked.
- Whether tennis-data.co.uk includes Betfair Exchange odds columns was not verified.

## 8. Live / in-play data feeds

### Takeaway
A hobbyist gets Betfair's own market stream (live key), a pollable race-status and scores service, and little else at low cost; official fast feeds (TPD GPS, Opta, IMG/Sportradar point-by-point) are commercial, with TPD's punter product sold through vendor apps via "race credits" at prices I could not confirm.

### Cited Findings
- Betfair Race Status API (`listRaceDetails`): status of horse/greyhound races before and after the off, via the scores JSON-RPC endpoint (polled, not streamed). `listScores` returns current scores for events. — [Betfair docs: Race Status API](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687662/Race+Status+API)
- The Betfair stream protocol has a race subscription (`raceSubscription`) returning Race Change Messages (`rcm`) carrying live TPD GPS tracking; supported by NautilusTrader's adapter. — [NautilusTrader Betfair integration](https://nautilustrader.io/docs/latest/integrations/betfair/)
- Total Performance Data (TPD): GPS trackers in saddle cloths giving position, sectional times, pace, stride data; 150 racecourses in 16 countries claimed; primarily sold B2B (bookmakers, via PA/BetMakers). A punter subscription portal (TPD.Zone) lets users view low-latency live race data in integrated apps; Bet Angel was the first Betfair vendor app to integrate it, using "TPD race credits". — [European Gaming: Bet Angel/TPD](https://europeangaming.eu/portal/latest-news/2021/11/09/103478/bet-angel-betting-platform-to-integrate-tpds-in-running-data/); [SourceForge TPD listing](https://www.sourceforge.net/software/product/Total-Performance-Data)
- Betfair attributed a 10-year high in in-play racing turnover share to TPD's in-running data (2021), i.e. in-running racing markets are contested by participants with GPS feeds. — [iGamingFuture](https://igamingfuture.com/betfair-signals-10-year-high-of-in-play-horse-racing-bets-thanks-to-tpds-in-running-data/)
- DataGolf live model: in-play finish probabilities and live stats at 5-minute update intervals (Scratch PLUS). — [DataGolf API access](https://datagolf.com/api-access)
- The Racing API: results/odds refresh every 3 minutes (not an in-running feed). — [The Racing API FAQ](https://www.theracingapi.com/faq)
- API-Football: live scores/events via polling, free at 100 requests/day. — [API-Football pricing](https://www.api-football.com/pricing)
- Historic Betfair files contain no race status or scores, so any in-play backtest needs those recorded separately at the time. — [Automation Hub: Historic stream files](https://betfair-datascientists.github.io/data/historicStreamFiles/)

### Inferences
- In-play horse racing is the least hobbyist-friendly segment: competitors have sub-second GPS and fast pictures; a hobbyist on TV pictures plus the market stream is structurally slower. Pre-off racing, and slower in-play sports (cricket, golf, football match odds), are less latency-sensitive.
- Cheap polled score APIs (seconds-to-minutes latency) are usable for state features but not for reacting to events faster than the market.

### Gaps
- TPD race-credit prices, API availability to individuals, and latency: not found.
- Whether the Betfair `raceSubscription`/RCM stream is enabled for ordinary personal live keys, on which courses, and at what cost: no Betfair primary source found.
- Latency and coverage of Betfair `listScores` by sport not confirmed; hobbyist-priced tennis point-by-point feeds not researched.

## 9. What a ~£500 budget realistically buys; best free-data sports

### Takeaway
£500 cannot cover both a live key (£499 per current docs) and any meaningful amount of PRO data; the realistic plans are (a) free data only plus delayed key for development, (b) live key and self-record, or (c) a few ADVANCED/PRO months plus a cheap fundamentals subscription. Horse racing (BSP archive) and football (football-data.co.uk) have the best free coverage for strategy research; cricket has the best free event-level data.

### Cited Findings
- Cost anchors: live key £499 one-off ([Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)); PRO £79-£230 and ADVANCED £39-£69 per sport-month ([Betfair support price table](https://support.developer.betfair.com/hc/en-us/articles/360019984158-Are-bulk-purchase-discounts-available)); HorseRaceBase £25/month ([FAQ](https://www.horseracebase.com/faq.php)); Proform £37.50/month ([sign-up](https://www.proformracing.com/sign-up)); Timeform Race Passes ~£30/month ([Timeform](https://www.timeform.com/horse-racing/race-passes)); DataGolf $270/year ([subscribe](https://datagolf.com/subscribe)); API-Football from $19/month ([RapidAPI](https://rapidapi.com/api-sports/api/api-football/pricing)).
- Free anchors: BSP files UK/IRE racing 2007-08 onward and greyhounds 2012 onward ([BSP directory](https://promo.betfair.com/betfairsp/prices)); BASIC historic files ([Automation Hub](https://betfair-datascientists.github.io/data/usingHistoricDataSite)); football-data.co.uk ([data page](https://www.football-data.co.uk/data.php)); tennis-data.co.uk ([data page](https://www.tennis-data.co.uk/data.php)); Cricsheet ([downloads](https://cricsheet.org/downloads/)); StatsBomb open data ([repo](https://github.com/statsbomb/open-data)).

### Inferences
- Option A, £0-£100: delayed key for code development; BSP files + a 1-2 month racing-database subscription (or The Racing API) for a BSP-based racing selection model; football-data.co.uk for football value models benchmarked on closing odds. Tests selection edge, not execution.
- Option B, ~£499: live key, run flumine market recorder for months, accumulate PRO-equivalent data for the specific markets of interest; no money left for fundamentals, so rely on free sources. Best long-run value if the user will actually bet/trade via API anyway (the key is needed to go live regardless).
- Option C, ~£450-£500 without live key: e.g. 2 months racing PRO (£460); or 3 months racing ADVANCED (£207) + 2 months golf PRO (£198) ; or DataGolf annual (~£210) + 7 months golf ADVANCED (£273). Gives immediate tick/1 s data for prototyping a simulator, but going live later still requires the £499 key.
- Free-data ranking by sport for a Betfair hobbyist (my assessment): horse racing (free BSP history is unique, deep liquidity pre-off, but form data is paywalled) > football (free results/odds, weaker advanced stats post-FBref) > cricket (free ball-by-ball, good liquidity on major matches) > tennis (free results/odds; Sackmann repos currently missing) > golf (paid but cheap and high quality) > greyhounds (free BSP but paywalled fundamentals and thin markets).
- Liquidity statements (racing/football deepest, greyhounds and minor sports thin) are general knowledge and were not sourced in this research.

### Gaps
- No sourced per-sport Betfair liquidity figures were gathered here (assumed covered by another researcher).
- GBP equivalents of USD prices depend on exchange rate at purchase.
- The £299 vs £499 live-key fee conflict should be resolved by checking the developer portal when applying; it changes which option is affordable.
