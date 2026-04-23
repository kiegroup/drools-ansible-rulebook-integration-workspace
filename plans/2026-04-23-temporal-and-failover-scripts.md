# Temporal + Failover Scripts Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Port `load_test_temporal.sh` and `load_test_failover.sh` from `drools-ansible-rulebook-integration-main/` into the new `drools-ansible-rulebook-integration-load-tests/` module as `load_test_temporal_HA-PG.sh` and `load_test_failover_HA-PG.sh`, fixing the legacy temporal rule's degenerate `meta.uuid` grouping.

**Architecture:** Both new scripts are multi-size, no-args drivers that run HA-PG only. `LoadTestMain` gains two flags (`--ha-uuid`, `--failover-recovery`); `HaLoadRunner` gains an optional `haUuidOverride` parameter; a new `HaFailoverRecoveryRunner` class handles the Phase-2 recovery timing. `MetricReporter` gains an optional `(failover-recovery)` tag. `PayloadGenerator` switches to pretty-printed output globally and emits three new `once_within_*_events.json` files. `lib/common.sh` gains one new helper (`size_to_int`).

**Tech Stack:** Java 17, Maven, JUnit 5 + AssertJ (unit tests), SLF4J simple, Drools 9.103.1 transitively via `-runtime`. Bash + Docker + `postgres:15-alpine` at script-run time.

**Working branches (do not deviate):**
- **Project repo:** `/home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x` — continue on branch `reorganize-load-test` (cut from `2.0.x`). **Never commit directly to `2.0.x`.**
- **Workspace repo:** `/home/tkobayas/claude/public/drools-ansible-rulebook-integration-2.0.x` — branch `reorganize-load-test` is already checked out and carries the spec + this plan.

**Hard constraint:** Do not modify any file under `drools-ansible-rulebook-integration-main/`. Keep every change inside the new module.

**Spec reference:** `specs/2026-04-23-temporal-and-failover-scripts-design.md` (in workspace). Read it before starting. The master module spec is `specs/2026-04-21-load-tests-module-design.md`.

**Commit conventions:** Conventional-commits prefix (`feat:`, `chore:`, `test:` etc.). Every commit ends with:
```
Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

All git commands in the plan target the **project repo**. Use `git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x <cmd>` or run from that directory.

---

## Task 1: Extend `MetricReporter` with `failoverRecovery` tag (TDD)

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporter.java`
- Modify: `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporterTest.java`
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadTestMain.java` (single call-site update — pass `false` pending wiring in Task 4)

### Step 1: Write the failing test

Append to `MetricReporterTest.java` (before the final `}`):

```java
    @Test
    void reportsFailoverRecoveryLine() {
        ByteArrayOutputStream buf = new ByteArrayOutputStream();
        try (PrintStream ps = new PrintStream(buf, true, StandardCharsets.UTF_8)) {
            MetricReporter.report(ps, "retention_100_events.json", true, true, 3_000_000L, 50L);
        }
        assertThat(buf.toString(StandardCharsets.UTF_8).trim())
                .isEqualTo("retention_100_events.json (HA-PG) (failover-recovery), 3000000, 50");
    }
