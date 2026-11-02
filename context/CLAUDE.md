> **Translated from Ukrainian.** This is the project's root `CLAUDE.md`, the file Claude Code reads at the start of
> every session. The git e-mail is removed. Everything else is as it is in the repository on 2 November 2026.

# Eldvir — an OOC-style game (single player, in Ukrainian)

Symfony 8 + PHP 8.5, PostgreSQL 16, FrankenPHP in Docker. The narrator is the Claude API (`ClaudeNarrator`, key
`ANTHROPIC_API_KEY` in `.env.local`; without a key, and in tests, `EchoNarrator`). The narrator's instructions are
`data/narrator.md`.

## Commands
- Game server: `bin/serve` → http://localhost:8080 (dev, code mounted, reads `.env.local`; port 80 is taken by the
  system Apache). Stop: `docker rm -f ooc-web`.
- Tests: `APP_ENV=test bin/dphp bin/phpunit` (`bin/dphp` is PHP with pdo_pgsql in Docker). One file:
  `... bin/phpunit tests/Tool/ToolsTest.php --filter Name`.
- Migrations: `bin/dphp bin/console doctrine:migrations:migrate -n` (and with `APP_ENV=test`).
- Atlas map (SVG): `bin/dphp bin/console app:atlas:map [--zoom=principality] [--out=var/atlas-map.svg]`.
- Balance: `bin/dphp bin/console app:simulate --runs=300 --seed=x [--only=combat|magic|economy|events|weather]`.
- New game: `bin/dphp bin/console app:game:new` (skill set: wanderer | warrior | mage; `--gender=m|f`;
  `--stranger` — not a local; `--gold=N` — money).

## Where things are
- `src/Engine` — pure rules without Symfony (combat, magic, rolls, economy, inventory).
- `src/Game` — game services (Growth, Skills, Archetypes, Bestiary, Scenes, Injuries, Aftermath, WorldSeeder,
  LivingWorld — world clocks and rumours, GreatEvents — events of the region and the world, Threads — story threads,
  Lore — the past for the narrator and legends of places; Bonds — NPC relationships with the player and their log;
  Romance — temperament, attraction, stages, flirting, looks; GameClock — the clock; Calendar — date, week, holidays;
  Law — jurisdiction, degree, punishments; Weather — climate and weather outdoors, light and travel hours per day,
  roads by season and sky, blizzards, nights outside, the farming year and the market by season, weather spells and
  fair wind, `data/climate.yaml`; Omens — tomorrow's forecast for those who can read it; WordForms — word forms: cases
  of names and nicknames for finding them in text, the ukrainian-declension library).
- `src/Tool` — the narrator's tools (`*Tool`, 18 of them) and their helpers; input is checked by `SchemaValidator`
  (we don't use the API's strict mode — its limits).
- `src/Turn` — the turn as a transaction (`TurnOrchestrator`), the budget, permission to skip time from the player's
  words (`TimeRequest`).
- `data/` — YAML reference data: items, skills, archetypes (NPC roles), names (NPC names by people), bestiary, scenes,
  services, injuries, clocks (event templates for the director: local, regional, world), atlas (the continent:
  regions, towns, roads, coordinates), worlds/ (starts), calendar (months, weekdays), laws (shared laws, culture codes,
  regions; engine — Law and Tool/Justice, narrator — `world` actions), world secrets (spoilers).
- `design/` — rules (rules.md), world, names (glossary.md: proper names only, no common nouns), economy, architecture,
  the backlog **gaps.md** (statuses and decisions), **ready.md** — designed but not built (packages with effort).
  **next-run.md** — what to check in the next paid evaluation run.

## Working rules
- Documents and comments in Ukrainian, short. The engine counts, the AI only chooses and describes.
- **Do not reveal** the contents of the secret files (not in chat, not in code, not in command descriptions).
- Don't touch the global git config; this repo has its own author e-mail. Commits end with a `Co-Authored-By` line.
- Don't save usage limits at the cost of code: between "better code" and "fewer limits" — better code. Save only the
  tokens of a game turn (state, tools, narrator.md) — measure and report.
