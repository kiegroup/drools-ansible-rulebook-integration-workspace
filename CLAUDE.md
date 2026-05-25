# drools-ansible-rulebook-integration-main Workspace

**Project repo:** /home/tkobayas/usr/work/eda-HA-PoC/drools-ansible-rulebook-integration-main
**Workspace type:** public

## Session Start

Run `add-dir /home/tkobayas/usr/work/eda-HA-PoC/drools-ansible-rulebook-integration-main` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

| Artifact   | Destination |
|------------|-------------|
| adr        | workspace   |
| blog       | workspace   |
| design     | workspace   |
| snapshots  | workspace   |

Valid destinations: `project` · `workspace` · `alternative ~/path/to/repo/`

## Context Management

If the conversation is getting very long or you notice context pressure,
proactively suggest writing a handover before continuing.

---

## Project Type

**type:** java
**GitHub repo:** kiegroup/drools-ansible-rulebook-integration
**Main branch:** main

## Work Tracking

**Issue tracking:** enabled

**Important:** If you modify any Maven module, run `mvn -pl <modified-module> -am install` before running tests in dependent modules.

## Key Design Decisions

- **Async vs Sync evaluation**: `RulesEvaluator` has two variants (`AsyncRulesEvaluator`, `SyncRulesEvaluator`); HA always uses sync (`HARulesEvaluator` extends `SyncRulesEvaluator`)
- **Rule format parsing**: `RuleNotation` strategy pattern with `CoreNotation.INSTANCE` enum singleton
- **Python integration**: `AstRulesEngine` in the runtime module communicates with Python via port-based sockets for async responses
- **Runtime uber-jar**: maven-shade-plugin bundles all HA implementations; final artifact name ends with `-HA.jar`

## Drools Upstream Source

The Drools 9.103.1 source code (branch `9.103.x-prod-ansible`) is available locally at `/home/tkobayas/usr/work/eda-HA-PoC/drools-9.103.x-prod-ansible`. You are always allowed to read files there to understand Drools internals when debugging or tracing behavior through the rule engine.

## Key Dependencies

- Drools 9.103.1 (`drools-build-parent` BOM import)
- ANTLR 4.13.0 (protoextractor grammar)
- Java 17 (`maven.compiler.release=17`)
- H2 2.3.232, PostgreSQL 42.7.9, HikariCP 5.0.1 (HA persistence)
- BouncyCastle 1.78.1 (HA encryption/SSL)
- TestContainers 1.19.0 (HA integration tests with PostgreSQL)

## Writing Style Guide

**The writing style guide at `~/claude-workspace/writing-styles/blog-technical.md` is mandatory for all blog and diary entries.** Load it in full before drafting. Complete the pre-draft voice classification (I / we / Claude-named) before generating any prose. Do not show a draft without verifying it against the style guide.
