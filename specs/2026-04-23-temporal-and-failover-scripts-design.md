# Design — Temporal + Failover Scripts for `drools-ansible-rulebook-integration-load-tests`

**Date:** 2026-04-23
**Branch:** `reorganize-load-test`
**Status:** Approved design; implementation planning next.
**Relates to:** `specs/2026-04-21-load-tests-module-design.md` (master spec for the module)

## 1. Context

The master spec (2026-04-21) ported three categories of load test into the new module — match, unmatch, retention — plus combined drivers. Two scripts from `main/` deliberately stayed deferred as non-goals at that time: `load_test_temporal.sh` (temporal-operator HA persistence cost) and `load_test_failover.sh` (HA failover recovery timing). They are both HA-PG only; noHA is structurally meaningless for either. This spec covers folding both into the new module.

Two other `main/` scripts (`load_test_90kb.sh`, `load_test_ha_compare.sh`) remain out of scope — 90KB is a payload-variant exercise deemed not important, and `ha_compare` exists to show H2 vs PG overhead which is also not required. Explicit confirmation: **neither will be ported**, ever.

## 2. Goals

- Two new multi-size HA-PG-only scripts: `load_test_temporal_HA-PG.sh` (3 sizes × 1 phase = 3 JVM invocations) and `load_test_failover_HA-PG.sh` (3 sizes × 2 phases = 6 JVM invocations).
- Fix the temporal rule's grouping so that `once_within` actually does suppression work — the legacy `load_test_temporal.sh` groups by `event.meta.uuid` (always unique in practice), which makes every event its own one-event group and renders `once_within` a no-op. The new design uses a common `group_id` field with a fixed small number of groups so multiple events share the same suppression window.
- Reuse the existing module infrastructure (`LoadTestMain`, `HaLoadRunner`, `Measurement`, `MetricReporter`, `OutcomeCheck`, `PayloadGenerator`, `lib/common.sh`) with minimal extension. No duplicated PG-setup or metric-parsing logic.
- Share the existing `retention_{100,500,1k}_events.json` payloads for failover (semantically identical rule shape); generate three new `once_within_<size>_events.json` files for temporal via `PayloadGenerator`.
- Preserve the bash/Java metric-line contract: three CSV fields (`<file+tags>, <mem>, <time>`) so `fmt_parse_metrics` keeps working.

## 3. Non-goals

- **`load_test_90kb.sh`** and **`load_test_ha_compare.sh`** — not ported. 90KB payload variants and H2 mode remain out of scope for this module.
- **H2 backend support** — no `--ha` bare flag, no H2 file cleanup. HA-PG only.
- **Wiring the two new scripts into a combined driver** — each stands alone. No `load_test_temporal_failover_*.sh` master driver.
- **Failover across multiple nodes with real socket-level failover** — Phase 1 exits cleanly and Phase 2 cold-starts against the same PG. No network-partition simulation.
- **CI wiring** — operator-run, same as every other script in the module.

## 4. Module additions (file tree delta)

```
drools-ansible-rulebook-integration-load-tests/
├── load_test_temporal_HA-PG.sh                    # NEW
├── load_test_failover_HA-PG.sh                    # NEW
├── lib/common.sh                                  # +size_to_int helper
└── src/
    ├── main/
    │   ├── java/org/drools/ansible/rulebook/integration/loadtests/
    │   │   ├── LoadTestMain.java                  # +flags, +dispatch
    │   │   ├── HaLoadRunner.java                  # +haUuidOverride
    │   │   ├── HaFailoverRecoveryRunner.java      # NEW
    │   │   ├── MetricReporter.java                # +failoverRecovery tag
    │   │   └── gen/
    │   │       └── PayloadGenerator.java          # +once_within files
    │   └── resources/
    │       ├── once_within_100_events.json        # NEW (generated)
    │       ├── once_within_500_events.json        # NEW (generated)
    │       └── once_within_1k_events.json         # NEW (generated)
    └── test/
        └── java/org/drools/ansible/rulebook/integration/loadtests/
            └── MetricReporterTest.java            # +recovery-tag case
```

No changes to `LoadRunner`, `Measurement`, `OutcomeCheck`, `ExpectedOutcome`, `Payload`, `Result`, `MemoryLeakAnalyzer`.

## 5. Java-side contracts

### 5.1 `LoadTestMain` delta

New CLI:
```
java -jar <jar> <events-json>
     [--ha-db-params <json>]
     [--ha-uuid <value>]
     [--failover-recovery]
```

Dispatch (evaluated top-down):
1. `--failover-recovery` present → requires `--ha-db-params` **and** `--ha-uuid`. Call `HaFailoverRecoveryRunner.runRecovery(rulesSet, rulesetJson, haDbParamsJson, haUuid)`. `OutcomeCheck` is **not** called (no payload execution).
2. `--ha-db-params` present (without `--failover-recovery`) → `HaLoadRunner.runLoad(..., haUuidOverride)` where `haUuidOverride` = the `--ha-uuid` value or null.
3. Else → `LoadRunner.run(...)`.

