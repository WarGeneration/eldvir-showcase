# Case studies

Four examples, each traced to commits in the private repository. They show the same loop at different scales:
something goes wrong in play or in review → the agent finds the cause and proposes options → I decide → code, tests,
docs → a paid run checks the model's behaviour → the numbers go back into the docs.

## 1. A trophy: from 1/4 to 3/4

**Goal.** A strong enemy should sometimes leave one special item: "the pack leader's copper collar", not just
"wolf hide ×3". The model is good at naming things and bad at keeping prices honest.

**The agent's proposal, three questions.** (1) Which enemies give a trophy? (2) Should invented people in free-form
fights be capped too? (3) Where does the trophy's value come from? My answer: «1 так 2 не знаю, треба подумати
3 згоден» — yes to 1 and 3, 2 left open. The agent designed it so that 2 doesn't block the rest: a trophy is given only
in fights with an engine source, so invented enemies can't produce one either way.

**What was built** (`33a724a`): the engine marks the trophy (a boss; an elite, leader or contract target from level 4)
and sets a budget of half that enemy's worth. The model picks a base from a short list (the enemy's own gear, hide,
jewellery within the budget), a name and a look. The trophy's price is checked everywhere items change hands:
sale, gift, theft.

**First mini-stand** (`8877663`, $0.067). Ordinary loot worked 6 of 6. The trophy was given 1 time of 4. The cause was
in the logs: the trophy was one clause inside a long "loot here" line, and the line always ended with "coins — money
loot" — so the model asked for coins from wolves all four times.

**Three fixes** (`5c70804`): the trophy got its own state line; the results of `inventory give` and `money loot` remind
about a trophy still lying there; "coins" is mentioned only when there are coins. Cost: +100 state tokens and +92 in
the reminder — only on the few turns after a strong enemy, nothing on an ordinary turn.

**Repeat run** ($0.025): trophy 3 of 4 — "the pack leader's copper collar", "the leader's fang" (taken right after the
engine refused coins, from the reminder in that refusal), "the chieftain's silver seal". The one failure ignored
reminders in two results. Still open: a sword the enemy never had was "found" once out of two, with no tool call at
all. The cause is in the scribe's memory of the place, so the next fix is there, not in the narrator.

## 2. From a playtest bug to a subsystem

**What I saw** (game 13, turns 101–105): I was looking for an NPC called Morna. She was in the hall of the mill, one
room away. The narrator didn't see her and invented a new character, a thief named Kalem, to fill the scene.

**The cause** (`3082718`): the state showed only people in exactly the same place. A room next door was invisible.
The agent recorded it as 🔥 in the backlog with the turn numbers and two other cases from the same game.

**Two packages followed** (1 November):
- **People one step away** (`909df0e`, ZK1–ZK7): a short "one step away" line in the state, who is here and who is
  near on arrival, the scribe told that "behind the wall" is not "absent". Tokens: +99 cached, +24 per person only
  when someone is there.
- **People return to their place** (`74335a7`, PM1–PM8): each person has a post separate from where they are now; when
  released out of sight they walk back in travel time; guests stay their term.

**Checked by a mini-stand** (`f73ff68`, $0.16): the scribe got 6 of 8 right with no false records; the misses led to two small
rule changes. The same run found that the place name "Old Mill" made models describe the inside of a mill building;
it was renamed "Old Mill Yard" (`34f7044`).

## 3. Fourteen issues from one game

After game 7 (41 turns) the agent read the turn log, every tool call and the state after each turn, and listed
14 issues with turn numbers, G7-1 to G7-14. Examples:

- G7-1: money written as "9 з 97 с" ("9 of 97 silver") was read by the model as "97 silver" — the format confused a small model.
- G7-2: two NPC names appeared in the story before the player could know them. This became the **name filter**
  ([llm-design.md §4](llm-design.md#4-what-the-model-is-not-allowed-to-know)).
- G7-6: arrows weren't spent in combat.
- G7-11: marked *not a bug* after checking: the validator already repairs that exact slip.

My decisions, the same day: keep game 7 as it is (no retroactive fixes), accept most, reject one change, choose
between options on the rest. The fixes went out as six packages, A–F, all committed within the day.

## 4. Codex reviews the daily routine

**The package:** a "daily routine" tool that lives several days of work, rest and study in one call (ten commits).

**Codex's review** of commit `2dec3cc`: 19 findings, ranked. Two were P1: an arrested player could still work through
the routine (it skipped the law check that the single-action tools have), and a workshop lost its type and plot of land
after it was inherited. Every finding came with a test that failed on the snapshot: 25 tests, 22 failing.

**Claude's check against the code** (`9384001`): 17 real defects; one accepted as a rule change (words like
"I study" must no longer count as permission for a week); one rejected — the strict time check would break the
"skip" button, and Claude explained why.

**Four fix packages** (R1–R4, `3029966` … `ee69c7e`), then a mini-stand on Haiku ($0.099, 9 probes × 2): the time
permission held 6 of 6. It also showed two new problems the review couldn't find, because they are model behaviour:
Haiku rolled a hiring check on its own instead of using the routine, and an NPC "agreed" to teach in words without the
tool. Both moved into the engine (`6cf82e3`): the engine now decides whether someone hires or teaches you.

**Why it matters:** the review found engine bugs; only the paid run found model bugs. Each needs the other.
