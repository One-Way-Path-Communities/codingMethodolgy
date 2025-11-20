# codeTrackerJS — Project Plan v2.3.0

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

---

## 6. Modules + Independent Test Requirements (with Explicit APIs)

### 6.0 Shared Types and Conventions

All modules use **named exports only** (no default exports).

#### 6.0.1 `FunctionMetadata` shape

The `sourceParser` is responsible for producing function metadata in the following shape, which is then consumed by `storeSnapshot` and used by the orchestrator to calculate summary counts:

```js
/**
 * @typedef {Object} JsDocParam
 * @property {string} name
 * @property {string} [type]
 * @property {string} [description]
 */

/**
 * @typedef {Object} JsDocReturns
 * @property {string} [type]
 * @property {string} [description]
 */

/**
 * @typedef {Object} FunctionMetadata
 * @property {string} filePath           // absolute or module-relative .mjs path
 * @property {string} exportName         // exported function name
 * @property {JsDocParam[]} params       // parameters from JSDoc (in order)
 * @property {JsDocReturns|null} returns // return info from JSDoc or null if absent
 * @property {string} [description]      // optional JSDoc description
 */
```

#### 6.0.2 `CodeTrackerSummary` shape

The orchestrator returns the following shape:

```js
/**
 * @typedef {Object} CodeTrackerSummary
 * @property {string} moduleSlug
 * @property {number} moduleId
 * @property {string} branchName
 * @property {string} releaseTag
 * @property {number} functionsCount
 * @property {number} parametersCount
 * @property {number} returnValuesCount
 */
```

---

### 6.1 `codeTrackerOrchestrator.mjs`

#### Responsibility

- Validate inputs.
- Derive `moduleSlug` from `moduleFolder`.
- Use `fileScanner` → list of `.mjs` files.
- Use `sourceParser` → array of `FunctionMetadata`.
- Use `pgClient` + `schemaInit` → ensure DB schema exists.
- Use `storeSnapshot` → persist versioned snapshot for `(moduleSlug, branchName, releaseTag)`.
- Compute & return `CodeTrackerSummary`.

#### Public API (exports)

```js
/**
 * Orchestrates a full tracking run for a given module folder.
 *
 * @param {string} moduleFolder   Absolute or relative path to the package/module folder.
 * @param {string} branchName     Git branch name (e.g. "main", "feature/code-tracker").
 * @param {string} releaseTag     Release tag or version (e.g. "v1.0.0").
 * @returns {Promise<CodeTrackerSummary>}
 */
export async function trackModuleCode(moduleFolder, branchName, releaseTag) {}
```

- **Return value**: a `Promise` that resolves to a `CodeTrackerSummary` object.

> The orchestrator is the **only module** that should export `trackModuleCode`. Other modules must not reuse this name.

#### Self-test

Self-test is triggered via:

```js
if (import.meta.url === `file://${process.argv[1]}`) {
  // self-test
}
```

It calls:

```js
await trackModuleCode(
  CODE_TRACKER_TEST_SAMPLE_PACKAGE || "./test-data/samplePackage",
  "test-branch",
  "v0.0.0-test"
);
```

and then asserts that the returned value matches `CodeTrackerSummary` (including non-negative integer counts).  
If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log that DB tests are disabled and exit `0` instead of hitting the DB.

---

### 6.2 `fileScanner.mjs`

#### Responsibility

- Given a root folder path, recursively return `.mjs` file paths.
- Exclude `node_modules`, `.git`, and any additional ignore list.
- No DB or parsing responsibility.

#### Public API (exports)

```js
/**
 * Recursively scans a folder for .mjs files.
 *
 * @param {string} rootFolder                 Root folder to start scanning from.
 * @param {string[]} [extraIgnoreFolders=[]]  Additional folder names to ignore at any depth.
 * @returns {Promise<string[]>}               Promise resolving to a list of absolute or
 *                                            root-relative .mjs file paths.
 */
