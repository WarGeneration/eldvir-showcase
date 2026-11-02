# How I work with AI agents

This is how I built Eldvir with Claude Code (the builder) and OpenAI Codex (the reviewer): who decides what, where
the agent's knowledge lives, how its work is checked, and why I keep myself in the loop instead of running many
agents in parallel. Commit hashes refer to the private repository and can be shown on a call.

I described the same basics for an earlier project, [Cipher Duel](https://github.com/WarGeneration/cipher-duel-showcase).
Here I focus on what is different: a much larger design, an LLM inside the product, and a second model as reviewer.

## 1. Roles

| Me | Claude Code | Codex |
|---|---|---|
| Game design: rules, world, economy, what the narrator may and may not do | Proposes designs with options, edge cases and costs | Audits large packages of commits on a frozen snapshot |
| Picks an option on every open question, or leaves it open on purpose | Writes the code, tests and design docs | Writes tests that fail on the bugs it finds |
| Plays the game and reports what went wrong, turn by turn | Analyses game logs, finds root causes | Reviews design documents before they are built |
| Approves every paid model run and every commit | Measures token cost of every prompt change | |
| Brings Codex reports to Claude, makes the final call on each finding | Checks each Codex finding against the code, says which are real | |

## 2. Design before code

The game has about 6,000 lines of design documents (rules, world, economy, architecture). Code follows them,
not the other way round. Large features go in **packages** with short IDs (for example ЗВ1–ЗВ3: loot from people,
loot one step away, trophies). For each package the agent writes a plan with numbered questions, and I answer them
one by one. A real answer from 2 November:

> «1 так 2 не знаю, треба подумати 3 згоден» — "1 yes, 2 I don't know, need to think, 3 agreed"

Point 2 stayed open and was recorded as open: the agent built 1 and 3 so that 2 could still go either way.

## 3. Context engineering

An agent starts every session with no memory, so its knowledge lives in files:

- **[`CLAUDE.md`](../context/CLAUDE.md)** — commands, where things are, working rules. Short on purpose.
- **A pipeline of design documents** that the agent updates in the same commit as the code:
  - `gaps.md` — the backlog: problems found, decisions taken, status;
  - `ready.md` — designed but not yet built, with an effort estimate per package;
  - `next-run.md` — what the next paid evaluation run must check.

  [One feature through all three documents](../context/design-docs-excerpt.md), translated.
- **Agent memory with reasons.** Each rule I gave the agent is stored with *why* and *how to apply*, so it can judge
  edge cases instead of following a phrase. Examples ([all of them, translated](../context/agent-memory.md)):
  - **Be critical.** Say plainly when my idea is wrong or over-engineered, and suggest a better one.
  - **Ask before every paid run, with the price.** An approved plan that contains runs is not permission to run them.
  - **The guild is one path of several.** Check every rule against a player who never visits the guild.
  - **Spend tokens on NPC memory, not on anything else.** Memory is the product's differentiator; the price of a turn
    must not grow 2–3×.
  - **Priorities: good code, then usage limits, then speed.** In the game itself, every turn's tokens are measured
    and saved.

Several of these rules exist because the agent got something wrong first (see [retrospective](retrospective.md)).

## 4. Guardrails

| Rule | Why |
|---|---|
| The agent may read and edit the files with world secrets, but never quotes or summarises them in chat, commits or commands | I am both the author and the player; spoilers would ruin my own playtests |
| Tests on secret content assert true/false, never print it | A failing test must not show the secret |
| Every paid API run (evaluation runs, even $0.01) needs a separate "yes" right before it, with the price | Costs stay predictable |
| Scripts run in the test environment, where the narrator is a free echo stub | A dev script must not make paid calls by accident |
| No commit without my OK, no push without my request, my personal git config untouched | Outward-facing actions stay mine |
| Secret files are not copied into the snapshot Codex reviews | The reviewer doesn't need them |

## 5. Verification

- **1,176 automated tests**: engine rules, every narrator tool, the turn transaction, the name filter.
- **Balance simulation**: a command plays hundreds of fights and months of economy and prints the distributions.
- **Evaluation runs of the narrator**: real recorded turns replayed on the model with real tools (see
  [llm-design.md](llm-design.md#6-evaluation)). Prompt changes are not "done" until a run shows they work.
- **My own play.** I play long games (10 games, ~800 turns; game 13 alone is 106 turns) and report problems with turn numbers.
  The agent then reads the turn logs and tool calls and finds the cause. Game 7 alone gave 14 issues; I confirmed
  or rejected each one, and they became six packages.

## 6. Why I don't use agent orchestration

Claude Code can split work across several sub-agents running in parallel. I chose not to, for three reasons:

1. **I want to digest every change myself.** I decide implementation details, not only the product. If several agents
   produce code in parallel, I either review less or become the bottleneck anyway, so the speed is lost.
2. **There is little to run in parallel.** A typical package touches the same core: a tool, the state the narrator
   sees, the narrator prompt, the tests and the design docs. Parallel agents would edit the same files and each would
   start cold, without the reasons behind earlier decisions.
3. **It trades usage for speed, and speed is my last priority.** My order is: good code first, usage limits second,
   speed third. Parallel agents each reload the project context, so they spend more of the limits to finish sooner,
   and sooner buys nothing here, because my own review sets the pace.

The agent agreed with this at the start of the project (15 October). Its advice then was to use separate agents only
for narrow, independent work, and above all for **independent review**. That is what I do — with a different model.

## 7. A second model as reviewer

After each large package (5–10 commits) I give the commit range to **Codex**. About ten rounds between 20 and 28 October
covered the law system, events and projects, building and families, daily routine, the item system, and a weather
design document before any code was written.

How a round works:

1. **Codex works on a frozen snapshot** (`review-source-<sha>`) and a separate test database, so its findings stay
   valid while the main code moves on.
2. **It proves findings with tests.** Daily routine round, commit `2dec3cc`: the regular suite passed (957 tests);
   Codex added 25 audit tests, 22 of which failed by design, and reported 19 findings ranked P1–P3 with reproductions.
3. **Claude checks every finding against the code** and proposes what to accept. Same round: 17 confirmed as defects,
   one accepted as a rule change, one rejected (the strict check would have broken the time-skip button).
   Other rounds: 10 of 13 accepted for property; the item audit led to a five-package rewrite of the item system.
4. **I decide**, then the fixes go into the normal pipeline: plan in `ready.md`, packages, tests, commits that cite
   the finding IDs (for example `3acae7c` "access and rollback (item audit 27.10: F01–F03, F05, F10, F14, F20)").

Why another vendor's model and not a Claude sub-agent: it doesn't share the builder's habits or assumptions, and the
builder can't quietly agree with itself. My prompt is almost always the same: *read the report, take what we need,
explain why, be critical.*

## 8. Time and numbers

| | |
|---|---|
| Calendar | 15 Oct – 2 Nov 2026, prompts on 16 days |
| My prompts | 834 |
| Session time | ~116 hours (gaps under 15 min joined), ~127 hours (under 30 min) |
| Commits | 383 |
| Code | ~55,000 lines of PHP, ~27,500 lines of tests, 1,176 tests |
| Games played by me | 10 games, ~800 turns |
| Claude API, total | $35.27 (games and evaluation runs; [split](llm-design.md#7-what-it-cost)) |

Session time is measured from the timestamps in the Claude Code logs. It includes time when the agent worked alone
(long test runs, evaluation runs) while I read or played. It doesn't include my playtesting outside the sessions.

