# `drools-ansible-rulebook-integration-load-tests` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a greenfield Maven sub-module `drools-ansible-rulebook-integration-load-tests` that houses load-test infrastructure (fat-jar CLI, shell scripts, payload generator, memory-leak analyzer) isolated from `drools-ansible-rulebook-integration-main`.

**Architecture:** A new module sits next to `-main` in the reactor, depends only on `-runtime`/`-api` (same as `-main`), and builds a `jar-with-dependencies` fat jar. Small single-responsibility Java classes (`LoadTestMain`, `LoadRunner`, `HaLoadRunner`, `Measurement`, `MetricReporter`, `OutcomeCheck`, `PayloadGenerator`, ported `MemoryLeakAnalyzer`) replace the organically-grown `main/Main.java`. Four shell scripts share prefixed helpers from `lib/common.sh`. Every scenario covers noHA and HA-PG.

**Tech Stack:** Java 17, Maven (maven-assembly-plugin for fat jar), JUnit 5 + AssertJ (unit tests), SLF4J simple (logging), Drools 9.105.0 transitively via `-runtime`. Bash + Docker + `postgres:15-alpine` at script-run time.

**Working branches (do not deviate):**
- **Project repo:** `/home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x` — use branch `reorganize-load-test` (cut from `2.0.x`). **Never commit directly to `2.0.x`.**
- **Workspace repo:** `/home/tkobayas/claude/public/drools-ansible-rulebook-integration-2.0.x` — branch `reorganize-load-test` is already checked out and carries this plan + the spec.

**Hard constraint:** Do not modify any file under `drools-ansible-rulebook-integration-main/`. The only permitted change outside the new module is one `<module>` line added to the root reactor `pom.xml`.

**Spec reference:** `specs/2026-04-21-load-tests-module-design.md` (in workspace). Read it before starting.

**Commit conventions:** Conventional-commits prefix (`feat:`, `chore:`, `test:` etc.). Every commit ends with:
```
Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
```

All git commands in the plan target the **project repo**. Use `git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x <cmd>` or run from that directory.

---

## Post-execution amendment — 2026-04-22

The plan below was executed end-to-end. Task 8's Step 5 smoke test surfaced a cross-cutting defect in Tasks 3–7 that the plan's task bodies do not anticipate:

- **Symptom:** `OutcomeCheck.verify` threw on every `24kb_*_events.json` match scenario even though the engine matched all events (`eventsMatched=1000` in session stats).
- **Root cause:** `Payload` (copied verbatim from `main/` in Task 3) skips accumulating matches when `discard_matched_events=true`, which all match scenarios set for memory reasons. The spec-designed `OutcomeCheck.verify(List<Map>, …)` therefore saw an empty list even when matches occurred.
- **Fix (commit `b6f1b338`):**
  - `PayloadRunner` tracks `matchCount` as a counter that increments regardless of `discardMatchedEvents`.
  - `Payload.execute` now returns `Payload.Execution` (wrapper holding `matches` list + `matchCount`).
  - `Measurement.TimedResult` gains `int matchCount`; `timeWork` takes `Supplier<Payload.Execution>`.
  - `OutcomeCheck.verify` signature changes to `(int matchCount, ExpectedOutcome, String eventsJson)`. The `NO_MATCH` error message drops the "first match" hint since the list is now authoritative for display only.
  - `LoadRunner` / `HaLoadRunner` pass `t.matchCount` to `OutcomeCheck`; still pass `t.matches` into `Result`.

If you are replaying this plan from scratch on a clean checkout, merge this fix into Task 3 (Payload) and Tasks 4, 5, 7 (OutcomeCheck, Measurement, LoadRunner/HaLoadRunner) code blocks before executing — don't copy verbatim. The spec (`specs/2026-04-21-load-tests-module-design.md`, §5.2, §5.3, §5.4, §5.6) has been updated to reflect the final signatures.

---

## Task 1: Cut `reorganize-load-test` in the project repo

**Files:** none yet. This task only manipulates branch state.

- [ ] **Step 1: Verify project repo is clean and on 2.0.x**

Run:
```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x status
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x branch --show-current
```
Expected: `working tree clean`; branch `2.0.x`. If not on `2.0.x` or there are uncommitted changes, STOP and surface to the user.

- [ ] **Step 2: Create and check out `reorganize-load-test`**

Run:
```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x checkout -b reorganize-load-test
```
Expected: `Switched to a new branch 'reorganize-load-test'`.

- [ ] **Step 3: Verify**

Run:
```bash
git -C /home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x branch --show-current
```
Expected: `reorganize-load-test`.

No commit in this task (branch cut only).

---

## Task 2: Maven module skeleton + reactor wiring

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/pom.xml`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/.keep`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/resources/simplelogger.properties`
- Create: `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/.keep`
- Modify: `pom.xml` (root reactor — one line added)

All paths from here on are relative to the project repo root: `/home/tkobayas/usr/work/eda-HA-PoC/load-test-organize/drools-ansible-rulebook-integration-2.0.x/`.

- [ ] **Step 1: Create `drools-ansible-rulebook-integration-load-tests/pom.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <parent>
    <artifactId>drools-ansible-rulebook-integration</artifactId>
    <groupId>org.drools</groupId>
    <version>2.0.0-SNAPSHOT</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>

  <artifactId>drools-ansible-rulebook-integration-load-tests</artifactId>

  <name>Drools :: Ansible Rulebook Integration :: Load Tests</name>

  <dependencies>
    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-simple</artifactId>
    </dependency>
    <dependency>
      <groupId>org.drools</groupId>
      <artifactId>drools-ansible-rulebook-integration-runtime</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>org.drools</groupId>
      <artifactId>drools-ansible-rulebook-integration-api</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter-api</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter-engine</artifactId>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <artifactId>maven-assembly-plugin</artifactId>
        <configuration>
          <finalName>drools-ansible-rulebook-integration-load-tests</finalName>
          <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
          </descriptorRefs>
          <archive>
            <manifest>
              <mainClass>org.drools.ansible.rulebook.integration.loadtests.LoadTestMain</mainClass>
            </manifest>
          </archive>
        </configuration>
        <executions>
          <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
              <goal>single</goal>
            </goals>
          </execution>
        </executions>
      </plugin>

      <plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <configuration>
          <argLine>-Xmx500m</argLine>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 2: Create empty package directories and a resources log config**

Create empty marker file `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/.keep` (empty file).

Create empty marker file `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/.keep` (empty file).

Create `drools-ansible-rulebook-integration-load-tests/src/main/resources/simplelogger.properties`:
```properties
org.slf4j.simpleLogger.defaultLogLevel=info
org.slf4j.simpleLogger.showDateTime=true
org.slf4j.simpleLogger.dateTimeFormat=yyyy-MM-dd HH:mm:ss.SSS
org.slf4j.simpleLogger.showThreadName=false
org.slf4j.simpleLogger.showLogName=false
```

- [ ] **Step 3: Add the new module to the root reactor**

Open `pom.xml` in the project root and add one line after the `-main` entry inside `<modules>`:

```xml
    <module>drools-ansible-rulebook-integration-protoextractor</module>
    <module>drools-ansible-rulebook-integration-api</module>
    <module>drools-ansible-rulebook-integration-runtime</module>
<!--    <module>drools-ansible-rulebook-integration-core-rest</module>-->
    <module>drools-ansible-rulebook-integration-benchmark</module>
    <module>drools-ansible-rulebook-integration-tests</module>
    <module>drools-ansible-rulebook-integration-main</module>
    <module>drools-ansible-rulebook-integration-load-tests</module>
    <module>drools-ansible-rulebook-integration-ha</module>
```

The only change to the root pom is inserting the single new `<module>` line between `-main` and `-ha`.

- [ ] **Step 4: Build-verify the empty module**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am install -DskipTests
```
Expected: `BUILD SUCCESS`. A (near-empty) jar and fat jar are produced under `drools-ansible-rulebook-integration-load-tests/target/`.

- [ ] **Step 5: Commit**

```bash
git add pom.xml drools-ansible-rulebook-integration-load-tests
git commit -m "$(cat <<'EOF'
chore: scaffold drools-ansible-rulebook-integration-load-tests module

Empty Maven module with pom, reactor entry, and resource-log config.
Follow-up commits add the Java classes, scripts, and generated resources.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Copy `Payload` into the new module

The new module can't reach `main`'s `Payload` (no dependency on `-main`, per the hard constraint). Copy it verbatim with the package header changed. No behavior change.

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Payload.java`

- [ ] **Step 1: Create the file**

Copy the full body of `drools-ansible-rulebook-integration-main/src/main/java/org/drools/ansible/rulebook/integration/main/Payload.java` into the new file. Change only:
- The `package` declaration from `org.drools.ansible.rulebook.integration.main` to `org.drools.ansible.rulebook.integration.loadtests`.

Nothing else: keep all imports, logic, `PayloadRunner` inner class, the `injectMetaUuid` helper, and the `SUPPORTED_SOURCE_NAMES` list byte-for-byte identical.

- [ ] **Step 2: Compile-verify**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am install -DskipTests
```
Expected: `BUILD SUCCESS`.

- [ ] **Step 3: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Payload.java
git commit -m "$(cat <<'EOF'
feat: copy Payload into load-tests module

Duplication with main/Payload is intentional — the new module cannot depend
on -main per the isolation constraint. Byte-identical body; only the package
declaration differs.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: `ExpectedOutcome` enum + `OutcomeCheck` (TDD)

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/ExpectedOutcome.java`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/OutcomeCheck.java`
- Test: `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/OutcomeCheckTest.java`

