# Phased plan

Durations assume 5 to 8 hours a week. Each phase ends at a gate. Do not start the next phase until the gate passes; the "if it fails" line says what to do instead.

| Phase | What | Duration | Spend | Real bets |
|---|---|---|---|---|
| 0 | Repo, rules and data splits | 1 week | £0 | No |
| 1 | Ingest the BSP archive | 1 to 2 weeks | £0 | No |
| 2 | Evaluation harness, then freeze it | 1 to 2 weeks | £0 | No |
| 3 | Baseline: what blind betting at BSP returns | 1 week | £0 | No |
| 4 | Price-only hypotheses | 2 to 4 weeks | £0 | No |
| 5 | Form data and the model | 4 to 8 weeks | £25 to £75 | No |
| 6 | Holdout verdict | 1 day | £0 | No |
| 7 | Execution bot and safety layer | 2 to 3 weeks | £0 | A handful |
| 8 | Forward test at minimum stakes | 3 to 6 months | VPS if used | Yes |
| 9 | Verdict | 1 week | £0 | No |

## Fixed design decisions

### Data splits

| Split | Dates | On disk | Used for |
|---|---|---|---|
| Train | 2008-01-01 to 2021-12-31 | Yes | Fitting models, exploring |
| Validation | 2022-01-01 to 2024-12-31 | Yes | Choosing between variants |
| Holdout | 2025-01-01 to 2026-10-31 | **No, until Phase 6** | One verdict run |
| Forward | From the first live bet | Recorded live | Phase 8 |

The Betfair archive starts in April 2007 for win markets; 2007 is dropped as a partial first year. The market has changed since 2008 (liquidity has fallen every year since 2016), so every result is also reported by year.

### The bet rule

For each runner the model outputs a win probability `p`. The fair price is `1/p`. The bot places a back bet at BSP with a minimum price of `fair price × (1 + margin)`. On history, the bet is counted if and only if the recorded BSP is at or above that minimum. This makes the backtest rule and the live rule the same, including the fact that you are only matched when the market disagrees with you.

Laying is out of scope until backing has a verdict. Lay liability at long odds makes stakes and risk much larger.

### Cost model

- Stake is flat: one unit per bet during all research. No Kelly sizing until a live edge is established.
- A winning back bet at price `o` returns `stake × (o − 1)`; a losing one returns `−stake`.
- Commission is 2% of **net winnings per market**, charged only when the market's net is positive.
- With 2% commission, a back bet at price `o` breaks even when the true probability is `1 / (1 + 0.98 × (o − 1))`.

### When each archive field is known

The model may only use fields known at decision time. The live bot decides a fixed number of minutes before the off.

| Field | Meaning | Known before the off? | Model input? |
|---|---|---|---|
| `EVENT_DT`, `MENU_HINT`, `EVENT_NAME` | Time, course, race description | Yes | Yes |
| Number of runners | Field size | Yes, bar late withdrawals | Yes |
| `MORNINGWAP`, `MORNINGTRADEDVOL` | Average price and volume matched before 11am | Yes, for bets placed after 11am | Yes, but often thin |
| `PPWAP`, `PPTRADEDVOL`, `PPMAX`, `PPMIN` | Pre-off average price, volume and extremes | Only partly: they include trading up to the off | Research only, flagged as optimistic |
| `BSP` | Starting price | No | Never as an input; it is the execution price |
| `IPMAX`, `IPMIN`, `IPTRADEDVOL` | In-play extremes and volume | No | Never |
| `WIN_LOSE` | Result | No | Never; it is the label |

A model using `PPWAP` can only be trusted after the forward test, where the live price at decision time replaces it.

### Default pass threshold

A hypothesis passes a split when both hold:

1. Return per unit staked after commission is positive, with the lower end of a 95% bootstrap interval above zero. The bootstrap resamples whole race days, not individual bets.
2. The return is positive in at least two-thirds of the calendar years in the split.

