---
name: vb6-etl-migration
description: End-to-end orchestration knowledge for migrating VB6 ETL pipelines to
  Java 8 using the 5-class BatchRun architecture. Defines 4 phases, handoff document
  formats, go/no-go gates, auto-repair loop specification, and risk taxonomy. Use
  this skill when orchestrating or executing the /vb6-migrate command.
origin: ECC
---

## When to Activate

Use this skill when:

- Running or implementing the `/vb6-migrate` workflow
- Deciding what inputs a phase needs or what outputs it must produce
- Determining how to route after a validation BLOCK
- Understanding what "done" means for each phase

---

## Workflow Overview

```
/vb6-migrate --vb6-src <path> --vb6-pkg-doc <path> --java-pkg-doc <path> --java-out <path>

Phase 1: vb6-analyzer          reads: vb6-src + vb6-pkg-doc
Phase 2: architect             reads: VB6_ANALYSIS_REPORT.md + java-pkg-doc
                               skill: vb6-to-java8-patterns
Phase 3: tdd-guide             reads: ARCHITECTURE.md + ADRs
      →  vb6-java-reviewer     reads: generated Java source
      →  java-build-resolver   reads: build errors (if any)
Phase 4: etl-validator         reads: VB6 reference output + Java output
                                    + VB6_ANALYSIS_REPORT.md
```

---

## Phase Details

### Phase 1 — VB6 Analysis

**Agent**: `vb6-analyzer`
**Inputs**: VB6 project path, VB6 internal package manual
**Outputs**: `VB6_ANALYSIS_REPORT.md`

**Success criteria** (go/no-go gate):

- All `.bas` and `.cls` files have been read and inventoried
- Every internal package method call has an entry in the DB API Call Inventory table
- Business Rules Catalog covers all modules that contain Transform logic
- Risk Register is complete

**Do not proceed to Phase 2 if**:

- Any module was skipped due to read errors
- The VB6 package manual could not be read (missing path or unreadable)

---

### Phase 2 — Architecture Design

**Agent**: `architect` (informed by skill: `vb6-to-java8-patterns`)
**Inputs**: `VB6_ANALYSIS_REPORT.md`, Java 8 internal package manual
**Outputs**: `ARCHITECTURE.md`, one ADR per major design decision

**Success criteria**:

- Each ETL step from Phase 1's ETL Step Inventory maps to one of the 5 Java classes
- A Java Package API Table exists (VB6 API call → Java package method)
- An ADR exists for each risk flagged as HIGH in the Risk Register
- `ARCHITECTURE.md` contains the BatchRun execution sequence

**Do not proceed to Phase 3 if**:

- Any VB6 DB API call has no Java equivalent identified
- Any HIGH risk item has no mitigation decision in an ADR

---

### Phase 3 — Implementation

**Agents**: `tdd-guide` → `vb6-java-reviewer` → `java-build-resolver`
**Inputs**: `ARCHITECTURE.md`, ADRs, Java package manual
**Outputs**: Java source files for all 5 classes + test suite

**tdd-guide instructions**:

- Use `mvn test` (not `npm test`) to run tests
- Test each of the 5 classes independently: `CreateTableTest`, `DownloadTest`, `ProcessTest`, `UploadTest`, `BatchRunTest`
- Target: 80%+ JaCoCo line coverage (`mvn verify` with JaCoCo plugin)
- Each business rule from the Business Rules Catalog must have a dedicated test case

**vb6-java-reviewer instructions**:

- Read `ARCHITECTURE.md` before reviewing
- Focus on migration correctness (data types, error handling) and 5-class compliance
- Do not block on Spring Boot patterns — this is not a Spring project

**java-build-resolver instructions**:

- Only invoked if `mvn compile` or `mvn test` fails
- Fix compilation errors and dependency issues, not logic

**Success criteria**:

- `mvn verify` passes with 80%+ line coverage
- `vb6-java-reviewer` verdict is APPROVE or WARNING (no BLOCK)
- All 5 class files exist in the output directory

---

### Phase 4 — Data Validation

