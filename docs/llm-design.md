# How Eldvir uses an LLM

Eldvir is a text RPG: the player writes what their character does, the narrator answers in prose. The narrator is
an LLM (Claude Haiku 5.5). The hard part is not making it write well. It is keeping a cheap, fast model honest over
hundreds of turns: no invented gold, no names it shouldn't know, no fights won in prose without the dice.

The rule behind every decision below:

> **The engine counts, the AI only chooses and describes.**

## 1. The boundary between engine and model

| The engine decides (PHP, tested) | The model decides |
|---|---|
| Dice, checks, damage, death | Which tool to call and with what intent |
| Prices, wages, loot and its value | How the scene looks and sounds |
| What a defeated enemy carried | What an NPC says, within their card |
| Time: how long an action takes, whether the player allowed a skip | Pacing: when to hand the turn back |
| Law: who saw a crime, what the sentence is | Names and looks of new minor things, within a budget |
| Titles, land, followers — only through projects | |

Examples of the boundary in practice:

- **Loot.** Each enemy has a worth computed from level and rank. Weapons and armour are rolled when the fight starts
  and shown in the fight roster. The narrator takes loot only through `inventory give source=loot`; asking for an item
  the enemy didn't carry is refused.
- **Trophies.** The engine marks a trophy only on strong enemies from engine sources (a boss; an elite or a leader from
  level 4; an event or contract target) and gives a price budget from that enemy's worth. The model names it and
  describes it — "the pack leader's copper collar" — but can't make it worth more.
- **Time.** Whether the player allowed a long time skip is parsed by the server from the player's own words
  (durations, place names, a consent button). Only what the parser can't read goes to a small model with
  structured output. The narrator can't grant itself a week.

## 2. Tools

- **18 tools** (`skill_check`, `start_combat`, `npc`, `inventory`, `money`, `world`, `advance_time`, …). Each has a
  JSON schema and a short description written for a small model. The narrator's own instructions are about 46 KB;
  [four sections, translated](../context/narrator-excerpt.md).
- **Validated by the server, not by the API's strict mode.** Strict mode has limits we exceed (number of tools,
  optional and nullable parameters). The server validates input, repairs obvious slips, and returns an error the
  model can act on. The model sees a slimmed schema without the parts only the validator needs.
- **A refusal changes nothing.** Before each call the registry takes a snapshot of the turn's state; a refusal or a
  bad input restores it.
- **A refusal says what to do next.** "The defeated had no short sword — it isn't here at all: don't describe it lying
  there or broken. Here is what they carried: …". Small models act much better on a concrete next step than on a bare "no".
