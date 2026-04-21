# Handoff — 2026-04-21

Mid-implementation of greenfield Maven sub-module `drools-ansible-rulebook-integration-load-tests`. Spec and plan approved; Tasks 1-7 of 12 committed; Task 8 partially done.

## Working locations

- **Project repo:** `/home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x`, branch `reorganize-load-test` (cut from `2.0.x`; **never commit to `2.0.x`**)
- **Workspace repo:** this repo, branch `reorganize-load-test`
- **Spec:** `specs/2026-04-21-load-tests-module-design.md`
- **Plan:** `plans/2026-04-21-load-tests-module.md` — authoritative task list (12 tasks)

## Hard constraints

- **Do not modify any file under `drools-ansible-rulebook-integration-main/`.** Only permitted change outside the new module was one `<module>` line in the root reactor `pom.xml`.
- Use `git -C <path> <cmd>` form for all git commands (user preference).
- Tasks 2-3 commits (`e4a966ad`, `df1c0c82`) are not issue-linked — accepted gap, start linking from Task 4 onward.

## Progress (project repo)

Commits on `reorganize-load-test`:
```
f6c5ccc9 Task 7 — LoadTestMain + LoadRunner + HaLoadRunner + Result  (Closes #3)
5fea3c22 Task 6 — MetricReporter                                      (Refs #3)
074b52e4 Task 5 — Measurement                                         (Refs #3)
14fe3736 Task 4 — ExpectedOutcome + OutcomeCheck                      (Refs #3)
df1c0c82 Task 3 — copy Payload                                        (unlinked)
e4a966ad Task 2 — module scaffold                                     (unlinked)
```

`#N` references are `kiegroup/drools-ansible-rulebook-integration-workspace#N`. Issue repo is **not** the code repo — it's the separate workspace repo: epic #1, children #2-#7 (see epic body for scope mapping).

## Task 8 in flight

- `PayloadGenerator.java` exists at `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/` (uncommitted, `??` in git status).
- Generator was run once; 11 JSON resources written to `src/main/resources/` (all uncommitted, sizes ~24KB each).
- Smoke-test of generated JSONs (`java -jar target/…-jar-with-dependencies.jar 24kb_1k_events.json`) was interrupted — never verified that the JSONs parse and run through the engine.

## Next step (specific)

1. `cd /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x && mvn -pl drools-ansible-rulebook-integration-load-tests -am package -DskipTests`
2. Smoke: `java -Xmx512m -jar drools-ansible-rulebook-integration-load-tests/target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar 24kb_1k_events.json` — expect stderr line `24kb_1k_events.json, <bytes>, <ms>`; no `OutcomeCheck` throw.
3. If OK, commit PayloadGenerator + 11 JSONs with `Closes kiegroup/drools-ansible-rulebook-integration-workspace#4` (see plan Task 8 Step 6 for exact message).
4. Continue to Task 9 (MemoryLeakAnalyzer port with HA-aware 4-group analysis) — spec §5.8, plan Task 9.

## Execution mode

User switched from subagent-driven to **inline execution** mid-session (subagents felt slow). Continue inline: write files → run tests/build → commit → next task. No spec/code-quality review dispatches.

## References (read on demand)

- Plan tasks remaining: 9, 10, 11, 12 — see `plans/2026-04-21-load-tests-module.md` for exact code and commands.
- GH epic: `https://github.com/kiegroup/drools-ansible-rulebook-integration-workspace/issues/1`