Missing-required-flag errors for case 1 propagate as `IllegalArgumentException` with a clear message naming the missing flag.

### 5.2 `HaLoadRunner` delta

Single signature change to `runLoad`: append `String haUuidOverride` as the last parameter. Inside the method:
```java
String haUuid = haUuidOverride != null
    ? haUuidOverride
    : "loadtest-ha-" + System.currentTimeMillis();
```
Rest of the method unchanged. Non-failover callers pass `null` and get the existing auto-generated UUID.

### 5.3 `HaFailoverRecoveryRunner` (new)

```java
public final class HaFailoverRecoveryRunner {
    public static Result runRecovery(RulesSet rulesSet,
                                     String rulesetJson,
                                     String haDbParamsJson,
                                     String haUuid);
}
```

Flow:
1. `engine.initializeHA(haUuid, "recovery-worker", haDbParamsJson, "{\"write_after\":1}");`
2. `long id = engine.createRuleset(rulesSet, rulesetJson);` — rebuilds `KieSession` from persisted state during `enableLeader()`.
3. Open socket to `localhost:engine.port()` (required for HA `isConnected()` checks).
4. `Measurement.TimedResult t = Measurement.timeWork(() -> { engine.enableLeader(); return Payload.Execution.empty(); });` — reuses the existing timing primitive. `Payload.Execution.empty()` is a new static factory (empty matches, matchCount=0) required for the `Supplier<Payload.Execution>` contract; downstream `matches` and `matchCount` are ignored in this path.
5. `long mem = Measurement.captureUsedMemoryAfterGc();`
6. `return new Result(List.of(), t.durationMs, mem);`
7. Close socket in `finally`; `WARN`-log close failures, do not fail the run.

Mirrors `HaLoadRunner`'s shape. No `OutcomeCheck` call.

### 5.4 `MetricReporter` delta

Signature becomes:
```java
public static void report(PrintStream err,
                          String eventsJson,
                          boolean haPg,
                          boolean failoverRecovery,
                          long usedMemoryBytes,
                          long timeMs);
```

Line format:
```
<eventsJson>[ (HA-PG)][ (failover-recovery)], <usedMemoryBytes>, <timeMs>
```

Bash contract unchanged: three CSV fields split on `,`. `fmt_parse_metrics` keeps using `grep "^<file>"` which matches both Load and Recovery lines without modification.

### 5.5 `PayloadGenerator` delta

Add emission of three `once_within_<size>_events.json` files. Total output set grows from 13 (after the 5k addition) to **16 files**.

Per-file shape:
- `source_args.repeat_count = size / 10`
- `source_args.payload` = array of **10 entries**, each a 24KB realistic event template (same skeleton as `24kb_*_events.json`) with an added top-level `group_id: 0..9` (one per array entry) and fixed `i: 1`.
- `source_args.discard_matched_events = true`
- Rule: one rule, `AllCondition: event.i == 1`, action `debug`, throttle `{ group_by_attributes: ["event.group_id"], once_within: "60 seconds" }`.

`Payload.parsePayload` (unchanged) expands the array × repeat_count into a round-robin interleaved event stream: `[T0,T1,…,T9, T0,T1,…,T9, …]` totalling `size` events. First 10 events fire (one per group), remaining events are suppressed within their 60s window. `MATCHING_EVENT` row count = 10 regardless of size; what scales is per-event HA-write/suppression overhead. This is a realistic usage pattern — many events collapse into a few groups.

Deterministic seeding matches the existing generator convention (`new Random(42L)` for filler text; fixed strings elsewhere).

**Output formatting.** `PayloadGenerator` is switched to **pretty-printed (indented) JSON output** for all files it emits — via Jackson's `ObjectMapper.writerWithDefaultPrettyPrinter()`. The 10-template `once_within` files benefit most (~240KB each with all 10 event templates), but the existing 13 files also get reformatted on next regeneration for consistency; they are functionally unchanged (Jackson's pretty-printer is deterministic for fixed-seed input), only their on-disk representation differs. The reformat is a one-shot diff — subsequent regenerations produce byte-identical output.

## 6. Scripts

### 6.1 `load_test_temporal_HA-PG.sh`

No args. Loops `SIZES=("100" "500" "1k")`. One PG container shared across all three runs with `pg_truncate` between each (fresh baseline per size). Each run is a single JVM invocation with `--ha-db-params`. Outputs a single table.

```
=== Temporal once_within Load Test (HA-PG) ===
File                             Events   Memory(bytes)   Time(ms)   Per-Event(KB)   MATCHING   BlobSize(B)
-----------------------------------------------------------------------------------------------------------
once_within_100_events.json      100      <mem>           <time>     <kb>            10         <bytes>
once_within_500_events.json      500      ...             ...        ...             10         ...
once_within_1k_events.json       1000     ...             ...        ...             10         ...
```

Columns mirror the retention script (`Per-Event(KB)`, `MATCHING`, `BlobSize(B)`) for consistency. `MATCHING` is expected to be 10 for every size by design (fixed group count); deviations signal a regression.