- **Results carry reminders where the model looks.** A trophy waiting to be taken is mentioned in the result of the
  loot call, not only in the system prompt (see [case study 1](case-studies.md#1-a-trophy-from-14-to-34)).
- **At most 24 tool calls per turn.**

## 3. The turn is a transaction

A turn is several model calls with tool calls between them. If the second call fails, the state has already changed
and there is no text. So all changes of a turn are buffered and written in **one database transaction** at the end.
On failure nothing is written, and the retry uses the same dice, so a retry is not a reroll. One turn at a time per
game: an advisory lock rejects a second tab or a double click.

The money spent on a failed turn is recorded outside the rollback, so the **daily budget cap** stays accurate.

## 4. What the model is not allowed to know

- **The name filter.** An NPC whose name the player hasn't heard appears to the model as "the miller's wife" or
  "a stranger", and their ID as a neutral `n<id>`. All text that goes to the model passes through the filter: the
  state and every tool result. IDs are mapped back in tool inputs. The model can't leak a name it never saw.
  The name becomes visible only after `npc introduce`, when it is spoken in the story. Ukrainian inflects names
  (Morna → Mornu, Kremin → Kremenia), so the filter matches every case form through a declension library plus our own
  stem rules.
- **Secrets stay out of the request.** Hidden NPC motives, world history, answers to mysteries: a test checks that
  hidden fields never reach the prompt.
- **What is near, not everything.** The model sees the people here and one step away, not the whole map. Game 13 showed
  why "one step away" matters: the narrator didn't see an NPC in the next room and invented a new character in her
  place ([case study 2](case-studies.md#2-from-a-playtest-bug-to-a-subsystem)).

## 5. Three model roles and the cost of a turn

| Role | Model | Job |
|---|---|---|
| Narrator | Claude Haiku 5.5 (effort high) | Prose and tool calls |
| Scribe | Claude Haiku 5.5, structured output | After the turn: facts of the world, places, which names the player heard, rule breaches the engine should look at |
| Chronicler | Claude Haiku 5.5, structured output | NPC memory: what each person said, saw or heard this turn (with a quote as proof), witnesses of engine events, old memories folded into short episodes |

The engine checks what the scribe and the chronicler return, too. The chronicler can record that a guard saw the
theft; it can't decide the sentence. It runs only when people with a card are near or named in the text, so most
quiet turns don't pay for it.

**Context and caching.** The request is ordered for the prompt cache: tools → narrator instructions and game settings
(cached for an hour) → past turns (the window moves in steps of ten, so its start doesn't change every turn) → this
turn's state. State tokens are rewritten to the cache each turn, so one token of state costs about **6×** a token of
instructions. That is why new state lines appear only when they matter: the trophy line costs +100 tokens, but only
while a trophy lies here or one step away.

**Every change is measured.** Any feature that grows the prompt is reported with its token count and price per turn,
measured with the free `count_tokens` endpoint, not estimated. A typical line from a commit: *narrator.md +48 tokens,
schema +8, both cached, ≈ +$0.00003/turn.*

A whole turn on Haiku 5.5, with the scribe and the chronicler, costs about **$0.005** (games 11–13); see
[§7](#7-what-it-cost) for how it got there.

One extra thousand tokens of NPC memory adds about 4% to a Haiku turn; one more model call adds about 18% and two
seconds. So memory gets tokens, and new model calls are avoided.

## 6. Evaluation

A unit test can check that a tool refuses a sword that isn't there. It can't check that the model then doesn't write
the sword into the story. For that there are paid evaluation runs ("stands"):

### Choosing the models

The narrator started on **Opus 5.5** (game 4, $0.069 per turn), then moved to **Sonnet 5.5** with **Haiku 4.5** as the
scribe (games 5–8, $0.045–0.070 per turn). That was good enough to build on, but too expensive for a game people would
pay for by the month. On 20–21 October the agent priced about a dozen cheaper models on the real token profile of our turns.
The main cost was cache reads, so a model with cheap tokens but expensive cache didn't help. The plan was to test, in
order, Haiku 5.5 and two models with an Anthropic-compatible API (DeepSeek V4.1 Flash, then MiniMax M3.1 Flash), and to
write an adapter for GPT or Gemini only if all three failed. Haiku 5.5 passed, so the list stopped at DeepSeek.

**The narrator (21–22 October).** 30 recorded turns of different kinds from my games.
For each one the game state is restored from a snapshot, and the same player message is replayed on Sonnet 5.5,
Haiku 5.5 and DeepSeek V4.1 Flash with the real tools. Measured: price, tool rounds, refusals, rule breaches (judged by
a separate model). The prose I rated **blind** in a small web page: which of the three is best and which worst, letters
instead of model names.

| | Sonnet 5.5 | Haiku 5.5 | DeepSeek V4.1 Flash |
|---|---|---|---|
| Narrator $ / turn | 0.0368 | 0.0026 | 0.0026 |
| Tool rounds / turn | 2.43 | 2.30 | 3.27 |
| Refusals (30 turns) | 9 | 8 | 19 |
| Rule breaches | 7 | 6 | 6 |
| My blind votes: best / worst | 12 / 9 | 10 / 9 | 8 / 11 |

Haiku 5.5 became the narrator: 14× cheaper than Sonnet, with similar breaches and prose I rated close. On 30 turns the
difference in votes is noise, and we said so. DeepSeek cost the same as Haiku with more than twice the refusals, and
once it closed a story event with an explanation it invented itself. Sonnet stays as a possible premium option.
The first narrator run was invalid (the stand didn't load turn history). We found it, fixed it and paid for a second
run rather than draw conclusions from it.

**The scribe.** A separate run on 141 recorded turns against the scribe we had (Haiku 4.5):

| | Haiku 4.5 | Haiku 5.5 | DeepSeek V4.1 Flash |
|---|---|---|---|
| $ / turn | 0.0068 | 0.0006 | 0.0004 |
| Time | ~4 s | ~2 s | |
| Notes | invented facts (a trading post that doesn't exist, a man "killed… breathing") | fewest errors in names | lost the main point of what people said |

All three mixed up unnamed people. The cause was the scribe's input, not the models: it saw people only as an ID and a
name, without their looks or nicknames. That was fixed in the engine. Haiku 5.5 became the scribe: 11× cheaper and
twice as fast as before.

**The chronicler** came later (23 October), already on Haiku 5.5. Here the question was not which model but whether
memory works at all. So it got its own test set: "gold" turns that list who must remember what and who must *not* know
it. First run 9 of 14; after fixes 13 of 14. Then probes on a replayed game, where an NPC is asked about the past:
the first version failed the threshold (18 of 24, worse than the old memory), and after a review of the failures
41 of 48. One finding from play changed the call itself: in game 12, 47 of 50 summarising calls spent their whole token
limit on reasoning and returned no text, and one cut-off summary replaced 16 memories of an NPC. Now the summary runs
without reasoning, and a cut-off summary is thrown away while the memories stay.

**Mini-stands.** Each change to narrator behaviour gets a few probes in YAML: a saved game, a setup (for example "a
defeated pack leader lies here with a trophy"), a player message, the expected tool calls, and forbidden words. Each
probe runs twice. The automatic verdict is checked by reading the text, because it misses things (see the
[retrospective](retrospective.md)). Typical cost: $0.01–0.16 per run.

## 7. What it cost

The whole project spent **$35.27** on the Claude API (Anthropic console), for development and evaluation runs together.
The game logs the cost of every turn, so the split is visible:

| Period | Narrator | Scribe / chronicler | Turns | Logged cost | Per turn |
|---|---|---|---|---|---|
| 16 Oct (game 4) | Opus 5.5 | — | 71 | $4.92 | $0.069 |
| 17–20 Oct (games 5–8) | Sonnet 5.5 | Haiku 4.5 | 392 | $23.07 | $0.059 |
| 22–28 Oct (games 11–13) | Haiku 5.5 | Haiku 5.5 + chronicler | 322 | $1.66 | $0.005 |

Most of the money went on the Sonnet and Haiku 4.5 period. After the switch to Haiku 5.5 the cost dropped sharply:
322 turns for less than $2, about **12× cheaper per turn**, although each turn now includes a third model call
for memory. The remaining $6–9 went on evaluation runs (the model comparison alone was about $6.5).
Until 21 October the game's counter priced Sonnet's cache reads too high (by about 15% on a turn), so the
Sonnet row is slightly overstated. The total comes from the console.

## 8. What we learned about small models

- **Prompt rules wear out; engine rules don't.** The pattern repeated many times: the model invents something →
  a rule in the prompt → it works most of the time, not always. When it matters, the fix moves into the engine (a tool that refuses,
  a value only the engine gives) or into the data.
- **Bad data beats good instructions.** A place called "Old Mill" made models describe the inside of a mill building,
  whatever the prompt said; renaming it to "Old Mill Yard" fixed it. Another example: the sword the enemy never had. The
  scribe had written "a short sword under the table" into the place's memory, and the narrator kept bringing it back
  without asking the engine. The next fix is in the scribe's rules, not the narrator's.
- **Put the reminder where the model is looking.** A note in the tool result works better than the same sentence in
  the system prompt.
- **A refusal works only if the model asks.** If the model never calls the tool, the engine has nothing to refuse. Then
  the problem is upstream: what the model saw in its context.
- **The model's past words are not a record.** In game 13 the narrator insisted three times, out of character, that
  only the leader's ears counted, then called the second pair of ears in the inventory "my mistake" and removed it.
  Now only the engine's records count as what happened, and nothing is removed "by mistake" without the player's OK.
