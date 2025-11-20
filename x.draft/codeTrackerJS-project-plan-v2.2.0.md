# codeTrackerJS — Project Plan v2.2.0

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
  scripts/
    install-deps.sh       # bash script to install required dependencies for this package
```

> **Install script requirement:** this package must include a `scripts/install-deps.sh` bash script which installs all Node dependencies (`npm install ...`) and any other required tools (e.g. Postgres client libs) needed by this package’s modules and self-tests.

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
  - `CODE_TRACKER_TEST_SAMPLE_PACKAGE` as the folder to scan; defaults to `./test-data/samplePackage` if not set.
  - `CODE_TRACKER_TEST_DATABASE` / `CODE_TRACKER_TEST_SCHEMA_PREFIX` if set, otherwise falls back to runtime DB vars.
  - `CODE_TRACKER_TEST_ENABLE_DB` to decide whether to hit the DB or skip DB work.
- Steps:
  1. If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log that DB tests are disabled and exit `0`.
  2. Otherwise, call:

     ```js
     trackModuleCode(
       CODE_TRACKER_TEST_SAMPLE_PACKAGE,
       "test-branch",
       "v0.0.0-test"
     );
     ```

  3. Log the summary object and assert basic shape (moduleSlug, moduleId, counts).
  4. Exit non-zero on failure.

---

### 6.2 fileScanner.mjs

**Responsibility**

- Given a root folder path, recursively return `.mjs` file paths.
- Exclude `node_modules`, `.git`, and any additional ignore list.

**Self-test**

- Uses `CODE_TRACKER_TEST_SCANNER_ROOT` as the root directory; defaults to `./test-data/fileScanner`.
- Steps:
  1. Ensure `CODE_TRACKER_TEST_SCANNER_ROOT` exists.
  2. Create a small test directory tree:
     - `a.mjs`, `b.mjs`, `ignore.txt`, `nested/nested.mjs`.
  3. Invoke the scanner on the root.
  4. Log results and confirm only `.mjs` files are included (assertions).
  5. Optionally clean up test files/directories.

---

### 6.3 sourceParser.mjs

**Responsibility**

- Given `.mjs` file paths and/or raw source, parse:
  - Exported function names.
  - Parameter names/types (from JSDoc).
  - Return type (from JSDoc, where possible).

**Self-test**

- Uses inline **sample source strings**; does **not** rely on filesystem.
- Steps:
  1. Define 2–3 simple source snippets with JSDoc comments and exported functions.
  2. Pass them into `parseSourceStrings([...])`.
  3. Log the parsed structures.
  4. Verify expected function names, parameter counts, and JSDoc types are present.

---

### 6.4 pgClient.mjs

**Responsibility**

- Expose a function that returns a configured Postgres client using the **runtime env vars** (with optional test overrides).

**Self-test**

- Reads:
  - `CODE_TRACKER_TEST_DATABASE` (preferred DB name for tests; falls back to `CODE_TRACKER_PG_DATABASE` if missing).
  - `CODE_TRACKER_TEST_ENABLE_DB` to decide whether to attempt a connection.
- Steps:
  1. If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log that DB tests are disabled and exit `0`.
  2. Otherwise, create a client pointing at the test DB.
  3. Attempt a simple `SELECT 1`.
  4. Log success or a clear failure message.
  5. Exit non-zero on error, but do not throw uncaught exceptions.

---

### 6.5 schemaInit.mjs

**Responsibility**

- Given a Postgres client, ensure required tables exist using `CREATE TABLE IF NOT EXISTS` / `CREATE SCHEMA IF NOT EXISTS`.
- Tables hold:
  - A `modules` table keyed by `moduleSlug` / `moduleId`.
  - A `snapshots` table keyed by `(moduleId, branchName, releaseTag)`.
  - A `functions` table with metadata for each function.

**Self-test**

- Uses same DB resolution rules as `pgClient.mjs`:
  - Prefer `CODE_TRACKER_TEST_DATABASE`.
  - Respect `CODE_TRACKER_TEST_ENABLE_DB`.
- Steps:
  1. If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log that DB tests are disabled and exit `0`.
  2. Otherwise:
     - Acquire a client (ideally via `getPgClient` with the test DB).
     - Run `initSchema(client)`.
     - Query `information_schema` or pg_catalog to confirm tables exist.
  3. Log results, exit non-zero on failure.

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
  1. Build minimal function metadata in-memory (1–2 fake functions).
  2. Call `storeSnapshot` with:
     - `branchName="selftest-branch"`, `releaseTag="v0.0.0-selftest"`.
  3. Query back the inserted rows.
  4. Log results and exit non-zero if nothing is inserted or shapes are wrong.
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

## 8. README Content Requirements

The README generated for `codeTrackerJS` MUST follow these rules.

### 8.1 Overview of Self-tests

- Explain that **each module includes an embedded self-test block** using:

  ```js
  if (import.meta.url === `file://${process.argv[1]}`) {
    // self-test
  }
  ```

- Explain that the **same `.env` file** configures both:
  - runtime behaviour; and
  - self-test behaviour (via `CODE_TRACKER_TEST_...` variables).

### 8.2 Per-module Self-test Documentation

For each module (`codeTrackerOrchestrator.mjs`, `fileScanner.mjs`, `sourceParser.mjs`, `pgClient.mjs`, `schemaInit.mjs`, `storeSnapshot.mjs`), the README must:

1. Describe in 1–2 sentences **what the self-test does**.
2. List exactly **which environment variables** it relies on, including relevant ones from:
   - `CODE_TRACKER_TEST_SAMPLE_PACKAGE`
   - `CODE_TRACKER_TEST_SCANNER_ROOT`
   - `CODE_TRACKER_TEST_DATABASE`
   - `CODE_TRACKER_TEST_SCHEMA_PREFIX`
   - `CODE_TRACKER_TEST_ENABLE_DB`
   - plus any runtime `CODE_TRACKER_PG_*` vars if they matter for that test.
3. Show **how to run the self-test from the CLI**, for example:

   ```bash
   node codeTrackerOrchestrator.mjs
   node fileScanner.mjs
   node sourceParser.mjs
   node pgClient.mjs
   node schemaInit.mjs
   node storeSnapshot.mjs
   ```

4. Briefly state **what a successful run looks like**, e.g.:
   - “Logs a summary object to the console.”
   - “Logs a list of found `.mjs` files.”
   - “Logs a successful `SELECT 1` against the database.”
   - “Logs inserted snapshot rows.”

### 8.3 How to Supply Self-test Variables

The README must include a section titled **“Configuring Self-tests”** which explains:

- Edit `.env` in the `codeTrackerJS/` folder.
- Set or override the `CODE_TRACKER_TEST_...` variables, for example:

  ```env
  CODE_TRACKER_TEST_SAMPLE_PACKAGE=./test-data/samplePackage
  CODE_TRACKER_TEST_SCANNER_ROOT=./test-data/fileScanner
  CODE_TRACKER_TEST_DATABASE=code_tracker_test
  CODE_TRACKER_TEST_SCHEMA_PREFIX=test_
  CODE_TRACKER_TEST_ENABLE_DB=true
  ```

- If `CODE_TRACKER_TEST_ENABLE_DB=false`, DB-dependent self-tests (`pgClient.mjs`, `schemaInit.mjs`, `storeSnapshot.mjs`, and the DB parts of the orchestrator self-test) will:
  - log that DB tests are disabled; and
  - skip DB work gracefully without throwing.

### 8.4 README and Versioning

- The README must mention that this package is implemented against **project plan v2.2.0**.
- It should remind the reader to:
  - run all module self-tests at least once after configuration; and
  - create a stable Git tag (e.g., `codeTrackerJS-v1.0.0`) once tests pass.

---

## 9. Versioning

- This file is **v2.2.0**.
- The builder should reference this exact filename in its `.env` configuration.
