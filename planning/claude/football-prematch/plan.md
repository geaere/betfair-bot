# Phased plan

Durations assume 5 to 8 hours a week. Each phase ends at a gate. Do not start the next phase until the gate passes; the "if it fails" line says what to do instead.

| Phase | What | Duration | Spend | Real bets |
|---|---|---|---|---|
| 0 | Repo, rules and data splits | 1 week | £0 | No |
| 1 | Ingest the archive and start the price recorder | 2 weeks | VPS if used | No |
| 2 | Evaluation harness, then freeze it | 2 weeks | £0 | No |
| 3 | Baseline and price reconciliation | 1 to 2 weeks | £0 | No |
| 4 | Price-only hypotheses | 2 to 3 weeks | £0 | No |
| 5 | Goals model and market blend | 4 to 8 weeks | £0 | No |
| 6 | Holdout verdict | 1 day | £0 | No |
| 7 | Execution bot and safety layer | 2 to 3 weeks | £0 | A handful |
| 8 | Forward test at minimum stakes | 3 to 6 months of season | VPS if used | Yes |
| 9 | Verdict | 1 week | £0 | No |

## Fixed design decisions

### What I checked in the data

On 15 November 2026 I fetched football-data.co.uk's `notes.txt`, its data page and season files for ten divisions. These facts are verified from those files; anything else about the site is not.

| Fact | Detail |
|---|---|
| File address | `football-data.co.uk/mmz4281/<season>/<division>.csv`, for example `2425/E0.csv` |
| Betfair Exchange prices at collection time | `BFEH`, `BFED`, `BFEA` (match odds); `BFE>2.5`, `BFE<2.5` |
| Betfair Exchange closing prices | `BFECH`, `BFECD`, `BFECA`; `BFEC>2.5`, `BFEC<2.5` |
| Seasons with those columns | 2024/25, 2025/26 and 2026/27. Absent from the Premier League files for 2018/19 to 2023/24 |
| Collection time | "Friday afternoons" for weekend games, "Tuesday afternoons" for midweek games. Exact times are on the site's fixtures page, which I did not read |
| Closing collection time | Not stated. Not stated either whether the prices are back prices, though they sum like back prices |
| Bookmaker average | `AvgH`, `AvgD`, `AvgA` at collection, `AvgCH`, `AvgCD`, `AvgCA` at close. The site says it has collected both sets since 2019/20; earlier seasons have one set |
| Pinnacle | Half empty in 2025/26 and absent from the 2026/27 Premier League file. Do not use |
| Expected goals | `HxG`, `AxG` appear in the 2026/27 Premier League file only. Source not stated |
| Coverage | In the six divisions used, Betfair match odds are filled for every match in 2024/25 and about 94% in 2025/26; over/under 2.5 has a few more gaps |

### Leagues and markets

