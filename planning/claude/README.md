# Plans

Two alternative plans for the Betfair bot experiment, based on the research in `research/claude/`. Each folder stands alone and holds the same three files: `README.md` (overview and what is needed from you), `plan.md` (phases and gates) and `prompts.md` (agent guardrails and one prompt per phase).

| Folder | Strategy | Rank |
|---|---|---|
| `racing-bsp/` | UK and Irish horse racing, backing runners at Betfair Starting Price when a model says the price is too big | First choice |
| `football-prematch/` | Football pre-match model, judged on whether the price taken beats the closing price | Second choice |

## How they differ

| | Racing at BSP | Football pre-match |
|---|---|---|
| Free exchange price history | Every race since 2007 | Two finished seasons (2024/25 and 2025/26) |
| Will the bet be matched? | Yes, at the starting price or not at all | Only if the price is still on offer; the bot takes it and pays the spread |
| Main measure | Profit after commission on a long history | Whether the price taken beats the closing price |
| Paid data | Form data, about £25 to £75 | None |
| Betfair login needed from | Phase 7 | Phase 1, for a price recorder |
| Bets needed for a verdict | Thousands, available in the archive | About 200 to 800 live or archived bets |

Pick one to start. The repo setup, evaluation harness conventions and safety layer are common to both, so the second plan is shorter to carry out if the first has been built.