- [ ] **Step 1: Write the failing test**

Create `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/OutcomeCheckTest.java`:

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThatCode;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class OutcomeCheckTest {

    @Test
    void matchExpected_withMatches_passes() {
        List<Map> matches = List.of(Map.of("rule", "r1"));
        assertThatCode(() -> OutcomeCheck.verify(matches, ExpectedOutcome.MATCH, "24kb_1k_events.json"))
                .doesNotThrowAnyException();
    }

    @Test
    void matchExpected_withZeroMatches_throws() {
        List<Map> matches = List.of();
        assertThatThrownBy(() -> OutcomeCheck.verify(matches, ExpectedOutcome.MATCH, "24kb_1k_events.json"))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Expected at least one match but got 0")
                .hasMessageContaining("24kb_1k_events.json");
    }

    @Test
    void noMatchExpected_withZeroMatches_passes() {
        List<Map> matches = List.of();
        assertThatCode(() -> OutcomeCheck.verify(matches, ExpectedOutcome.NO_MATCH, "retention_100_events.json"))
                .doesNotThrowAnyException();
    }

    @Test
    void noMatchExpected_withSomeMatches_throws() {
        Map<String, String> firstMatch = Map.of("rule", "r-unexpected");
        List<Map> matches = List.of(firstMatch, Map.of("rule", "r2"));
        assertThatThrownBy(() -> OutcomeCheck.verify(matches, ExpectedOutcome.NO_MATCH, "24kb_1k_events_unmatch.json"))
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("Expected no matches but got 2")
                .hasMessageContaining("24kb_1k_events_unmatch.json")
                .hasMessageContaining("r-unexpected");
    }
}
```

- [ ] **Step 2: Run the test — expect FAIL**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: compile failure (`ExpectedOutcome` and `OutcomeCheck` do not exist yet).

- [ ] **Step 3: Create `ExpectedOutcome.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

public enum ExpectedOutcome {
    MATCH,
    NO_MATCH
}
```

- [ ] **Step 4: Create `OutcomeCheck.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.util.List;
import java.util.Map;

public final class OutcomeCheck {

    private OutcomeCheck() {}

    public static void verify(List<Map> matches, ExpectedOutcome expected, String eventsJson) {
        int count = matches.size();
        if (expected == ExpectedOutcome.MATCH && count == 0) {
            throw new RuntimeException(
                    "Expected at least one match but got 0 (events: " + eventsJson + ")");
        }
        if (expected == ExpectedOutcome.NO_MATCH && count > 0) {
            throw new RuntimeException(
                    "Expected no matches but got " + count
                            + " (events: " + eventsJson
                            + ", first match: " + matches.get(0) + ")");
        }
    }
}
```

- [ ] **Step 5: Run the test — expect PASS**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: `BUILD SUCCESS`, 4 tests run, 0 failures.

- [ ] **Step 6: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/ExpectedOutcome.java \
        drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/OutcomeCheck.java \
        drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/OutcomeCheckTest.java
git commit -m "$(cat <<'EOF'
feat: add ExpectedOutcome + OutcomeCheck

Pure, deterministic check used by LoadRunner/HaLoadRunner to fail loudly when
a match scenario produces zero matches or an unmatch/retention scenario
produces any match.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: `Measurement` (TDD)

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Measurement.java`
- Test: `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/MeasurementTest.java`