Six divisions: `E0`, `E1`, `D1`, `SP1`, `I1`, `F1`, which is 2,304 matches a season. I chose them because their archived Betfair prices are tightest. The overround (the sum of `1/price` across a market's outcomes; 1.00 means no margin) is about 1.006 to 1.010 at collection time. In the four other divisions I checked (`E2`, `D2`, `N1`, `SC0`) it is 1.018 to 1.027 and the price data is noisier, so they are out of scope. The research found no figures for money matched by league; the recorder supplies them.

Two markets per match: match odds and over/under 2.5 goals. One goals model prices both. Asian handicap is out of scope because I could not tell which Betfair market its columns describe.

### Data splits

| Split | Seasons | On disk | Used for |
|---|---|---|---|
| Train | 2005/06 to 2023/24 | Yes | Fitting models; proxy tests against bookmaker average prices |
| Validation | 2024/25 | Yes | Choosing between variants at Betfair prices |
| Holdout | 2025/26 | **No, until Phase 6** | One verdict run |
| Spare holdout | 2026/27 up to the day the recorder starts | No | Kept unseen in case the first holdout is spent |
| Forward | From the day the recorder starts | Recorded live | Phases 1, 3 and 8 |

The holdout is enforced by season folder: the downloader refuses any season after the last validation season unless run with a holdout flag. The same gate applies to every other source, such as expected-goals data.

Train has no Betfair prices, so bets there are scored against the bookmaker average with its margin removed. That price could not have been traded. It tests one thing on five seasons from 2019/20: whether the model predicts which way prices move before the close. It is always labelled as a proxy.

### The bet rule

**When.** Friday afternoon for matches from Friday to Monday, Tuesday afternoon for matches from Tuesday to Thursday. This matches the archive's collection times, so the backtest and the live bot make the same decision at the same moment.

**What.** For each outcome the model gives a probability `p`; the fair price is `1/p`; the minimum price is `fair price × (1 + margin)`. The bot backs an outcome when the price on offer is at or above the minimum, with at most one bet per market.

**On history.** The price on offer is the collection-time Betfair column reduced by a haircut: `BFEH × (1 − haircut)`. The haircut stands for the gap between a snapshot and a price you could have traded. Every report shows haircuts of 0%, 1% and 2%. The default is 1% until Phase 3 measures it.

**Live.** The bot sends a limit order to back at the minimum price and cancels whatever is not matched within a minute. If the market offers more than the minimum, Betfair should match at the better price (to be confirmed in Phase 7). The order never rests in the queue.

**Why take the price.** Orders that wait in the queue fill mostly when the market has moved against them. In the research, Yapp's waiting orders filled 18.5% of the time and mainly when the bot was wrong, and no free data can simulate a queue. Taking costs the spread, and Whelan (2025) finds takers lose before kick-off while those who wait roughly break even. So this rule starts on the losing side and the model has to earn its way out. Waiting orders are a later study, after a live key and recorded data.

### The closing price and CLV

The closing price is the last price before kick-off. On history it is the `BFEC` columns. Live, the bot polls each market once a minute from ten minutes before the scheduled kick-off until Betfair marks it in-play (underway), and keeps the last back prices seen before that.

For a bet at price `o`, with `q` the closing probability (each `1/closing price` divided by their sum):

- CLV is `o × q − 1`.
- Close-implied return is `q × 0.98 × (o − 1) − (1 − q)`: the bet's expected return after commission if the closing price is right.

The delayed key's prices are 1 to 180 seconds old, so the live close is up to three minutes early. I expect that to add an error of a tick (one price step) or two, which is noise against a per-bet CLV spread of about 7%. That is an expectation, not a measurement. Phase 8 checks it by comparing the bot's close with `BFEC` for the same matches.

### Cost model

- Stake is flat: one unit per bet during all research.
- A winning back bet at price `o` returns `stake × (o − 1)`; a losing one returns `−stake`.
- Commission is 2% of net winnings per market, charged only when that market's net is positive. Match odds and over/under 2.5 are separate markets.
- A taker also pays the spread. On the 2024/25 files, backing every outcome at the collection-time price gave a CLV of about −0.9% in match odds and −2.2% in over/under 2.5. With commission worth 1% to 1.5% of stake, a bet needs roughly 2% to 3.5% of real mispricing to break even.

### When each field is known

Bet time is the collection time. A feature for a match may use only matches played before that match's collection day.

| Field | Known at bet time? | Model input? |
|---|---|---|
| `Div`, `Date`, `Time`, `HomeTeam`, `AwayTeam` | Yes; kick-offs can be moved | Yes |
| `FTHG`, `FTAG`, `FTR` and half-time fields of this match | No | Never; they are the label |
| Shots, corners, cards, `HxG`, `AxG` of this match | No | Never |
| `Referee` | Unverified when appointed | No |
| Results and statistics of earlier matches | Yes, if played before the collection day | Yes |
| League position, form, ratings, season averages | Only if built from earlier matches | Yes, rebuilt for each match as of its collection day |
| Ratings or expected goals from another site | Only if dated; a page fetched today includes later matches | Only with a per-match date, through the same cut-off |
| Collection-time odds (`BFEH`, `AvgH`, `BFE>2.5`) | Yes | Yes; the Betfair one is also the execution price |
| Closing odds (any column with `C` after the bookmaker code) | No | Never; `BFEC` is the benchmark |

Team strengths fitted on a whole season and then applied to matches inside it are the commonest football leak. Every fit is walk-forward: refit using only matches before each collection day.

### Default pass threshold

A hypothesis passes a split when all hold, at the chosen haircut:

1. Mean close-implied return is positive, with the lower end of a 95% bootstrap interval above zero. The bootstrap resamples whole match weeks (Friday to Thursday).
2. It is positive in at least four of the six leagues.
3. Realised return after commission is consistent with it: the realised 95% interval contains the close-implied mean.

A mean CLV above 3%, or a realised return above 5% after commission, is treated as a bug until a leakage audit says otherwise.

### Sample size

These are my estimates. The 7% spread comes from the 2024/25 files; the bet rate is a guess until Phase 5.

| Measure | Bets for a verdict | Time at 300 to 800 bets a season |
|---|---|---|
| CLV, 1% edge after costs | About 200 | Part of one season |
| CLV, 0.5% edge after costs | About 800 | One to three seasons |
| Profit, 2% edge | 10,000 to 20,000 | Decades; not reachable |

### Data terms

football-data.co.uk says its data is "intended for private individuals only", not for commercial or data-training products "using automated bots/scrapers/AI". My reading is that a one-off download of about 120 files for personal analysis is within that, but the wording names bots and AI, so read it yourself. If in doubt, download by hand or email the owner. Never redistribute the files or commit them. Understat has no official API and its terms were not read; treat it as optional and check first.

### Spend

The plan avoids the £499 live key. What that gives up:

- An exact closing price. The delayed one is up to three minutes early.
- Traded volume per outcome and prices beyond the best three.
- Tick-level recording, which is what a later study of waiting orders would need.
- Certainty. It depends on Betfair accepting months of real bets on the delayed key.

## Phase 0: repo, rules and data splits

**Deliverables**
- Python 3.12 project managed with `uv`, with `ruff` and `pytest`.
- Layout: `src/`, `data/raw/` and `data/derived/` (both git-ignored), `registry/` for the trial log and hypotheses, `reports/`.
- `CLAUDE.md` and `AGENTS.md` containing the guardrails from `prompts.md`.
- `config/splits.toml` with the seasons above, the six division codes and the haircut.
- Secrets policy: credentials live outside the repo; `.gitignore` covers `.env`, certificates and keys.

**Gate:** `uv run pytest` passes and a fresh clone sets up with one command.
**If it fails:** fix it; no judgement is involved.

## Phase 1: ingest the archive and start the price recorder

Do items 5 to 7 in `README.md` first.

**Deliverables**
- A first step that re-checks the table under "What I checked in the data" against the live site, including the fixtures page, and reports any difference.
- Train and validation season files in `data/raw/`, by the method you chose. A scripted downloader makes one request a second, never re-downloads, and refuses later seasons without the holdout flag.
- A parser to typed Parquet, and a data quality report: matches per league and season, missing or duplicate fixtures, missing odds, overround distributions, team names that change spelling.
- A price recorder using the delayed key. For every match in the six leagues it stores the best three back and lay prices and amounts in both markets at each collection time and every minute over the last ten before kick-off. It places no bets.

**Gate:** match counts equal the known league sizes, anomalies are listed, and the recorder has captured one full weekend unattended.
**If it fails:** fix the parser or document the anomaly. If the Betfair columns differ from the table above, stop and revise this plan first.

## Phase 2: evaluation harness, then freeze it

**Deliverables**
- One function that takes proposed bets (match, market, outcome, stake, minimum price) and a split name, applies the bet rule and cost model, and returns settled results.
- It refuses splits whose data is not permitted and logs every run to `registry/trials.jsonl` with timestamp, git commit, hypothesis ID, split, number of bets and headline result.
- A feature-time check: every feature row carries the latest match date it used, and the harness rejects any row where that is on or after the collection day.
- Output per run: bets, mean CLV, close-implied return, realised return before and after commission, 95% intervals by match-week bootstrap, results by league, market, odds band and half-season, all at haircuts of 0%, 1% and 2%.
- A proxy mode for train that scores against bookmaker average prices and labels its output as proxy.
- Tests on synthetic data: a known edge, zero edge, commission netting inside one market but not across the two markets of a match, a bet whose minimum price is above the price on offer, and a feature that breaks the cut-off.

**Gate:** the synthetic tests pass, a second model has reviewed the harness, and you have read it. Then tag the commit `harness-v1`.
**If it fails:** fix and re-review. After the tag, changes need your approval and a new tag, and earlier trials are re-run.

## Phase 3: baseline and price reconciliation

**Deliverables**
- Baseline on validation: backing every outcome at the collection-time Betfair price. CLV, close-implied and realised return by league, market and odds band, plus a calibration table of implied probability against observed frequency.
- The same on train in proxy mode, and a comparison of Betfair and bookmaker-average probabilities on validation, to show how far the proxy can be trusted.
- Price reconciliation. You run a tool that fetches the current season file and keeps only fixture names and Betfair price columns for matches the recorder captured. The report compares them with the recorded prices: which side the columns are, when they were collected, how far the recorded best back price was from the column, and how much money was on offer there.
- You set the haircut in `config/splits.toml` from that report, before Phase 4.

**Gate:** blind backing shows a small negative CLV and a negative return after commission, and reconciliation covers at least three match weeks.
**If it fails:** a positive blind result means a bug in ingest or settlement; go back. If recorded prices are worse than the columns by more than 2%, the archive cannot stand in for tradable prices: stop, or record for a season and return.

## Phase 4: price-only hypotheses

**Deliverables**
- Three to five hypotheses written in `registry/hypotheses/` before any is run. Candidates:
  - Reverse favourite-longshot bias: Angelini et al. (2022) report favourites are underpriced in football match odds. Is it large enough to clear the costs?
  - Exchange against bookmakers: when the Betfair price is well above the bookmaker average with margin removed, does it fall back by the close? A positive result here may only be snapshot noise, so it is also a test of the data.
  - Consistency between match odds and over/under 2.5 for the same match.
  - Draw and home-side pricing by league.
- Each run on validation through the harness, and on train in proxy mode where it applies.

**Gate:** informational. This phase teaches the data and exercises the process.
**If nothing passes:** expected. Continue. If something passes, carry it to Phase 6 alongside the model.

## Phase 5: goals model and market blend

**Deliverables**
- A first-stage goals model from results only, in the Dixon and Coles style: attack and defence strength per team, home advantage, more weight on recent matches. It is refitted walk-forward and outputs probabilities for both markets.
- A second-stage model that blends the first-stage probability with the collection-time market probability, fitted on train. The research is clear that any edge sits in the blend, not in the model alone.
- A report of how much the blend improves on the market price alone, by log loss (a score for probability forecasts; lower is better). Expect the gain to be near zero.
- Optional: expected-goals features from Understat for the five leagues it covers, only after item 9 in `README.md` and through the same season gate and cut-off.
- A leakage audit by a second model, plus a manual read of 50 randomly chosen bets from the bet log.
- A pre-registered hypothesis for the final model, margin and markets, with the pass threshold.

**Gate:** the pre-registered model passes the default threshold on validation, shows positive proxy CLV on train, and the leakage audit is clean.
**If it fails:** revise and re-register up to a number of attempts fixed in advance (suggest five), each counted in the trial log. After that, stop. Do not open the holdout to "have a look".

## Phase 6: holdout verdict

**Deliverables**
- You download the 2025/26 files with the holdout flag and run the registered hypotheses once.
- A one-page verdict report committed with the trial log.

**Gate:** the hypothesis passes the default threshold on the holdout.
**If it fails:** the strategy is not taken live. The spare holdout allows one more registered attempt, after you write down what changed and why. After that, further research waits for new seasons.

## Phase 7: execution bot and safety layer

**Deliverables**
- Verified against Betfair's docs before coding: the commission rate on your account and on football markets; how to cancel the unmatched part of a limit order at once; whether a limit order matches at a better price when one is on offer; whether price requests include cross-matched ("virtual") prices; the market type names for match odds and over/under 2.5.
- A team-name table between football-data.co.uk and Betfair, reviewed by you. The bot refuses any match it cannot map.
- A twice-weekly job: fetch the latest results, refit, read the markets, apply the bet rule, place and cancel as the rule says. Then the closing-price poll for every match bet on or considered.
- Safety layer: maximum stake, maximum bets per run, one bet per market ever, daily and total loss stops, refusal when results are stale, a market is in-play or a kick-off has moved, and a dry-run mode.
- Reconciliation each morning against Betfair's cleared orders and account statement, with alerts on any difference, any exception, and any collection time that passes with no run.

**Gate:** two weeks of dry-run with no errors, then five to ten real minimum-stake bets that reconcile to the penny.
**If it fails:** fix it. If Betfair refuses delayed-key betting, decide between the £499 key and stopping.

## Phase 8: forward test at minimum stakes

**Deliverables**
- The bot runs unattended at flat £1 or £2 stakes.
- A weekly report with three gaps per bet: the price the backtest would assume (the archive column less the haircut) against the price matched; the bot's closing price against `BFEC`; and the CLV of orders that did not match against those that did.
- A sample size and verdict rule fixed before the first bet. Suggested: 300 matched bets. Edge if the lower bound of mean close-implied return is above zero, no edge if the upper bound is below zero, otherwise inconclusive.

**Gate:** the pre-set sample is reached. Expect a zero-edge run to lose 2% to 4% of turnover, about £5 to £25.
**Stop early if:** the loss stop is hit, reconciliation fails twice, matched prices fall short of the assumed price by more than the haircut over a month, or unmatched orders show clearly better CLV than matched ones.

## Phase 9: verdict

**Deliverables**
- A written verdict against the pre-set rule: edge, no edge, or inconclusive.
- If there is an edge: stakes raised in steps, each conditional on the three gaps staying small.
- If not: a note on what is cheap to try next, such as the racing plan on the same harness, or buying the live key to record tick data.

## Hypothesis template

Copy to `registry/hypotheses/H-NNN-short-name.md` and fill in before running anything.

```markdown
# H-NNN: short name

- Date written:
- Claim: one sentence stating what is mispriced
- Mechanism: who is on the other side and why they are wrong
- Bet rule: leagues, markets, which outcomes are backed, margin
- Inputs: each field used, and when it is known
- Splits to run: train (proxy), validation
- Haircut: the configured value, or state the change and the reason
- Pass threshold: default, or state the change and the reason
- Maximum variants: how many parameter settings will be tried
- What would make me abandon it:
```
