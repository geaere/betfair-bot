# Prompts

One prompt per phase, ready to paste. Each assumes the agent is working in this repo and can read `planning/claude/football-prematch/plan.md`.

## How to run the sessions

- **One phase per session.** Start each in plan mode, read the plan the agent proposes, and only then let it build. A fresh session per phase keeps the context clean.
- **Writer and reviewer are different models.** If Claude writes a phase, give Codex the review prompt for it, and the reverse. The reviewer gets the diff and the plan, not the writer's explanation.
- **You run the gates.** Check each gate in `plan.md` yourself before moving on. Do not accept "the gate passes" from the agent without seeing the evidence.
- **If an agent reports an edge, your next prompt is the leakage audit**, not the next phase.
- **You run anything that touches the holdout or the current season file**: the holdout download, the reconciliation fetch and the verdict run.

## Proposed CLAUDE.md

Put this at the repo root in Phase 0. Codex reads `AGENTS.md`; use the same text there. If the racing plan's file already exists, add this as a second section and keep both sets of rules.

```markdown
# betfair-bot

Research and execution code for a football pre-match strategy on the Betfair Exchange.
The plan is in planning/claude/football-prematch/plan.md. Follow the phase you are asked to
work on and no other.

## Rules that are never broken

1. Never read, download or query data for a season after the last validation season in
   config/splits.toml, from any source. Only the human runs the holdout download and the
   reconciliation fetch.
2. Never edit the evaluation harness, the cost model, config/splits.toml or
   registry/trials.jsonl once the harness-v1 tag exists. If one looks wrong, stop and explain.
3. Every backtest goes through the harness. Never compute strategy returns any other way.
4. Never run a hypothesis that has no file in registry/hypotheses/ written before the run.
5. A feature for a match may use only matches played before that match's collection day, as
   plan.md defines it. Closing odds and anything from the match itself are never inputs.
6. Never type in knowledge about teams, players or seasons. Every team-specific number is
   fitted by code from permitted data. You were trained on text that includes past results.
7. Never assume an order that waits in the queue gets matched. Bets are taken at the price on
   offer or not placed.
8. Never place, change or cancel a real bet unless the human has asked for it in this session.
   Default to dry-run.
9. Never write credentials, keys or certificates into the repo, a log or a prompt.
10. A mean closing line value above 3%, or a return above 5% after commission, is a suspected
    bug. Say so and look for leakage first.

## Working conventions

- Python 3.12, managed with uv. Run tests with `uv run pytest`.
- data/raw is immutable and never committed. Derived tables go in data/derived and must be
  rebuildable from raw.
- Report results with the number of bets, the interval and the haircut, never a bare percentage.
- Results on train are proxy results against bookmaker prices. Always label them as proxy.
- Say plainly when something failed, was skipped or was not verified.
```

## Phase 0: repo, rules and data splits

```text
Set up this repo for Phase 0 of planning/claude/football-prematch/plan.md.

Create a Python 3.12 project managed with uv, with ruff and pytest configured. Create the
directory layout the plan describes, a .gitignore that excludes data/, .env, and any key or
certificate files, and config/splits.toml with the seasons, the six division codes and the
default haircut from the plan. Put the "Proposed CLAUDE.md" text from
planning/claude/football-prematch/prompts.md at the repo root as CLAUDE.md and as AGENTS.md.

Done means: a fresh clone can be set up with one documented command, `uv run pytest` passes,
and the README explains the layout in under 30 lines. Do not write any data or strategy code.
```

## Phase 1: ingest the archive and start the price recorder

Before prompting, read the site's terms and choose how the files are fetched.

```text
Build the ingest half of Phase 1 of planning/claude/football-prematch/plan.md.

Start by fetching football-data.co.uk's notes.txt, its data page and its fixtures page, plus
the 2024/25 file for one division. Compare what you find with the table "What I checked in
the data" in the plan and tell me every difference, and the exact collection times the
fixtures page gives, before you design anything. Do not fetch any later season.

The season files are fetched <by a script | by me, by hand, into data/raw/>. If by script:
one request a second, resumable, never re-downloads, and refuses any season after the last
validation season in config/splits.toml unless passed an explicit --holdout flag.

Then build a parser that writes typed Parquet tables to data/derived, and a data quality
report in reports/ covering the checks listed under Phase 1 in the plan. Report anomalies;
do not filter them out silently.

Done means: train and validation data are on disk, the report exists, tests cover the parser
on files from an early, a middle and a recent season, and a test proves the season gate
refuses holdout seasons.
```

