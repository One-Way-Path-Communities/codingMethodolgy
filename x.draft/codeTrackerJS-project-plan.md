# codeTrackerJS — Full Package Project Plan (Postgres Backend, Version-Aware)

## 1. Purpose

The **codeTrackerJS package** automatically analyzes another package’s `.mjs` files and stores structured information about its exported functions in a **Postgres database**.

It supports **version-aware snapshots**, so each run stores a snapshot tied to:

- `module_slug` (derived from folder name)
- `branch_name`
- `release_tag`

Snapshots are stored without overwriting other versions.

No HTTP router is included in **v0.1**.  
This package is fully automated, code-driven, and requires **zero manual database schema setup** beyond supplying Postgres credentials.

---

## 2. High-Level Flow

1. User runs the orchestrator with:
   ```js
   trackModuleCode(moduleFolder, branchName, releaseTag)
   ```
2. The package scans the target folder, finds all `.mjs` files.
3. Parses each file:
   - exported functions  
   - parameters and return types from JSDoc
4. Builds a structured descriptor.
5. Connects to Postgres using env vars you supply.
6. Ensures tables exist.
7. Deletes existing snapshot for `(module_slug, branch, tag)` only.
8. Inserts a fresh snapshot.
9. Returns summary.

---

## 3. Inputs & Outputs

### Inputs

```js
trackModuleCode(
  moduleFolder,
  branchName,
  releaseTag
)
```

### Output Summary

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

---

## 4. Environment Variables

```env
CODE_TRACKER_PG_HOST=localhost
CODE_TRACKER_PG_PORT=5432
CODE_TRACKER_PG_DATABASE=code_tracker
CODE_TRACKER_PG_USER=code_tracker_user
CODE_TRACKER_PG_PASSWORD=supersecret
CODE_TRACKER_PG_SSL=false
CODE_TRACKER_LOG_LEVEL=info
```

---

## 5. Package Structure

```
codeTrackerJS/
  codeTrackerOrchestrator.mjs
  fileScanner.mjs
  sourceParser.mjs
  pgClient.mjs
  schemaInit.mjs
  storeSnapshot.mjs
  README.md
  .env.codetracker
```

---

## 6. Description of ES Modules

### 6.1 codeTrackerOrchestrator.mjs
Main entry point.  
Builds descriptor, calls storeSnapshot, returns summary.

### 6.2 fileScanner.mjs
Scans for `.mjs` files.

### 6.3 sourceParser.mjs
Parses exports and JSDoc using AST.

### 6.4 pgClient.mjs
Connects to Postgres using env vars.

### 6.5 schemaInit.mjs
Creates necessary tables with IF NOT EXISTS.

### 6.6 storeSnapshot.mjs
Handles deletion + insertion of versioned snapshot within a transaction.

---

## 7. Database Schema

### modules

```sql
CREATE TABLE IF NOT EXISTS modules (
  id SERIAL PRIMARY KEY,
  module_slug TEXT NOT NULL,
  name TEXT,
  folder_path TEXT,
  branch_name TEXT NOT NULL,
  release_tag TEXT NOT NULL,
  commit_sha TEXT,
  last_scanned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (module_slug, branch_name, release_tag)
);
```

### functions

```sql
CREATE TABLE IF NOT EXISTS functions (
  id SERIAL PRIMARY KEY,
  module_id INTEGER NOT NULL REFERENCES modules(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  file_path TEXT NOT NULL,
  export_type TEXT NOT NULL,
  is_async BOOLEAN NOT NULL DEFAULT FALSE,
  description TEXT,
  jsdoc_raw TEXT,
  order_in_file INTEGER
);
```

### parameters

```sql
CREATE TABLE IF NOT EXISTS parameters (
  id SERIAL PRIMARY KEY,
  function_id INTEGER NOT NULL REFERENCES functions(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  position INTEGER NOT NULL,
  type TEXT,
  is_optional BOOLEAN NOT NULL DEFAULT FALSE,
  is_rest BOOLEAN NOT NULL DEFAULT FALSE,
  default_value TEXT,
  description TEXT
);
```

### return_values

```sql
CREATE TABLE IF NOT EXISTS return_values (
  id SERIAL PRIMARY KEY,
  function_id INTEGER NOT NULL REFERENCES functions(id) ON DELETE CASCADE,
  type TEXT,
  description TEXT
);
```

---

## 8. Deletion Logic

Delete only snapshot with matching:

- module_slug
- branch_name
- release_tag

Cascading FK deletes child rows.

---

## 9. Testing

- Self-tests inside each module  
- End-to-end:
  ```
  node codeTrackerOrchestrator.mjs ./newsletterJS main v1.3.1
  ```

---

## 10. Release Plan

### v0.1.0
- Full Postgres backend  
- Version-aware snapshotting  
- Automatic table creation  
- CLI only  
- Six ES modules inside one package

### v0.2.0-pre
- Optional git auto-checkout  
- Optional HTTP route wrapper  
- Optional export-to-Airtable view

### v1.0.0
- Integrate with documentation API  
- Diff view  
- Stable schema

---
