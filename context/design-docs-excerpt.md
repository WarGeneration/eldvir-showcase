> **Translated excerpts.** One feature — loot from enemies and trophies — as it moves through the three working
> documents: the backlog (`gaps.md`), the plan (`ready.md`) and the evaluation-run list (`next-run.md`). The agent
> writes all three and updates them in the same commit as the code. Shortened where marked (…). The originals are
> dense on purpose: they are read by the agent at the start of a session, so every line carries a fact, a number or a
> decision. Code identifiers and file names are left as they are.

## 1. `gaps.md` — the backlog: what went wrong, what was decided, status

From the section "Game 13 (28 Oct)":

> - **Turns 31, 33:** the engine didn't give the bandit's short sword (40 silver) — the loot was 50 copper; the bandit's
>   bag in the corridor — "no loot here" (the fight was in the hall) → ready.md "🔥 Enemy loot: people's gear, a strong
>   enemy's trophy" (ЗВ1–ЗВ4, reviewed 02.11; ЗВ1–ЗВ3 ✅ 02.11, ЗВ4 — later).
>   - **Mini-stand ЗВ1–ЗВ3, 02.11** (`tests/Stand/loot-probes.yaml`, × 2, $0.067). **Gear works:** turn 28 — "the axe
>     rolled under the table" 2/2; 31 — the axe and coins from the loot 2/2; 33 — the engine refused lock picks and a
>     whetstone invented from the bag, text says "the bag is empty" 2/2. **Refusal covered by fiction 1/2:** the sword
>     was refused, and the text said "it lies under the table, the blade cracked — you leave it". **Trophy 1/4:** … the
>     line inside "loot here" goes unnoticed.
>   - **Fix and repeat, 02.11** (3 probes × 2, $0.025): the trophy gets its own state line (+100 tokens, only with a
>     trophy nearby), a `trophy_waiting` reminder in the result of `inventory give` / `money loot` (+92, only while the
>     trophy lies there), the refusal says "this sword isn't here at all — don't describe it"; "coins — money loot" only
>     when there are coins. **Trophy 3/4** (was 1/4) … **Sword 1/2** (as before): once "you take the short sword from
>     under the table, bent" with no tool call at all — a refusal doesn't help when the narrator doesn't ask the engine;
>     the source is "a short sword under the table" in the place's facts and the history. 📌 Scribe: don't write the
>     weapons and armour of the defeated into place facts (the engine knows them) — separately, with a scribe mini-stand.

## 2. `ready.md` — designed, then built: the package

> ### 🔥 Enemy loot: people's gear, a strong enemy's trophy (game 13, 28.10; reviewed 02.11) ✅ ЗВ1–ЗВ3 (ЗВ4 — later)
>
> - **Game 13, turns 28–36.** After the fight the engine said about the bandit "hides, carcasses, goods for 1 s 50 c;
>   coins 1 s 50 c": an animal's label on a person, and a sum without a single item. What was in it, the narrator
>   decided: a short sword for 40 s (13 times the whole loot), and it covered the refusal with "a cracked hilt".
> - **Cause** — not missing weapons in the bestiary but an abstract sum: the narrator fills it in itself. **Rule:** the
>   engine names the enemy's gear, before the fight and after; the narrator describes what is there.
>
> **ЗВ3 — a strong enemy's trophy ✅ (02.11).**
> - **Who:** only in a fight with an engine source — an event, a contract, an encounter on the road: every boss; the
>   strongest from level 4 (`StartCombatTool::TROPHY_LEVEL`) — elite, leader, the strongest in a pack of two or more,
>   or the target of a contract. A "mercenary boss" invented in a free fight gives no trophy.
> - **Budget — out of the sum:** half the enemy's worth; an animal has fewer parts by the same amount, a person pays it
>   from trifles, then coins.
> - **State:** a separate "trophy" line (+100 tokens, only when a trophy is here or one step away); until taken —
>   `trophy_waiting` in the result of `inventory give` / `money loot` (+92). Mini-stand 02.11: inside "loot here" it
>   went unnoticed 1/4 → separate and with a reminder 3/4.
> - **The item:** `inventory give source=loot` with `trophy` (the look), `name`, `template` — the base: this enemy's own
>   item, hide or parts for an animal, jewellery within the budget; anything else is refused with the list; one per
>   enemy. Its own price (`Item::price`: sale, gift, theft, betrayal use it) = budget + the enemy's item when that is
>   the base.
> - **Not done:** fame — only the existing rumour about a foiled event; reinforcements mid-fight give no trophy.
> - **Open (the user's decision, later):** cap people in free fights the way animals are capped. The trophy no longer
>   depends on it.
> - Tokens: `inventory` description +69 and narrator.md +75 — cached (≈ $0.000003 per turn); fight roster +4 per
>   trophy enemy.

## 3. `next-run.md` — what the next paid run must check

> ## Enemy loot ЗВ1–ЗВ3 — mini-stand and repeat 02.11 ($0.067 + $0.025): gear ✅, trophy 3/4 ✅; sword from place memory 1/2 — after the scribe rule
>
> Probes — `tests/Stand/loot-probes.yaml` (game 13, turns 26–33 — the bandit from the corridor; 31 and 77 — the
> chieftain's and the pack leader's trophy). Free first with `--dry` (it prints the state the model will see). 7 probes
> × 2 ≈ $0.17, ≈ 10 min:
> ```
> bin/dphp bin/console app:stand:narrator claude-haiku-5-5 --probes=tests/Stand/loot-probes.yaml --repeat=2 --tag=loot
> ```
> Judge by: `gear_named` — items from the fight roster in the text; `no_ok` — the sword not given from loot, nothing
> "found" in the bag; trophy (31, 77) — `inventory give` with `trophy` passed; by eye — no "cracked hilt". The place's
> memory (31) holds "a short sword under the table" from the scribe: if the run passes, the engine is stronger than
> place memory; if not, the scribe or the state needs a rule "loot only from the engine".
