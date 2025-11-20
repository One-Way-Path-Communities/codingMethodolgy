# Coding Policy and Procedures for One Way Path Communities  
### Version 1.0.1 — Terminology Update (Packages vs Modules)

## Terminology Update (New in 1.0.1)
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
- Release plan  

### 3.2 Package-Level Plans
Each **package** needs a plan with:
- Purpose  
- Internal modules  
- Exports / Public API  
- Required environment variables  
- Any API routes or CLI entry points  
- Testing approach  

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

### 6.2 JSDoc for Exported Functions
Every exported function in a module requires JSDoc:

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
- Add/update API routing using Hybrid Helper Pattern  
- Create/update README  
- Maintain proper `.env` loading  
- Suggest version tagging  
- Validate compliance with this policy  

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

# End of Version 1.0.1