```

Also update the **two existing tests** in that file to pass the new `failoverRecovery=false` argument. The existing `reportsNoHaLine` and `reportsHaPgLine` calls become:

```java
MetricReporter.report(ps, "24kb_1k_events.json", false, false, 5_200_000L, 195L);
// ...
MetricReporter.report(ps, "24kb_1k_events.json", true, false, 7_100_000L, 240L);
```

### Step 2: Run the tests to verify they fail (compilation error expected)

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test -Dtest=MetricReporterTest
```
Expected: **BUILD FAILURE** with `method report in class MetricReporter cannot be applied to given types` (signature mismatch — `MetricReporter.report` doesn't yet accept the extra `boolean` parameter).

### Step 3: Update `MetricReporter.report` signature

Replace the entire body of `MetricReporter.java` with:

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.io.PrintStream;

public final class MetricReporter {

    private MetricReporter() {}

    public static void report(PrintStream err, String eventsJson, boolean haPg,
                              boolean failoverRecovery,
                              long usedMemoryBytes, long timeMs) {
        StringBuilder sb = new StringBuilder();
        sb.append(eventsJson);
        if (haPg) {
            sb.append(" (HA-PG)");
        }
        if (failoverRecovery) {
            sb.append(" (failover-recovery)");
        }
        sb.append(", ").append(usedMemoryBytes);
        sb.append(", ").append(timeMs);
        err.println(sb.toString());
    }
}
```

### Step 4: Update the sole production call site in `LoadTestMain.java`

Find this line (currently line 60):

```java
        MetricReporter.report(System.err, eventsJson, haPg, result.usedMemoryBytes, result.durationMs);
```

Change it to (pass `false` as the new `failoverRecovery` argument — we'll flip this per-branch in Task 4):

```java
        MetricReporter.report(System.err, eventsJson, haPg, false, result.usedMemoryBytes, result.durationMs);
```

### Step 5: Run the tests to verify they pass

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test -Dtest=MetricReporterTest
```
Expected: **BUILD SUCCESS**, 3 tests pass.

### Step 6: Run the full module test suite as a regression fence

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: **BUILD SUCCESS**, all tests pass.

No commit yet — we'll group the Java surface changes into one commit at the end of Task 5.

---

## Task 2: Add `haUuidOverride` parameter to `HaLoadRunner`

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaLoadRunner.java`
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadTestMain.java` (sole call site)

### Step 1: Update `HaLoadRunner.runLoad` signature

Open `HaLoadRunner.java`. Change the method signature and the UUID-generation line inside. Final form:

```java
public static Result runLoad(RulesSet rulesSet, String rulesetJson, Map rulesSetMap,
                             String haDbParamsJson, ExpectedOutcome expected, String eventsJson,
                             String haUuidOverride) {
    try (AstRulesEngine engine = new AstRulesEngine()) {
        String haUuid = haUuidOverride != null
                ? haUuidOverride
                : "loadtest-ha-" + System.currentTimeMillis();
        engine.initializeHA(haUuid, "loadtest-worker", haDbParamsJson, "{\"write_after\":1}");
        // ... rest of method unchanged ...
```

Nothing else in the method changes. The `engine.initializeHA(...)` call already uses the local `haUuid` variable.

### Step 2: Update the LoadTestMain call site to pass `null` (preserves current behavior)

In `LoadTestMain.java`, find the existing call:

```java
Result result = haPg
        ? HaLoadRunner.runLoad(rulesSet, rulesetJson, rulesSetMap, haDbParamsJson, expected, eventsJson)
        : LoadRunner.run(rulesSet, rulesSetMap, expected, eventsJson);
```

Change to:

```java
Result result = haPg
        ? HaLoadRunner.runLoad(rulesSet, rulesetJson, rulesSetMap, haDbParamsJson, expected, eventsJson, null)
        : LoadRunner.run(rulesSet, rulesSetMap, expected, eventsJson);
```

### Step 3: Verify compilation

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests compile
```
Expected: **BUILD SUCCESS**.

### Step 4: Run full test suite

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: **BUILD SUCCESS**, all tests pass.

No commit yet.

---

## Task 3: Add `Payload.Execution.empty()` and create `HaFailoverRecoveryRunner`

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Payload.java`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaFailoverRecoveryRunner.java`

### Step 1: Add `Payload.Execution.empty()` static factory

In `Payload.java`, find the `Execution` inner class (around line 125):

```java
    public static final class Execution {
        public final List<Map> matches;
        public final int matchCount;

        public Execution(List<Map> matches, int matchCount) {
            this.matches = matches;
            this.matchCount = matchCount;
        }
    }
```

Add a static factory method inside the class, right after the existing constructor:

```java
    public static final class Execution {
        public final List<Map> matches;
        public final int matchCount;

        public Execution(List<Map> matches, int matchCount) {
            this.matches = matches;
            this.matchCount = matchCount;
        }

        public static Execution empty() {
            return new Execution(List.of(), 0);
        }
    }
```

### Step 2: Create `HaFailoverRecoveryRunner.java`

Create `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaFailoverRecoveryRunner.java` with this exact content:

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.io.IOException;
import java.net.Socket;
import java.util.List;

import org.drools.ansible.rulebook.integration.api.domain.RulesSet;
import org.drools.ansible.rulebook.integration.core.jpy.AstRulesEngine;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Runs the Phase 2 (Recovery) half of a two-JVM failover load test.
 *
 * Cold-starts a new AstRulesEngine against the same PG state written by a
 * prior {@link HaLoadRunner} invocation with the same {@code haUuid}, times
 * {@code engine.enableLeader()} (which triggers session recovery from PG),
 * and returns the recovery duration + post-GC memory snapshot.
 *
 * No payload execution. No OutcomeCheck call.
 */
public final class HaFailoverRecoveryRunner {

    private static final Logger LOGGER = LoggerFactory.getLogger(HaFailoverRecoveryRunner.class);

    private HaFailoverRecoveryRunner() {}

    public static Result runRecovery(RulesSet rulesSet, String rulesetJson,
                                     String haDbParamsJson, String haUuid) {
        try (AstRulesEngine engine = new AstRulesEngine()) {
            engine.initializeHA(haUuid, "recovery-worker", haDbParamsJson, "{\"write_after\":1}");

            long id = engine.createRuleset(rulesSet, rulesetJson);
            int port = engine.port();

            Socket haSocket;
            try {
                haSocket = new Socket("localhost", port);
            } catch (IOException e) {
                throw new RuntimeException("Failed to connect HA socket", e);
            }

            try {
                LOGGER.info("*** Start measuring recovery time (enableLeader)");
                Measurement.TimedResult t = Measurement.timeWork(() -> {
                    engine.enableLeader();
                    return Payload.Execution.empty();
                });
                LOGGER.info("*** End measuring recovery time, duration = {} ms", t.durationMs);

                String stats = engine.sessionStats(id);
                LOGGER.info(stats);

                long mem = Measurement.captureUsedMemoryAfterGc();

                return new Result(List.of(), t.durationMs, mem);
            } finally {
                try {
                    haSocket.close();
                } catch (IOException e) {
                    LOGGER.warn("Failed to close HA socket", e);
                }
            }
        }
    }
}
```

### Step 3: Verify compilation

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests compile
```
Expected: **BUILD SUCCESS**.

No commit yet.

---

## Task 4: Parse new flags + wire dispatch in `LoadTestMain`

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadTestMain.java`

### Step 1: Extend argument parsing

Replace the current `main(String[] args)` body (lines 23-61 of `LoadTestMain.java`) with:

```java
    public static void main(String[] args) {
        String haDbParamsJson = null;
        String haUuid = null;
        boolean failoverRecovery = false;
        List<String> positional = new ArrayList<>();
        for (int i = 0; i < args.length; i++) {
            if ("--ha-db-params".equals(args[i])) {
                if (i + 1 >= args.length) {
                    System.err.println("ERROR: --ha-db-params requires a JSON argument");
                    System.exit(1);
                }
                haDbParamsJson = args[++i];
            } else if ("--ha-uuid".equals(args[i])) {
                if (i + 1 >= args.length) {
                    System.err.println("ERROR: --ha-uuid requires a value");
                    System.exit(1);
                }
                haUuid = args[++i];
            } else if ("--failover-recovery".equals(args[i])) {
                failoverRecovery = true;
            } else {
                positional.add(args[i]);
            }
        }
        String eventsJson = positional.isEmpty() ? DEFAULT_JSON : positional.get(0);

        // Outcome derived from filename convention:
        //   contains "unmatch"       -> NO_MATCH
        //   starts with "retention_" -> NO_MATCH
        //   otherwise                -> MATCH
        ExpectedOutcome expected = (eventsJson.contains("unmatch") || eventsJson.startsWith("retention_"))
                ? ExpectedOutcome.NO_MATCH
                : ExpectedOutcome.MATCH;

        String rulesJsonRaw = readRulesJson(eventsJson);
        Map jsonObject = rulesJsonRaw.startsWith("[")
                ? (Map) JsonMapper.readValueAsListOfObject(rulesJsonRaw).get(0)
                : JsonMapper.readValueAsMapOfStringAndObject(rulesJsonRaw);
        Map rulesSetMap = (Map) jsonObject.get("RuleSet");
        String rulesetJson = JsonMapper.toJson(rulesSetMap);
        RulesSet rulesSet = RuleNotation.CoreNotation.INSTANCE.toRulesSet(RuleFormat.JSON, rulesetJson);

        boolean haPg = haDbParamsJson != null;
        Result result;
        if (failoverRecovery) {
            if (haDbParamsJson == null) {
                throw new IllegalArgumentException("--ha-db-params is required for --failover-recovery mode");
            }
            if (haUuid == null) {
                throw new IllegalArgumentException("--ha-uuid is required for --failover-recovery mode");
            }
            result = HaFailoverRecoveryRunner.runRecovery(rulesSet, rulesetJson, haDbParamsJson, haUuid);
        } else if (haPg) {
            result = HaLoadRunner.runLoad(rulesSet, rulesetJson, rulesSetMap, haDbParamsJson, expected, eventsJson, haUuid);
        } else {
            result = LoadRunner.run(rulesSet, rulesSetMap, expected, eventsJson);
        }

        MetricReporter.report(System.err, eventsJson, haPg, failoverRecovery, result.usedMemoryBytes, result.durationMs);
    }
```

Note: the `haUuid` variable is always passed into `HaLoadRunner.runLoad(...)` — it's `null` if the caller didn't supply `--ha-uuid` (preserving auto-generation), and a real value otherwise.

### Step 2: Verify compilation

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests compile
```
Expected: **BUILD SUCCESS**.

### Step 3: Run full test suite

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: **BUILD SUCCESS**, all tests pass (3 MetricReporter + 4 OutcomeCheck + 1 Measurement + 3 MemoryLeakAnalyzer — 11 tests total).

No commit yet.

---

## Task 5: Build fat jar + commit Java surface changes

**Files:** none (verification + commit only)

### Step 1: Rebuild fat jar

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am package -DskipTests
```
Expected: **BUILD SUCCESS**. The file `drools-ansible-rulebook-integration-load-tests/target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar` exists and has a recent mtime.

### Step 2: Quick CLI-flag sanity check (no PG)

Run:
```bash
java -jar drools-ansible-rulebook-integration-load-tests/target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar \
     24kb_1k_events.json --failover-recovery 2>&1 | head -5
```
Expected: the error message
```
Exception in thread "main" java.lang.IllegalArgumentException: --ha-db-params is required for --failover-recovery mode
```
confirms the flag parsing + validation works. (No PG container is started — the error fires before HA initialization.)

### Step 3: Commit

```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x add \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporter.java \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaLoadRunner.java \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaFailoverRecoveryRunner.java \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadTestMain.java \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Payload.java \
    drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporterTest.java

git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x commit -m "$(cat <<'EOF'
feat(load-tests): add failover-recovery runner + CLI flags

LoadTestMain gains --ha-uuid and --failover-recovery flags and
dispatches to a new HaFailoverRecoveryRunner in recovery mode.
HaLoadRunner gains an optional haUuidOverride parameter so Phase 1
(load) can share the same HA UUID with Phase 2 (recovery).
MetricReporter tags recovery lines with " (failover-recovery)" after
the existing " (HA-PG)" suffix; bash contract preserved at 3 CSV
fields. MetricReporterTest gets a regression case for the new tag;
existing cases pass the new failoverRecovery=false argument.

Payload.Execution gains a static empty() factory so the recovery path
can reuse Measurement.timeWork without duplicating a timing primitive.

Prereq for load_test_failover_HA-PG.sh (forthcoming).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Verify:
```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x log --oneline -1
```
Expected: `<hash> feat(load-tests): add failover-recovery runner + CLI flags`.

---

## Task 6: Switch `PayloadGenerator` to pretty-printed output + regenerate 13 existing files

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/PayloadGenerator.java`
- Regenerate (no hand edits): all 13 `24kb_*.json` + `retention_*.json` files under `drools-ansible-rulebook-integration-load-tests/src/main/resources/`.

### Step 1: Switch the write path to pretty-printed JSON

In `PayloadGenerator.java`, replace the top imports block and the `write(...)` method.

First, **add** this import (alongside the existing `org.drools.ansible.rulebook.integration.api.io.JsonMapper` import):

```java
import com.fasterxml.jackson.databind.ObjectMapper;
```

Second, **add** a static `ObjectMapper` field. Place it inside the class, immediately above the existing `private static final long SEED = 42L;` line:

```java
    private static final ObjectMapper PRETTY_MAPPER = new ObjectMapper();
```

Third, **replace** the existing `write(...)` method body (currently uses `JsonMapper.toJson(...)`) with a pretty-print writer:

```java
    private static void write(Path path, Map<String, Object> content) throws IOException {
        String json = PRETTY_MAPPER.writerWithDefaultPrettyPrinter()
                .writeValueAsString(List.of(content));
        Files.writeString(path, json + System.lineSeparator(), StandardCharsets.UTF_8);
        System.out.println("  wrote " + path + " (" + json.length() + " bytes)");
    }
```

**Leave `buildMessage(...)` untouched** — it uses `JsonMapper.toJson(...)` to compute the compact-JSON overhead for the 24KB target budget. That budget is a logical event-content size measurement; whitespace-free serialization keeps the generated event template byte-identical to pre-change output.

### Step 2: Install the updated module artifacts

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am install -DskipTests
```
Expected: **BUILD SUCCESS**.

### Step 3: Regenerate every JSON file (13 existing — 3 new come in Task 7)

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests exec:java \
    -Dexec.mainClass=org.drools.ansible.rulebook.integration.loadtests.gen.PayloadGenerator \
    -Dexec.classpathScope=compile
```
Expected: `Wrote 11 payload JSON files to ...` (the existing generator message still says 11 because the 5k pair is emitted via the same loop; it's actually 13 files on disk after this run — `24kb_{1k,5k,10k,100k,1m}_events{,_unmatch}.json` = 10 plus `retention_{100,500,1k}_events.json` = 3).

### Step 4: Confirm the diff is whitespace-only (sanity check)

Run:
```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x diff --stat drools-ansible-rulebook-integration-load-tests/src/main/resources/
```
Expected: 13 files modified, `+N / -M` insertions/deletions (N/M both in hundreds — whitespace-only reformat expands line count).

Spot-check that the *content* is unchanged — compare canonicalized JSON of one file before/after:
```bash
# from project root
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x show HEAD:drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_1k_events.json \
  | python3 -c 'import json,sys; print(json.dumps(json.load(sys.stdin), sort_keys=True))' > /tmp/before.json
python3 -c 'import json,sys; print(json.dumps(json.load(sys.stdin), sort_keys=True))' \
  < drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_1k_events.json > /tmp/after.json
diff /tmp/before.json /tmp/after.json
```
Expected: no output (files are semantically identical).

### Step 5: Run the MemoryLeakAnalyzer unit test as a regression fence

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: **BUILD SUCCESS**, all 11 tests pass. No test reads these JSON files directly; this is just making sure nothing else broke.

### Step 6: Commit the reformat as a standalone "functionally no-op" commit

```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x add \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/PayloadGenerator.java \
    drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_*.json \
    drools-ansible-rulebook-integration-load-tests/src/main/resources/retention_*.json

git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x commit -m "$(cat <<'EOF'
chore(load-tests): pretty-print PayloadGenerator output

PayloadGenerator.write() now emits indented JSON via Jackson's default
pretty-printer, rather than JsonMapper.toJson's compact output.
Regenerating the 13 existing payload JSONs to match produces a
whitespace-only diff — the events are byte-identical after canonical
reparse (verified with python json.dumps sort_keys).

Rationale: the forthcoming once_within_*_events.json files carry 10
distinct event templates each (240KB on disk pretty-printed) and are
genuinely easier to read indented. Consistency beats a bifurcated
generator that emits two formats.

buildMessage() intentionally still uses JsonMapper.toJson(compact) to
compute the 24KB event-content budget — that's a logical content-size
measurement, not file-display size, and changing it would shrink the
filler message.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Emit `once_within_*_events.json` files via `PayloadGenerator`

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/PayloadGenerator.java`
- Create (via generator): `drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_{100,500,1k}_events.json`

### Step 1: Add a `temporalRuleset(...)` ruleset builder

Inside `PayloadGenerator.java`, add this private helper method (place it immediately after the existing `retentionRuleset(...)` method):

```java
    /**
     * Builds a once_within rule set that groups by a common event field.
     *
     * Shape:
     *   - payload array: 10 entries, one per group_id 0..9
     *   - repeat_count: repeatCount (which will be N/10 for total events = N)
     *   - rule: AllCondition(event.i == 1), action=debug,
     *           throttle { group_by_attributes: ["event.group_id"], once_within: "60 seconds" }
     *   - discard_matched_events: true
     *
     * With 10 groups and first-event-per-group firing, MATCHING_EVENT rows
     * stay at 10 regardless of size; what scales is per-event HA-write and
     * suppression overhead.
     */
    private static Map<String, Object> temporalRuleset(String name, Map<String, Object> event, int repeatCount) {
        List<Map<String, Object>> conditions = List.of(equalsCondition("a", 1));

        LinkedHashMap<String, Object> condition = new LinkedHashMap<>();
        condition.put("AllCondition", conditions);

        LinkedHashMap<String, Object> actionBody = new LinkedHashMap<>();
        actionBody.put("action", "debug");
        actionBody.put("action_args", new LinkedHashMap<>());
        LinkedHashMap<String, Object> action = new LinkedHashMap<>();
        action.put("Action", actionBody);

        LinkedHashMap<String, Object> throttle = new LinkedHashMap<>();
        throttle.put("group_by_attributes", List.of("event.group_id"));
        throttle.put("once_within", "60 seconds");

        LinkedHashMap<String, Object> ruleBody = new LinkedHashMap<>();
        ruleBody.put("name", "r1");
        ruleBody.put("condition", condition);
        ruleBody.put("action", action);
        ruleBody.put("enabled", true);
        ruleBody.put("throttle", throttle);
        LinkedHashMap<String, Object> rule = new LinkedHashMap<>();
        rule.put("Rule", ruleBody);

        // Ten event templates, one per group_id 0..9. Each carries the 24KB
        // realistic payload plus a top-level group_id field.
        List<Map<String, Object>> templates = new ArrayList<>(10);
        for (int g = 0; g < 10; g++) {
            LinkedHashMap<String, Object> copy = new LinkedHashMap<>(event);
            copy.put("group_id", g);
            templates.add(copy);
        }

        LinkedHashMap<String, Object> sourceArgs = new LinkedHashMap<>();
        sourceArgs.put("discard_matched_events", true);
        sourceArgs.put("repeat_count", repeatCount);
        sourceArgs.put("payload", templates);

        LinkedHashMap<String, Object> eventSource = new LinkedHashMap<>();
        eventSource.put("name", "generic");
        eventSource.put("source_name", "generic");
        eventSource.put("source_args", sourceArgs);
        eventSource.put("source_filters", List.of());

        LinkedHashMap<String, Object> sourceEntry = new LinkedHashMap<>();
        sourceEntry.put("EventSource", eventSource);

        LinkedHashMap<String, Object> ruleSet = new LinkedHashMap<>();
        ruleSet.put("name", name);
        ruleSet.put("hosts", List.of("all"));
        ruleSet.put("sources", List.of(sourceEntry));
        ruleSet.put("rules", List.of(rule));

        LinkedHashMap<String, Object> wrapper = new LinkedHashMap<>();
        wrapper.put("RuleSet", ruleSet);
        return wrapper;
    }
```

### Step 2: Emit the three temporal files from `main(...)`

In `PayloadGenerator.main(...)`, find the existing retention emission block:

```java
        for (int n : new int[] { 100, 500, 1000 }) {
            String label = n == 1000 ? "1k" : String.valueOf(n);
            write(resourcesDir.resolve("retention_" + label + "_events.json"),
                    retentionRuleset("retention " + label + " events", event, n));
        }
```

**Immediately after** that block, add:

```java
        for (int n : new int[] { 100, 500, 1000 }) {
            String label = n == 1000 ? "1k" : String.valueOf(n);
            // 10 groups, N/10 iterations → N total events per file.
            int repeatCount = n / 10;
            write(resourcesDir.resolve("once_within_" + label + "_events.json"),
                    temporalRuleset("once_within " + label + " events", event, repeatCount));
        }
```

Also update the final summary `System.out.println("Wrote 11 payload JSON files ...")` to reflect the new count — change `11` to `16`:

```java
        System.out.println("Wrote 16 payload JSON files to " + resourcesDir.toAbsolutePath());
```

Also update the class Javadoc (lines 17-25) — change `Generates the 11 test-event JSON files` to `Generates the 16 test-event JSON files`.

### Step 3: Reinstall + regenerate

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am install -DskipTests
mvn -pl drools-ansible-rulebook-integration-load-tests exec:java \
    -Dexec.mainClass=org.drools.ansible.rulebook.integration.loadtests.gen.PayloadGenerator \
    -Dexec.classpathScope=compile
```
Expected: `Wrote 16 payload JSON files to ...`.

### Step 4: Verify the three new files exist and parse

Run:
```bash
ls -la drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_*.json
python3 -c 'import json; [json.load(open(p)) for p in __import__("glob").glob("drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_*.json")] ; print("all three parse")'
```
Expected: three files (100, 500, 1k variants), `all three parse` printed.

### Step 5: Spot-check shape

Run:
```bash
python3 -c '
import json
data = json.load(open("drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_100_events.json"))
rs = data[0]["RuleSet"]
src = rs["sources"][0]["EventSource"]["source_args"]
print("repeat_count:", src["repeat_count"])
print("templates:", len(src["payload"]))
print("group_ids:", [t["group_id"] for t in src["payload"]])
rule = rs["rules"][0]["Rule"]
print("throttle:", rule.get("throttle"))
'
```
Expected:
```
repeat_count: 10
templates: 10
group_ids: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
throttle: {'group_by_attributes': ['event.group_id'], 'once_within': '60 seconds'}
```

### Step 6: Commit temporal payloads

```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x add \
    drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/PayloadGenerator.java \
    drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_100_events.json \
    drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_500_events.json \
    drools-ansible-rulebook-integration-load-tests/src/main/resources/once_within_1k_events.json

git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x commit -m "$(cat <<'EOF'
feat(load-tests): generate once_within payloads for temporal test

PayloadGenerator emits three new JSON files:
  once_within_{100,500,1k}_events.json

Each carries 10 event templates (group_id = 0..9) with repeat_count
N/10, so Payload.parsePayload's existing "templates × repeat_count"
expansion produces a round-robin stream of N total events with
deterministic group assignment.

Rule shape: AllCondition(event.i == 1), once_within 60s grouped by
event.group_id. First event of each group fires, the rest are
suppressed within the 60s window. MATCHING_EVENT row count stays at
10 regardless of size; what scales is per-event HA-write and
suppression overhead — a realistic usage pattern (many events
collapse into a few groups).

Fixes the degenerate grouping of main/load_test_temporal.sh which
groups by event.meta.uuid (always unique → every event is its own
one-event group → once_within is a no-op).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Add `size_to_int` to `lib/common.sh`

**Files:**
- Modify: `drools-ansible-rulebook-integration-load-tests/lib/common.sh`

### Step 1: Add the helper

Open `drools-ansible-rulebook-integration-load-tests/lib/common.sh`. Find the `fmt_per_event_kb` function (around line 145). **After** that function (at the end of the file), append:

```bash
# size_to_int <100|500|1k>
# Echoes the integer event count for a retention/temporal size label.
# Prints "ERR" and returns non-zero for an unknown label.
size_to_int() {
  case "$1" in
    100)  echo 100 ;;
    500)  echo 500 ;;
    1k)   echo 1000 ;;
    *)    echo "ERR"; return 1 ;;
  esac
}
```

### Step 2: Smoke-test the helper in isolation

Run:
```bash
bash -c 'source drools-ansible-rulebook-integration-load-tests/lib/common.sh && for s in 100 500 1k; do echo "$s -> $(size_to_int $s)"; done'
```
Expected:
```
100 -> 100
500 -> 500
1k -> 1000
```

No commit yet (grouped with script commits in Task 11).

---

## Task 9: Create `load_test_temporal_HA-PG.sh`

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/load_test_temporal_HA-PG.sh`

### Step 1: Write the script

Create `drools-ansible-rulebook-integration-load-tests/load_test_temporal_HA-PG.sh` with this exact content:

```bash
#!/usr/bin/env bash
# Usage: ./load_test_temporal_HA-PG.sh
#
# Measures HA persistence cost for a once_within rule under rapid ingress.
# 10 groups × (size/10) events per group. First event per group fires
# (10 MATCHING_EVENT rows), the rest are suppressed within a 60s window.
# HA-PG only. Requires Docker for PostgreSQL.

set -euo pipefail
source "$(dirname "$0")/lib/common.sh"

SIZES=("100" "500" "1k")

OUT="result_temporal_HA-PG.txt"
LOG="out_temporal_HA-PG.log"
> "$OUT"
> "$LOG"

require_jar
trap pg_cleanup EXIT
pg_setup

# Header
{
  echo "=== Temporal once_within Load Test (HA-PG) ==="
  printf "%-32s %8s %14s %9s %14s %10s %12s\n" \
    "File" "Events" "Memory(bytes)" "Time(ms)" "Per-Event(KB)" "MATCHING" "BlobSize(B)"
  printf -- "-%.0s" {1..105}; echo
} | tee -a "$OUT"

for size in "${SIZES[@]}"; do
  pg_truncate
  file="once_within_${size}_events.json"
  events=$(size_to_int "$size")
  label="$file (HA-PG)"
  echo "Running $label..."
  jvm_run "$label" "$file" --ha-db-params "$PG_PARAMS"
  fmt_parse_metrics "$_run_stderr" "$file"
  kb=$(fmt_per_event_kb "$_mem" "$events")
  matching=$(pg_count drools_ansible_matching_event)
  blob=$(pg_blob_size)
  printf "%-32s %8d %14s %9s %14s %10s %12s\n" \
    "$file" "$events" "$_mem" "$_time" "$kb" "$matching" "$blob" | tee -a "$OUT"
done

echo ""
echo "Results written to $OUT"
echo "Full logs in $LOG"
```

No commit yet.

---

## Task 10: Create `load_test_failover_HA-PG.sh`

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/load_test_failover_HA-PG.sh`

### Step 1: Write the script

Create `drools-ansible-rulebook-integration-load-tests/load_test_failover_HA-PG.sh` with this exact content:

```bash
#!/usr/bin/env bash
# Usage: ./load_test_failover_HA-PG.sh
#
# Measures HA failover recovery time across two JVM runs sharing one PG:
#   Phase 1 (Load):     --ha-db-params $PG_PARAMS --ha-uuid $HA_UUID
#   Phase 2 (Recovery): + --failover-recovery, times engine.enableLeader()
# Uses retention_{size}_events.json (2-condition join, partial matches retained).
# HA-PG only. Requires Docker for PostgreSQL.

set -euo pipefail
source "$(dirname "$0")/lib/common.sh"

SIZES=("100" "500" "1k")

OUT="result_failover_HA-PG.txt"
LOG="out_failover_HA-PG.log"
> "$OUT"
> "$LOG"

require_jar
trap pg_cleanup EXIT
pg_setup

# Header
{
  echo "=== Failover Recovery Load Test (HA-PG) ==="
  printf "%-34s %-14s %14s %9s\n" "File" "Phase" "Memory(bytes)" "Time(ms)"
  printf -- "-%.0s" {1..75}; echo
} | tee -a "$OUT"

declare -A LOAD_MS RECOVERY_MS

for size in "${SIZES[@]}"; do
  pg_truncate  # baseline per size; NOT between Phase 1 and Phase 2
  file="retention_${size}_events.json"
  HA_UUID="failover-loadtest-${size}-$(date +%s%N)"

  # Phase 1: Load events into PG under $HA_UUID
  label="$file (load)"
  echo "Running $label..."
  jvm_run "$label" "$file" --ha-db-params "$PG_PARAMS" --ha-uuid "$HA_UUID"
  fmt_parse_metrics "$_run_stderr" "$file"
  LOAD_MS[$size]="$_time"
  printf "%-34s %-14s %14s %9s\n" "$file" "Load(PG)" "$_mem" "$_time" | tee -a "$OUT"

  # Phase 2: Cold-start Node2, time enableLeader() recovery against the same $HA_UUID
  label="$file (recovery)"
  echo "Running $label..."
  jvm_run "$label" "$file" --ha-db-params "$PG_PARAMS" --ha-uuid "$HA_UUID" --failover-recovery
  fmt_parse_metrics "$_run_stderr" "$file"
  RECOVERY_MS[$size]="$_time"
  printf "%-34s %-14s %14s %9s\n" "$file" "Recovery(PG)" "$_mem" "$_time" | tee -a "$OUT"
done

# Ratio table
{
  echo ""
  echo "=== Recovery Ratio ==="
  printf "%-6s %10s %14s %8s\n" "Size" "Load(ms)" "Recovery(ms)" "Ratio"
  printf -- "-%.0s" {1..42}; echo
  for size in "${SIZES[@]}"; do
    lm="${LOAD_MS[$size]}"
    rm_val="${RECOVERY_MS[$size]}"
    ratio="N/A"
    if [ "$lm" != "FAILED" ] && [ "$rm_val" != "FAILED" ] && [ "$lm" -gt 0 ] 2>/dev/null; then
      ratio=$(awk "BEGIN { printf \"%.1f%%\", $rm_val / $lm * 100 }")
    fi
    printf "%-6s %10s %14s %8s\n" "$size" "$lm" "$rm_val" "$ratio"
  done
} | tee -a "$OUT"

echo ""
echo "Results written to $OUT"
echo "Full logs in $LOG"
```

Note: the local variable is named `rm_val` (not `rm`) to avoid shadowing the `rm` command under `set -u`.

---

## Task 11: Syntax-check + chmod + commit scripts

**Files:** the two new scripts + `lib/common.sh`.

### Step 1: Make executable and syntax-check

Run:
```bash
chmod +x drools-ansible-rulebook-integration-load-tests/load_test_temporal_HA-PG.sh \
         drools-ansible-rulebook-integration-load-tests/load_test_failover_HA-PG.sh

for f in drools-ansible-rulebook-integration-load-tests/load_test_temporal_HA-PG.sh \
         drools-ansible-rulebook-integration-load-tests/load_test_failover_HA-PG.sh; do
  bash -n "$f" && echo "$f OK"
done
```
Expected: `… OK` line for each script (no syntax errors).

### Step 2: Commit

```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x add \
    drools-ansible-rulebook-integration-load-tests/lib/common.sh \
    drools-ansible-rulebook-integration-load-tests/load_test_temporal_HA-PG.sh \
    drools-ansible-rulebook-integration-load-tests/load_test_failover_HA-PG.sh

git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x commit -m "$(cat <<'EOF'
feat(load-tests): add temporal + failover HA-PG scripts

Two new no-args multi-size (100/500/1k) HA-PG-only scripts:

  load_test_temporal_HA-PG.sh: 3 JVM invocations. Per-size row shows
  memory, time, per-event(KB), MATCHING row count, blob size. Stress
  test for once_within HA persistence under rapid ingress.

  load_test_failover_HA-PG.sh: 6 JVM invocations (3 sizes x 2 phases).
  Phase 1 loads events under a per-size ha-uuid; Phase 2 cold-starts
  and times engine.enableLeader() recovery against the same ha-uuid.
  Main table + recovery-ratio summary table. Reuses retention_*
  payloads.

lib/common.sh gains size_to_int for the 100/500/1k -> int mapping.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Verify:
```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x log --oneline -4
```
Expected: the 4 new commits (Java surface, pretty-print reformat, temporal payloads, scripts + common.sh), topped by `reorganize-load-test`'s existing history.

---

## Task 12: End-to-end smoke test (no commit)

Verify that both scripts actually work. This is an acceptance gate before declaring the task done.

### Step 1: Fresh fat-jar build

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am package -DskipTests
```
Expected: **BUILD SUCCESS**. The fat jar under `drools-ansible-rulebook-integration-load-tests/target/` has a very recent mtime.

### Step 2: Run `load_test_temporal_HA-PG.sh`

Run (from module root — `cd` into the module):
```bash
cd drools-ansible-rulebook-integration-load-tests
./load_test_temporal_HA-PG.sh
```
Expected:
- Script completes without shell errors.
- `result_temporal_HA-PG.txt` has a header, separator, and **three rows** (one per size). No row has `FAILED` in the Memory or Time columns.
- Each row's `MATCHING` column shows `10` (the designed fixed group count). If any row shows a different number, stop and investigate the rule shape / payload generation.
- `out_temporal_HA-PG.log` contains full stdout+stderr for all three runs.
- PG container is stopped automatically at EXIT.

Inspect:
```bash
cat result_temporal_HA-PG.txt
grep "^once_within" out_temporal_HA-PG.log | tail -3
```

### Step 3: Run `load_test_failover_HA-PG.sh`

Still inside the module root:
```bash
./load_test_failover_HA-PG.sh
```
Expected:
- Script completes without shell errors.
- `result_failover_HA-PG.txt` has the main table (6 rows — 3 sizes × Load/Recovery) followed by the Ratio table (3 rows). No `FAILED` values.
- Ratio values are reasonable (recovery time should be some positive % of load time — typical values 5%-100% depending on payload size; any value over 200% is surprising and worth investigating).
- `out_failover_HA-PG.log` contains all six JVM invocations.
- PG container stopped automatically.

Inspect:
```bash
cat result_failover_HA-PG.txt
grep "^retention_" out_failover_HA-PG.log | head -12
```

### Step 4: Spot-check the metric-line tags

Run:
```bash
grep "(failover-recovery)" out_failover_HA-PG.log | head -3
```
Expected: three lines, one per size, each of the form:
```
retention_<size>_events.json (HA-PG) (failover-recovery), <mem>, <time>
```

### Step 5: Report to the user

Surface any anomalies (FAILED rows, MATCHING != 10 for temporal, recovery-ratio outliers, exceptions in `out_*.log`) before declaring the task done. If everything passes, report the two output files generated and the total wall-clock time.

**No commit in this task** — smoke is an acceptance gate; no new files produced.
