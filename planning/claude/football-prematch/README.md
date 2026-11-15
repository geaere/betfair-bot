# Betfair bot plan: football pre-match model

This folder plans a football experiment using the research in `research/claude/`. It is the equivalent of `planning/claude/racing-bsp/` and stands alone, so you can do it instead of the racing plan or after it. Read this file first, then `plan.md`, then use `prompts.md` when you start each phase.

| File | What it is |
|---|---|
| `README.md` | The strategy in brief, what I need from you, and the steps that are easy to miss |
| `plan.md` | Ten phases, each with deliverables, a gate, and the rule for what happens if the gate fails |
| `prompts.md` | A proposed `CLAUDE.md` for the repo and one ready-to-paste prompt per phase |

## The experiment in brief

**Question:** can a football model, blended with the market price, find bets on the Betfair Exchange whose price is better than the price the same market closes at, by more than the costs?

**Method:** build a goals model that gives a probability for each outcome in two markets: match odds (home, draw, away) and over/under 2.5 goals. Blend it with the market's own price. Twice a week, back (bet for) an outcome when the price on offer beats the blended fair price by a set margin. Judge each bet first on closing line value (CLV): how the price taken compares with the closing price, the last price before kick-off. Profit is reported but is too slow a measure to decide anything.

**Why this shape:**
- Football has no Betfair Starting Price, so unlike the racing plan a bet is not guaranteed to match. The bot therefore only takes prices already on offer and never leaves an order waiting. That costs the spread (the gap between the best price to back at and the best price to lay at) but stops the backtest being flattered by fills that would not have happened.
- CLV needs about 200 to 800 bets to give a verdict. Profit needs 10,000 to 20,000, which is decades at this bet rate.
- The data is free, and the delayed API key is enough because decisions are made hours or days before kick-off.

**Expected result:** roughly break-even before costs and a small loss after them. On the 2024/25 files I checked, backing every outcome at the archived Betfair price lost about 2% before commission and 3% after. No published model beats the Betfair closing price after commission. The experiment is worth running because it answers the question cheaply and leaves you a research system you can trust.

**Spend:** £0 on data. The £499 live key is not needed. A VPS at about £5 a month is the only likely cost. What you give up by avoiding the live key is listed in `plan.md` under "Spend".

## What I need from you

These are decisions and actions only you can take. The first six are needed before or during Phase 1, earlier than in the racing plan, because the price recorder needs a Betfair login from the start.

| # | Needed | When | Why |
|---|---|---|---|
| 1 | **Hours per week** you expect to give this | Now | Phase durations assume 5 to 8 hours a week |
| 2 | **Betting bankroll** and the total loss at which you stop | Now | £100 to £200 is ample at £1 or £2 stakes |
| 3 | **Where the recorder and bot run**: an always-on Mac or a small VPS | Now | The recorder starts in Phase 1 and must run around every kick-off |
| 4 | **Agreement to the guardrails** in `prompts.md` | Now | They only work if you enforce them |
| 5 | A **verified Betfair account**, a **delayed application key** and an **API login certificate** | Phase 1 | The recorder reads prices with them. Funding and the Basic rewards package (2% commission, unconfirmed) can wait until Phase 7 |
| 6 | A read of **football-data.co.uk's terms** and a choice: scripted download or by hand | Phase 1 | The site says its data is for private individuals and names automated bots, scrapers and AI. See "Data terms" in `plan.md` |
| 7 | A **written answer from Betfair Developer Support** that placing real bets on the delayed key for months is acceptable | Ask in Phase 1, needed before Phase 7 | Betfair describes that key as for development. If they say no, the choice is the £499 key or stopping |
| 8 | The **price haircut** and the **pass threshold**, written before results are seen | Phases 3 to 5 | The plan supplies defaults; you own the final numbers |
| 9 | A read of **Understat's terms**, if you want expected-goals data | Phase 5, optional | It has no official API; the data is scraped from its pages |
| 10 | **You** run the holdout download and the verdict run | Phase 6 | The holdout only means something if no agent has seen it |
| 11 | **You** review the table matching team names between the data and Betfair | Phase 7 | A wrong match means a bet on the wrong team |

## Steps that are easy to miss

- **The archive has Betfair prices for two finished seasons only.** I checked the files: the Betfair Exchange columns start in 2024/25. One season is validation and one is the holdout. Earlier seasons can train the model but cannot test bets at exchange prices.
- **Bet when the archive's prices were collected.** The site says it collects odds on Friday afternoons for weekend games and Tuesday afternoons for midweek games. The live bot must bet at those times, or the backtest describes a different strategy.
- **Cut features off at bet time, not kick-off.** A Sunday match bet on Friday must not use Saturday's results. This is the football version of look-ahead and it is easy to get wrong.
- **An archived price is a snapshot, not a fill.** It carries no depth, the amount of money on offer. The recorder and a price haircut handle this; see `plan.md`.
- **Capture your own closing price.** Nothing publishes it for your bets in time. The bot polls each market until kick-off.
- **Agents already know past results.** Coding models were trained on text that includes recent seasons. Every team-specific number must be fitted by code from permitted data, never typed in.
- **Lock the evaluation code before any strategy research**, count every trial, and keep the holdout off the disk, as in the racing plan.
- **Charge commission per market.** Match odds and over/under 2.5 are separate markets for the same match and do not net against each other.
- **Verify the Betfair rules the plan depends on**: the commission rate on football markets, how a limit order that cannot match is cancelled, and what the delayed key returns. The research could not read every Betfair page.
- **Build the safety layer before the first real bet**: stake cap, loss stops, dry-run mode, daily reconciliation against Betfair's statement.
- **Use a second model as reviewer**, and have it read the bet log, not only the code.
- **Plan around the season.** The six leagues stop from late May to early August. A forward test started in March gets about ten weeks before the break.
- **Set a deposit limit on the Betfair account** at the bankroll figure. Net deposits above £150 in 30 days also trigger a routine affordability check.

## Things I have assumed

- Python 3.12 managed with `uv`; work lands on `main`; repo conventions as in the racing plan.
- Six leagues, chosen because their archived Betfair prices are the tightest: the English Premier League and Championship, the German Bundesliga, Spanish La Liga, Italian Serie A and French Ligue 1. That is 2,304 matches a season.
- Backing only. Laying (betting against an outcome) is left until backing has a verdict.
- You have not built the racing plan. If you have, Phase 0 reduces to adding the football splits and rules, Phase 2 to adding football settlement and CLV to the existing harness under a new tag, and Phase 7 to the order logic and closing-price capture, since login, the safety layer, reconciliation and alerts already exist.
