# drools-ansible-rulebook-integration-2.0.x Workspace

**Project repo:** /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x
**Workspace type:** public

## Session Start

Run `add-dir /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x` before any other work.

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

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Type

**type:** java
**GitHub repo:** kiegroup/drools-ansible-rulebook-integration
**Main branch:** 2.0.x

## Work Tracking

**Issue tracking:** enabled
## Build Commands

```bash
# Full build (all modules)
mvn clean install

# Build a specific module and its dependencies
mvn -pl <module-name> -am install

# Run tests for a specific module
mvn -pl <module-name> test

# Run a single test class
mvn -pl <module-name> -Dtest=TestClassName test

# Run a single test method
mvn -pl <module-name> -Dtest=TestClassName#methodName test

# Run memory leak tests (excluded by default)
mvn -pl drools-ansible-rulebook-integration-tests -Pmemoryleak-tests test

# Run REST service in dev mode
cd drools-ansible-rulebook-integration-core-rest && mvn compile quarkus:dev
```

**Important:** If you modify any Maven module, run `mvn -pl <modified-module> -am install` before running tests in dependent modules.

## Module Structure

| Module | Purpose |
|--------|---------|
| `api` | Core public APIs: `RulesExecutor`, `RulesExecutorFactory`, `RulesEvaluator`, `RulesSet`, `Rule`, `RuleNotation` |
| `protoextractor` | ANTLR4 grammar (`Protoextractor.g4`) for parsing event data extraction expressions (e.g., `event.data['key'][0].value`) |
| `runtime` | jpy bridge layer (`AstRulesEngine`). Produces uber-jar via maven-shade including all HA modules |
| `main` | CLI entry point (`Main`) for standalone testing. Supports `--ha` and `--ha-db-params` flags |
| `core-rest` | Quarkus REST API (currently commented out in reactor) |
| `benchmark` | JMH performance benchmarks |
| `tests` | Integration tests for the rule engine |
| `ha/` | High Availability meta-module containing 4 submodules (see below) |

## HA Subsystem Architecture

The HA subsystem enables multi-node rule engine deployment with database-backed state persistence and leader election.

**Submodules:**
- **ha-core** — Interfaces (`HAStateManager`), base implementation (`AbstractHAStateManager`), HA-aware executor (`HARulesExecutor`, `HARulesEvaluator`), encryption (`HAEncryption`), state models (`SessionState`, `MatchingEvent`, `EventRecord`, `ActionInfo`)
- **ha-h2** — `H2StateManager` for embedded/test scenarios
- **ha-postgres** — `PostgreSQLStateManager` with SSL/TLS, PEM certificate support (`PemToKeyStoreConverter`), JSONB columns
- **ha-tests** — Integration tests using dual-engine (node1/node2) failover scenarios

**Key patterns:**
- `HAStateManagerFactory` uses reflection-based class loading to instantiate the DB-specific implementation
- `AbstractHAStateManager` is a template method base — subclasses implement SQL/transaction details
- Four DB tables: `SESSION_STATE`, `MATCHING_EVENT`, `ACTION_INFO`, `HA_STATS`
- Leader election: explicit `enableLeader()`/`disableLeader()` with leader ID tracked in `HA_STATS`
- State recovery: on failover, new leader reads `SessionState` (rulesSet hash, event records, processed event IDs) and recreates KieSession
- Encryption: PBKDF2 with primary/secondary key rotation support

**HA test base classes:**
- `AbstractHATestBase` — static DB initialization (H2 or PostgreSQL via `test.db.type` system property)
- `HAStateManagerTestBase` — lower-level state manager tests
- `HAIntegrationTestBase` — sets up dual `AstRulesEngine` instances with `AsyncConsumer` mock

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
