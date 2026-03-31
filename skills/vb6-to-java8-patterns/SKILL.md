---
name: vb6-to-java8-patterns
description: Knowledge base for VB6 ETL to Java 8 migration. Covers the 5-class
  BatchRun target architecture, language construct mapping, data type conversion,
  error handling translation, and the process for deriving DB API mappings from
  internal company package manuals.
origin: ECC
---

## When to Use

Reference this skill when:

- Designing a Java 8 architecture for a migrated VB6 ETL system
- Deciding how to map VB6 constructs, data types, or error handling patterns to Java
- Determining which Java class in the 5-class model a piece of VB6 logic belongs to
- Reading an internal Java package manual and mapping its APIs to VB6 DB calls

---

## 5-Class Target Architecture

All migrated ETL systems use this fixed structure:

```
BatchRun                     ← Coordinator (no business logic)
  ├─ CreateTable.execute()   ← Setup: DDL, truncate staging, schema prep
  ├─ Download.execute()      ← Extract: read from source DB or files
  ├─ Process.execute()       ← Transform: calculate, map, filter
  └─ Upload.execute()        ← Load: write to destination DB or files
```

### Class Responsibilities

| Class         | Responsibility                                                                                                                       | Must NOT contain                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `BatchRun`    | Instantiate the other 4 classes and call their `execute()` methods in order. Handle top-level exceptions and report status.          | Data reads, transforms, writes, business logic    |
| `CreateTable` | Create or truncate destination tables. Create temporary staging structures. Verify preconditions (source table exists, etc.).        | Business logic, data reads from source            |
| `Download`    | Read data from source using the internal DB package. Produce an intermediate representation (list of objects, staging table insert). | Transformations, destination writes               |
| `Process`     | Apply all business rules: type coercion, status mapping, calculations, filtering. Read from intermediate, write to intermediate.     | Source DB reads, destination DB writes            |
| `Upload`      | Write the processed data to the destination using the internal DB package.                                                           | Business logic, intermediate data transformations |

### BatchRun Template

```java
public class BatchRun {
    public void execute() throws Exception {
        new CreateTable().execute();
        new Download().execute();
        new Process().execute();
        new Upload().execute();
    }
}
```

---

## How to Derive DB API Mappings from the Java Package Manual

When you have the Java package manual (`--java-pkg-doc`) but no pre-defined mapping:

1. **Read the manual in full** — identify every method that reads from or writes to a database
2. **Classify each method** by operation type:
   - Methods that take a SELECT-like query or table name and return rows/objects → `Download` class
   - Methods that INSERT, UPDATE, or UPSERT rows → `Upload` class
   - Methods that CREATE, DROP, or TRUNCATE tables → `CreateTable` class
3. **Match to VB6 API calls** from `VB6_ANALYSIS_REPORT.md` DB API Call Inventory:
   - VB6 SELECT call → Java read method (goes in `Download`)
   - VB6 INSERT/UPDATE call → Java write method (goes in `Upload`)
   - VB6 DDL call → Java DDL method (goes in `CreateTable`)
4. **Note parameter differences**: VB6 may pass raw SQL strings; the Java package may use typed parameters, connection objects, or builder patterns — translate accordingly
5. **Document your mapping** in the `ARCHITECTURE.md` as a Java Package API Table

---

## Language Construct Mapping

