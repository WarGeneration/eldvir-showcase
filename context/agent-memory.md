> **Translated from Ukrainian and English.** Claude Code keeps a persistent memory outside the repository: one file
> per fact, each with *why* and *how to apply*. These are the rules about how to work, in the agent's own words,
> lightly shortened. Session IDs, file paths and the e-mail are removed; quotes from the user are kept in Ukrainian
> with a translation.

## Be critical
The user asked me to push back on their product and design ideas so the product gets better.
**Why:** they want their design ideas stress-tested (15 Oct, game design discussion).
**How to apply:** say plainly when an idea is wrong or over-engineered, point out failure modes, and suggest better
alternatives, with a recommendation.

## Ask before every paid run
Before every run that spends money on the API (evaluation stands, mini-stands, any model calls), ask the user in a
separate question right before starting, with the price and the time, and wait for "yes". Even a rerun after a fix,
even for $0.01.
**Why:** on 21 Oct I reran the narrator stand myself after fixing a bug; the user: «завжди, коли запускаєш
тестування, що потребує грошей, спочатку питай мене» ("always ask me first when you run tests that cost money").
On 23 Oct again: after "start, do as you advised" (a plan with stands and prices) I started the mini-stands and the full
probe stand myself, ≈ $0.24. An approved plan that contains paid runs is not permission to run them.
**How to apply:** prepare everything free (code, tests, dry runs), then stop and ask "run <what>? ≈ $X, ≈ N min" —
each run separately, each iteration of a mini-stand too.

## Scripts run in the test environment
Temporary scripts must run with `APP_ENV=test`. Without it the kernel loads `.env.local`, the API key is present, and
services make real paid calls.
**Why:** it happened on 17 Oct in a world simulation: hundreds of calls, and a simulated week took ~5 s instead of ~30 ms.
**How to apply:** always prefix `APP_ENV=test`; a quick check is that a world week takes milliseconds, not seconds.

## Never reveal the secrets
The user is both the author and the player of this game, so hidden content must stay hidden: world history, answers
to mysteries, hidden NPC motives.
**Why:** knowing the secrets would spoil discovery during play.
**How to apply:** I may open, read and edit the secret files when a task needs it (allowed 16 Oct), but never quote,
list or summarise their content in chat, reasoning, commit messages or tool descriptions. Report only counts,
pass/fail, or "updated N places in the secret file". Test assertions on secret content are boolean, so a failing
test doesn't print it.

## The guild is one path of several
The game must work for a player who never enters the guild: talking to NPCs, helping them, following events. Engine
rules (sources, limits, rewards) must not assume guild jobs.
**Why:** on 17 Oct the user called out that I had designed the story-scale guard around guild jobs only, so NPC
requests got squeezed. Both real playtests at that point went through the guild, so the evidence itself was biased.
**How to apply:** walk every new rule through as a wanderer who never visits the guild before proposing it.

## Documentation in the same change
Adding, changing or removing a feature → update the design docs in the same commit (rules, architecture, economy,
world, the backlog status, CLAUDE.md if the structure changes). Removed or renamed things must not linger in the docs.
**Why:** the user asked on 17 Oct: «не забувай нові фічі додавати в документацію або ті, що прибираєш чи змінюєш»
("don't forget to add new features to the docs, or the ones you remove or change").
**How to apply:** before committing, list which docs describe the touched area; grep the docs for removed concepts.

## Report the cost of a turn
When a feature adds to what the narrator gets each turn (tool schemas, narrator.md, state lines, extra tool rounds),
say so in the report: tokens added and $/turn, plus any extra rounds.
**Why:** the cost per turn is a key constraint of the product.
**How to apply:** measure with the free `count_tokens` endpoint, don't guess. Cached prefix tokens are cheap; state
tokens are rewritten to the cache every turn and cost about 6× more; an extra tool round is the most expensive.

## Priorities: good code, then usage limits, then speed
**Why:** on 16 Oct the user hit usage limits; on 21 Oct they said: «ми не економимо ліміти, тож якщо питання між
"кращий код" та "більше лімітів", ти вибираєш кращий код» ("we don't save limits: between better code and more
limits, choose better code"). Clarified on 2 Nov: «між швидкістю, лімітами та гарним кодом я вибираю 1 гарний код,
2 ліміти, 3 швидкість» ("1 good code, 2 limits, 3 speed").
**How to apply:** never trade quality for limits, but don't spend limits just to go faster (parallel sub-agents,
reruns with no reason). Still suggest the right effort level and `/compact` when the context grows long.

## NPC memory is the differentiator
NPC memory is what will set this game apart. It may spend tokens, but the price of a turn must not grow 2–3×.
**Why:** competitors also claim memory; the difference is the boundary of knowledge (a person doesn't know what they
didn't see), witnesses, rumours, long memory with reasons — and the player noticing it.
**How to apply:** measured 23 Oct: +1,000 tokens of memory ≈ +4% of a Haiku turn, one more model call ≈ +18% and ≈ +2 s.
So memory gets tokens; new model calls are avoided. Check quality with recall probes, not only with fact hygiene.

## Projects are physical work only
Investigations and other social interactions (negotiation, questioning, persuasion) are never "projects". A project
is only work with hands and craft over shifts: building, repair, healing.
**Why:** on 22 Oct, after I had added an investigation template, the user said social interactions must not be projects.
**How to apply:** investigation and questioning go through checks, event NPCs and findings on the spot.