export async function scanMjsFiles(rootFolder, extraIgnoreFolders = []) {}
```

- **Return value**: a `Promise` resolving to an array of `.mjs` file paths.

> The orchestrator will always call **`scanMjsFiles`** and expect a `Promise<string[]>`.

#### Self-test

Uses `CODE_TRACKER_TEST_SCANNER_ROOT` (or `./test-data/fileScanner`) to:

1. Ensure the root exists (create if necessary).
2. Create a small test tree:
   - `a.mjs`, `b.mjs`, `ignore.txt`, `nested/nested.mjs`.
3. Call `scanMjsFiles(root, [])`.
4. Assert that only `.mjs` files are returned.
5. Optionally clean up after itself.

---

### 6.3 `sourceParser.mjs`

#### Responsibility

- Parse exported functions and their JSDoc metadata from `.mjs` sources.
- Provide both:
  - a file-oriented parsing API (`parseFiles`), and
  - a direct string-based API (`parseSourceStrings`) used by the self-test.

#### Public API (exports)

```js
/**
 * Parses a list of .mjs files from disk and returns function metadata.
 *
 * @param {string[]} filePaths           List of .mjs file paths to parse.
 * @returns {Promise<FunctionMetadata[]>} Promise resolving to function metadata.
 */
export async function parseFiles(filePaths) {}

/**
 * Parses in-memory source strings and returns function metadata.
 *
 * @param {{ filePath: string, source: string }[]} sources
 *        Array of objects with a pseudo filePath label and the full source code.
 * @returns {FunctionMetadata[]}        Synchronous parse result.
 */
export function parseSourceStrings(sources) {}
```

- **Return values**:
  - `parseFiles`: `Promise<FunctionMetadata[]>`.
  - `parseSourceStrings`: `FunctionMetadata[]`.

> The orchestrator calls **`parseFiles`**; the self-test calls **`parseSourceStrings`**.

#### Self-test

Defines a few small source snippets, e.g.:

```js
const samples = [
  {
    filePath: "sample1.mjs",
    source: `
      /**
       * Adds two numbers.
       * @param {number} a
       * @param {number} b
       * @returns {number}
       */
      export function add(a, b) { return a + b; }
    `
  },
  // ...
];

const result = parseSourceStrings(samples);
```

Then asserts each `FunctionMetadata` has the expected `exportName`, `params`, and `returns`.

---

### 6.4 `pgClient.mjs`

#### Responsibility

- Create and return a configured Postgres client using env vars from section 4.
- Centralize DB connection configuration logic (including test overrides).

#### Public API (exports)

```js
/**
 * Creates and connects a Postgres client using env configuration.
 *
 * Env resolution:
 * - If options.databaseOverride is provided, use that database name.
 * - Else if CODE_TRACKER_TEST_DATABASE is set, use that for tests.
 * - Else fall back to CODE_TRACKER_PG_DATABASE.
 *
 * @param {{ databaseOverride?: string }=} options
 * @returns {Promise<import("pg").Client>}  Connected PG client. Caller must call client.end().
 */
export async function getPgClient(options = {}) {}
```

- **Return value**: a `Promise` resolving to a **connected** `pg.Client`.

> All other modules that need a DB connection should call **`getPgClient`** instead of using `pg` directly.

#### Self-test

- If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log and exit `0`.
- Otherwise, call:

  ```js
  const client = await getPgClient();
  const { rows } = await client.query("SELECT 1 AS ok");
  await client.end();
  ```

- Assert that `rows[0].ok === 1`.  
- Exit non-zero on failure, but do not throw uncaught exceptions.

---

### 6.5 `schemaInit.mjs`

#### Responsibility

- Ensure required schema and tables exist:
  - `modules`
  - `snapshots`
  - `functions`
- Optionally apply a schema/table prefix, e.g., `test_`.

#### Public API (exports)

```js
/**
 * Ensures the database schema and required tables exist.
 *
 * @param {import("pg").Client} client
 *        Connected Postgres client (e.g. from getPgClient()).
 * @param {{ schemaPrefix?: string }=} options
 *        Optional prefix for schemas/tables (e.g. "test_").
 * @returns {Promise<void>}
 */