| VB6                              | Java 8                                                | Notes                                          |
| -------------------------------- | ----------------------------------------------------- | ---------------------------------------------- |
| `Dim x As String`                | `String x;`                                           | —                                              |
| `Dim x As Variant`               | `Object x` or specific type                           | Resolve type from usage context                |
| `Dim x As Long`                  | `int x`                                               | VB6 Long is 32-bit                             |
| `Dim x As Integer`               | `short x` or `int x`                                  | VB6 Integer is 16-bit                          |
| `Dim x As Currency`              | `BigDecimal x`                                        | Never `double` or `float`                      |
| `Dim x As Date`                  | `LocalDate` or `LocalDateTime`                        | Choose based on whether time component is used |
| `Dim x As Boolean`               | `boolean x`                                           | —                                              |
| `Dim x As Object`                | `Object x`                                            | Narrow type if possible                        |
| `Public Sub Name()`              | `public void name()`                                  | Rename to camelCase                            |
| `Public Function Name() As Type` | `public Type name()`                                  | —                                              |
| `Private Sub Name()`             | `private void name()`                                 | —                                              |
| `Module` (`.bas`)                | `public final class` with `static` methods            | Or instance class if state is needed           |
| `Class Module` (`.cls`)          | `public class`                                        | —                                              |
| `Set obj = New ClassName`        | `ClassName obj = new ClassName();`                    | —                                              |
| `Set obj = Nothing`              | (no action — GC handles it)                           | —                                              |
| `IIf(cond, a, b)`                | `cond ? a : b`                                        | —                                              |
| `IsNull(x)`                      | `x == null`                                           | —                                              |
| `IsEmpty(x)`                     | `x == null \|\| x.toString().isEmpty()`               | —                                              |
| `Len(s)`                         | `s.length()`                                          | —                                              |
| `Left(s, n)`                     | `s.substring(0, n)`                                   | —                                              |
| `Right(s, n)`                    | `s.substring(s.length() - n)`                         | —                                              |
| `Mid(s, start, len)`             | `s.substring(start-1, start-1+len)`                   | VB6 Mid is 1-indexed                           |
| `Trim(s)`                        | `s.trim()`                                            | —                                              |
| `UCase(s)`                       | `s.toUpperCase()`                                     | —                                              |
| `LCase(s)`                       | `s.toLowerCase()`                                     | —                                              |
| `CStr(x)`                        | `String.valueOf(x)`                                   | —                                              |
| `CLng(x)`                        | `(int) x` or `Integer.parseInt(x)`                    | —                                              |
| `CDbl(x)`                        | `Double.parseDouble(x)`                               | Avoid for currency                             |
| `CCur(x)`                        | `new BigDecimal(String.valueOf(x))`                   | —                                              |
| `CDate(x)`                       | `LocalDate.parse(x)` or `LocalDateTime.parse(x)`      | Use explicit formatter                         |
| `Format(d, "yyyy-mm-dd")`        | `DateTimeFormatter.ofPattern("yyyy-MM-dd").format(d)` | —                                              |
| `DateAdd("d", n, d)`             | `d.plusDays(n)`                                       | —                                              |
| `DateDiff("d", d1, d2)`          | `ChronoUnit.DAYS.between(d1, d2)`                     | —                                              |
| `Now`                            | `LocalDateTime.now()`                                 | —                                              |
| `Date`                           | `LocalDate.now()`                                     | —                                              |

---

## Error Handling Translation

| VB6 Pattern                     | Java Equivalent                                           | Notes                                  |
| ------------------------------- | --------------------------------------------------------- | -------------------------------------- |
| `On Error Resume Next` (ignore) | `try { ... } catch (Exception e) { log.warn("...", e); }` | Must log — never silently swallow      |
| `On Error GoTo ErrHandler`      | `try { ... } catch (Exception e) { /* handler */ }`       | —                                      |
| `Err.Number` check after call   | Check exception type in catch                             | —                                      |
| `Err.Description`               | `e.getMessage()`                                          | —                                      |
| `Resume Next`                   | `continue` (in loop) or fall-through                      | —                                      |
| `Exit Sub` on error             | `return` or `throw`                                       | Prefer `throw` to make failure visible |

**Rule**: Every `On Error Resume Next` in VB6 that silently skipped errors must become a logged warning in Java. The migration must never introduce truly silent failures.

---

## Data Type Edge Cases

### Currency / Decimal

VB6 `Currency` is a 64-bit scaled integer (4 decimal places). Map to `BigDecimal` with explicit scale:

```java
// VB6: Dim amount As Currency = 1234.5678
BigDecimal amount = new BigDecimal("1234.5678");
```

Never use `double` for monetary values. Floating-point imprecision will cause Tier 3 validation failures.

### Dates Without Time

VB6 `Date` stores only the date portion. Use `LocalDate`, not `LocalDateTime`, unless the original field included time. Check the VB6 analysis report's DB API inventory to confirm.

### VB6 String "Null" vs Empty

VB6 distinguishes `Null` (database null) from `""` (empty string). When reading from DB results:

- If the VB6 code used `IsNull(rs!Field)` — use `rs.getString() == null` check
- If VB6 treated `""` and `Null` identically — use `StringUtils.isEmpty()` equivalent

---

## Naming Convention Bridge

| VB6 Style                     | Java Style                    | Example              |
| ----------------------------- | ----------------------------- | -------------------- |
| `strCustomerName` (Hungarian) | `customerName`                | Remove type prefix   |
| `intOrderCount`               | `orderCount`                  | —                    |
| `dtStartDate`                 | `startDate`                   | —                    |
| `bIsActive`                   | `isActive`                    | —                    |
| `MODULE_LEVEL_CONST`          | `SCREAMING_SNAKE_CASE` (keep) | Constants stay       |
| `GetCustomerById`             | `getCustomerById`             | Lowercase first char |
| `UpdateOrderStatus`           | `updateOrderStatus`           | —                    |
