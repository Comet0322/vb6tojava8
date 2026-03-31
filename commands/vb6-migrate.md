---
description: Full VB6 ETL to Java 8 migration workflow. Analyses VB6 source with an
  internal package manual, designs a 5-class BatchRun architecture, implements with
  TDD, reviews, and validates output with auto-repair loop on failure.
---

Migrate a VB6 ETL codebase to Java 8 using the full 4-phase workflow defined in `skill: vb6-etl-migration`.

## Usage

```
/vb6-migrate \
  --vb6-src      <path-to-vb6-project>         # required \
  --vb6-pkg-doc  <path-to-vb6-package-manual>  # required \
  --java-pkg-doc <path-to-java-package-manual> # required \
  --java-out     <path-to-java-output-dir>      # required \
  --max-retries  <n>                            # optional, default 3
```

## Reference

Before executing, load `skill: vb6-etl-migration` for phase details, handoff formats, go/no-go gates, and repair routing rules. Load `skill: vb6-to-java8-patterns` for language mapping knowledge.

## Execution

### Phase 1 — VB6 Analysis

Invoke agent: `vb6-analyzer`

Pass:

- `VB6_SRC` = value of `--vb6-src`
- `VB6_PKG_DOC` = value of `--vb6-pkg-doc`

Wait for `VB6_ANALYSIS_REPORT.md` and the HANDOFF section. Check Go/No-Go before continuing.

---

### Phase 2 — Architecture Design

Invoke agent: `architect` with `skill: vb6-to-java8-patterns`

Pass:

- `VB6_ANALYSIS_REPORT.md` from Phase 1
- `--java-pkg-doc` path (architect reads this to derive Java API mappings)
- REPAIR_ATTEMPT_COUNT from previous BLOCK (if re-entering Phase 2 due to Tier 4 BLOCK, include the VALIDATION_REPORT.md as additional context)

Wait for `ARCHITECTURE.md`, ADR files, and the HANDOFF section. Check Go/No-Go before continuing.

---

### Phase 3 — Implementation

Run agents in sequence:

**3a. tdd-guide**

Pass:

- `ARCHITECTURE.md` and all ADR files
- Java package manual path
- If re-entering due to BLOCK: include `VALIDATION_REPORT.md` and REPAIR_ROUTING instructions

Instructions to tdd-guide:

- Use `mvn test` to run tests (not `npm test`)
- Generate all 5 class files: `BatchRun.java`, `CreateTable.java`, `Download.java`, `Process.java`, `Upload.java`
- Write test classes: `BatchRunTest.java`, `CreateTableTest.java`, `DownloadTest.java`, `ProcessTest.java`, `UploadTest.java`
- Each business rule from VB6_ANALYSIS_REPORT.md Business Rules Catalog must have a test case
- Target 80%+ JaCoCo line coverage (`mvn verify`)

**3b. vb6-java-reviewer**

Pass:

- Generated Java source directory
- `ARCHITECTURE.md`

If verdict is BLOCK: stop Phase 3, surface the findings to the user. Do not proceed to Phase 4.
If verdict is APPROVE or WARNING: continue.

**3c. java-build-resolver** (conditional)

Invoke only if `mvn compile` or `mvn test` failed during 3a or 3b.

Pass:

- Build error output
- Java source directory

---

### Phase 4 — Data Validation

Invoke agent: `etl-validator`

Pass:

- `VB6_OUTPUT`: path to VB6 ETL reference output
- `JAVA_OUTPUT`: value of `--java-out`
- `VB6_ANALYSIS_REPORT.md`
- `REPAIR_ATTEMPT_COUNT`: current count (start at 0)

**On CERTIFY**: write `MIGRATION_REPORT.md` (see `skill: vb6-etl-migration` for format) and stop. Migration complete.

**On INVESTIGATE**: stop and present the `VALIDATION_REPORT.md` to the user. Do not auto-retry for INVESTIGATE verdicts.

**On BLOCK**:

1. Increment `REPAIR_ATTEMPT_COUNT`
2. If `REPAIR_ATTEMPT_COUNT` > `--max-retries` (default 3): stop with ESCALATION message
3. Read `REPAIR_ROUTING` from the VALIDATION_REPORT:
   - `route_to_phase: 3` → re-enter Phase 3 at step 3a with the BLOCK report as additional context
   - `route_to_phase: 2` → re-enter Phase 2 with the BLOCK report as additional context
4. After re-running the routed phase, return to Phase 4 with the updated `REPAIR_ATTEMPT_COUNT`

**On ESCALATION** (emitted by etl-validator after max retries): stop and present ESCALATION section to user.

---

## Agent Chain Summary

```
vb6-analyzer
    ↓ VB6_ANALYSIS_REPORT.md
architect (+ skill: vb6-to-java8-patterns)
    ↓ ARCHITECTURE.md + ADRs
tdd-guide → vb6-java-reviewer → [java-build-resolver if build fails]
    ↓ Java source + tests
etl-validator
    ├─ CERTIFY   → MIGRATION_REPORT.md  ✓
    ├─ INVESTIGATE → stop for human review
    └─ BLOCK → auto-repair loop (max 3 retries)
               Tier 1-3 → Phase 3
               Tier 4   → Phase 2
```