- [ ] **Step 1: Write the failing test**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.util.List;
import java.util.Map;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MeasurementTest {

    @Test
    void timeWork_returnsMatchesAndNonNegativeDuration() {
        List<Map> stubMatches = List.of(Map.of("k", "v"));
        Measurement.TimedResult t = Measurement.timeWork(() -> {
            // simulate a little work
            try {
                Thread.sleep(5);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            return stubMatches;
        });
        assertThat(t.matches).isSameAs(stubMatches);
        assertThat(t.durationMs).isGreaterThanOrEqualTo(0L);
    }

    @Test
    void captureUsedMemoryAfterGc_returnsPositive() {
        // After running System.gc(), used = totalMemory - freeMemory; must be > 0
        // on a real JVM (classloader, test runner, JIT scaffolding are all alive).
        long used = Measurement.captureUsedMemoryAfterGc();
        assertThat(used).isGreaterThan(0L);
    }
}
```

- [ ] **Step 2: Run — expect FAIL (compile error)**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: compile failure (`Measurement` undefined).

- [ ] **Step 3: Create `Measurement.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.function.Supplier;

public final class Measurement {

    private Measurement() {}

    public static TimedResult timeWork(Supplier<List<Map>> work) {
        Instant start = Instant.now();
        List<Map> matches = work.get();
        long durationMs = Duration.between(start, Instant.now()).toMillis();
        return new TimedResult(matches, durationMs);
    }

    public static long captureUsedMemoryAfterGc() {
        System.gc();
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
        System.gc();
        Runtime r = Runtime.getRuntime();
        return r.totalMemory() - r.freeMemory();
    }

    public static final class TimedResult {
        public final List<Map> matches;
        public final long durationMs;

        public TimedResult(List<Map> matches, long durationMs) {
            this.matches = matches;
            this.durationMs = durationMs;
        }
    }
}
```

- [ ] **Step 4: Run — expect PASS**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 5: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Measurement.java \
        drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/MeasurementTest.java
git commit -m "$(cat <<'EOF'
feat: add Measurement (timeWork + captureUsedMemoryAfterGc)

Two responsibilities split into two methods so the outcome check can run
between them: time the work, check matches, then capture used memory after
the GC dance.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: `MetricReporter` (TDD)

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporter.java`
- Test: `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporterTest.java`

- [ ] **Step 1: Write the failing test**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.nio.charset.StandardCharsets;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class MetricReporterTest {

    @Test
    void reportsNoHaLine() {
        ByteArrayOutputStream buf = new ByteArrayOutputStream();
        try (PrintStream ps = new PrintStream(buf, true, StandardCharsets.UTF_8)) {
            MetricReporter.report(ps, "24kb_1k_events.json", false, 5_200_000L, 195L);
        }
        assertThat(buf.toString(StandardCharsets.UTF_8).trim())
                .isEqualTo("24kb_1k_events.json, 5200000, 195");
    }

    @Test
    void reportsHaPgLine() {
        ByteArrayOutputStream buf = new ByteArrayOutputStream();
        try (PrintStream ps = new PrintStream(buf, true, StandardCharsets.UTF_8)) {
            MetricReporter.report(ps, "24kb_1k_events.json", true, 7_100_000L, 240L);
        }
        assertThat(buf.toString(StandardCharsets.UTF_8).trim())
                .isEqualTo("24kb_1k_events.json (HA-PG), 7100000, 240");
    }
}
```

- [ ] **Step 2: Run — expect FAIL (compile error)**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: compile failure.

- [ ] **Step 3: Create `MetricReporter.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.io.PrintStream;

public final class MetricReporter {

    private MetricReporter() {}

    public static void report(PrintStream err, String eventsJson, boolean haPg,
                              long usedMemoryBytes, long timeMs) {
        StringBuilder sb = new StringBuilder();
        sb.append(eventsJson);
        if (haPg) {
            sb.append(" (HA-PG)");
        }
        sb.append(", ").append(usedMemoryBytes);
        sb.append(", ").append(timeMs);
        err.println(sb.toString());
    }
}
```

- [ ] **Step 4: Run — expect PASS**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: `BUILD SUCCESS`, all tests pass.

- [ ] **Step 5: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporter.java \
        drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/MetricReporterTest.java
git commit -m "$(cat <<'EOF'
feat: add MetricReporter

Writes the single stderr metric line consumed by bash fmt_parse_metrics.
Regression-tested because this is the Java/bash contract.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: `LoadRunner`, `HaLoadRunner`, `LoadTestMain`, `Result`

No unit tests for these — they pull the full `AstRulesEngine` and DB. Exercised by the shell scripts later. Compile-verify and commit.

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Result.java`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadRunner.java`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaLoadRunner.java`
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadTestMain.java`

- [ ] **Step 1: Create `Result.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.util.List;
import java.util.Map;

public final class Result {

    public final List<Map> matches;
    public final long durationMs;
    public final long usedMemoryBytes;

    public Result(List<Map> matches, long durationMs, long usedMemoryBytes) {
        this.matches = matches;
        this.durationMs = durationMs;
        this.usedMemoryBytes = usedMemoryBytes;
    }
}
```

- [ ] **Step 2: Create `LoadRunner.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.util.Map;

import org.drools.ansible.rulebook.integration.api.domain.RulesSet;
import org.drools.ansible.rulebook.integration.core.jpy.AstRulesEngine;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class LoadRunner {

    private static final Logger LOGGER = LoggerFactory.getLogger(LoadRunner.class);

    private LoadRunner() {}

    public static Result run(RulesSet rulesSet, Map rulesSetMap,
                             ExpectedOutcome expected, String eventsJson) {
        try (AstRulesEngine engine = new AstRulesEngine()) {
            long id = engine.createRuleset(rulesSet);

            LOGGER.info("*** Start measuring execution time");
            Measurement.TimedResult t = Measurement.timeWork(() -> {
                Payload payload = Payload.parsePayload(rulesSetMap);
                return payload.execute(engine, id);
            });
            LOGGER.info("*** End measuring execution time, duration = {} ms", t.durationMs);

            OutcomeCheck.verify(t.matches, expected, eventsJson);

            String stats = engine.sessionStats(id);
            LOGGER.info(stats);

            long mem = Measurement.captureUsedMemoryAfterGc();

            return new Result(t.matches, t.durationMs, mem);
        }
    }
}
```

Why Payload is created inside the `timeWork` lambda: the lambda scope is the sole strong reference to the Payload. Once `timeWork` returns, the lambda is unreachable and so is Payload (`t.matches` contains fresh Jackson maps produced by `Payload.execute` — no back-reference to Payload internals). This lets `captureUsedMemoryAfterGc` reclaim Payload's event-list without any `payload = null` dance and without LoadTestMain holding a competing reference.

- [ ] **Step 3: Create `HaLoadRunner.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.io.IOException;
import java.net.Socket;
import java.util.Map;

import org.drools.ansible.rulebook.integration.api.domain.RulesSet;
import org.drools.ansible.rulebook.integration.core.jpy.AstRulesEngine;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class HaLoadRunner {

    private static final Logger LOGGER = LoggerFactory.getLogger(HaLoadRunner.class);

    private HaLoadRunner() {}

    public static Result runLoad(RulesSet rulesSet, String rulesetJson, Map rulesSetMap,
                                 String haDbParamsJson, ExpectedOutcome expected, String eventsJson) {
        try (AstRulesEngine engine = new AstRulesEngine()) {
            String haUuid = "loadtest-ha-" + System.currentTimeMillis();
            engine.initializeHA(haUuid, "loadtest-worker", haDbParamsJson, "{\"write_after\":1}");

            long id = engine.createRuleset(rulesSet, rulesetJson);
            int port = engine.port();

            Socket haSocket;
            try {
                haSocket = new Socket("localhost", port);
            } catch (IOException e) {
                throw new RuntimeException("Failed to connect HA socket", e);
            }

            try {
                engine.enableLeader();

                LOGGER.info("*** Start measuring execution time");
                Measurement.TimedResult t = Measurement.timeWork(() -> {
                    Payload payload = Payload.parsePayload(rulesSetMap);
                    return payload.execute(engine, id);
                });
                LOGGER.info("*** End measuring execution time, duration = {} ms", t.durationMs);

                OutcomeCheck.verify(t.matches, expected, eventsJson);

                String stats = engine.sessionStats(id);
                LOGGER.info(stats);

                long mem = Measurement.captureUsedMemoryAfterGc();

                return new Result(t.matches, t.durationMs, mem);
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

Same Payload-in-lambda discipline as `LoadRunner`.

- [ ] **Step 4: Create `LoadTestMain.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.drools.ansible.rulebook.integration.api.RuleFormat;
import org.drools.ansible.rulebook.integration.api.RuleNotation;
import org.drools.ansible.rulebook.integration.api.domain.RulesSet;
import org.drools.ansible.rulebook.integration.api.io.JsonMapper;

public final class LoadTestMain {

    private static final String DEFAULT_JSON = "24kb_1k_events.json";

    private LoadTestMain() {}

    public static void main(String[] args) {
        String haDbParamsJson = null;
        List<String> positional = new ArrayList<>();
        for (int i = 0; i < args.length; i++) {
            if ("--ha-db-params".equals(args[i])) {
                if (i + 1 >= args.length) {
                    System.err.println("ERROR: --ha-db-params requires a JSON argument");
                    System.exit(1);
                }
                haDbParamsJson = args[++i];
            } else {
                positional.add(args[i]);
            }
        }
        String eventsJson = positional.isEmpty() ? DEFAULT_JSON : positional.get(0);

        // Outcome is derived from the filename convention:
        //   contains "unmatch"          -> NO_MATCH
        //   starts with "retention_"    -> NO_MATCH
        //   otherwise                   -> MATCH
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
        Result result = haPg
                ? HaLoadRunner.runLoad(rulesSet, rulesetJson, rulesSetMap, haDbParamsJson, expected, eventsJson)
                : LoadRunner.run(rulesSet, rulesSetMap, expected, eventsJson);

        MetricReporter.report(System.err, eventsJson, haPg, result.usedMemoryBytes, result.durationMs);
    }

    private static String readRulesJson(String name) {
        try (InputStream is = LoadTestMain.class.getClassLoader().getResourceAsStream(name)) {
            if (is != null) {
                return new String(is.readAllBytes(), StandardCharsets.UTF_8);
            }
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        try (InputStream is = new FileInputStream(name)) {
            return new String(is.readAllBytes(), StandardCharsets.UTF_8);
        } catch (FileNotFoundException e) {
            throw new RuntimeException("Rules JSON not found on classpath or filesystem: " + name, e);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- [ ] **Step 5: Build-verify**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am install
```
Expected: `BUILD SUCCESS`. Tests still pass (the three existing unit tests). A fat jar `drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar` is produced under `target/`.

- [ ] **Step 6: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/Result.java \
        drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadRunner.java \
        drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/HaLoadRunner.java \
        drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/LoadTestMain.java
git commit -m "$(cat <<'EOF'
feat: add LoadTestMain + LoadRunner + HaLoadRunner + Result

CLI entry plus the two execution paths. LoadTestMain parses --ha-db-params,
derives ExpectedOutcome from the filename, and delegates to the appropriate
runner. Both runners invoke OutcomeCheck between timeWork and the memory
capture so semantic failures never emit a metric line.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: `PayloadGenerator` + generated JSON resources

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/PayloadGenerator.java`
- Create: 11 files under `drools-ansible-rulebook-integration-load-tests/src/main/resources/`.

- [ ] **Step 1: Create `PayloadGenerator.java`**

```java
package org.drools.ansible.rulebook.integration.loadtests.gen;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;

import org.drools.ansible.rulebook.integration.api.io.JsonMapper;

/**
 * Generates the 11 test-event JSON files under src/main/resources/.
 * Deterministic: the only randomness is a fixed-seed word shuffle used to
 * build the bulky message field. Run manually:
 *
 *   mvn -pl drools-ansible-rulebook-integration-load-tests exec:java \
 *       -Dexec.mainClass=org.drools.ansible.rulebook.integration.loadtests.gen.PayloadGenerator \
 *       -Dexec.classpathScope=compile
 */
public final class PayloadGenerator {

    private static final long SEED = 42L;
    private static final int TARGET_EVENT_BYTES = 24_000;

    // ~200 words, rough "ops/log chatter" vocabulary. Deliberately bland.
    private static final List<String> WORDS = List.of(
            "request", "response", "latency", "backoff", "retry", "timeout",
            "connection", "pool", "saturated", "drained", "stale", "fresh",
            "cache", "hit", "miss", "eviction", "lease", "expired",
            "cluster", "node", "replica", "leader", "follower", "quorum",
            "partition", "offset", "commit", "rollback", "checkpoint",
            "handler", "listener", "worker", "thread", "queue", "dispatch",
            "event", "payload", "message", "record", "batch", "stream",
            "source", "sink", "topic", "channel", "subscriber", "publisher",
            "rule", "condition", "action", "match", "fire", "evaluate",
            "session", "state", "persistence", "recovery", "failover", "replay",
            "memory", "leak", "allocation", "gc", "heap", "offheap",
            "cpu", "throttle", "saturation", "contention", "lock", "unlock",
            "deadlock", "starvation", "fairness", "priority", "queue",
            "disk", "buffer", "flush", "sync", "fsync", "journal",
            "database", "query", "transaction", "isolation", "snapshot",
            "backend", "frontend", "service", "endpoint", "token", "session",
            "authentication", "authorization", "role", "policy", "rule",
            "metric", "counter", "gauge", "histogram", "summary", "percentile",
            "trace", "span", "correlation", "context", "propagation",
            "probe", "readiness", "liveness", "health", "ping", "pong",
            "socket", "port", "bind", "listen", "accept", "close",
            "decode", "encode", "serialize", "deserialize", "marshal", "unmarshal",
            "shard", "rebalance", "promote", "demote", "drain", "detach",
            "attach", "mount", "unmount", "rotate", "archive", "purge",
            "apply", "revert", "restore", "snapshot", "migrate", "backfill",
            "scheduler", "trigger", "tick", "cron", "job", "task",
            "observed", "expected", "threshold", "breach", "alarm", "warning",
            "info", "debug", "trace", "notice", "critical", "fatal",
            "version", "release", "build", "sha", "branch", "tag",
            "configuration", "parameter", "override", "default", "profile",
            "region", "zone", "cluster", "namespace", "environment", "tier",
            "quiesce", "resume", "pause", "continue", "abort", "cancel"
    );

    private PayloadGenerator() {}

    public static void main(String[] args) throws IOException {
        Path resourcesDir = Paths.get("drools-ansible-rulebook-integration-load-tests/src/main/resources");
        if (!Files.isDirectory(resourcesDir)) {
            // Fallback when exec:java runs from the module root
            resourcesDir = Paths.get("src/main/resources");
        }
        if (!Files.isDirectory(resourcesDir)) {
            throw new IOException("Cannot locate src/main/resources (run from project root or module root)");
        }

        Map<String, Object> event = buildEvent();

        for (String size : List.of("1k", "10k", "100k", "1m")) {
            int repeatCount = sizeToRepeatCount(size);
            write(resourcesDir.resolve("24kb_" + size + "_events.json"),
                    matchRuleset("24kb " + size + " events", event, repeatCount));
            write(resourcesDir.resolve("24kb_" + size + "_events_unmatch.json"),
                    unmatchRuleset("24kb " + size + " events unmatch", event, repeatCount));
        }

        for (int n : new int[] { 100, 500, 1000 }) {
            String label = n == 1000 ? "1k" : String.valueOf(n);
            write(resourcesDir.resolve("retention_" + label + "_events.json"),
                    retentionRuleset("retention " + label + " events", event, n));
        }

        System.out.println("Wrote 11 payload JSON files to " + resourcesDir.toAbsolutePath());
    }

    private static int sizeToRepeatCount(String size) {
        switch (size) {
            case "1k":   return 1_000;
            case "10k":  return 10_000;
            case "100k": return 100_000;
            case "1m":   return 1_000_000;
            default: throw new IllegalArgumentException(size);
        }
    }

    private static Map<String, Object> buildEvent() {
        LinkedHashMap<String, Object> event = new LinkedHashMap<>();
        event.put("event_id", "evt-0001");
        event.put("timestamp", "2026-04-21T10:15:30.123Z");

        LinkedHashMap<String, Object> source = new LinkedHashMap<>();
        source.put("host", "host-1");
        source.put("component", "ansible-rulebook");
        source.put("region", "us-east-1");
        source.put("cluster", "cluster-1");
        event.put("source", source);

        event.put("severity", "INFO");
        event.put("tags", List.of("eda", "rulebook", "load-test", "synthetic"));

        LinkedHashMap<String, Object> labels = new LinkedHashMap<>();
        labels.put("env", "prod");
        labels.put("team", "platform");
        labels.put("tier", "critical");
        labels.put("batch", "1");
        event.put("labels", labels);

        LinkedHashMap<String, Object> metrics = new LinkedHashMap<>();
        metrics.put("cpu_pct", 42.7);
        metrics.put("mem_mb", 1337);
        metrics.put("disk_io_mb", 88.2);
        metrics.put("net_rx_kb", 1024);
        metrics.put("net_tx_kb", 2048);
        metrics.put("req_count", 17);
        metrics.put("latency_ms_p50", 12);
        metrics.put("latency_ms_p95", 87);
        metrics.put("latency_ms_p99", 201);
        event.put("metrics", metrics);

        LinkedHashMap<String, Object> trace = new LinkedHashMap<>();
        trace.put("trace_id", "0123456789abcdef0123456789abcdef");
        trace.put("span_id", "fedcba9876543210");
        trace.put("parent_span_id", "89abcdef01234567");
        event.put("trace", trace);

        event.put("message", buildMessage(event));
        event.put("a", 1);
        return event;
    }

    private static String buildMessage(Map<String, Object> eventWithoutMessage) {
        Random rnd = new Random(SEED);
        List<String> shuffled = new ArrayList<>(WORDS);
        Collections.shuffle(shuffled, rnd);

        // Serialize a working copy with an empty "message" so we know the
        // fixed overhead; fill message with sentences until serialized size
        // reaches TARGET_EVENT_BYTES.
        Map<String, Object> probe = new LinkedHashMap<>(eventWithoutMessage);
        probe.put("message", "");
        int overhead = JsonMapper.toJson(probe).length();
        int targetMessageLen = Math.max(0, TARGET_EVENT_BYTES - overhead);

        StringBuilder sb = new StringBuilder(targetMessageLen + 128);
        int wordIdx = 0;
        int sentenceWordBudget = 0;
        while (sb.length() < targetMessageLen) {
            if (sentenceWordBudget == 0) {
                if (sb.length() > 0) sb.append(". ");
                sentenceWordBudget = 6 + rnd.nextInt(10); // 6-15 words/sentence
            }
            sb.append(shuffled.get(wordIdx % shuffled.size()));
            wordIdx++;
            sentenceWordBudget--;
            if (sentenceWordBudget > 0) sb.append(' ');
        }
        // Truncate at last whitespace so the message is readable.
        int lastSpace = sb.lastIndexOf(" ");
        if (lastSpace > 0 && sb.length() - lastSpace < 40) {
            sb.setLength(lastSpace);
        }
        return sb.toString();
    }

    private static Map<String, Object> matchRuleset(String name, Map<String, Object> event, int repeatCount) {
        return buildRuleset(name, event, repeatCount,
                buildCondition("event.a == 1"),
                /* discardMatchedEvents= */ true);
    }

    private static Map<String, Object> unmatchRuleset(String name, Map<String, Object> event, int repeatCount) {
        return buildRuleset(name, event, repeatCount,
                buildCondition("event.a == 2"),
                /* discardMatchedEvents= */ true);
    }

    private static Map<String, Object> retentionRuleset(String name, Map<String, Object> event, int repeatCount) {
        // 2-condition join: condition 1 matches (event.a == 1), condition 2 never matches (event.b == 1).
        // Partial matches accumulate, the rule never fires.
        LinkedHashMap<String, Object> condA = buildCondition("event.a == 1");
        LinkedHashMap<String, Object> condB = buildCondition("event.b == 1");
        LinkedHashMap<String, Object> all = new LinkedHashMap<>();
        all.put("AllCondition", List.of(condA, condB));
        return buildRulesetWithRawCondition(name, event, repeatCount, all,
                /* discardMatchedEvents= */ false);
    }

    private static LinkedHashMap<String, Object> buildCondition(String expression) {
        LinkedHashMap<String, Object> cond = new LinkedHashMap<>();
        cond.put("EqualsExpression", expressionToEqualsNode(expression));
        return cond;
    }

    private static Map<String, Object> expressionToEqualsNode(String expression) {
        // Parses "event.a == 1" (or == 2) and returns the EqualsExpression body.
        String[] parts = expression.split("==");
        String lhs = parts[0].trim();
        String rhs = parts[1].trim();
        LinkedHashMap<String, Object> eq = new LinkedHashMap<>();
        LinkedHashMap<String, Object> lhsNode = new LinkedHashMap<>();
        lhsNode.put("Event", lhs.substring("event.".length()));
        eq.put("lhs", lhsNode);
        LinkedHashMap<String, Object> rhsNode = new LinkedHashMap<>();
        rhsNode.put("Integer", Integer.parseInt(rhs));
        eq.put("rhs", rhsNode);
        return eq;
    }

    private static Map<String, Object> buildRuleset(String name, Map<String, Object> event, int repeatCount,
                                                    Map<String, Object> condition, boolean discardMatchedEvents) {
        return buildRulesetWithRawCondition(name, event, repeatCount, wrapAllCondition(condition), discardMatchedEvents);
    }

    private static Map<String, Object> wrapAllCondition(Map<String, Object> condition) {
        LinkedHashMap<String, Object> all = new LinkedHashMap<>();
        all.put("AllCondition", List.of(condition));
        return all;
    }

    private static Map<String, Object> buildRulesetWithRawCondition(String name, Map<String, Object> event, int repeatCount,
                                                                    Map<String, Object> condition, boolean discardMatchedEvents) {
        LinkedHashMap<String, Object> rule = new LinkedHashMap<>();
        LinkedHashMap<String, Object> ruleBody = new LinkedHashMap<>();
        ruleBody.put("name", "r1");
        ruleBody.put("condition", condition);
        LinkedHashMap<String, Object> action = new LinkedHashMap<>();
        LinkedHashMap<String, Object> debug = new LinkedHashMap<>();
        debug.put("action", "debug");
        debug.put("action_args", new LinkedHashMap<>());
        action.put("Action", debug);
        ruleBody.put("action", action);
        ruleBody.put("enabled", true);
        rule.put("Rule", ruleBody);

        LinkedHashMap<String, Object> sourceArgs = new LinkedHashMap<>();
        sourceArgs.put("discard_matched_events", discardMatchedEvents);
        sourceArgs.put("repeat_count", repeatCount);
        sourceArgs.put("payload", List.of(event));

        LinkedHashMap<String, Object> eventSource = new LinkedHashMap<>();
        eventSource.put("name", "generic");
        eventSource.put("source_name", "generic");
        eventSource.put("source_args", sourceArgs);

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

    private static void write(Path path, Map<String, Object> content) throws IOException {
        String json = JsonMapper.toJson(List.of(content));
        Files.writeString(path, json, StandardCharsets.UTF_8);
        System.out.println("  wrote " + path + " (" + json.length() + " bytes)");
    }
}
```

IMPORTANT — before running the generator, eyeball the **existing** `drools-ansible-rulebook-integration-main/src/main/resources/24kb_1k_events.json` to confirm the EqualsExpression JSON shape the engine actually accepts. If the existing format uses different field names (e.g., `Event` vs `EventLookup`, `Integer` vs `Number`), align `expressionToEqualsNode` and the retention `AllCondition` builder accordingly before running the generator. Use `Read` on the existing file and compare.

- [ ] **Step 2: Verify shape of existing resources and adjust generator if needed**

Read one match file, one unmatch file, and one failover (retention-like) file from `drools-ansible-rulebook-integration-main/src/main/resources/` and confirm the AST node names used for conditions. If the generator's shape differs, edit `expressionToEqualsNode`, `wrapAllCondition`, and `retentionRuleset` to match exactly. Fail fast here — do not proceed until the generator emits a RuleSet whose structure parses through `RuleNotation.CoreNotation.INSTANCE.toRulesSet`.

Quick sanity probe (does not require running the generator):
```bash
head -80 drools-ansible-rulebook-integration-main/src/main/resources/24kb_1k_events.json
head -80 drools-ansible-rulebook-integration-main/src/main/resources/failover_100_events.json
```

- [ ] **Step 3: Build (compiles the generator into `target/classes`)**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am install -DskipTests
```

- [ ] **Step 4: Run the generator**

Run from the project root:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests exec:java \
    -Dexec.mainClass=org.drools.ansible.rulebook.integration.loadtests.gen.PayloadGenerator \
    -Dexec.classpathScope=compile
```
Expected: 11 `wrote ...` lines plus the final summary. Inspect the output directory:
```bash
ls -l drools-ansible-rulebook-integration-load-tests/src/main/resources/
```
Expected: 11 JSON files plus `simplelogger.properties`. Each JSON file should be on the order of 26KB (same ballpark as the existing `main` resources).

- [ ] **Step 5: Parse-verify at runtime**

Run a quick smoke test: invoke `LoadTestMain` with a small file and without HA. This will validate the generated JSON is well-formed and that the whole pipeline (read → RulesSet → Payload → engine → MetricReporter) works.

```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am package -DskipTests
java -Xmx512m -Dorg.slf4j.simpleLogger.logFile=System.out \
     -jar drools-ansible-rulebook-integration-load-tests/target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar \
     24kb_1k_events.json
```
Expected: program completes without exception; final stderr line matches `24kb_1k_events.json, <bytes>, <ms>`. If the program fails (e.g., OutcomeCheck throws), the RuleSet or Payload shape in the generator needs fixing — go back to Step 2 and revise.

- [ ] **Step 6: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/gen/PayloadGenerator.java \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_1k_events.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_10k_events.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_100k_events.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_1m_events.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_1k_events_unmatch.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_10k_events_unmatch.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_100k_events_unmatch.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/24kb_1m_events_unmatch.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/retention_100_events.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/retention_500_events.json \
        drools-ansible-rulebook-integration-load-tests/src/main/resources/retention_1k_events.json
git commit -m "$(cat <<'EOF'
feat: add PayloadGenerator and committed 24KB event payloads

Generator is the source of truth for the 11 committed resource files: 8
match/unmatch variants at 1k/10k/100k/1m and 3 retention variants at
100/500/1k. Template event is multi-field (event_id, source, metrics,
trace, message, labels) with a ~22KB message body built from a fixed-seed
word shuffle — deterministic across runs, readable in diffs.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Port + enhance `MemoryLeakAnalyzer` with tests (TDD-ish)

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/analyze/MemoryLeakAnalyzer.java`
- Test: `drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/analyze/MemoryLeakAnalyzerTest.java`

- [ ] **Step 1: Write the failing test**

```java
package org.drools.ansible.rulebook.integration.loadtests.analyze;

import java.nio.file.Files;
import java.nio.file.Path;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import static org.assertj.core.api.Assertions.assertThat;

class MemoryLeakAnalyzerTest {

    @Test
    void cleanAcrossAllFourGroups_reportsNoLeak(@TempDir Path tmp) throws Exception {
        Path f = tmp.resolve("result.txt");
        Files.writeString(f, String.join("\n",
                // match / noHA
                "24kb_1k_events.json, 5200000, 100",
                "24kb_10k_events.json, 5210000, 300",
                "24kb_100k_events.json, 5225000, 1800",
                "24kb_1m_events.json, 5280000, 17000",
                // match / HA-PG
                "24kb_1k_events.json (HA-PG), 6100000, 180",
                "24kb_10k_events.json (HA-PG), 6130000, 600",
                "24kb_100k_events.json (HA-PG), 6175000, 3500",
                "24kb_1m_events.json (HA-PG), 6240000, 32000",
                // unmatch / noHA
                "24kb_1k_events_unmatch.json, 5100000, 80",
                "24kb_10k_events_unmatch.json, 5115000, 250",
                "24kb_100k_events_unmatch.json, 5135000, 1500",
                "24kb_1m_events_unmatch.json, 5175000, 14000",
                // unmatch / HA-PG
                "24kb_1k_events_unmatch.json (HA-PG), 6000000, 140",
                "24kb_10k_events_unmatch.json (HA-PG), 6020000, 500",
                "24kb_100k_events_unmatch.json (HA-PG), 6050000, 2800",
                "24kb_1m_events_unmatch.json (HA-PG), 6100000, 26000"
        ));

        MemoryLeakAnalyzer.AnalyzeResult r = new MemoryLeakAnalyzer().analyzeFile(f.toString());

        assertThat(r.exceptionFound).isFalse();
        assertThat(r.hasLeak).isFalse();
    }

    @Test
    void hugeAbsoluteSpike_reportsLeak(@TempDir Path tmp) throws Exception {
        Path f = tmp.resolve("result.txt");
        Files.writeString(f, String.join("\n",
                "24kb_1k_events.json, 5000000, 100",
                "24kb_10k_events.json, 5010000, 300",
                "24kb_100k_events.json, 5020000, 1800",
                "24kb_1m_events.json, 300000000, 17000" // 295MB jump
        ));

        MemoryLeakAnalyzer.AnalyzeResult r = new MemoryLeakAnalyzer().analyzeFile(f.toString());

        assertThat(r.hasLeak).isTrue();
    }

    @Test
    void exceptionSubstringInResultFile_isFlagged(@TempDir Path tmp) throws Exception {
        Path f = tmp.resolve("result.txt");
        Files.writeString(f, String.join("\n",
                "24kb_1k_events.json, 5200000, 100",
                "RuntimeException at line 42",
                "24kb_10k_events.json, 5210000, 300"
        ));

        MemoryLeakAnalyzer.AnalyzeResult r = new MemoryLeakAnalyzer().analyzeFile(f.toString());

        assertThat(r.exceptionFound).isTrue();
    }
}
```

- [ ] **Step 2: Run — expect FAIL (compile error)**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: compile failure.

- [ ] **Step 3: Create `MemoryLeakAnalyzer.java`**

Port the body from `drools-ansible-rulebook-integration-main/src/main/java/org/drools/ansible/rulebook/integration/main/MemoryLeakAnalyzer.java` with these changes:

1. Package `org.drools.ansible.rulebook.integration.loadtests.analyze`.
2. Extract an `analyzeFile(String filename)` method returning an `AnalyzeResult` struct with `hasLeak` and `exceptionFound`. Keep `main` for CLI invocation; `main` calls `analyzeFile` and maps to `System.exit` codes.
3. Replace the two-group split (match/unmatch) with a four-way grouping: derive `haPg = testName.contains(" (HA-PG)")` and bucket into `match/noHA`, `match/HA-PG`, `unmatch/noHA`, `unmatch/HA-PG`. Run the same analysis on each bucket independently; any bucket flagging `hasLeak` sets the overall `hasLeak`.
4. Thresholds, sort-by-size, and text output are unchanged per the spec.

```java
package org.drools.ansible.rulebook.integration.loadtests.analyze;

import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/**
 * See spec section 5.8. Groups test results four ways
 * (match|unmatch) x (noHA|HA-PG), sorts each group by event count, applies the
 * same absolute-increase / consecutive-acceleration / total-increase thresholds
 * used in the main module, and flags a leak if any group trips a threshold.
 */
public class MemoryLeakAnalyzer {

    private static final double INCREASE_GROWTH_THRESHOLD = 3.0;
    private static final long ABSOLUTE_INCREASE_THRESHOLD = 50_000_000;

    public static void main(String[] args) {
        if (args.length != 1) {
            System.err.println("Usage: java MemoryLeakAnalyzer <result_file>");
            System.exit(1);
        }
        try {
            AnalyzeResult r = new MemoryLeakAnalyzer().analyzeFile(args[0]);
            if (r.hasLeak || r.exceptionFound) {
                System.err.println("\n❌ MEMORY LEAK DETECTED OR EXCEPTION FOUND!");
                System.err.println("  Review the result file for details.\n");
                System.exit(1);
            }
            System.out.println("\n✅ No memory leak detected.");
            System.exit(0);
        } catch (Exception e) {
            System.err.println("Error analyzing results: " + e.getMessage());
            e.printStackTrace();
            System.exit(2);
        }
    }

    public AnalyzeResult analyzeFile(String filename) throws IOException {
        ParseResult pr = parseResultFile(filename);
        boolean hasLeak = analyzeResults(pr.results);
        return new AnalyzeResult(hasLeak, pr.exceptionFound);
    }

    private ParseResult parseResultFile(String filename) throws IOException {
        List<TestResult> results = new ArrayList<>();
        boolean exceptionFound = false;

        try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
            String line;
            while ((line = reader.readLine()) != null) {
                line = line.trim();
                if (line.isEmpty()) continue;
                if (line.contains("Exception") || line.contains("exception")) {
                    exceptionFound = true;
                }
                String[] parts = line.split(",");
                if (parts.length == 3) {
                    try {
                        String testName = parts[0].trim();
                        long mem = Long.parseLong(parts[1].trim());
                        long dur = Long.parseLong(parts[2].trim());
                        results.add(new TestResult(testName, mem, dur));
                    } catch (NumberFormatException ignored) {
                    }
                }
            }
        }

        if (exceptionFound) {
            System.err.println("\n⚠️  EXCEPTION FOUND IN RESULTS (potentially caused by a memory leak)\n");
        }

        if (results.isEmpty()) {
            throw new IOException("No valid test results found in file. Please check the result file format.");
        }
        return new ParseResult(results, exceptionFound);
    }

    private boolean analyzeResults(List<TestResult> results) {
        System.out.println("Memory Leak Analysis Report");
        System.out.println("===========================\n");

        Map<String, List<TestResult>> groups = new LinkedHashMap<>();
        groups.put("match/noHA", new ArrayList<>());
        groups.put("match/HA-PG", new ArrayList<>());
        groups.put("unmatch/noHA", new ArrayList<>());
        groups.put("unmatch/HA-PG", new ArrayList<>());

        for (TestResult r : results) {
            boolean unmatch = r.testName.contains("unmatch");
            boolean haPg = r.testName.contains(" (HA-PG)");
            String key = (unmatch ? "unmatch" : "match") + "/" + (haPg ? "HA-PG" : "noHA");
            groups.get(key).add(r);
        }

        boolean hasLeak = false;
        for (Map.Entry<String, List<TestResult>> entry : groups.entrySet()) {
            List<TestResult> tests = entry.getValue();
            if (tests.isEmpty()) continue;
            System.out.println(entry.getKey() + ":");
            hasLeak |= analyzeTestGroup(tests);
            System.out.println();
        }
        return hasLeak;
    }

    private boolean analyzeTestGroup(List<TestResult> tests) {
        if (tests.isEmpty()) return false;

        tests.sort((a, b) -> Integer.compare(extractEventCount(a.testName), extractEventCount(b.testName)));

        System.out.println("Test Name                                 Memory (bytes)    Duration (ms)");
        System.out.println("------------------------------------------------------------------------");
        for (TestResult t : tests) {
            System.out.printf("%-41s %,13d    %,12d%n", t.testName, t.memoryUsage, t.duration);
        }

        System.out.println("\nMemory Increase Analysis:");
        boolean hasLeak = false;
        Long previousIncrease = null;
        int consecutive = 0;

        for (int i = 1; i < tests.size(); i++) {
            TestResult prev = tests.get(i - 1);
            TestResult curr = tests.get(i);
            long increase = curr.memoryUsage - prev.memoryUsage;

            System.out.printf("  %s → %s:%n",
                    formatEventCount(extractEventCount(prev.testName)),
                    formatEventCount(extractEventCount(curr.testName)));
            System.out.printf("    Memory increase: %,d bytes", increase);

            if (curr.memoryUsage == 0) {
                System.out.printf(" ⚠️  TEST FAILED (likely due to memory threshold)!%n");
                hasLeak = true;
            } else if (Math.abs(increase) > ABSOLUTE_INCREASE_THRESHOLD) {
                System.out.printf(" ⚠️  LARGE INCREASE!%n");
                hasLeak = true;
            } else if (previousIncrease != null && increase > 0 && previousIncrease > 0) {
                double ratio = (double) increase / previousIncrease;
                System.out.printf(" (%.2fx previous increase)", ratio);
                if (ratio > INCREASE_GROWTH_THRESHOLD) {
                    consecutive++;
                    if (consecutive >= 2) {
                        System.out.printf(" ⚠️  CONSECUTIVE ACCELERATING GROWTH!%n");
                        hasLeak = true;
                    } else {
                        System.out.printf(" ⚠ (noted, but not a leak if it's not consecutive)%n");
                    }
                } else {
                    consecutive = 0;
                    System.out.printf(" ✓%n");
                }
            } else {
                consecutive = 0;
                System.out.printf(" ✓%n");
            }
            previousIncrease = increase;
        }

        if (tests.size() >= 2) {
            long total = tests.get(tests.size() - 1).memoryUsage - tests.get(0).memoryUsage;
            System.out.printf("\nTotal memory increase (first → last): %,d bytes", total);
            if (total > ABSOLUTE_INCREASE_THRESHOLD * 3) {
                System.out.printf(" ⚠️  EXCESSIVE TOTAL INCREASE!%n");
                hasLeak = true;
            } else {
                System.out.printf(" ✓%n");
            }
        }
        return hasLeak;
    }

    private int extractEventCount(String name) {
        if (name.contains("1m_")) return 1_000_000;
        if (name.contains("100k_")) return 100_000;
        if (name.contains("10k_")) return 10_000;
        if (name.contains("1k_")) return 1_000;
        return 0;
    }

    private String formatEventCount(int c) {
        if (c >= 1_000_000) return (c / 1_000_000) + "M";
        if (c >= 1_000) return (c / 1_000) + "k";
        return String.valueOf(c);
    }

    public static final class AnalyzeResult {
        public final boolean hasLeak;
        public final boolean exceptionFound;

        public AnalyzeResult(boolean hasLeak, boolean exceptionFound) {
            this.hasLeak = hasLeak;
            this.exceptionFound = exceptionFound;
        }
    }

    private static final class TestResult {
        final String testName;
        final long memoryUsage;
        final long duration;

        TestResult(String testName, long memoryUsage, long duration) {
            this.testName = testName;
            this.memoryUsage = memoryUsage;
            this.duration = duration;
        }
    }

    private static final class ParseResult {
        final List<TestResult> results;
        final boolean exceptionFound;

        ParseResult(List<TestResult> results, boolean exceptionFound) {
            this.results = results;
            this.exceptionFound = exceptionFound;
        }
    }
}
```

- [ ] **Step 4: Run — expect PASS**

Run:
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests test
```
Expected: all tests pass (including the three new `MemoryLeakAnalyzerTest` cases).

- [ ] **Step 5: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/src/main/java/org/drools/ansible/rulebook/integration/loadtests/analyze/MemoryLeakAnalyzer.java \
        drools-ansible-rulebook-integration-load-tests/src/test/java/org/drools/ansible/rulebook/integration/loadtests/analyze/MemoryLeakAnalyzerTest.java
git commit -m "$(cat <<'EOF'
feat: port MemoryLeakAnalyzer with HA-aware four-group analysis

Ported from main, enhanced to bucket results into match/noHA,
match/HA-PG, unmatch/noHA, unmatch/HA-PG and run the same thresholds on
each group. Exposes analyzeFile() so tests can assert outcomes without
going through System.exit.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: `lib/common.sh`

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/lib/common.sh`

- [ ] **Step 1: Create `lib/common.sh`**

```bash
#!/usr/bin/env bash
# Shared helpers for drools-ansible-rulebook-integration-load-tests scripts.
# Source from a load_test_*.sh script:
#
#   set -euo pipefail
#   source "$(dirname "$0")/lib/common.sh"
#
# Functions are prefixed by subsystem: require_*, pg_*, jvm_*, fmt_*.

JAR="${JAR:-target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar}"
PG_CONTAINER=""
PG_PARAMS=""
_run_stderr=""
_mem=""
_time=""

# ---- require_* -------------------------------------------------------------

require_jar() {
  if [ ! -f "$JAR" ]; then
    echo "ERROR: Fat JAR not found at $JAR" >&2
    echo "Build it with:" >&2
    echo "  mvn -pl drools-ansible-rulebook-integration-load-tests -am package -DskipTests" >&2
    exit 1
  fi
}

require_docker() {
  if ! docker info >/dev/null 2>&1; then
    echo "ERROR: Docker is not available. PostgreSQL is required." >&2
    exit 1
  fi
}

require_python3() {
  if ! command -v python3 >/dev/null 2>&1; then
    echo "ERROR: python3 is required for free-port discovery." >&2
    exit 1
  fi
}

# ---- pg_* ------------------------------------------------------------------

pg_setup() {
  require_docker
  require_python3
  local pg_port
  pg_port=$(python3 -c 'import socket; s=socket.socket(); s.bind(("",0)); print(s.getsockname()[1]); s.close()')
  echo "Starting PostgreSQL container on port $pg_port..."
  PG_CONTAINER=$(docker run -d --rm \
    -e POSTGRES_USER=loadtest \
    -e POSTGRES_PASSWORD=loadtest \
    -e POSTGRES_DB=loadtest \
    -p "${pg_port}:5432" \
    postgres:15-alpine)

  echo "Waiting for PostgreSQL to be ready on port $pg_port..."
  local retries=30
  while ! docker exec "$PG_CONTAINER" pg_isready -U loadtest -q 2>/dev/null; do
    retries=$((retries - 1))
    if [ "$retries" -le 0 ]; then
      echo "ERROR: PostgreSQL failed to start within timeout." >&2
      docker stop "$PG_CONTAINER" >/dev/null 2>&1 || true
      exit 1
    fi
    sleep 1
  done

  local conn_retries=10
  while ! docker exec "$PG_CONTAINER" psql -U loadtest -d loadtest -c "SELECT 1" >/dev/null 2>&1; do
    conn_retries=$((conn_retries - 1))
    if [ "$conn_retries" -le 0 ]; then
      echo "ERROR: PostgreSQL authentication not ready within timeout." >&2
      docker stop "$PG_CONTAINER" >/dev/null 2>&1 || true
      exit 1
    fi
    sleep 1
  done

  PG_PARAMS="{\"db_type\":\"postgres\",\"host\":\"localhost\",\"port\":${pg_port},\"database\":\"loadtest\",\"user\":\"loadtest\",\"password\":\"loadtest\",\"sslmode\":\"disable\"}"
  echo "PostgreSQL ready on port $pg_port"
}

pg_cleanup() {
  if [ -n "$PG_CONTAINER" ]; then
    echo "Stopping PostgreSQL container..."
    docker stop "$PG_CONTAINER" >/dev/null 2>&1 || true
    PG_CONTAINER=""
  fi
}

pg_truncate() {
  docker exec "$PG_CONTAINER" psql -U loadtest -d loadtest -c \
    "TRUNCATE drools_ansible_session_state, drools_ansible_matching_event, drools_ansible_action_info, drools_ansible_ha_stats" >/dev/null 2>&1 || true
}

pg_count() {
  local table="$1"
  docker exec "$PG_CONTAINER" psql -U loadtest -d loadtest -tAc "SELECT COUNT(*) FROM $table" 2>/dev/null || echo "ERR"
}

pg_blob_size() {
  docker exec "$PG_CONTAINER" psql -U loadtest -d loadtest -tAc \
    "SELECT COALESCE(MAX(length(partial_matching_events)), 0) FROM drools_ansible_session_state" 2>/dev/null || echo "ERR"
}

# ---- jvm_* -----------------------------------------------------------------

# jvm_run <label> <args...>
# Runs the fat jar with -Xmx1g, appends stdout+stderr to $LOG, captures stderr into $_run_stderr.
# Tolerates non-zero JVM exit — caller inspects _run_stderr / fmt_parse_metrics.
jvm_run() {
  local label="$1"; shift
  local tmpstderr
  tmpstderr=$(mktemp)
  echo "=== $label ===" >> "$LOG"
  java -Xmx1g -Dorg.slf4j.simpleLogger.logFile=System.out \
       -jar "$JAR" "$@" >> "$LOG" 2>"$tmpstderr" || true
  cat "$tmpstderr" >> "$LOG"
  _run_stderr=$(cat "$tmpstderr")
  rm -f "$tmpstderr"
  echo "" >> "$LOG"
}

# ---- fmt_* -----------------------------------------------------------------

# fmt_parse_metrics <stderr> <file>
# Finds the last stderr line beginning with <file> and splits on ",".
# Sets $_mem and $_time (or "FAILED"/"FAILED" if the line wasn't emitted).
fmt_parse_metrics() {
  local stderr_output="$1"
  local filename="$2"
  local metrics_line
  metrics_line=$(echo "$stderr_output" | grep "^${filename}" | tail -1)
  if [ -z "$metrics_line" ]; then
    _mem="FAILED"
    _time="FAILED"
  else
    _mem=$(echo "$metrics_line" | cut -d',' -f2 | tr -d ' ')
    _time=$(echo "$metrics_line" | cut -d',' -f3 | tr -d ' ')
  fi
}

# fmt_per_event_kb <bytes> <count> -> prints "%.1f" KB or "N/A".
fmt_per_event_kb() {
  local bytes="$1"
  local count="$2"
  if [ "$bytes" = "FAILED" ] || [ "$count" -le 0 ] 2>/dev/null; then
    echo "N/A"
    return
  fi
  awk "BEGIN { printf \"%.1f\", $bytes / $count / 1024 }"
}
```

- [ ] **Step 2: Syntax-check the file**

Run:
```bash
bash -n drools-ansible-rulebook-integration-load-tests/lib/common.sh
echo "syntax ok"
```
Expected: `syntax ok`.

- [ ] **Step 3: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/lib/common.sh
git commit -m "$(cat <<'EOF'
feat: add lib/common.sh with prefixed helpers

Single file sourced by all load-test scripts. Groups: require_* (preflight),
pg_* (Docker PostgreSQL lifecycle + inspection), jvm_* (fat-jar invocation
with stderr capture), fmt_* (metric parsing and per-event KB formatting).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 11: The four shell scripts

**Files:**
- Create: `drools-ansible-rulebook-integration-load-tests/load_test_match.sh`
- Create: `drools-ansible-rulebook-integration-load-tests/load_test_unmatch.sh`
- Create: `drools-ansible-rulebook-integration-load-tests/load_test_retention.sh`
- Create: `drools-ansible-rulebook-integration-load-tests/load_test_all.sh`

All scripts must be executable (`chmod +x`). All are designed to be run from the module root: `cd drools-ansible-rulebook-integration-load-tests && ./load_test_match.sh 1k`.

- [ ] **Step 1: Create `load_test_match.sh`**

```bash
#!/usr/bin/env bash
# Usage: ./load_test_match.sh [1k|10k|100k|1m]
#
# Runs 24kb_<size>_events.json once with noHA, once with HA-PG.
# Requires Docker for PostgreSQL.

set -euo pipefail
source "$(dirname "$0")/lib/common.sh"

size="${1:-1k}"
case "$size" in
  1k|10k|100k|1m) ;;
  *)
    echo "Invalid size: $size"
    echo "Usage: $0 [1k|10k|100k|1m]"
    exit 1
    ;;
esac

file="24kb_${size}_events.json"
count_for_size() {
  case "$1" in
    1k) echo 1000 ;;
    10k) echo 10000 ;;
    100k) echo 100000 ;;
    1m) echo 1000000 ;;
  esac
}
count=$(count_for_size "$size")

OUT="result_match_${size}.txt"
LOG="out_match.log"
> "$LOG"

require_jar
trap pg_cleanup EXIT
pg_setup

{
  echo "=== Match Load Test (size=${size}) ==="
  printf "\n%-8s %-8s %14s %9s %14s\n" "Mode" "Events" "Memory(bytes)" "Time(ms)" "Per-Event(KB)"
  printf "%s\n" "$(head -c 60 < /dev/zero | tr '\0' '-')"
} | tee "$OUT"

# noHA
echo "Running $file (noHA)..."
jvm_run "$file (noHA)" "$file"
fmt_parse_metrics "$_run_stderr" "$file"
per_event=$(fmt_per_event_kb "$_mem" "$count")
printf "%-8s %-8s %14s %9s %14s\n" "noHA" "$size" "$_mem" "$_time" "$per_event" | tee -a "$OUT"

# HA-PG
echo "Running $file (HA-PG)..."
pg_truncate
jvm_run "$file (HA-PG)" "$file" --ha-db-params "$PG_PARAMS"
fmt_parse_metrics "$_run_stderr" "$file"
per_event=$(fmt_per_event_kb "$_mem" "$count")
printf "%-8s %-8s %14s %9s %14s\n" "HA-PG" "$size" "$_mem" "$_time" "$per_event" | tee -a "$OUT"

echo "Results written to $OUT"
echo "Full logs in $LOG"
```

- [ ] **Step 2: Create `load_test_unmatch.sh`**

Identical to `load_test_match.sh` except:
- Header text: `=== Unmatch Load Test (size=${size}) ===`
- File name: `file="24kb_${size}_events_unmatch.json"`
- Output file: `OUT="result_unmatch_${size}.txt"`
- Log file: `LOG="out_unmatch.log"`
- "Match" in echoes replaced by "Unmatch".

```bash
#!/usr/bin/env bash
# Usage: ./load_test_unmatch.sh [1k|10k|100k|1m]
#
# Runs 24kb_<size>_events_unmatch.json once with noHA, once with HA-PG.
# Requires Docker for PostgreSQL.

set -euo pipefail
source "$(dirname "$0")/lib/common.sh"

size="${1:-1k}"
case "$size" in
  1k|10k|100k|1m) ;;
  *)
    echo "Invalid size: $size"
    echo "Usage: $0 [1k|10k|100k|1m]"
    exit 1
    ;;
esac

file="24kb_${size}_events_unmatch.json"
count_for_size() {
  case "$1" in
    1k) echo 1000 ;;
    10k) echo 10000 ;;
    100k) echo 100000 ;;
    1m) echo 1000000 ;;
  esac
}
count=$(count_for_size "$size")

OUT="result_unmatch_${size}.txt"
LOG="out_unmatch.log"
> "$LOG"

require_jar
trap pg_cleanup EXIT
pg_setup

{
  echo "=== Unmatch Load Test (size=${size}) ==="
  printf "\n%-8s %-8s %14s %9s %14s\n" "Mode" "Events" "Memory(bytes)" "Time(ms)" "Per-Event(KB)"
  printf "%s\n" "$(head -c 60 < /dev/zero | tr '\0' '-')"
} | tee "$OUT"

echo "Running $file (noHA)..."
jvm_run "$file (noHA)" "$file"
fmt_parse_metrics "$_run_stderr" "$file"
per_event=$(fmt_per_event_kb "$_mem" "$count")
printf "%-8s %-8s %14s %9s %14s\n" "noHA" "$size" "$_mem" "$_time" "$per_event" | tee -a "$OUT"

echo "Running $file (HA-PG)..."
pg_truncate
jvm_run "$file (HA-PG)" "$file" --ha-db-params "$PG_PARAMS"
fmt_parse_metrics "$_run_stderr" "$file"
per_event=$(fmt_per_event_kb "$_mem" "$count")
printf "%-8s %-8s %14s %9s %14s\n" "HA-PG" "$size" "$_mem" "$_time" "$per_event" | tee -a "$OUT"

echo "Results written to $OUT"
echo "Full logs in $LOG"
```

- [ ] **Step 3: Create `load_test_retention.sh`**

Keeps the shape of the existing `main/load_test_retention.sh`: 6 rows (3 sizes × 2 modes), plus a delta table between sizes per mode, plus an HA-overhead delta table (HA-PG minus noHA per size). Uses `retention_<N>_events.json` (not `failover_*`).

```bash
#!/usr/bin/env bash
# Usage: ./load_test_retention.sh
#
# Measures memory growth when events accumulate as partial matches.
# Uses retention_<N>_events.json (2-condition join, only condition 1 satisfied).
# Runs 100, 500, 1000 events in noHA and HA-PG modes.
# Requires Docker for PostgreSQL.

set -euo pipefail
source "$(dirname "$0")/lib/common.sh"

SIZES=("100" "500" "1k")
FILES=("retention_100_events.json" "retention_500_events.json" "retention_1k_events.json")
COUNTS=(100 500 1000)

OUT="result_retention.txt"
LOG="out_retention.log"
> "$LOG"

require_jar
trap pg_cleanup EXIT
pg_setup

{
  echo "=== Event Retention Memory Analysis ==="
  echo "Event payload: ~24KB JSON each (2-condition join, all retained as partial matches)"
  echo ""
  printf "%-10s %-8s %14s %9s %14s %10s %14s\n" \
    "Mode" "Events" "Memory(bytes)" "Time(ms)" "Per-Event(KB)" "MATCHING" "BlobSize(B)"
  printf "%s\n" "$(head -c 88 < /dev/zero | tr '\0' '-')"
} | tee "$OUT"

declare -A MEM_RESULTS

for run_mode in noHA HA-PG; do
  for idx in 0 1 2; do
    file="${FILES[$idx]}"
    count="${COUNTS[$idx]}"
    size="${SIZES[$idx]}"

    echo "Running $file ($run_mode)..."

    if [ "$run_mode" = "noHA" ]; then
      jvm_run "$file ($run_mode)" "$file"
      matching_rows="-"
      blob_size="-"
    else
      pg_truncate
      jvm_run "$file ($run_mode)" "$file" --ha-db-params "$PG_PARAMS"
      matching_rows=$(pg_count "drools_ansible_matching_event")
      blob_size=$(pg_blob_size)
    fi

    fmt_parse_metrics "$_run_stderr" "$file"
    mem="$_mem"
    time="$_time"
    per_event=$(fmt_per_event_kb "$mem" "$count")

    printf "%-10s %-8s %14s %9s %14s %10s %14s\n" \
      "$run_mode" "$size" "$mem" "$time" "$per_event" "$matching_rows" "$blob_size" | tee -a "$OUT"

    MEM_RESULTS["${run_mode}_${idx}"]="$mem"
  done
  echo "" | tee -a "$OUT"
done

{
  echo "=== Incremental Per-Event Cost (delta between sizes) ==="
  echo ""
  printf "%-10s %-12s %14s %14s\n" "Mode" "Range" "Delta(bytes)" "Per-Event(KB)"
  printf "%s\n" "$(head -c 54 < /dev/zero | tr '\0' '-')"

  for run_mode in noHA HA-PG; do
    for pair in "0:1:100-500:400" "1:2:500-1k:500"; do
      IFS=':' read -r from_idx to_idx label delta_count <<< "$pair"
      from_mem="${MEM_RESULTS["${run_mode}_${from_idx}"]}"
      to_mem="${MEM_RESULTS["${run_mode}_${to_idx}"]}"

      if [ "$from_mem" != "FAILED" ] && [ "$to_mem" != "FAILED" ] 2>/dev/null; then
        delta=$((to_mem - from_mem))
        per_event_delta=$(awk "BEGIN { printf \"%.1f\", $delta / $delta_count / 1024 }")
        printf "%-10s %-12s %14s %14s\n" "$run_mode" "$label" "$delta" "$per_event_delta"
      else
        printf "%-10s %-12s %14s %14s\n" "$run_mode" "$label" "FAILED" "N/A"
      fi
    done
  done

  echo ""
  echo "--- HA overhead (HA-PG minus noHA) ---"
  printf "%-12s %14s %14s\n" "Range" "Delta(bytes)" "Per-Event(KB)"
  printf "%s\n" "$(head -c 42 < /dev/zero | tr '\0' '-')"

  for pair in "0:100:100" "1:500:500" "2:1k:1000"; do
    IFS=':' read -r idx label count <<< "$pair"
    noha_mem="${MEM_RESULTS["noHA_${idx}"]}"
    ha_mem="${MEM_RESULTS["HA-PG_${idx}"]}"

    if [ "$noha_mem" != "FAILED" ] && [ "$ha_mem" != "FAILED" ] 2>/dev/null; then
      overhead=$((ha_mem - noha_mem))
      per_event_overhead=$(awk "BEGIN { printf \"%.1f\", $overhead / $count / 1024 }")
      printf "%-12s %14s %14s\n" "$label" "$overhead" "$per_event_overhead"
    else
      printf "%-12s %14s %14s\n" "$label" "FAILED" "N/A"
    fi
  done

  echo ""
  echo "--- Notes ---"
  echo "Incremental per-event cost eliminates JVM baseline overhead."
  echo "  noHA delta   = M (Java Map in Drools working memory)"
  echo "  HA-PG delta  = M + J (Map + JSON string in EventRecord)"
  echo "  HA overhead  = J (the EventRecord JSON copy)"
  echo ""
} | tee -a "$OUT"

echo "Results written to $OUT"
echo "Full logs in $LOG"
```

- [ ] **Step 4: Create `load_test_all.sh`**

```bash
#!/usr/bin/env bash
# Usage: ./load_test_all.sh
#
# Loops 4 sizes x {match, unmatch} x {noHA, HA-PG} = 16 runs.
# Emits one combined result_all.txt (metric lines) and runs MemoryLeakAnalyzer.
# Requires Docker for PostgreSQL.

set -euo pipefail
source "$(dirname "$0")/lib/common.sh"

SIZES=("1k" "10k" "100k" "1m")
SCENARIOS=("match" "unmatch")
MODES=("noHA" "HA-PG")

OUT="result_all.txt"
LOG="out_all.log"
> "$OUT"
> "$LOG"

require_jar
trap pg_cleanup EXIT
pg_setup

for size in "${SIZES[@]}"; do
  for scenario in "${SCENARIOS[@]}"; do
    if [ "$scenario" = "match" ]; then
      file="24kb_${size}_events.json"
    else
      file="24kb_${size}_events_unmatch.json"
    fi
    for mode in "${MODES[@]}"; do
      if [ "$mode" = "noHA" ]; then
        label="$file (noHA)"
        echo "Running $label..."
        jvm_run "$label" "$file"
      else
        pg_truncate
        label="$file (HA-PG)"
        echo "Running $label..."
        jvm_run "$label" "$file" --ha-db-params "$PG_PARAMS"
      fi
      # Append the metric line from this run to result_all.txt.
      # Metric line begins with the events-json name (optionally followed by " (HA-PG)").
      echo "$_run_stderr" | grep "^${file}" | tail -1 >> "$OUT" || echo "$file (${mode}), FAILED, FAILED" >> "$OUT"
    done
  done
done

echo ""
echo "All 16 runs complete. Result lines:"
cat "$OUT"
echo ""

echo "Running MemoryLeakAnalyzer..."
java -cp "target/classes:target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar" \
     org.drools.ansible.rulebook.integration.loadtests.analyze.MemoryLeakAnalyzer "$OUT"
ANALYZER_EXIT=$?
echo "Analyzer exit code: $ANALYZER_EXIT"
exit $ANALYZER_EXIT
```

- [ ] **Step 5: Make all four scripts executable and syntax-check**

Run:
```bash
chmod +x drools-ansible-rulebook-integration-load-tests/load_test_match.sh \
         drools-ansible-rulebook-integration-load-tests/load_test_unmatch.sh \
         drools-ansible-rulebook-integration-load-tests/load_test_retention.sh \
         drools-ansible-rulebook-integration-load-tests/load_test_all.sh
for f in drools-ansible-rulebook-integration-load-tests/load_test_*.sh; do
  bash -n "$f" && echo "$f OK"
done
```
Expected: four `... OK` lines.

- [ ] **Step 6: Commit**

```bash
git add drools-ansible-rulebook-integration-load-tests/load_test_match.sh \
        drools-ansible-rulebook-integration-load-tests/load_test_unmatch.sh \
        drools-ansible-rulebook-integration-load-tests/load_test_retention.sh \
        drools-ansible-rulebook-integration-load-tests/load_test_all.sh
git commit -m "$(cat <<'EOF'
feat: add load_test_match / unmatch / retention / all shell scripts

All four scripts source lib/common.sh, require Docker, and cover noHA +
HA-PG. load_test_all.sh drives 4 sizes x {match, unmatch} x {noHA, HA-PG}
and hands off to MemoryLeakAnalyzer at the end; its exit code becomes the
script's.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 12: End-to-end smoke test (no commit)

Verify that the whole pipeline works for at least one small script, both noHA and HA-PG. No commit — this is an acceptance gate before shipping.

- [ ] **Step 1: Build the fat jar**

Run (from project root):
```bash
mvn -pl drools-ansible-rulebook-integration-load-tests -am package -DskipTests
```
Expected: `BUILD SUCCESS`. `drools-ansible-rulebook-integration-load-tests/target/drools-ansible-rulebook-integration-load-tests-jar-with-dependencies.jar` exists.

- [ ] **Step 2: Run `load_test_match.sh 1k`**

Run (from module root):
```bash
cd drools-ansible-rulebook-integration-load-tests
./load_test_match.sh 1k
```
Expected:
- The script completes without error.
- `result_match_1k.txt` contains two rows (noHA and HA-PG) with non-`FAILED` memory/time numbers.
- `out_match.log` contains the full stdout from both runs.
- The Docker PG container is started and stopped automatically.

- [ ] **Step 3: Spot-check the metric lines**

Run:
```bash
cat result_match_1k.txt
grep "^24kb_1k_events.json" out_match.log
```
Expected: the grep shows two metric lines, one ending `(noHA)`-less and one with `(HA-PG)`, with positive memory (7-digit bytes) and positive time (3-4 digit ms). Numbers should be realistic (not 0/FAILED).

- [ ] **Step 4: Run `load_test_unmatch.sh 1k` as a second smoke**

```bash
./load_test_unmatch.sh 1k
cat result_unmatch_1k.txt
```
Expected: two non-FAILED rows (noHA and HA-PG). If either row is FAILED, the likely cause is the Unmatch rule shape in the generator — re-inspect Task 8 and iterate.

- [ ] **Step 5: Run `load_test_retention.sh`**

```bash
./load_test_retention.sh
cat result_retention.txt
```
Expected: 6 rows + delta table + HA-overhead table. MATCHING and BlobSize(B) columns show `-` for noHA and integers for HA-PG.

- [ ] **Step 6: Report results to the user**

Surface any anomalies (FAILED rows, large unexpected memory, exceptions in `out_*.log`) before declaring the task done. If everything passes: report the five output files that were generated and the total duration.

---

## Notes on running `load_test_all.sh`

`load_test_all.sh` executes 16 JVM runs and is the slowest script. It's deliberately not part of the smoke test (Task 12). Run it manually once the three single-size scripts pass. Expect ~5-10 minutes wall-clock time on a developer machine.
