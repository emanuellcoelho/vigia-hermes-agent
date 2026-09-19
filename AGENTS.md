# Vigia — AGENTS.md

A price-watching Plow agent, and the foundation for future agents. Read this
before changing anything; it says who owns what and where a change goes.

## The one test

**Who else would have to change if this fact changed?** That owner is where
the change goes. Everything here follows the same rule the Plow repos use:
one owner, one place.

| Path | Owns | Never here |
| --- | --- | --- |
| `plow-pbc/plow-hermes-agent` (base) | boot, `plow-init`, gateway config, base persona, plugin pin | anything Vigia-specific |
| `persona.md` | Vigia's voice: language mirroring, first-value rule, honesty about data, alert etiquette | per-turn plumbing, tool how-tos |
| `skills/vigia-watch/` | the engine: models, sources, alerts, sweep, store — and its SKILL.md | chat delivery (post_chat.py is its only exception, mechanically) |
| `skills/vigia-digest/` | the morning digest conversation and "compro ou espero?" | alert decisions (engine owns those) |
| `skills/vigia-onboarding/` | first contact, config questions, cron registration | the engine's defaults (config.py owns those) |
| `skills/vigia-watch/scripts/kit/` | generic infrastructure: clock, http, jsonio — domain-free | anything that knows what a price is |
| `image/` | TZ cont-init | gateway config, plow-init, agent-index reporter — the base's |
| `Dockerfile` / `compose.yml` | how this content ships | base-image behavior |

A Hermes bug goes upstream (`srosro/hermes-agent`), never patched here. A
base-image improvement is a PR there, then a digest bump in this Dockerfile.

## Conventions

- **Scripts are mechanical.** They answer one JSON object on stdout, exit 0/1/2,
  and never compose human words. The SKILL.md files are the voice. No prompt
  text in Python.
- **stdlib only.** No new dependencies in scripts or the image. The base adds
  what it adds; this repo adds none.
- **Alert logic is pure** (`vigia/engine/alerts.py`): no IO, no clock reads.
  Time comes from `kit.clock`, injectable in tests. Sources are adapters
  behind `vigia/sources/base.py`; the registry picks readers per store.
- **Writes are atomic** (`kit.jsonio.save_json_atomic`). The store file is a
  contract: `vigia/models.py` owns its shape, explicitly serialized.
- **State lives in `/var/lib/hermes/vigia/`** (override with `VIGIA_HOME` for
  tests). Nothing under this tree may carry a credential, a chat id, or a
  person's data.
- **TZ is fixed at boot** from `vigia/config.json` (`image/cont-init.d/
  10-vigia-timezone`). `hermes cron create` takes no per-job zone, so
  onboarding refuses to register schedules while the container's zone and the
  config disagree — a restart applies the zone first.

## Commits

Conventional, scoped by the table above, imperative, one concern per commit:
`feat(engine):`, `feat(sources):`, `feat(skills):`, `feat(report):`,
`fix(...)`, `docs:`, `chore:`. The body says **why**; the diff says what.
Never in a commit: `plow-credentials`, state files, anything under a
`VIGIA_HOME`. A pin bump (the base digest) is its own
commit naming what moved and why.

## Tests

    python3 -m pytest tests/ -q

Pure and fixture-fed: no network, no clock sleeps. A change to alert rules,
URL canonicalization, or store shape starts in `tests/`.

## Schedules (the registered spec)

| name | schedule (container TZ) | deliver |
| --- | --- | --- |
| `vigia-sweep` | `0 9,15,21 * * *` | post_chat.py per alert; quiet = NO_REPLY |
| `vigia-digest` | `30 8 * * *` (onboarding writes the user's time) | native `--deliver` to `plow_chat:${PLOW_HOME_CHANNEL}` |

Changing these rows is an edit to `vigia-onboarding/SKILL.md` and this table
together.

## Forking this into a new agent

The structure is deliberately the template:

1. Copy the tree; change `persona.md` and the skills' domain layer
   (`vigia/` → your domain; `kit/` stays).
2. Swap `sources/` for your domain's readers behind the same
   `Source`-shaped seam; keep `alerts.py` pure.
3. New `AGENT_ID`, new registration (`agent_index_client.py --register`),
   new compose `AGENT_ID`.
4. LICENSE stays MIT; NOTICE keeps the Apache attributions.

If a second agent confirms the pattern, `kit/` graduates into its own
package — not before.
