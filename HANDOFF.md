# Handover — 2026-04-23

**Previous handover:** `git show HEAD~7:HANDOFF.md` · diff: `git diff HEAD~7 HEAD -- HANDOFF.md`

## What Changed This Session

- Ported the last two meaningful `main/` scripts into `-load-tests` as HA-PG-only multi-size no-args drivers:
  - `load_test_temporal_HA-PG.sh` (3 sizes × 1 phase) — fixes the legacy `once_within` bug. Legacy grouped by `event.meta.uuid` (always unique → degenerate 1-event groups). New design: 10 fixed groups via `event.group_id`, `size/10` events per group. MATCHING rows stay at 10 regardless of size.
  - `load_test_failover_HA-PG.sh` (3 sizes × 2 phases = 6 JVM runs). Phase 1 loads state under `--ha-uuid`; Phase 2 cold-starts with `--failover-recovery` and times `engine.enableLeader()`.
- Dropped two `main/` scripts explicitly (not porting): `load_test_90kb.sh`, `load_test_ha_compare.sh` (latter needs H2 backend, not deployed).
- Java surface additions: `HaFailoverRecoveryRunner`, `Payload.Execution.empty()` factory, `MetricReporter` gains `failoverRecovery` tag, `HaLoadRunner.runLoad` gains `haUuidOverride`, `LoadTestMain` gains `--ha-uuid` / `--failover-recovery` flags + validation.
- `PayloadGenerator` switched to pretty-printed JSON globally. 13 existing files reformatted in a standalone "semantically zero-op" commit (proven via `json.dumps(sort_keys=True)` canonical diff = empty). 3 new `once_within_*_events.json` files generated. 16 files total.
- Renamed `load_test_match_unmatch_noHA-PGHA.sh` → `…_noHA_HA-PG.sh` and `load_test_retention_noHA-PGHA.sh` → `…_noHA_HA-PG.sh`. Underscore reads as `{noHA, HA-PG}` set separator; previous compact form fused modes. Spec/plan/CLAUDE/HANDOFF synced.
- Smoke results: MATCHING=10 at all temporal sizes; failover recovery ratios 7.4% / 1.2% / 0.5% (recovery is near-flat; load scales linearly).

## State Right Now

Project-repo `reorganize-load-test` is **20 commits ahead** of `2.0.x`, pushed to `origin`. Workspace branch is up to date with its origin. **7 load-test scripts** in the module — `load_test_{match,unmatch}.sh`, `load_test_retention_noHA_HA-PG.sh`, `load_test_match_unmatch_{noHA,noHA_HA-PG}.sh`, `load_test_temporal_HA-PG.sh`, `load_test_failover_HA-PG.sh`. No PR opened.

## Immediate Next Step

User's call. Three live housekeeping items:
1. Open PR against `kiegroup/drools-ansible-rulebook-integration` `2.0.x` when ready.
2. Master spec (`specs/2026-04-21-load-tests-module-design.md`) still doesn't cross-reference the 2026-04-23 spec for temporal+failover. Deferred housekeeping.
3. Recovery ratios (7.4→1.2→0.5%) may be worth a follow-up blog entry once the numbers are reproduced on a different machine.

## Open Questions / Blockers

None.

## References

| Context | Where | Retrieve with |
|---|---|---|
| Temporal+failover spec (authoritative for today's work) | `specs/2026-04-23-temporal-and-failover-scripts-design.md` | `cat` |
| Temporal+failover plan | `plans/2026-04-23-temporal-and-failover-scripts.md` | `cat` |
| Master module spec | `specs/2026-04-21-load-tests-module-design.md` | `cat` |
| This session's blog | `blog/2026-04-23-tk01-temporal-and-failover-scripts.md` | `cat` |
| Previous blog | `blog/2026-04-22-tk01-load-tests-smoke-and-split.md` | `cat` |

## Environment

*Unchanged — `git show HEAD~7:HANDOFF.md`*