Anything above about 5% return after commission is treated as a bug until a leakage audit says otherwise.

## Phase 0: repo, rules and data splits

**Deliverables**
- Python 3.12 project managed with `uv`, with `ruff` and `pytest`.
- Layout: `src/` for code, `data/raw/` and `data/derived/` (both git-ignored), `registry/` for the trial log and hypotheses, `reports/` for outputs.
- `CLAUDE.md` containing the guardrails from `prompts.md`.
- `config/splits.toml` holding the split dates above, read by everything else.
- Secrets policy: credentials live outside the repo; `.gitignore` covers `.env`, certificates and keys.

**Gate:** `uv run pytest` passes on an empty test suite and a fresh clone can be set up with one command.
**If it fails:** fix it; nothing else depends on judgement here.

## Phase 1: ingest the BSP archive

**Deliverables**
- A downloader for `promo.betfair.com/betfairsp/prices` covering UK and Irish win and place files. It refuses any date after the validation end date unless run with an explicit holdout flag. It is polite: about one request a second, resumable, and never re-downloads a file it has.
- Raw files kept unmodified. A parser writes one Parquet table per market type with typed columns.
- A data quality report: files per year, missing days, duplicate rows, null or zero BSP, values seen in `WIN_LOSE`, races with no winner or more than one.

**Gate:** all of these hold on train and validation.
- Every win market has exactly one winner, or is listed as a dead heat or void.
- The sum of `1/BSP` across runners in a race is close to 1 for the great majority of races. The report shows the distribution.
- Row counts by year look smooth, with no unexplained gaps.

**If it fails:** fix the parser or document the anomaly. Do not paper over it with filters.

## Phase 2: evaluation harness, then freeze it

This comes before any strategy work on purpose.

**Deliverables**
- One function that takes a table of proposed bets (market, runner, side, stake, minimum price) and a split name, and returns settled results using the cost model above.
- It refuses to run on a split whose data is not permitted, and it appends every run to `registry/trials.jsonl` with the timestamp, git commit, hypothesis ID, split, number of bets and headline result. Strategy code cannot bypass the log.
- Output per run: number of bets, return per unit staked before and after commission, 95% interval by race-day bootstrap, results by year and by odds band, and the share of profit coming from the best 1% of days.
- Tests on synthetic data with a known built-in edge and with zero edge, plus a test that commission nets per market.

**Gate:** the synthetic tests recover the known edge within the interval, a second model has reviewed the harness, and you have read it. Then tag the commit `harness-v1`.
**If it fails:** fix and re-review. After the tag, changes need your explicit approval and a new tag, and prior trials are re-run.

## Phase 3: baseline

**Deliverables**
- Return from backing every runner at BSP, before and after commission, overall, by odds band, by year, by field size and by race type parsed from `EVENT_NAME`.
- A calibration table: for each odds band, the implied probability against the observed win rate.

**Gate:** backing everything returns close to zero before commission and negative after. That confirms the pipeline. The research found no published version of this table, so it is also a small original result.
**If it fails:** a clearly positive blind return means a bug in ingest or settlement. Go back to Phase 1 or 2.

## Phase 4: price-only hypotheses

**Deliverables**
- Three to five hypotheses written in `registry/hypotheses/` before any is run, using the template at the end of this file. Candidates:
  - Favourite-longshot bias: are short prices under-bet and long prices over-bet at BSP, by enough to clear 2%?
  - Race-type effects: do handicaps, maidens or large fields show different calibration?
  - Morning price moves: does the change from `MORNINGWAP` to the price near the off predict mispricing at BSP?
  - Place markets: are place BSPs consistent with win BSPs for the same runners?
- Each is run on train, then on validation, through the harness.

**Gate:** informational. This phase teaches the data and exercises the process.
**If nothing passes:** expected. Continue to Phase 5. If something passes, carry it forward as a candidate for Phase 6 alongside the model.

## Phase 5: form data and the model

