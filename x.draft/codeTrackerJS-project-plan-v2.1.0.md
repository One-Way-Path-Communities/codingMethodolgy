# codeTrackerJS — Project Plan v2.1.0

## 1. Purpose
The **codeTrackerJS** package analyzes another package’s `.mjs` files and stores structured information about exported functions into a **Postgres database**, with proper version-aware snapshots for `branchName` + `releaseTag`.  
It should run entirely programmatically with zero manual schema setup.

## 2. High-Level Flow
1. Orchestrator receives:
   - `moduleFolder`
   - `branchName`
   - `releaseTag`
2. Scans folder for `.mjs` files.
3. Parses functions + JSDoc.
4. Prepares Postgres tables.
5. Writes a snapshot (safe delete+insert).
6. Returns a summary object.

## 3. Inputs / Outputs

### 3.1 Orchestrator Input

```js
trackModuleCode(moduleFolder, branchName, releaseTag)
```

### 3.2 Orchestrator Output

```js
{
  moduleSlug,
  moduleId,
  branchName,
  releaseTag,
  functionsCount,
  parametersCount,
  returnValuesCount
}
```

## 4. Environment Variables (.env inside codeTrackerJS/)

The **runtime** and **self-test** configuration live in a single `.env` file inside the `codeTrackerJS/` folder.

### 4.1 Runtime Configuration

These variables are required for normal operation (used by `pgClient.mjs`, `schemaInit.mjs`, `storeSnapshot.mjs`, and the orchestrator):

```env
CODE_TRACKER_PG_HOST=localhost
CODE_TRACKER_PG_PORT=5432
CODE_TRACKER_PG_DATABASE=code_tracker
CODE_TRACKER_PG_USER=code_tracker_user
CODE_TRACKER_PG_PASSWORD=supersecret
CODE_TRACKER_PG_SSL=false
CODE_TRACKER_LOG_LEVEL=info
```

### 4.2 Self-test Configuration

These variables are optional, but if provided they are used by the **self-test blocks** in each module so the tests can be run in a controlled way without editing the code:

```env
# Root folder containing sample .mjs files used by orchestrator + source parser tests.
CODE_TRACKER_TEST_SAMPLE_PACKAGE=./test-data/samplePackage

# A folder the fileScanner self-test can safely create/delete subfolders/files in.
CODE_TRACKER_TEST_SCANNER_ROOT=./test-data/fileScanner

# Database name to use specifically for self-tests (optional).
# If omitted, tests fall back to CODE_TRACKER_PG_DATABASE.
CODE_TRACKER_TEST_DATABASE=code_tracker_test

# Optional: schema / table prefix used by tests so they don't collide with production tables.
CODE_TRACKER_TEST_SCHEMA_PREFIX=test_

# Optional: whether self-tests should attempt to connect to Postgres at all.
# If set to "false", pgClient and schemaInit should log a clear message and skip DB work.
CODE_TRACKER_TEST_ENABLE_DB=true
```

**Rules for module implementation:**

- Modules must **read these self-test env vars only inside their self-test blocks**.
- Default behaviour (when these vars are missing) must still be safe:
  - either skip self-test with a clear log message, or
  - fall back to obvious defaults like `./test-data/...`.

## 5. Package Structure

```text
codeTrackerJS/
  codeTrackerOrchestrator.mjs
  fileScanner.mjs
  sourceParser.mjs
  pgClient.mjs
  schemaInit.mjs
  storeSnapshot.mjs
  README.md
  .env
  test-data/
    samplePackage/        # small example .mjs files for parsing/orchestrator
    fileScanner/          # scratch space for fileScanner tests
```

## 6. Modules + Independent Test Requirements

### 6.1 codeTrackerOrchestrator.mjs

**Responsibility**

- Validate inputs.
- Derive `moduleSlug` from `moduleFolder`.
- Use `fileScanner` → file list.
- Use `sourceParser` → function metadata.
- Use `pgClient` + `schemaInit` → DB ready.
- Use `storeSnapshot` → write versioned snapshot.
- Return the summary object shown above.

**Self-test**

