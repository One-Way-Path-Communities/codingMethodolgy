# Coding Policy and Procedures for One Way Path Communities  
### Version 1.0.4 — Project Plan, Export Contracts, DB Schema & Install Script Requirements

## Terminology Update (From 1.0.1)
- **Package** = a folder containing multiple ES modules that implement a complete feature.  
- **Module** = a single `.mjs` ES module within a package.

---

## 1. Purpose & Scope
This document defines coding standards, naming conventions, workflow practices, and documentation rules for all OWP software. Applies to:
- Node/Express applications
- JavaScript packages
- Python packages
- Scripts inside hosted apps (Airtable, Asana, Office)

---

## 2. Development Process & “Don’t Reinvent the Wheel”

### 2.1 Tool Selection Order
Use the simplest viable tool:
1. Existing app features  
2. Automations inside apps  
3. Zapier zaps  
4. Custom API coding  

### 2.2 When to Use Custom Code
Only when simpler tools cannot meet complexity, reliability, or integration requirements.

---

## 3. Coding Plan & Repository Setup

### 3.1 Coding Plan Requirements
Every repo must start with a `CODING_PLAN.md` containing:
- Problem statement  
- Planned **packages**  
- For each package: list of ES **modules**  
- API surfaces  
- Environment variables  
- Testing strategy  
- README requirements (see 3.2)  
- Release plan  

### 3.2 Package-Level Plans
Each **package** needs a package-level plan document (e.g. `codeTrackerJS-project-plan-vX.Y.Z.md`) with:

- **Purpose** of the package.  
- **Internal modules** and their responsibilities.  
- **Exports / Module Contracts**:
  - For each module, the project plan must specify the **exact exported function names** that other modules/orchestrators are allowed to call.
  - For each exported function, specify the **parameters and return shape** (e.g., key fields on returned objects or summary types).
  - For packages with an orchestrator, the orchestrator must rely **only** on these documented exports. If a function is not in the project plan, the orchestrator should not call it.
- **Database Schema (if the package uses a database)**:
  - The project plan must define the **canonical schema** for all tables the package depends on: table names, column names, data types, primary keys, foreign keys, and any uniqueness constraints.
  - If prefixes (e.g., `schemaPrefix` or test prefixes like `test_`) are used, the plan must specify exactly how they are applied to table names.
  - Helper modules such as `schemaInit` and persistence modules (e.g., `storeSnapshot`) must be written to match the schema defined in the project plan. The DB schema must not be left for ChatGPT to infer.

- **Required environment variables**, split into:
  - Runtime configuration (e.g. DB connection, API keys)
  - Self-test configuration (e.g. `*_TEST_*` variables)  
- **Self-test behaviour per module**, including:
  - What each module’s self-test does.
  - Which environment variables it reads.
  - Whether it can run without external services.
- **README content requirements**, including:
  - How to run each module’s self-test from the command line.
  - What variables need to be supplied (via `.env`) for each self-test.
  - What a successful run looks like.
- **Install script requirement**:
  - Each package must define a bash script named e.g. `scripts/install-deps.sh` that installs all required dependencies for that package (Node modules like `pg`, `dotenv`, `comment-parser`, etc., and any other tools the self-tests rely on).
  - The project plan must list what that script is responsible for installing.

### 3.3 Relationship to ChatGPT and the Package Builder
When using ChatGPT or the package builder to generate code:

- The **project plan** is the single source of truth for:
  - module responsibilities,
  - self-test expectations,
  - env var usage, and
  - README content.
  - **module export contracts** (the exact exported function names, parameters, and return values that orchestrators and helper modules must use).
  - **database schema** details for any package that uses a database (tables, columns, keys, constraints, and prefixes).
- The builder must provide:
  - this Coding Policy,
  - the package-level project plan,
  - and any debugging-notes file (if present).
- ChatGPT must follow:
  - the self-test env rules in the plan,
  - the README documentation requirements,
  - and ensure that the install script covers the dependencies implied by the generated code.

---

## 4. Code Organization, Structure & Naming

### 4.1 Packages and Modules
- A **package** is a folder named `*JS` (JavaScript) or `*PT` (Python).  
- A **module** is a single `.mjs` file inside a package.  
- Each module must handle one responsibility.

### 4.2 Naming Conventions
- **Packages:** `newsletterJS`, `codeTrackerJS`, `golfScorePostJS`  
- **Modules:** camelCase, e.g., `fileScanner.mjs`  
- **Routes:** kebab-case  
- **Database:** lower_snake_case  
- **URLs:** kebab-case  

### 4.3 Releases & Versioning
Every repo and each significant package requires at least one stable release (e.g., `v1.0.0`).  
Use semantic versioning.

---

## 5. API Integration & Routing

### 5.1 Packages Expose an API Surface
Any package meant for broader use MUST expose:
- A set of exported functions, **and/or**
- An Express router (via the Hybrid Helper Pattern)

### 5.2 Hybrid Helper Pattern (Unchanged)
Routes are registered centrally using `routes/registerRoutes.mjs`.

### 5.3 Environment Loading
Each package should load its own `.env` file.  
This keeps `server.mjs` unchanged when adding packages.

---

## 6. Documentation Requirements

### 6.1 README Files
Each package must include a README with:
- Table of Contents  
- Purpose  
- Installation & usage  
- API surface  
- **Self-test instructions**, as dictated by that package’s project plan:
  - How to run each module’s self-test (exact `node` command).
  - Which env vars must be supplied for each.
  - Expected outputs and success criteria.

### 6.2 JSDoc for Exported Functions
Every exported function in a module requires JSDoc, and its JSDoc **must match the export contract defined in the package’s project plan** (name, parameters, and return shape):

```js
/**
 * Summary...
 * @param {...}
 * @returns {...}
 * Example:
 */
```

### 6.3 Embedded Self-Tests
Each module must include:

```js
if (import.meta.url === `file://${process.argv[1]}`) {
  // Self-test
}
```

Self-tests must respect the self-test env vars defined in the project plan (`*_TEST_*`), and avoid crashing with unhandled errors.

---

## 7. Code in Hosted Apps  
(Airtable, Asana, Office)

Scripts there must:
- Follow modular structure  
- Include function-level JSDoc  
- Include inline self-tests (when possible)  

---

## 8. ChatGPT Procedural Checklist
ChatGPT must always:
- Check simpler options first  
- Produce/update the package-level coding plan  
- Use correct terminology (package vs module)  
- Follow naming conventions  
- Ensure modular structure  
- Add/update API routing using Hybrid Helper Pattern (where relevant)  
- Create/update README per project-plan instructions  
- Maintain proper `.env` loading  
- Suggest version tagging  
- Validate compliance with this policy  
- Ensure that for any package with an orchestrator, the package-level project plan defines explicit **module export contracts** (exported function names, parameters, and return values) for each module, and that generated code adheres to them.

- Ensure each package has:
  - a defined self-test strategy and env vars in the project plan; and
  - a `scripts/install-deps.sh` (or equivalent) to install required dependencies.

---

## 9. Appendix: Example Snippet

### Example JSDoc + Self-Test

```js
export function parseRoundInfo(text) {
  return {};
}

if (import.meta.url === `file://${process.argv[1]}`) {
  console.log(parseRoundInfo("demo text"));
}
```

---

# End of Version 1.0.4
