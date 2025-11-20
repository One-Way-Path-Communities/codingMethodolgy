# Package Builder Script — Instruction Specification  
### Version 1.0.1 (Updated Terminology: Packages vs Modules)

## Terminology Update (1.0.1)
- **Package** = a folder containing multiple ES modules implementing a complete feature.  
- **Module** = a single `.mjs` ES module inside a package.

---

## 1. Purpose
Create a **Package Builder** script that uses the OpenAI API to generate an entire **PACKAGE** for the project.

A **PACKAGE** includes:  
- Several `.mjs` modules  
- One orchestrator module  
- A `.env` template  
- A README snippet  

(No routing, no API wiring, no docker changes.)

---

## 2. Inputs

### Defined as constants at the top of the script:

#### 1. Project Plan
- Short name of the **package**.
- File path to a **project plan** containing:
  - Numbered list of modules (1..N)
  - Requirements for each module
  - Orchestrator module description  
- Script loads full text **without parsing**.

#### 2. Coding Policy  
- File path to the coding policy (e.g., `coding-policy-and-procedures.md`).  
- Full text included in system/developer prompt for **every** OpenAI call.

#### 3. Package Folder Name  
- E.g., `"./newsletterJS"`  
- **Must end with `JS`**  
- Created automatically by this script if missing.

#### 4. Number of Modules  
- `NUM_MODULES` = # of module files to generate (excluding orchestrator)  
- Orchestrator is generated separately.

---

## 3. Outputs

### Inside the package folder:

#### 1. Numbered modules (`1..NUM_MODULES`)
Each must follow coding policy:
- One responsibility per module  
- JSDoc for all exported functions  
- Self-test block:
  ```js
  if (import.meta.url === `file://${process.argv[1]}`) {
    ...
  }
  ```
- camelCase functions, `.mjs` file naming from plan or rules.

#### 2. Orchestrator Module
- Implements orchestration logic from project plan  
- JSDoc + self-test  
- No express routing  

#### 3. `.env` Template  
Named:  
```
.env.<packageName>.example
```

Includes only required environment variables with placeholder values.

#### 4. README Snippet  
Contains:
- Package purpose  
- Explanation of each module  
- Env vars  
- Self-test instructions  
- Reminder to create a stable tag (v1.0.0)  

---

## 4. OpenAI Usage

Each generation call includes:
- **Coding policy** (system/developer)  
- **Project plan** (user)

### Module N instruction:
> Build module N as described.  
> Return **only** the `.mjs` file content.

### Orchestrator:
> Build the orchestrator module.  
> Return **only** the `.mjs` content.

### Env template:
> Return **only** the `.env.<packageName>.example` content.

### README:
> Return **only** the README markdown.

All calls run **in parallel** using `Promise.all`.

---

## 5. Script Requirements

- Written in Node.js ES modules (`.mjs`)  
- Uses `fs/promises`  
- Official OpenAI client  
- Reads API key from `process.env.OPENAI_API_KEY`  
- Defines all inputs as constants  
- Creates package folder if missing  
- Builds a promise list:
  - One per generated module  
  - One for orchestrator  
  - One for `.env`  
  - One for README  
- Saves all files when promises resolve  
- Logs file creation  

---

# End of Version 1.0.1