**Agent**: `etl-validator`
**Inputs**: VB6 ETL reference output, Java ETL output, `VB6_ANALYSIS_REPORT.md`
**Outputs**: `VALIDATION_REPORT.md` with verdict

**Success criteria** (CERTIFY):

- Tier 1: row count match within 0.1%, all output files present
- Tier 2: null rates within 1%, distinct key counts match
- Tier 3: numeric sums within 0.001%, date ranges match
- Tier 4: all documented business rules produce identical output

---

## Handoff Document Template

Use this format between every phase transition:

```markdown
## HANDOFF: [source-agent] → [target-agent]

### Context

[2-3 sentences: what this ETL does and what was accomplished in this phase]

### Key Findings

[Bulleted list: top items the receiving agent must know]

### Artifacts Produced

- [list of files written]

### Open Questions

[Items requiring architect or developer judgment before proceeding]

### REPAIR_ATTEMPT_COUNT

[integer: 0 on first run, increments each time this phase is re-entered due to BLOCK]

### Go/No-Go

PROCEED — [brief reason]
HOLD — [reason and what is needed to unblock]
```

---

## Auto-Repair Loop

When `etl-validator` emits `BLOCK`, the `/vb6-migrate` command re-routes automatically:

| Failed Tier           | Route To | Agent       | What to Fix                            |
| --------------------- | -------- | ----------- | -------------------------------------- |
| Tier 1 (structural)   | Phase 3  | `tdd-guide` | Logic producing wrong row/column count |
| Tier 2 (completeness) | Phase 3  | `tdd-guide` | Null handling, de-duplication          |
| Tier 3 (statistical)  | Phase 3  | `tdd-guide` | Numeric precision, boundary conditions |
| Tier 4 (semantic)     | Phase 2  | `architect` | Business rule misunderstood in design  |

**Repair loop rules**:

- `REPAIR_ATTEMPT_COUNT` is incremented before re-entering the routed phase
- The BLOCK report from `etl-validator` is passed as additional context to the receiving agent
- Maximum retries: 3 (configurable via `--max-retries` flag)
- After 3 failures: `etl-validator` emits `ESCALATION` verdict; workflow stops

---

## Risk Taxonomy

These are the most common VB6 migration risks. Each should appear in the Phase 1 Risk Register and be addressed by an ADR in Phase 2.

| Risk                                              | Consequence if ignored                       | Mitigation                                                             |
| ------------------------------------------------- | -------------------------------------------- | ---------------------------------------------------------------------- |
| `Variant` type used in calculations               | Implicit coercion may change numeric results | Resolve all `Variant` to explicit types in `Process` class             |
| `On Error Resume Next` without `Err.Number` check | Silent data corruption carried forward       | Convert to logged warnings; add test cases for error paths             |
| VB6 `Currency` mapped to `double`                 | Floating-point drift in financial sums       | Use `BigDecimal` throughout `Process` and `Upload`                     |
| Timezone-naive `Date` fields                      | Date boundary bugs at midnight or DST change | Use `LocalDate`; document timezone assumptions in ADR                  |
| COM DLL with no Java equivalent                   | Untranslatable functionality                 | Identify Java library substitute or defer to human                     |
| Stored procedure called via package               | Proc logic not visible in VB6 source         | Document proc name; include proc SQL in `ARCHITECTURE.md` if available |
| Global module-level state                         | Non-obvious coupling between ETL steps       | Convert to explicit method parameters in Java                          |

---

## Final Migration Report Format

```markdown
# Migration Report

## Summary

Migration: [VB6 project name] → Java 8
Date: [ISO date]
Verdict: COMPLETE / PARTIAL / BLOCKED

## Coverage

VB6 modules analysed: N
ETL steps migrated: N/N
Business rules covered: N/N

## Validation Status

| Tier           | Result    |
| -------------- | --------- |
| 1 Structural   | PASS/FAIL |
| 2 Completeness | PASS/FAIL |
| 3 Statistical  | PASS/FAIL |
| 4 Semantic     | PASS/FAIL |

## Java Project Structure

[List of generated files]

## Open Items

[Anything deferred or requiring human follow-up]
```