```text
Build the price recorder from Phase 1 of planning/claude/football-prematch/plan.md.

Before writing code, read the current Betfair API documentation and tell me, with links:
what the delayed application key returns for football markets, the market type names for
match odds and over/under 2.5 goals, how a market is flagged as in-play, and whether price
requests can include cross-matched prices. Flag anything that differs from the plan.

Then build a recorder with betfairlightweight that, for every match in the six leagues,
stores the best three back and lay prices and amounts in both markets at each collection
time and once a minute over the ten minutes before kick-off until the market is in-play.
Store raw responses unmodified with the time each was received. Credentials come from the
environment. The recorder contains no code path that can place a bet; add a test for that.

Done means: it runs unattended as a scheduled job, alerts me if a collection time passes
with nothing stored, and has captured today's markets.
```

## Phase 2: evaluation harness

```text
Build Phase 2 of planning/claude/football-prematch/plan.md: the evaluation harness. This
code will be frozen after review, so correctness matters more than speed or elegance.

Implement the bet rule, the haircut, the closing line value definitions and the cost model
exactly as the plan states them. Commission nets within a market and never across the two
markets of one match. The harness takes proposed bets and a split name, settles them, checks
that no feature used a match on or after the collection day, and appends a record of the run
to registry/trials.jsonl. It must refuse splits that are not permitted and must not be
bypassable by strategy code. Train runs use proxy mode and say so in their output.

Write tests first, covering every case listed under Phase 2 in the plan, plus a postponed
match and a match with a missing Betfair price.

Done means: all tests pass, the output includes every figure listed under Phase 2 in the
plan at haircuts of 0%, 1% and 2%, and you have written a short note of every assumption you
made that the plan did not specify, for me to confirm before the harness is tagged.
```

Review prompt for the other model:

```text
Review the evaluation harness in this repo against the fixed design decisions and Phase 2 in
planning/claude/football-prematch/plan.md. You are looking for ways it could overstate
results: closing odds or results reaching bet selection, the feature cut-off using kick-off
time instead of the collection day, the haircut not applied, bets counted when the price on
offer is below the minimum, commission netted across markets, the closing probability
computed wrongly, the bootstrap resampling bets instead of match weeks, proxy results not
labelled, or any path that skips the trial log. For each finding give the file and line, a
concrete input that triggers it, and the wrong output. Do not fix anything.
```

## Phase 3: baseline and price reconciliation

```text
Run the baseline from Phase 3 of planning/claude/football-prematch/plan.md using the frozen
harness.

Register it as hypothesis H-000 "back every outcome at the collection-time Betfair price"
first. Produce the report on validation, and on train in proxy mode: closing line value,
close-implied return and realised return by league, market and odds band, the calibration
table, and the comparison of Betfair with bookmaker-average probabilities on validation.

If backing everything shows a positive closing line value or a positive return before
commission, stop and tell me, because that points to a bug in ingest or settlement.
```

After you have run the reconciliation fetch yourself:

```text
Write the price reconciliation report from Phase 3 of the plan. The extract of archive
prices for recorded matches is at <path>; use nothing else from the current season.

For each recorded match and outcome, compare the archive's collection-time and closing
Betfair prices with what the recorder stored. Tell me: whether the archive prices look like
back prices, lay prices or last traded prices; the time of day at which the recorder's
prices agree best with each column; the distribution of the gap between the archive price
and the recorded best back price; and how much money was on offer at the best price, by
league and market. Recommend a haircut and show the evidence. Do not change any config.
```

## Phase 4: price-only hypotheses

```text
We are in Phase 4 of planning/claude/football-prematch/plan.md. Read reports/ for the
baseline and the reconciliation first.

Propose five hypotheses about mispricing in the collection-time Betfair prices that use only
fields the plan lists as known at bet time. For each, write a file in registry/hypotheses/
using the template in the plan, including the mechanism: who is on the other side of the bet
and why they would be wrong. Stop there and show me the five files. Do not run anything
until I approve them.
```

After you approve:

