# Handover — 2026-04-22

**Previous handover:** `git show HEAD~1:HANDOFF.md` | diff: `git diff HEAD~1 HEAD -- HANDOFF.md`

## What Changed This Session

- Finished Tasks 8–12 of the load-tests plan. Bug found during Task 8 smoke (OutcomeCheck saw empty match list when `discard_matched_events=true` — the spec's list-based check was wrong). Fix: thread `int matchCount` through `Payload.Execution → Measurement.TimedResult`, change `OutcomeCheck.verify(int, …)`. Commits `b6f1b338` (fix), `47fc7636` … `8f744277` (Tasks 8–11), `bad70a29` (Task 9).
- Post-plan: added **5k** event-count variant (`PayloadGenerator` + `MemoryLeakAnalyzer` + two new JSONs; existing 11 JSONs byte-identical). Commit `198c77db`.
- Post-plan: replaced `load_test_all.sh` with two scripts by cost profile. Commit `364cb289`.
  - `load_test_match_unmatch_noHA.sh` — 4 sizes × match/unmatch × noHA = 8 runs, no Docker.
  - `load_test_match_unmatch_noHA-PGHA.sh` — 3 sizes (1k/5k/10k) × match/unmatch × {noHA, HA-PG} = 12 runs. (Renamed from `load_test_match_unmatch_HA.sh` in `3f4206f2`; `load_test_retention.sh` renamed to `load_test_retention_noHA-PGHA.sh` in the same commit.)
- User ran and verified both new scripts. Branch pushed to `origin/reorganize-load-test`.
- Workspace-repo: spec §5.2/5.3/5.4/5.6/§9 + plan amendment note synced for OutcomeCheck signature change (`9a4c06d`). CLAUDE.md gained load-tests module row + fat-jar build command (`a3f4f6a`). First blog entry written (`0d7cc2c`).

## State Right Now

All 12 plan tasks complete. Branch pushed, no PR. 5 load-test scripts (`load_test_match.sh`, `load_test_unmatch.sh`, `load_test_retention_noHA_HA-PG.sh`, `load_test_match_unmatch_noHA.sh`, `load_test_match_unmatch_noHA_HA-PG.sh`) verified.

Project repo `reorganize-load-test` is 12 commits ahead of `2.0.x` since branch cut.

## Immediate Next Step

User's call — no hard next. Three open housekeeping items if picked up:

1. Workspace spec/plan still reference `load_test_all.sh` and don't mention the 5k variant. Follow the same amendment-note pattern as `9a4c06d`.
2. `.gitignore` the load-test result/log files (`drools-ansible-rulebook-integration-load-tests/result_*.txt`, `out_*.log`) — currently untracked in project repo.
3. Open PR against `kiegroup/drools-ansible-rulebook-integration` `2.0.x` when ready.

## Open Questions / Blockers

None.

## References

| Context | Where | Retrieve with |
|---------|-------|---------------|
| Spec (authoritative system design) | `specs/2026-04-21-load-tests-module-design.md` | `cat` that file |
| Plan (post-execution amended) | `plans/2026-04-21-load-tests-module.md` | `cat` that file |
| This session's narrative | `blog/2026-04-22-tk01-load-tests-smoke-and-split.md` | `cat` that file |
| Previous handover | git history | `git show HEAD~1:HANDOFF.md` |

## Environment

Project-repo `CLAUDE.md` is a **symlink** to the workspace `CLAUDE.md` — edits land in the workspace repo and commit there. The project repo does not track CLAUDE.md content directly.
