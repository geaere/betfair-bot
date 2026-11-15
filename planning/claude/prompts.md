# Prompts

One prompt per phase, ready to paste. Each assumes the agent is working in this repo and can read `planning/claude/plan.md`.

## How to run the sessions

- **One phase per session.** Start each in plan mode, read the plan the agent proposes, and only then let it build. A fresh session per phase keeps the context clean.
- **Writer and reviewer are different models.** If Claude writes a phase, give Codex the review prompt for it, and the reverse. The reviewer gets the diff and the plan, not the writer's explanation.
- **You run the gates.** At the end of each phase, check the gate in `plan.md` yourself before moving on. Do not accept "the gate passes" from the agent without seeing the evidence.
- **If an agent reports an edge, your next prompt is the leakage audit**, not the next phase.

## Proposed CLAUDE.md

Put this at the repo root in Phase 0. Codex reads `AGENTS.md`; use the same text there.

```markdown
# betfair-bot

Research and execution code for a horse racing strategy that bets at Betfair Starting Price.
The plan is in planning/claude/plan.md. Follow the phase you are asked to work on and no other.

## Rules that are never broken

1. Never read, download or query data dated after the validation end date in config/splits.toml.
   Only the human runs the holdout download.
2. Never edit the evaluation harness, the cost model, config/splits.toml or registry/trials.jsonl
   once the harness-v1 tag exists. If one looks wrong, stop and explain; do not change it.
3. Every backtest goes through the harness. Never compute strategy returns any other way.
4. Never run a hypothesis that has no file in registry/hypotheses/ written before the run.
5. Never use a field as a model input unless plan.md lists it as known before the off.
   BSP, in-play fields and the result are never inputs.
6. Never place, change or cancel a real bet unless the human has asked for it in this session.
   Default to dry-run.
7. Never write credentials, keys or certificates into the repo, a log or a prompt.
8. A return above 5% after commission is a suspected bug. Say so and look for leakage first.

## Working conventions

- Python 3.12, managed with uv. Run tests with `uv run pytest`.
- data/raw is immutable. Derived tables go in data/derived and must be rebuildable from raw.
- Report results with the number of bets and the interval, never a bare percentage.
- Say plainly when something failed, was skipped or was not verified.
```

## Phase 0: repo, rules and data splits

```text
Set up this repo for Phase 0 of planning/claude/plan.md.

Create a Python 3.12 project managed with uv, with ruff and pytest configured. Create the
directory layout the plan describes, a .gitignore that excludes data/, .env, and any key or
certificate files, and config/splits.toml with the split dates from the plan. Put the
"Proposed CLAUDE.md" text from planning/claude/prompts.md at the repo root as CLAUDE.md and
as AGENTS.md.

Done means: a fresh clone can be set up with one documented command, `uv run pytest` passes,
and the README explains the layout in under 30 lines. Do not write any data or strategy code.
```

## Phase 1: ingest the BSP archive

```text
Build Phase 1 of planning/claude/plan.md: ingest of the Betfair BSP price files.

Start by fetching the directory listing at promo.betfair.com/betfairsp/prices and three or
four sample files from different years, and tell me the file naming scheme, the columns, the
date format and anything that differs between years before you design anything.

Then build: a resumable downloader for UK and Irish win and place files that makes about one
request a second, never re-downloads a file, and refuses any date after the validation end
date in config/splits.toml unless passed an explicit --holdout flag; a parser that writes
typed Parquet tables to data/derived; and a data quality report in reports/.

The quality report must cover the checks listed under Phase 1 in the plan, including the
distribution of the sum of 1/BSP per race and every race with zero or multiple winners.
Report anomalies; do not filter them out silently.

Done means: train and validation data are on disk, the quality report exists, tests cover the
parser on the sample files, and the downloader has a test proving it refuses holdout dates.
```

## Phase 2: evaluation harness

```text
Build Phase 2 of planning/claude/plan.md: the evaluation harness. This code will be frozen
after review, so correctness matters more than speed or elegance.

Implement the cost model exactly as the plan states it, including commission charged on net
winnings per market. The harness takes a table of proposed bets and a split name, settles
them against the BSP data, and appends a record of the run to registry/trials.jsonl. It must
refuse splits that are not permitted and must not be bypassable by strategy code.

Write tests first: synthetic races with a known built-in edge, synthetic races with zero
edge, two bets in one market that net against each other for commission, a dead heat, and a
bet whose minimum price is above the BSP and so is not matched.

Done means: all tests pass, the output includes every figure listed under Phase 2 in the
plan, and you have written a short note of every assumption you made that the plan did not
specify, for me to confirm before the harness is tagged.
```

Review prompt for the other model:

```text
Review the evaluation harness in this repo against the cost model and Phase 2 in
planning/claude/plan.md. You are looking for ways it could overstate returns: wrong commission
netting, bets counted that would not be matched, results leaking into bet selection, the
bootstrap resampling bets instead of race days, or any path that skips the trial log.
For each finding give the file and line, a concrete input that triggers it, and the wrong
output. Do not fix anything.
```

## Phase 3: baseline

```text
Run Phase 3 of planning/claude/plan.md using the frozen harness.

Register it as hypothesis H-000 "back every runner at BSP" first. Produce the baseline report
on train and validation: returns before and after commission overall and by odds band, year,
field size and race type parsed from EVENT_NAME, plus the calibration table.

Tell me what you parsed from EVENT_NAME and how often parsing failed. If backing everything
shows a clearly positive return before commission, stop and tell me, because that points to a
bug in ingest or settlement.
```

## Phase 4: price-only hypotheses

```text
We are in Phase 4 of planning/claude/plan.md. Read reports/ for the baseline first.

Propose five hypotheses about mispricing at BSP that use only fields the plan lists as known
before the off. For each, write a file in registry/hypotheses/ using the template in the plan,
including the mechanism: who is on the other side of the bet and why they would be wrong.
Stop there and show me the five files. Do not run anything until I approve them.
```

After you approve:

```text
Run the approved hypotheses through the harness on train, then validation. Report each
against the default pass threshold in the plan, with bets, return after commission, interval
and results by year. Include the ones that failed. Do not tune any parameter beyond the
maximum variants stated in each hypothesis file.
```

## Phase 5: form data and the model

Before prompting, subscribe to the form data source and note what its terms allow.

```text
We are in Phase 5 of planning/claude/plan.md. The form data is at <path> and its terms allow
<what you may do with it>.

Step 1 only for now: match runners in the form data to the BSP archive by date, course, off
time and horse name. Report the match rate by year and list the commonest reasons for
failure. Do not build features yet.
```

```text
Step 2: build the feature table. Every feature must carry the time it became known, and a
test must prove no feature for a race uses anything from that race's day or later. List each
feature with a one-line reason it might predict winning.
```

```text
Step 3: write the hypothesis file for the model, then build it. First stage: win probability
from form features only, normalised within each race. Second stage: combine the first-stage
probability with MORNINGWAP as the market input. Fit on train only. Apply the bet rule from
the plan with the registered margin and run it through the harness on train and validation.
Report against the default pass threshold. Also report how much the second stage improves log
loss over the market price alone, since that is the real test of whether the model adds
information.
```

Leakage audit prompt for the other model:

```text
Audit this repo's feature pipeline and model for look-ahead. The rule is in
planning/claude/plan.md under "When each archive field is known". Check every feature's
availability time, every join, every normalisation and every train or validation boundary.
Then take 50 bets at random from the latest bet log and, for each, verify by hand from the
raw files that every input was known before that race. Report each problem with file, line
and an example race. Do not fix anything.
```

## Phase 6: holdout verdict

No agent prompt for the download; you run it. Then:

```text
The holdout data is now on disk and I am authorising one run. Run only the hypotheses listed
in registry/hypotheses/ as registered for the holdout, once each, through the harness on the
holdout split. Write the verdict report against the pass threshold. Do not change any code,
parameter or threshold, and do not run anything a second time.
```

## Phase 7: execution bot and safety layer

```text
Build Phase 7 of planning/claude/plan.md. Before writing code, read the current Betfair API
documentation and tell me, with links: the exact behaviour of LIMIT_ON_CLOSE back bets, the
minimum stake and liability for BSP bets in GBP, what the delayed application key can and
cannot do, and how commission is reported on cleared orders. Flag anything that differs from
the plan.

Then build the daily job, safety layer, reconciliation and alerts the plan lists, using
betfairlightweight. Credentials come from the environment and never appear in the repo or
logs. Dry-run is the default; placing real bets needs an explicit flag and still respects
every limit. Write tests for each limit that prove it blocks a bet.

Done means: dry-run works end to end on today's markets and the tests pass. Do not place a
real bet.
```

```text
Review the execution bot for ways it could lose money through a bug: a stake larger than the
cap, a bet on the wrong runner or market, a duplicate bet after a retry, betting on stale or
missing inputs, an exception that is swallowed, or a ledger that can drift from Betfair's
record. Give a concrete triggering scenario for each. Do not fix anything.
```

## Phase 8: forward test

```text
Write the weekly forward-test report. For every race the bot acted on in the past week,
compare what was placed and settled live with what the harness predicts for the same race
from the published BSP files. List every race where they differ and explain each difference.
Then give cumulative live results against the pre-set sample size and verdict rule in
registry/hypotheses/. Do not suggest changes to the strategy.
```