Output: `result_temporal_HA-PG.txt` + `out_temporal_HA-PG.log`.

### 6.2 `load_test_failover_HA-PG.sh`

No args. Loops `SIZES=("100" "500" "1k")` against `retention_<size>_events.json`. For each size:
1. `pg_truncate` (baseline per size; not between Phase 1 and Phase 2 — Phase 2 needs Phase 1's state).
2. Generate a unique per-size `HA_UUID="failover-loadtest-${size}-$(date +%s%N)"`.
3. Phase 1: `jvm_run` with `--ha-db-params "$PG_PARAMS" --ha-uuid "$HA_UUID"` — load events into PG.
4. Phase 2: `jvm_run` with `--ha-db-params "$PG_PARAMS" --ha-uuid "$HA_UUID" --failover-recovery` — cold-start and time `enableLeader()`.
5. Capture load/recovery mem+time via `fmt_parse_metrics`.

Main table (6 rows, size × phase):
```
=== Failover Recovery Load Test (HA-PG) ===
File                                Phase           Memory(bytes)   Time(ms)
----------------------------------------------------------------------------
retention_100_events.json           Load(PG)        <mem>           <time>
retention_100_events.json           Recovery(PG)    <mem>           <time>
retention_500_events.json           Load(PG)        ...             ...
retention_500_events.json           Recovery(PG)    ...             ...
retention_1k_events.json            Load(PG)        ...             ...
retention_1k_events.json            Recovery(PG)    ...             ...
```

Followed by a summary table:
```
=== Recovery Ratio ===
Size   Load(ms)   Recovery(ms)   Ratio
--------------------------------------
100    <lm>       <rm>           <rm/lm * 100>%
500    ...        ...            ...
1k     ...        ...            ...
```

Ratio printed as `N/A` when either phase FAILED or load_time == 0.

Output: `result_failover_HA-PG.txt` + `out_failover_HA-PG.log`.

### 6.3 `lib/common.sh` delta

One new helper:
```bash
# Usage: size_to_int <100|500|1k>; echoes the integer event count.
size_to_int() {
  case "$1" in
    100)  echo 100 ;;
    500)  echo 500 ;;
    1k)   echo 1000 ;;
    *)    echo "ERR"; return 1 ;;
  esac
}
```

Rationale: both new scripts need this for `Per-Event(KB)` computation (temporal) and potential future use (failover can print it too if we decide to add a column). Kept in `common.sh` rather than duplicated.

Everything else (`require_jar`, `require_docker`, `pg_setup`/`cleanup`/`truncate`/`count`/`blob_size`, `jvm_run`, `fmt_parse_metrics`, `fmt_per_event_kb`) is reused as-is.

## 7. Error handling (delta)

- `LoadTestMain` rejects `--failover-recovery` without `--ha-db-params` or `--ha-uuid` with `IllegalArgumentException`, naming the missing flag. The JVM exits non-zero with a stack trace — the bash script sees a missing metric line and records `FAILED, FAILED` for that row.
- Recovery phase `OutcomeCheck` is intentionally skipped. No matches to verify; the measurement is purely timing.
- Ratio computation in the summary table guards against zero/FAILED load time and prints `N/A`.

All other error-handling rules from the master spec §8 apply unchanged.

## 8. Testing (delta)

- `MetricReporterTest` gains one regression case: the recovery line `"retention_100_events.json (HA-PG) (failover-recovery), 3000000, 50"`. Existing cases (noHA, HA-PG) stay.
- No new tests for `HaFailoverRecoveryRunner` — follows the existing exclusion for runners that pull in `AstRulesEngine` + DB.
- No change to `OutcomeCheckTest`, `MeasurementTest`, `MemoryLeakAnalyzerTest`.
- End-to-end smoke (no commit): one full run of each new script at all three sizes. Acceptance gate before declaring the work done.

## 9. Branch, commit hygiene

- **Project repo branch:** continues on `reorganize-load-test` (same branch the master-spec work is on).
- **Workspace repo branch:** continues on `reorganize-load-test`.
- Commits along logical seams:
  1. `LoadTestMain` + `HaLoadRunner` + `MetricReporter` + `HaFailoverRecoveryRunner` (+ `MetricReporterTest` case) — Java surface for failover.
  2. `PayloadGenerator` pretty-print switch + the 13 regenerated existing JSONs — **pure reformat, functionally no-op**. Standalone commit so the diff is auditably "whitespace only".
  3. `PayloadGenerator` temporal-emission branch + the 3 new `once_within_*_events.json` files — temporal payloads.
  4. `lib/common.sh` `size_to_int` + both shell scripts — `load_test_temporal_HA-PG.sh` and `load_test_failover_HA-PG.sh`.
  5. (Optional, if anything needs smoke-driven adjustment.) Fix commit.
- Master spec (`specs/2026-04-21-load-tests-module-design.md`) gets a cross-reference amendment pointing at this spec once implementation lands — deferred housekeeping, not part of this session's scope.
