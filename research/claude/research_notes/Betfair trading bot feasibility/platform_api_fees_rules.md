# Betfair Exchange: platform mechanics, API access, fees and rules (UK individual, as of 15 Nov 2026)

RESEARCHER'S NOTE ON RELIABILITY (read first). Betfair's customer-facing pages (support.betfair.com charges page, general terms, betting.betfair.com Expert Fee FAQ) returned HTTP 403 to every fetch, and several developer-docs pages came back truncated. Each finding is therefore tagged:
- [read directly] = the page was opened and its content extracted in this session.
- [search summary] = the claim comes from a search engine's summary of the linked page; the page itself was not opened. Treat as probable, not confirmed.
Anything I know only from background knowledge is confined to the Gaps subsections and labelled UNVERIFIED. Web research was interrupted once by a usage limit and then resumed.

## 1. How does the exchange order book and matching work?

### Takeaway
The API-level mechanics that were confirmed: a fixed price ladder with widening tick sizes, three persistence types for unmatched bets, and a per-market in-play bet delay exposed as `betDelay`. Queue priority, cross-matching and suspension rules could not be confirmed from a primary page in this session.

### Cited Findings
- Price increments by price band per the developer docs: 1.01 to 2 in steps of 0.01; 2 to 3 in 0.02; 3 to 4 in 0.05; 4 to 6 in 0.1; 6 to 10 in 0.2; 10 to 20 in 0.5; 20 to 30 in 1. A bet outside these increments returns INVALID_ODDS. Bands above 30 were not returned. Some betting-guide sites give different figures for the 3 to 4 and 20 to 30 bands; the developer docs should be treated as authoritative. [search summary; exact docs page not opened, link is one of the developer-docs results returned] — [Betfair developer docs](https://betfair-developer-docs.atlassian.net/wiki/x/eRkp)
- The price ladder used by a market is defined by a PriceLadderType in the market definition (so not every market necessarily uses the classic ladder). [search summary] — [Betfair developer docs, Exchange Stream API](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687396/Exchange+Stream+API)
- Persistence types for unmatched orders: LAPSE (cancelled when the market turns in-play), PERSIST (carried into the in-play market), MARKET_ON_CLOSE (goes into the Betfair Starting Price auction at the turn in-play). [search summary] — [Betfair docs, Betting Enums](https://docs.developer.betfair.com/display/1smk3cen4v3lu3yomq5qye0ni/Betting+Enums)
- LAPSE is documented as the default in one client library. [search summary] — [abettor R client docs](https://rdrr.io/github/phillc73/abettor/man/placeOrders.html)
- A forum thread reports that orders which match instantly can cause an invalid persistence type error; persistence only matters for the unmatched remainder. [search summary] — [Betfair developer forum](https://forum.developer.betfair.com/forum/sports-exchange-api/exchange-api/33396-getting-invalid_persistence_type-when-placing-bet-before-in-play)
- In-play delay: the stream exposes `betDelay` in seconds, usually non-zero once a market is in-play. A PASSIVE mode lets orders that cannot match immediately skip the delay. [search summary] — [Betfair developer docs, Exchange Stream API](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687396/Exchange+Stream+API)
- A third-party write-up uses a 5-second in-play delay as its example (anecdotal, not an official figure). [search summary] — [Trinity Logic](https://www.trinitylogic.co.uk/blog/betfair-in-play-automation-latency-execution/)
- The transaction count includes "all unmatched, matched, cancelled, and lapsed bets", which is relevant to how order churn is designed. [read directly] — [Betfair developer support](https://support.developer.betfair.com/hc/en-us/articles/20029343399836-Transaction-Charge-how-are-transactions-counted)
- Betfair advises making the minimal number of transactions/changes possible. [search summary] — [Betfair developer docs, Best Practice](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687730/Best+Practice)

### Inferences
- A bot that wants resting orders to survive the off must set PERSIST explicitly; otherwise they lapse at the turn in-play.
- In-play strategies must model the bet delay per market (read `betDelay`), since a taking order is exposed to price movement for that many seconds.

### Gaps
- Not confirmed from any retrieved source: queue priority rule, cross-matching/virtual bets, best-execution behaviour, minimum stake, BSP order types and reconciliation, suspension behaviour, market types, tick bands above 30, and typical bet-delay values by sport. The placeOrders docs page was truncated on fetch.
- UNVERIFIED background (prior knowledge only, may be outdated): odds run 1.01 to 1000 with further bands 30 to 50 in 2, 50 to 100 in 5, 100 to 1000 in 10; matching is price-then-time (FIFO at each price) with best execution giving a better price when available; cross-matching creates virtual bets from the other selections' books and appears when virtual prices are requested; minimum stake for GBP was historically £2 and was reduced to £1 in 2022; BSP order types are MARKET_ON_CLOSE and LIMIT_ON_CLOSE; in-play delays are commonly 1 to 12 seconds depending on sport; unmatched LAPSE bets are cancelled on suspension for material events in some sports. Verify each before use.

## 2. What are the API products, delayed vs live keys, live key cost and conditions?

### Takeaway
The live application key costs a one-off £499 as of the Betfair developer docs page read on 15 Nov 2026 (page marked "Updated Nov 02"). Older and third-party sources still quote £299, so the fee appears to have risen; the delayed key is free, allows real bet placement, but gives data delayed 1 to 180 seconds.

### Cited Findings
- Live Application Key activation fee is £499: "Check that your Betfair account is funded to cover the £499 activation fee." It is debited from the Betfair account once the Live App Key has been activated. The comparison table lists Activation Fee "None" for Delayed and "£499" for Live. Page shows "Updated Nov 02" (no year shown). The page carries no note about a fee change. [read directly, twice] — [Betfair developer docs, Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- CONFLICT / RECENT CHANGE: other sources, including the Betfair Automation Hub guide and a January 2026 third-party blog, give the fee as £299. The search engine's own summary of Betfair docs also said £299. The directly read docs page says £499, which I treat as current; the date of the increase was not found. [search summary] — [Betfair Automation Hub, app key guide](https://betfair-datascientists.github.io/api/apiappkey/); [oddspapi blog](https://oddspapi.io/blog/?p=112)
- Delayed key vs Live key (from the docs table): Delayed is for development/functional testing; price data delayed by 1 to 180 seconds; 3 price levels; no total matched by selection; no BSP far/near price; bet placement is allowed and goes to the live exchange (there is no sandbox); Stream API allowed but limited to 2 connections and 200 markets; transaction charges apply. Live gives live prices, all price levels, total matched by selection and BSP near/far. [read directly] — [Betfair developer docs, Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- Delayed-key Stream API access is default only for keys created after 8 April 2020; older keys need Developer Support to enable it. [read directly] — same Application Keys page
- Read-only use of the Live key is not permitted; testing should be completed on the Delayed key first. [read directly] — same Application Keys page
- Conditions to obtain a live key: apply via the developer.betfair.com "get-started" page choosing Exchange API and "For My Personal Betting"; account fully verified under KYC; account funded to cover the fee. [read directly] — same Application Keys page
- Personal vs commercial: "App Key generation is for personal betting purposes only." Commercial use of data/API needs Betfair approval; "Unauthorised commercial usage will be identified & blocked." [read directly] — same Application Keys page
- Key creation: via the Accounts API Demo Tool using createDeveloperAppKeys; one Live and one Delayed key per account; only one set of keys per account; application name must be unique and must not contain the Betfair username. [read directly] — same Application Keys page
- A Live key can still return delayed data if the account is not funded or an override has been applied. [search summary] — [Betfair developer support, Stream API access](https://support.developer.betfair.com/hc/en-us/articles/115003887871-How-do-I-get-access-to-the-Stream-API)
- Delayed stream data is conflated: changes are delivered in a single message every 3 minutes. [search summary] — [Betfair developer docs, Exchange Stream API](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687396/Exchange+Stream+API)
- On 29 July 2026 the Exchange joined the Sportsbook in a unified mobile app; customers cannot opt out. No commission or API change is mentioned. [search summary] — [Betfair support, "Betfair Exchange - What's Changed?"](https://support.betfair.com/app/answers/detail/a_id/10792/)

### Inferences
- Fixed entry cost for live automated trading is £499, non-recurring. At hobby stakes this is likely the largest single cost and needs to be weighed against expected profit.
- The delayed key is good enough to build and functionally test the whole bot (including placing small real bets), but not to evaluate a price-sensitive strategy.
- Third-party software (for example Bet Angel) is a route to live data without a personal live key, since vendors hold their own keys. This is my inference from the personal/vendor split; vendor terms were not retrieved.

### Gaps
- Date of the £299 to £499 change not found.
- Not retrieved: Historical Data service tiers and prices, Accounts API detail, vendor licence terms, live-key approval timescale, login/session rules.
- UNVERIFIED background: Historical Data has Basic (free), Advanced and Pro (paid) tiers; login is via interactive or certificate-based non-interactive endpoints.

## 3. Current commission rates, calculation and loyalty/discount schemes

### Takeaway
Commission is set by the "My Betfair Rewards" package the customer selects: Basic 2%, Rewards 5% (the default), Rewards+ 8%. No reliable source confirms a 2026 change to these rates.

### Cited Findings
- Betfair's charges page says the Basic package carries a 2% commission rate, with a worked example of £2 on £100 of net winnings; the rate is determined by the package chosen in "My Betfair Rewards" in My Account. [search summary; page returned 403 on direct fetch] — [Betfair Charges](https://support.betfair.com/app/answers/detail/betfair-charges)
- Three packages: Rewards+ 8%, Rewards 5% (default), Basic 2%. Package can be switched at the start of each month. [search summary] — [Sporting Life](https://www.sportinglife.com/free-bets/guides/betfair-exchange-education/how-does-commission-work)
- Choosing Basic (2%) makes the customer ineligible for most exchange promotions. [search summary] — [Outplayed, Betfair review](https://outplayed.com/betfair-review)
- Betfair states the average commission rate on the Exchange is now approximately 2.5%. [search summary] — [Betfair, Expert Fee FAQ](https://betting.betfair.com/betfair-announcements/exchange-news/the-betfair-exchange-expert-fee-faq-111224-6.html)
- CONFLICT: one low-reliability guide claims base commission rates increased from June 2026; another review says base commission remains 5% of net winnings. A targeted follow-up search found no source reporting a 2026 change to the Basic 2% rate. [search summary] — [stakeshorseracingbet.com (low reliability)](https://stakeshorseracingbet.com/betfair-exchange-premium-charge-explained/); [trueprob.com](https://www.trueprob.com/reviews/betfair-exchange/fees)
- CONFLICT: an unofficial guide says Market Base Rate varies by market (UK/Irish racing 2%, most others 5%), shown on each market's Rules tab; another says the old discount-rate scheme is retired for most accounts. [search summary] — [betfairsquare (low reliability)](https://betfairsquare.pages.dev/guides/betfair-commission-explained)

### Inferences
- A bot operator would normally choose Basic at 2%, because the extra 3 to 6 points of commission on the other packages buy promotions a bot does not use. Inference, not a Betfair statement.
- The older "5% Market Base Rate less a points-based Discount Rate" model in many trading guides appears superseded by the package model; I could not date the switch.
- With commission charged on net winnings per market, a round-trip trade's edge must exceed 2% of the net market profit to break even; losing markets pay nothing.

### Gaps
- Exact official wording of the calculation and the full current rate table could not be read (403).
- Whether individual markets carry rates different from the package rate is unresolved.

## 4. Premium Charge / Expert Fee status and likely impact on a hobbyist

### Takeaway
The Premium Charge was replaced by the Expert Fee from 6 January 2025. It applies only above £25,000 gross profit over the last 52 active weeks, so a hobby-scale bot will not be affected.

### Cited Findings
- From 6 January (2025) the Expert Fee replaced the Premium Charge, measuring gross profit over a rolling 52-week period instead of lifetime. [search summary] — [Racing Post](https://www.racingpost.com/news/britain/betfair-exchange-to-introduce-new-commission-system-for-2025-as-premium-charge-is-dropped-a7wbg0v4GCAJ)
- Rates by Last 52 Active Week Gross P&L: under £25,000, 0%; £25,000 to £100,000, 20%; over £100,000, 40%. [read directly] — [Bet Angel, Expert Fee explained](https://www.betangel.com/betfair-expert-fee/); matches [Racing Post](https://www.racingpost.com/news/britain/betfair-exchange-to-introduce-new-commission-system-for-2025-as-premium-charge-is-dropped-a7wbg0v4GCAJ)
- Conditions listed (all must be met): 52-active-week gross P&L above £25,000; bets placed in more than 100 Exchange markets; lifetime profit; commission generated lower than the Expert Fee rate. [read directly] — [Bet Angel, Expert Fee explained](https://www.betangel.com/betfair-expert-fee/)
- Gross P&L is total won excluding commission minus total lost across all Exchange markets. Commission generated is half the commission paid on winning bets plus implied commission. Assessed weekly, every Monday, on the previous week. [read directly] — [Bet Angel, Expert Fee explained](https://www.betangel.com/betfair-expert-fee/)
- Implied commission was cut from 3% to 2.5% of market losses. [search summary] — [Betfair, Expert Fee FAQ](https://betting.betfair.com/betfair-announcements/exchange-news/the-betfair-exchange-expert-fee-faq-111224-6.html)
- For accounts with fewer than 52 active weeks, the measure is gross P&L since account creation. [search summary] — [Betfair Charges](https://support.betfair.com/app/answers/detail/betfair-charges)
- The former 50% and 60% tiers are gone (cap 40%); the lifetime allowance is replaced by the rolling 52-week test. [search summary] — [Bet Angel](https://www.betangel.com/betfair-expert-fee-will-you-pay/)
- CONFLICT (minor): the Bet Angel explainer page lists lifetime profit as a condition, while a search summary of another Bet Angel page suggested prior lifetime losses do not matter. The directly read page (lifetime profit required) and Traderline agree. [read directly / search summary] — [Bet Angel](https://www.betangel.com/betfair-expert-fee/); [Traderline](https://traderline.com/education/betfair-premium-charge-explained)

### Inferences
- A hobbyist would need more than £25,000 gross profit in a year before the Expert Fee applies. It is irrelevant at experimental stakes; commission and the £499 key fee are what matter.
- Above the threshold, a 2% commission payer generates less "commission generated" than a 5% payer and so pays more Expert Fee. Follows from the definitions above; my inference.

### Gaps
- Betfair's own Expert Fee FAQ and charges page could not be read (403): the precise buffer formula, definition of an "active week", linked-account treatment and any 2026 amendments are unconfirmed.

## 5. Transaction charges, failed transactions, request limits and stream limits

### Takeaway
The free allowance is 5,000 transactions per clock hour (the 1,000 figure is outdated), with a sub-penny charge per excess transaction that is offset against commission. Market data requests are capped at 200 weight points per request.

### Cited Findings
- Free allowance of 5,000 transactions per hour; a 2020 announcement priced each excess transaction at £0.002. [search summary] — [Betfair developer forum](https://forum.developer.betfair.com/forum/sports-exchange-api/exchange-api/33242-transaction-charges)
- CONFLICT: Betfair's charges page is summarised as listing failed transactions at £0.001 each, duplicate failed transactions at £0.005, and a standard £0.001 in the hourly calculation. [search summary; page 403 on fetch] — [Betfair Charges](https://support.betfair.com/app/answers/detail/betfair-charges)
- What counts: "all unmatched, matched, cancelled, and lapsed bets". Place then successfully cancel = 1 transaction in total. A failed cancellation counts as a failed transaction (placement + failed cancel = 1 transaction + 1 failed transaction). [read directly] — [Betfair developer support](https://support.developer.betfair.com/hc/en-us/articles/20029343399836-Transaction-Charge-how-are-transactions-counted)
- Transaction fees are offset against total commission; if commission exceeds the fee nothing is charged. Not offset against Premium Charge/Expert Fee. [search summary] — [Betfair Charges](https://support.betfair.com/app/answers/detail/betfair-charges); [Betfair Automation Hub](https://betfair-datascientists.github.io/gettingStarted/commission/)
- Counted per clock hour, resetting on the hour. [search summary] — [Bet Angel help](https://betangel.freshdesk.com/support/solutions/articles/80001073150-number-of-bets-transactions-allowed)
- Charges show as items in getAccountStatement. [search summary] — [Betfair developer support](https://support.developer.betfair.com/hc/en-us/articles/360018362418-How-can-I-identify-Transaction-Charges-via-getAccountStatement)
- Transaction charges apply to both Delayed and Live keys. [read directly] — [Betfair developer docs, Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- Market data request limit: sum(weight) x number of marketIds must not exceed 200 points per request, else TOO_MUCH_DATA. [read directly] — [Betfair developer docs, Market Data Request Limits](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687478/Market+Data+Request+Limits)
- Weights, listMarketBook/listRunnerBook: none set 2; SP_AVAILABLE 3; SP_TRADED 7; EX_BEST_OFFERS 5; EX_ALL_OFFERS 17; EX_TRADED 17; EX_BEST_OFFERS+EX_TRADED 20; EX_ALL_OFFERS+EX_TRADED 32. With exBestOffersOverrides the weight is multiplied by requestedDepth/3. listMarketCatalogue: MARKET_DESCRIPTION 1, RUNNER_METADATA 1, others 0. listMarketProfitAndLoss: 4. Example: EX_BEST_OFFERS across 20 markets = 100 points. [read directly] — same Market Data Request Limits page
- listMarketBook should be called at most 5 times per second per marketId. [search summary] — [Betfair docs, listMarketBook](https://docs.developer.betfair.com/display/1smk3cen4v3lu3yomq5qye0ni/listMarketBook)
- Stream: default cap of 200 markets per subscription; exceeding it returns SUBSCRIPTION_LIMIT_EXCEEDED (the one error that does not close the connection). Whether 200 is per connection, key or account is not stated in primary docs; secondary sources say per connection with up to 10 connections, uncited. [search summary] — [Betfair developer docs, Exchange Stream API](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687396/Exchange+Stream+API); [Betfair developer forum](https://forum.developer.betfair.com/forum/sports-exchange-api/exchange-api/3477-stream-api-subscription-limit)
- Delayed key stream limit: 2 connections and 200 markets. [read directly] — [Betfair developer docs, Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)
- A "2026 pricing guide" claiming £0.01 per bet above 1,000 per hour contradicts all other sources and looks unreliable. [search summary] — [tryfix.it.com](https://tryfix.it.com/how-much-does-betfair-api-cost-the-complete-2026-pricing-guide/)

### Inferences
- At 5,000 free transactions an hour (about 83 a minute), a low-frequency hobby bot will not incur transaction charges; even if it did, the charge is netted against commission.
- Using the Stream API for prices avoids the polling weight limits entirely.

### Gaps
- Current per-excess-transaction price (£0.001 vs £0.002) unconfirmed from a directly read Betfair page.
- Live-key stream connection limit, login rate limits, and any separate data request charge were not confirmed.

## 6. Betfair terms on bots, multiple accounts and restricting winners

### Takeaway
Betfair's terms recognise bots and permit API betting, but reserve a right to restrict bots used "to the detriment of our business", prohibit screen scraping, and allow only one account per customer. These points come from search snippets of the terms, not a full read.

### Cited Findings
- The terms acknowledge that some customers use programs to place bets automatically within parameters they set ("bots"), and Betfair may restrict bots generally or any specific bot if it believes a bot has been used to the detriment of its business. [search summary; terms page 403 on fetch] — [Betfair General Terms and Conditions](https://support.betfair.com/app/answers/detail/betfair-general-terms-and-conditions)
- Screen scraping, web scraping or other automated or manual collection of Betfair Data is prohibited; one version of the wording covers commercial or personal use, while the current UK version is reported as narrowed to commercial use and use by a Restricted Person. Regional versions differ. [search summary] — [Betfair General Terms and Conditions](https://support.betfair.com/app/answers/detail/betfair-general-terms-and-conditions)
- One account only: additional accounts are "Duplicate Accounts"; Betfair may close accounts and void winnings. [search summary] — [Betfair General Terms and Conditions](https://support.betfair.com/app/answers/detail/betfair-general-terms-and-conditions)
- A clause lets Betfair act where it reasonably suspects bets were placed robotically or by automated means rather than manually; the snippet notes API bets are excluded from this. [search summary] — [Betfair General Terms and Conditions](https://support.betfair.com/app/answers/detail/betfair-general-terms-and-conditions)
- API keys are for personal betting only and unauthorised commercial use is blocked. [read directly] — [Betfair developer docs, Application Keys](https://betfair-developer-docs.atlassian.net/wiki/spaces/1smk3cen4v3lu3yomq5qye0ni/pages/2687105/Application+Keys)

### Inferences
- A personal bot placing bets through the official API on the user's single account is within the terms; scraping the website or app, or running a second account, is not.
- The "detriment of our business" clause is discretionary, so there is no contractual guarantee against restriction.

### Gaps
- The full current UK terms were not read; clause numbers, exact wording and last-updated date are unconfirmed. The search tool noted the indexed official pages were roughly 18 months old.
- No source retrieved on whether Betfair in practice restricts or closes winning exchange accounts. UNVERIFIED background: the exchange is generally understood not to stake-limit winners as the Sportsbook does, with the Expert Fee being the mechanism for large winners.

## 7. UK tax, HMRC, UKGC checks and 2025-2026 duty changes

### Takeaway
HMRC's position is that betting as such is not a trade, even when systematic or the bettor's sole livelihood, so an individual's winnings are not taxed. Operator duty on remote betting rises from 15% to 25% from 1 April 2027, and UKGC financial checks at hobby deposit levels are light-touch.

### Cited Findings
- "The basic position is that betting and gambling, as such, do not constitute trading." An organised activity to make profits out of the gambling public (for example bookmaking) will normally amount to trading. [read directly] — [HMRC BIM22015](https://www.gov.uk/hmrc-internal-manuals/business-income-manual/bim22015)
- A betting system, or success sufficient to live on, does not by itself make the activity a trade. Graham v Green [1925] 9 TC 309: sole livelihood from betting on horses, not taxable; Rowlatt J: "There is no tax on a habit." Habitual or systematic betting does not create a trade. [read directly] — [HMRC BIM22017](https://www.gov.uk/hmrc-internal-manuals/business-income-manual/bim22017)
- Exception: a professional gambler providing services for reward (for example appearance money) may be trading; whether winnings are part of that trade depends on the facts. [read directly] — [HMRC BIM22017](https://www.gov.uk/hmrc-internal-manuals/business-income-manual/bim22017)
- Remote Gaming Duty up from 21% to 40% from 1 April 2026 (Budget, 26 November 2025). [search summary] — [GOV.UK, Changes to gambling duties](https://www.gov.uk/government/publications/changes-to-gambling-duties)
- New remote rate of General Betting Duty of 25% (from 15%) from 1 April 2027; excludes self-service betting terminals, pool betting and spread betting; remote bets on UK horse racing stay at 15%. [search summary] — [GOV.UK, Changes to gambling duties](https://www.gov.uk/government/publications/changes-to-gambling-duties); [Deloitte](https://taxscape.deloitte.com/measures-autumn-budget-2025/gambling-duties.aspx)
- UKGC light-touch financial vulnerability checks: threshold £500 net deposits in a rolling 30 days from 30 August 2024, reduced to £150 from February 2025. [search summary] — [Gambling Commission, financial vulnerability and financial risk summary](https://www.gamblingcommission.gov.uk/guidance/regulatory-decisions-procedures-and-guidance-for-regulatory-hearings/summary-financial-vulnerability-and-financial-risk)
- Financial risk assessments (FRAs): pilot found 95% of checks frictionless. A 7 July 2026 Commission decision is reported as a staged rollout: first stage flags net deposits above £5,000 in a rolling 24 hours (under-25s: £750 in 24 hours or £2,000 in a rolling 90 days); a later phase for age 25+ at £1,000 in 24 hours or £3,000 in a rolling 90 days. Rollout dates not confirmed on the Commission's own site. [search summary] — [SBC News](https://sbcnews.co.uk/social-responsibility/2026/05/21/commission-gambling-affordability-checks/); [iGaming Business](https://igamingbusiness.com/legal-compliance/regulation/document-checks-not-required-for-financial-risk-assessments/)
- The Commission has said FRAs will not require document checks such as bank statements. [search summary] — [iGaming Business](https://igamingbusiness.com/legal-compliance/regulation/document-checks-not-required-for-financial-risk-assessments/)
- CONFLICT: one article claims FRAs trigger at £150 and have applied since February 2026; this contradicts the July 2026 reporting and looks unreliable. [search summary] — [brightsideofnews](https://brightsideofnews.com/gambling/news/uk-gambling-affordability-checks-financial-risk-assessments-2026/)

### Inferences
- A hobbyist's exchange profits are outside income tax and CGT on HMRC's stated position, and losses are not relievable. The manual does not discuss API/bot trading specifically; this applies the general rule.
- Depositing more than £150 net in 30 days is likely to trigger a background vulnerability check, which is automated and should not need action from the customer.
- The April 2027 duty rise falls on Betfair, not the customer, but is a plausible driver of future commission or fee increases. No confirmed Betfair response found.

### Gaps
- Whether exchange commission revenue is within the 25% remote betting rate was not addressed in sources found.
- Any Autumn 2026 Budget change to these rates was not checked.
- Operator-level AML source-of-funds thresholds at Betfair were not found.

## 8. Alternative exchanges with APIs

### Takeaway
Betdaq is no longer an option: it was sold to Matchbook in October 2026 and its exchange closed. Smarkets and Matchbook remain, both quoted around 2% and both with APIs, but Matchbook's fee structure is inconsistent across sources.

### Cited Findings
- Matchbook (Triplebet Limited) announced the acquisition of the Betdaq brand on 14 October 2026; Betdaq's operator began an orderly closure. Trading ended 21 October 2026; accounts stayed open for withdrawals until 4 November 2026. Betdaq accounts do not transfer; customers must open a new Matchbook account. Matchbook may relaunch the Betdaq brand later. [search summary] — [iGaming Business](https://igamingbusiness.com/strategy/ma/matchbook-completes-acquisition-rival-exchange-betdaq/); [European Gaming](https://europeangaming.eu/portal/latest-news/2026/10/15/216005/matchbook-acquires-betdaq-brand/)
- 2026 comparisons list Smarkets 2% and Matchbook 2% (and Betdaq 2% before closure). [search summary] — [laybetaccess comparison](https://laybetaccess.com/betting-exchange-commission-comparison)
- CONFLICT on Matchbook: sources variously give 0.75% maker / 1.5% taker, 0% for makers, or 2.5% base with volume discounts; none confirmed on Matchbook's own page. [search summary] — [botblog comparison](https://botblog.co.uk/matchbook-vs-smarkets-vs-betdaq/); [punter2pro](https://punter2pro.com/best-betting-exchange-betfair-comparison/)
- Smarkets has a documented REST API and WebSocket streaming, with an application required; Matchbook has a REST API; third-party trading software reportedly does not natively support Matchbook. [search summary] — [botblog, UK exchanges compared](https://botblog.co.uk/comparing-top-5-uk-sports-exchanges/)
- Matchbook's owner was previously fined £740,000 by the Gambling Commission. [search summary] — [Racing Post](https://www.racingpost.com/news/matchbook-owner-fined-740000-by-gambling-commission-over-serious-failings-a63Tn1x9oenK/)

### Inferences
- Review pages dated November 2026 that still list Betdaq as an operating exchange are stale.
- Liquidity was not assessed; Betfair is widely described as the deepest market, but I have no retrieved figure.

### Gaps
- No primary fee or API-terms pages for Smarkets or Matchbook were read; API cost, approval terms and rate limits are unconfirmed.
