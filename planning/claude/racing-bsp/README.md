# Betfair bot plan: racing at Betfair Starting Price

This folder plans the experiment chosen from the research in `research/claude/`. Read this file first, then `plan.md`, then use `prompts.md` when you start each phase.

| File | What it is |
|---|---|
| `README.md` | The strategy in brief, what I need from you, and the steps that are easy to miss |
| `plan.md` | Ten phases, each with deliverables, a gate, and the rule for what happens if the gate fails |
| `prompts.md` | A proposed `CLAUDE.md` for the repo and one ready-to-paste prompt per phase |

## The experiment in brief

**Question:** can a model pick UK and Irish horse racing runners that the Betfair Starting Price (BSP) misprices by more than commission?

**Method:** build a model that outputs a fair price for each runner, then back a runner only when the BSP is better than that fair price by a set margin. Betfair has an order type for exactly this (a BSP bet with a price limit, `LIMIT_ON_CLOSE`), so the rule tested on history is the same rule executed live.

**Why this shape:** it removes the two things that sink most hobby bots. There is no fill problem, because a BSP bet is matched at the reconciled price or not at all. And there is no data cost to start, because Betfair publishes the BSP and result for every UK and Irish runner since 2007 for free.

**Expected result:** roughly break-even. BSP is the sharpest price in the market. The experiment is worth running because it gives a trustworthy answer cheaply, and the evaluation harness carries over to harder strategies.

**Spend:** £0 until Phase 5, then about £25 to £75 on racing form data. The £499 live API key is not in this plan; it is only needed if you later move to strategies that depend on live prices.

## What I need from you

These are decisions and actions only you can take. The first four are needed before Phase 0; the rest can wait until the phase named.

| # | Needed | When | Why |
|---|---|---|---|
| 1 | **Hours per week** you expect to give this | Now | The phase durations in `plan.md` assume about 5 to 8 hours a week |
| 2 | **Betting bankroll**, separate from the £500 tooling budget, and the total loss at which you stop | Now | The forward test places real bets. A few hundred pounds is ample at minimum stakes |
| 3 | **Where the bot runs** in Phase 8: an always-on Mac at home, or a small VPS (about £5 a month) | Now, can revise | It must run unattended every racing day for months |
| 4 | **Agreement to the guardrails** in `prompts.md` | Now | They restrict what coding agents may touch. They only work if you enforce them |
| 5 | A **pass threshold** for each hypothesis, written before the data is touched | Phases 4 and 5 | The plan supplies a default; you own the final number |
| 6 | A **form data subscription** (about £25 to £40 a month) and a read of its terms on export and automated use | Phase 5 | This is the only paid input. I could not read The Racing API's prices; check them yourself |
| 7 | A **verified, funded Betfair account** set to the Basic rewards package (2% commission), a **delayed application key**, and an **API login certificate** | Phase 7 | Account verification can take days, so start it early |
| 8 | A **written answer from Betfair Developer Support** that placing real BSP bets on the delayed key over months is acceptable | Before Phase 7 | Betfair describes the delayed key as for development. If they say no, the choice is the £499 key or stopping |
| 9 | **You** run the holdout download and the verdict run | Phase 6 | The holdout only means something if no agent has seen it |

## Steps that are easy to miss

- **Lock the evaluation code before any strategy research.** The harness, cost model and date splits are written and tested first, then frozen. Agents may propose changes but not make them. Without this, an agent searching for edge will find bugs in the scoring instead.
- **Keep the holdout off the disk.** The plan never downloads 2025 or 2026 data until the single verdict run. A holdout that exists only on Betfair's server cannot leak.
- **Count every trial.** Each backtest run is appended to a log by the harness. The best of fifty noise strategies always looks good, and the count is what lets you discount it.
- **Audit when each field is known.** Several columns in the BSP files are only known after the race starts. `plan.md` has the table. Using one as a model input is the commonest way to create a fake edge.
- **Charge commission per race, not per bet.** Betfair takes commission on net winnings in each market. Two bets in one race net against each other first.
- **Verify the Betfair rules the plan depends on** before building on them: the `LIMIT_ON_CLOSE` order type's exact behaviour, the minimum stake for BSP bets, and your account's commission rate. The research could not read every Betfair page directly.
- **Build the safety layer before the first real bet.** That means a maximum stake, a daily loss stop, a dry-run mode, and a daily reconciliation of the bot's ledger against Betfair's account statement. One bot in the research lost $2,304 on a single bet through a parsing bug with no stake cap.
- **Use a second model as reviewer.** You have Claude and Codex. Have one write and the other audit for look-ahead and review the bet log. In the one published case study, bet-log review caught bugs that code review missed.
- **Decide the stop rule in advance.** Each gate in `plan.md` says what failing means. Deciding at the time, with results in front of you, is how experiments drift.
- **Set a deposit limit on the Betfair account** at the bankroll figure from item 2.

## Things I have assumed

- Python 3.12 managed with `uv` (the machine's system Python is 3.9.6, which is too old for current `flumine` and `betfairlightweight` releases).
- Work lands on `main`, as you chose.
- You are sport-agnostic, so racing was picked for its data, not for interest. If you would rather follow a sport you know, say so before Phase 4; football is the second-best fit.
