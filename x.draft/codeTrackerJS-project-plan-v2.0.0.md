# codeTrackerJS — Project Plan v2.0.0

## 1. Purpose
The **codeTrackerJS** package analyzes another package’s `.mjs` files and stores structured information about exported functions into a **Postgres database**, with proper version-aware snapshots for branch + releaseTag.  
It should run entirely programmatically with zero manual schema setup.

## 2. High-Level Flow
1. Orchestrator receives:
   - moduleFolder
   - branchName
   - releaseTag
2. Scans folder for `.mjs` files.
3. Parses functions + JSDoc.
4. Prepares Postgres tables.
5. Writes a snapshot (safe delete+insert).
6. Returns summary.

## 3. Inputs / Outputs
### Input
trackModuleCode(moduleFolder, branchName, releaseTag)

### Output
{
  moduleSlug,
  moduleId,
  branchName,
  releaseTag,
  functionsCount,
  parametersCount,
  returnValuesCount
}

## 4. Environment Variables (.env inside codeTrackerJS/)
CODE_TRACKER_PG_HOST=localhost
CODE_TRACKER_PG_PORT=5432
CODE_TRACKER_PG_DATABASE=code_tracker
CODE_TRACKER_PG_USER=code_tracker_user
CODE_TRACKER_PG_PASSWORD=supersecret
CODE_TRACKER_PG_SSL=false
CODE_TRACKER_LOG_LEVEL=info

## 5. Package Structure
codeTrackerJS/
  orchestrator.mjs
  fileScanner.mjs
  sourceParser.mjs
  pgClient.mjs
  schemaInit.mjs
  storeSnapshot.mjs
  README.md
  .env

## 6. Modules + Independent Test Requirements

### 6.1 orchestrator.mjs
- Validate inputs.
- Derive moduleSlug.
- Use fileScanner → file list.
- Use sourceParser → function metadata.
- Use pgClient + schemaInit → DB ready.
- Use storeSnapshot → write versioned snapshot.
- Self-test:
  - Use ./test-data/samplePackage with tiny .mjs files.
  - Use safe env vars.
  - Call trackModuleCode("./test-data/samplePackage", "test-branch", "v0.0.0-test").
  - Log summary.

### 6.2 fileScanner.mjs
- Recursively returns `.mjs` files.
- Excludes node_modules, .git.
- Self-test:
  - Create tmp folder structure in ./test-data/fileScanner.
  - Include a.mjs, b.mjs, ignore.txt, nested/nested.mjs.
  - Verify only mjs files returned.

### 6.3 sourceParser.mjs
- Parse each `.mjs` file.
- Extract exported function names, params, return values from JSDoc.
- Self-test:
  - Provide raw source strings directly.
  - Include 2–3 simple exports with JSDoc.
  - Assert parsed shapes.

### 6.4 pgClient.mjs
- Create and expose a Postgres client with env vars.
- Self-test:
  - Connect + simple SELECT 1 query.
  - If DB missing, log graceful error and exit non-zero.

### 6.5 schemaInit.mjs
- Ensure required tables exist (CREATE TABLE IF NOT EXISTS).
- Self-test:
  - Run against DB.
  - Confirm table existence via SELECT.

### 6.6 storeSnapshot.mjs
- Delete old snapshot for (moduleSlug, branchName, releaseTag).
- Insert new snapshot and detail records.
- Self-test:
  - Use temporary test identifiers.
  - Insert minimal snapshot.
  - Query back and log results.

## 7. Test Philosophy
- **Each module must be independently testable** without requiring the others.
- Tests should supply **local sample data** or **mocked inputs**.
- Only orchestrator test uses the integrated flow.
- Modules must not crash if DB is unreachable—log and safe-exit only.

## 8. Versioning
This file is **v2.0.0** and should be referenced by builder scripts.