**Deliverables**
- A form data source chosen and its terms read. Options from the research: The Racing API (permits use in betting models; price unconfirmed), HorseRaceBase (£25 a month), Proform (£37.50 a month). Subscribe for one or two months and export what the terms allow.
- Entity matching between the form data and the BSP archive by date, course, off time and horse name, with a measured match rate and a list of failures.
- A feature table where every feature carries the time it became known.
- A first-stage model predicting win probability from form alone, normalised within each race. Then a second-stage model that combines it with a market price known before the off. The research is clear that the combination, not the model alone, is where any edge sits.
- A leakage audit by a second model, plus a manual read of 50 randomly chosen bets from the bet log.
- A pre-registered hypothesis for the final model and margin, with the pass threshold.

**Gate:** the pre-registered model passes the default threshold on validation, and the leakage audit is clean.
**If it fails:** you may revise and re-register up to a number of attempts you fix in advance (suggest five), each counted in the trial log. After that, stop or change sport. Do not open the holdout to "have a look".

## Phase 6: holdout verdict

**Deliverables**
- You run the downloader with the holdout flag, then run the registered hypotheses once.
- A one-page verdict report committed with the trial log.

**Gate:** the hypothesis passes the default threshold on the holdout.
**If it fails:** the strategy is not taken live. The holdout is now spent; any further research needs a new holdout, which means waiting for new data. You have spent under £100 and have a working research system.

## Phase 7: execution bot and safety layer

Start the account steps (items 7 and 8 in `README.md`) during Phase 5; they take calendar time.

**Deliverables**
- Login with `betfairlightweight` using the certificate, the delayed key and credentials from outside the repo.
- A daily job: fetch the day's UK and Irish win markets, build features, produce fair prices, and place `LIMIT_ON_CLOSE` back bets a fixed number of minutes before each off.
- Safety layer: hard maximum stake per bet, maximum bets per day, daily and total loss stops, refusal to bet when any input is missing or stale, and a dry-run mode that logs intended bets without placing them.
- Reconciliation: each morning, compare the bot's ledger with Betfair's cleared orders and account statement, and alert on any difference.
- Alerts to your phone on any exception, any reconciliation difference and any day with no bets when racing was on.
- Verified against Betfair's docs before coding: `LIMIT_ON_CLOSE` semantics, minimum stake and liability for BSP bets, and the commission rate on your account.

**Gate:** one week of dry-run with no errors, then five to ten real minimum-stake bets that reconcile to the penny.
**If it fails:** fix it. No strategy judgement is involved, only correctness.

## Phase 8: forward test at minimum stakes

**Deliverables**
- The bot runs unattended every racing day at flat minimum stakes.
- A weekly report comparing live results with what the harness predicts for the same races once the archive files for those days are published. The two should match almost exactly; any gap is a bug or a rule difference.
- A sample size and verdict rule fixed before the first bet.

**Gate:** the pre-set sample is reached. Expect a zero-edge strategy to lose around 2% of turnover, which at minimum stakes is tens of pounds over the whole test.
**Stop early if:** the loss stop is hit, reconciliation fails twice, or live results diverge from the harness for a reason you cannot explain.

## Phase 9: verdict

**Deliverables**
- A written verdict against the pre-set rule: edge, no edge, or inconclusive.
- If there is an edge: a staking plan that raises stakes in steps, each step conditional on live results still matching the harness.
- If not: a note on what the harness and data make cheap to try next, such as football pre-match models, or buying the live key to record tick data for pre-race research.

## Hypothesis template

Copy to `registry/hypotheses/H-NNN-short-name.md` and fill in before running anything.

```markdown
# H-NNN: short name

- Date written:
- Claim: one sentence stating what is mispriced
- Mechanism: who is on the other side and why they are wrong
- Bet rule: exactly which runners are backed and at what minimum price
- Inputs: each field used, and when it is known
- Splits to run: train, validation
- Pass threshold: default, or state the change and the reason
- Maximum variants: how many parameter settings will be tried
- What would make me abandon it:
```
