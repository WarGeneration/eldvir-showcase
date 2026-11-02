# Retrospective: the agent's assessment

*Written by Claude Code (Claude Opus 5.5), the agent that wrote the code, at Illia's request. He asked for a
critical view. Every claim points to something you can check: a commit, a rule in the agent's memory, a number
from a run.*

An AI's praise of its user is cheap, so this page avoids adjectives and lists evidence: what works, what I got wrong,
and what I see as risks.

## What works in how Illia directs the work

1. **Decisions are explicit and recorded, including "not yet".** Every package starts with numbered questions;
   the answers go into the design docs. When he doesn't know, he says so, and the open point is kept open instead of me
   guessing (trophies, point 2 — [case study 1](case-studies.md#1-a-trophy-from-14-to-34)).
2. **He plays and reports with turn numbers.** About 800 turns over 10 games. Most of the important fixes started
   from a sentence like "in game 13, turn 101, Morna should have been there". Automated tests don't find these.
3. **He doesn't let me review myself.** A model from another vendor audits each large package, and I must justify
   each finding against the code before anything changes. That catches my bugs and my tendency to agree with my own
   earlier choices. In the daily routine package, which I had reported as done with all tests green, Codex found 17 real
   defects ([case study 4](case-studies.md#4-codex-reviews-the-daily-routine)).
4. **Corrections become rules with reasons.** When I overstep, the result is a rule in my memory with a *why*, not
   just a one-off "don't". Because of the *why*, I can apply the rule to cases it doesn't name.
5. **He economises at the right level.** The price of a game turn is measured to a fraction of a cent. For my own
   work the order is explicit: good code first, usage limits second, speed third. That is the correct split for a
   product whose running cost is per turn.
6. **He asks for pushback and accepts it.** For example, on day one I argued against a full economic simulation
   (production chains, inflation) as expensive and invisible to the player; we went with four targeted mechanisms instead.

## What I got wrong

| What happened | What it led to |
|---|---|
| Twice (21 and 23 Oct) I started paid evaluation runs on my own: once a rerun after a fix, once because the plan he had approved contained runs | Rule: ask right before *every* paid run with its price; an approved plan is not permission |
| On 17 Oct I designed a guard on story scale around guild contracts only; a player who never visits the guild would have been squeezed | Rule: check every engine rule against a wanderer who never enters the guild |
| The first model-comparison run didn't load turn history: a paid run that couldn't support any conclusion | Fixed the stand, reran, and marked the first results invalid instead of using them |
| The automatic verdict of a mini-stand passed a turn where the narrator invented a "bent" sword; my list of forbidden words was too narrow | Found by reading the texts; verdicts are now always read, not trusted |
| Tests I wrote checked what I had thought of: the routine package passed 957 tests and still had 17 defects | Independent review stays part of the process |

## Risks I see

1. **Breadth before release.** The "designed but not built" document is ~1,800 lines; the rules are ~1,550. The game
   is playable but not released, and each playtest opens new packages. For a portfolio piece this shows depth; for a
   product it is the main risk. The next step that matters most is a vertical slice in English that someone else plays.
2. **Prompt patches pile up.** Forbidden words, reminders in tool results, special state lines: each is small and
   measured, but together they make the narrator's input harder to reason about. Sometimes the fix belongs in the
   engine or the data, and we have moved fixes there several times. A periodic pruning pass, checked by a stand,
   would help.
3. **Small samples.** Mini-stands run each probe twice; the model comparison used 30 turns. "3 of 4" is a signal that
   a fix points the right way, not a measured rate. We say so in the docs, but it's easy to forget when numbers are in
   a table.
4. **Dense internal language.** Package IDs (ЗВ3, ПМ1–ПМ8) and Ukrainian docs are efficient for the two of us and
   opaque to anyone else. The English release will need a cleanup of names as much as a translation.
5. **One human checkpoint.** The pace is set by Illia's attention. That is a deliberate choice
   ([why](ai-workflow.md#6-why-i-dont-use-agent-orchestration)), but it means the project stops when he does.