- Triggered via:

  ```js
  if (import.meta.url === `file://${process.argv[1]}`) {
    // self-test
  }
  ```

- Uses:
  - `CODE_TRACKER_TEST_SAMPLE_PACKAGE` (if set) as the folder to scan; otherwise defaults to `./test-data/samplePackage`.
  - `CODE_TRACKER_TEST_DATABASE` and/or `CODE_TRACKER_TEST_SCHEMA_PREFIX` if set, otherwise falls back to runtime DB vars.
- Steps:
  1. Call `trackModuleCode(CODE_TRACKER_TEST_SAMPLE_PACKAGE, "test-branch", "v0.0.0-test")`.
  2. Log the summary to the console.
  3. If DB is unavailable AND `CODE_TRACKER_TEST_ENABLE_DB=false`, log a clear message and exit gracefully.

---

### 6.2 fileScanner.mjs

**Responsibility**

- Given a root folder path, recursively return `.mjs` file paths.
- Exclude `node_modules`, `.git`, and any additional ignore list.

**Self-test**

- Uses `CODE_TRACKER_TEST_SCANNER_ROOT` as the root directory; defaults to `./test-data/fileScanner`.
- Steps:
  1. Create a small test directory tree:
     - `a.mjs`, `b.mjs`, `ignore.txt`, `nested/nested.mjs`.
  2. Invoke the scanner on the root.
  3. Log results and confirm only `.mjs` files are included.
  4. Clean up test files/directories if practical.

---

### 6.3 sourceParser.mjs

**Responsibility**

- Given `.mjs` file paths and/or raw source, parse:
  - Exported function names.
  - Parameter names/types (from JSDoc).
  - Return type (from JSDoc, where possible).

**Self-test**

- Uses inline **sample source strings** so it does not depend on real files.
- Steps:
  1. Define 2–3 simple source snippets with JSDoc comments and exported functions.
  2. Pass them into the parser.
  3. Log the parsed structures.
  4. Verify they match expected shapes (e.g., via basic checks).

---

### 6.4 pgClient.mjs

**Responsibility**

- Expose a function that returns a configured Postgres client using the **runtime env vars** (with optional test overrides).

**Self-test**

- Reads:
  - `CODE_TRACKER_TEST_DATABASE` (if present) and uses it instead of `CODE_TRACKER_PG_DATABASE`.
  - `CODE_TRACKER_TEST_ENABLE_DB` to decide whether to attempt a connection.
- Steps:
  1. If `CODE_TRACKER_TEST_ENABLE_DB==="false"`, log that DB tests are disabled and return early.
  2. Otherwise, attempt a `SELECT 1` query.
  3. Log success or a clear failure message.
  4. Never throw uncaught errors from self-test; exit non-zero but with a clear log if connection fails.

---

### 6.5 schemaInit.mjs

**Responsibility**

- Given a Postgres client, ensure required tables exist using `CREATE TABLE IF NOT EXISTS` / `CREATE SCHEMA IF NOT EXISTS`.
- Tables hold:
  - A `modules` table keyed by `moduleSlug` / `moduleId`.
  - A `snapshots` table keyed by `(moduleId, branchName, releaseTag)`.
  - A `functions` table with metadata for each function.

**Self-test**

- Uses the same DB resolution rules as `pgClient.mjs`:
  - Prefer `CODE_TRACKER_TEST_DATABASE` when present.
  - Skip DB work entirely if `CODE_TRACKER_TEST_ENABLE_DB==="false"`.
- Steps:
  1. Get a Postgres client (ideally via `pgClient`).
  2. Run `schemaInit` against the test DB/schema.
  3. Perform minimal verification (e.g., `SELECT` against information_schema).
  4. Log results.

---

### 6.6 storeSnapshot.mjs

**Responsibility**

- Accepts:
  - `moduleSlug`, `branchName`, `releaseTag`.
  - Parsed function metadata (from `sourceParser`).
  - A Postgres client / connection.
- Deletes any existing snapshot rows for `(moduleSlug, branchName, releaseTag)`.
- Inserts:
  - Module row (if not present).
  - Snapshot row (for this branch/tag).
  - Function metadata rows.

**Self-test**

- Uses:
  - `CODE_TRACKER_TEST_DATABASE` / `CODE_TRACKER_TEST_SCHEMA_PREFIX` if available.
  - A synthetic `moduleSlug`, e.g., `codeTrackerJS-selftest`.
- Steps:
  1. Construct minimal, hard-coded function metadata in-memory.
  2. Call `storeSnapshot` with:
     - `branchName="selftest-branch"`, `releaseTag="v0.0.0-selftest"`.
  3. Query back the inserted data.
  4. Log the results.
  5. Optionally clean up test data.

---

## 7. Test Philosophy

- **Each module must be independently testable** without requiring others (except where explicitly noted, e.g., `schemaInit` using `pgClient`).
- Self-tests should:
  - Use **local sample data** or **mocked inputs**.
  - Respect the **self-test env vars** defined in section 4.2.
- Modules must not crash if DB is unreachable:
  - Log a clear message.
  - Exit with a non-zero code in self-tests instead of throwing unhandled exceptions.

---

## 8. Versioning

- This file is **v2.1.0**.
- The builder should reference this exact filename in its `.env` configuration.