```text
Run the approved hypotheses through the harness on validation, and on train in proxy mode
where the hypothesis allows. Report each against the default pass threshold in the plan,
with bets, closing line value, close-implied return, realised return, intervals and results
by league. Include the ones that failed. Do not tune any parameter beyond the maximum
variants stated in each hypothesis file.
```

## Phase 5: goals model and market blend

```text
We are in Phase 5 of planning/claude/football-prematch/plan.md.

Step 1: build the first-stage goals model from results only, as the plan describes. It must
be refitted walk-forward, so that the probabilities for a match come from a fit that used
only matches before that match's collection day. Write a test that proves this by changing
a later result and showing earlier predictions do not move. Report log loss on train and
validation against the market price alone. Do not place any bets through the harness yet.
```

```text
Step 2: write the hypothesis file for the blended model, then build it. Fit the second
stage on train, combining the first-stage probability with the collection-time market
probability. Apply the bet rule with the registered margin and run it through the harness on
train in proxy mode and on validation. Report against the default pass threshold. Also
report the weight the blend gives the model and how much it improves log loss over the
market alone, since that is the real test of whether the model adds information. If the
weight is near zero, say so plainly.
```

Optional, after you have read Understat's terms:

```text
Step 3: add expected-goals features. The data is at <path> and its terms allow <what you may
do with it>. It must pass the same season gate and collection-day cut-off as everything
else; a season total fetched today is not usable. Register a new hypothesis before running.
```

Leakage audit prompt for the other model:

```text
Audit this repo's feature pipeline and model for look-ahead. The rule is in
planning/claude/football-prematch/plan.md under "When each field is known". Check in
particular: team strengths or season averages fitted on matches at or after the match they
are applied to; weekend matches using results from earlier the same weekend; postponed
matches ordered by round instead of date; league tables computed at season end; promoted
teams given values from later matches; any closing-odds column reaching a feature; any
external data without a per-match date; and any hard-coded team, player or season value.
Then take 50 bets at random from the latest bet log and, for each, verify by hand from the
raw files that every input predates that match's collection day. Report each problem with
file, line and an example match. Do not fix anything.
```

## Phase 6: holdout verdict

No agent prompt for the download; you run it. Then:

```text
The holdout season is now on disk and I am authorising one run. Run only the hypotheses
listed in registry/hypotheses/ as registered for the holdout, once each, through the harness
on the holdout split. Write the verdict report against the pass threshold. Do not change any
code, parameter, haircut or threshold, and do not run anything a second time.
```

## Phase 7: execution bot and safety layer

```text
Build Phase 7 of planning/claude/football-prematch/plan.md. Before writing code, read the
current Betfair API documentation and tell me, with links: how to place a limit back order
whose unmatched part is cancelled at once, whether such an order is matched at a better
price when one is on offer, the minimum stake in GBP, what the delayed application key can
and cannot do, and how commission is reported on cleared orders. Flag anything that differs
from the plan.

Then build the team-name table for me to review, the twice-weekly job, the closing-price
poll, the safety layer, reconciliation and alerts the plan lists, using betfairlightweight.
Credentials come from the environment and never appear in the repo or logs. Dry-run is the
default; placing real bets needs an explicit flag and still respects every limit. Write
tests for each limit that prove it blocks a bet, and a test that no order can remain
unmatched after the run ends.

Done means: dry-run works end to end on this week's markets and the tests pass. Do not place
a real bet.
```

```text
Review the execution bot for ways it could lose money through a bug: a stake larger than the
cap, a bet on the wrong team, outcome or market, home and away swapped, a duplicate bet
after a retry, an order left waiting in the queue, betting on stale results or a market
already in-play, an exception that is swallowed, or a ledger that can drift from Betfair's
record. Give a concrete triggering scenario for each. Do not fix anything.
```

## Phase 8: forward test

```text
Write the weekly forward-test report. For every bet the bot attempted in the past week, show
the price it observed, the minimum price, what was matched and at what price, and the closing
price it captured. Then, using the archive extract at <path> for those matches, report the
three gaps in Phase 8 of the plan: assumed price against matched price, the bot's closing
price against the archive's, and the closing line value of unmatched orders against matched
ones. Explain every bet where the harness and the live record disagree. Then give cumulative
results against the pre-set sample size and verdict rule in registry/hypotheses/. Do not
suggest changes to the strategy.
```