export async function initSchema(client, options = {}) {}
```

- **Return value**: `Promise<void>`.

> The orchestrator will always call **`initSchema(client, { schemaPrefix })`** before `storeSnapshot`.

#### Self-test

- If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log and exit `0`.
- Otherwise:
  1. Obtain client via `getPgClient({ databaseOverride: CODE_TRACKER_TEST_DATABASE })` if set, otherwise default.
  2. Call `await initSchema(client, { schemaPrefix: CODE_TRACKER_TEST_SCHEMA_PREFIX })`.
  3. Query `information_schema` (or `pg_catalog`) to confirm tables exist.
  4. Log results, exit non-zero on failure.
  5. Close client.

---

### 6.6 `storeSnapshot.mjs`

#### Responsibility

- Given `moduleSlug`, `(branchName, releaseTag)`, and `FunctionMetadata[]`:
  - Ensure a module row exists (or create it).
  - Delete any existing snapshot rows for that `(moduleId, branchName, releaseTag)`.
  - Insert:
    - a new snapshot row, and
    - function metadata rows linked to that snapshot.

#### Public API (exports)

```js
/**
 * Stores a versioned snapshot of function metadata for a module.
 *
 * @param {import("pg").Client} client
 *        Connected Postgres client.
 * @param {string} moduleSlug
 *        Slug derived from moduleFolder (e.g. "serverjs" or "newsletterjs").
 * @param {string} branchName
 * @param {string} releaseTag
 * @param {FunctionMetadata[]} functions
 *        Parsed function metadata from sourceParser.
 * @param {{ schemaPrefix?: string }=} options
 *        Optional schema/table prefix (e.g. "test_").
 * @returns {Promise<{
 *   moduleId: number,
 *   snapshotId: number,
 *   functionsCount: number
 * }>}
 */
export async function storeSnapshot(
  client,
  moduleSlug,
  branchName,
  releaseTag,
  functions,
  options = {}
) {}
```

- **Return value**: a `Promise` resolving to an object containing:
  - `moduleId` (primary key from `modules` table),
  - `snapshotId` (primary key from `snapshots` table),
  - `functionsCount` (number of function rows inserted for this snapshot).

> The orchestrator uses **`functions.length`** plus `functions[*].params` and `functions[*].returns` (from `FunctionMetadata`) to compute `parametersCount` and `returnValuesCount`; it does **not** require the DB to return these aggregates.

#### Self-test

- If `CODE_TRACKER_TEST_ENABLE_DB === "false"`, log and exit `0`.
- Otherwise:
  1. Get a client via `getPgClient({ databaseOverride: CODE_TRACKER_TEST_DATABASE })`.
  2. Call `initSchema(client, { schemaPrefix: CODE_TRACKER_TEST_SCHEMA_PREFIX })`.
  3. Build a simple in-memory `FunctionMetadata[]`, e.g.:

     ```js
     const functions = [
       {
         filePath: "selftest.mjs",
         exportName: "exampleFn",
         params: [{ name: "x", type: "number" }],
         returns: { type: "number", description: "example" },
         description: "Self-test function"
       }
     ];
     ```

  4. Call:

     ```js
     const result = await storeSnapshot(
       client,
       "codeTrackerJS-selftest",
       "selftest-branch",
       "v0.0.0-selftest",
       functions,
       { schemaPrefix: CODE_TRACKER_TEST_SCHEMA_PREFIX }
     );
     ```

  5. Assert `result.functionsCount === functions.length`.
  6. Query the DB back for this module/branch/tag to confirm rows exist.
  7. Optionally clean up test rows, then `client.end()`.

---

### 6.7 How the Orchestrator Uses These APIs (Summary)

Inside `trackModuleCode`, the call sequence is **fixed** and depends on the exact function names/signatures defined above:

```js
import { scanMjsFiles } from "./fileScanner.mjs";
import { parseFiles } from "./sourceParser.mjs";
import { getPgClient } from "./pgClient.mjs";
import { initSchema } from "./schemaInit.mjs";
import { storeSnapshot } from "./storeSnapshot.mjs";

export async function trackModuleCode(moduleFolder, branchName, releaseTag) {
  // 1. derive moduleSlug from moduleFolder
  // 2. const filePaths = await scanMjsFiles(moduleFolder);
  // 3. const functions = await parseFiles(filePaths);
  // 4. const client = await getPgClient(...);
  // 5. await initSchema(client, ...);
  // 6. const { moduleId } = await storeSnapshot(
  //        client,
  //        moduleSlug,
  //        branchName,
  //        releaseTag,
  //        functions,
  //        ...
  //    );
  // 7. compute functionsCount, parametersCount, returnValuesCount from `functions`
  // 8. return CodeTrackerSummary
}
```

Because **every exported function name, parameter list, and return value is now explicitly specified**, the builder/orchestrator generator can safely construct `codeTrackerOrchestrator.mjs` directly from this plan without any guesswork about module APIs.

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

### 8.3 How to Supply Self-tests Variables

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

- The README must mention that this package is implemented against **project plan v2.3.0**.
- It should remind the reader to:
  - run all module self-tests at least once after configuration; and
  - create a stable Git tag (e.g., `codeTrackerJS-v1.0.0`) once tests pass.

---

## 9. Versioning

- This file is **v2.3.0**.
- The builder should reference this exact filename in its `.env` configuration.
