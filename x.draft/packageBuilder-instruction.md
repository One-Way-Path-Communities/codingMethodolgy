# Module Builder Script — Instruction Specification

You are a senior Node.js engineer. Create a “module builder” script that uses the OpenAI API to generate a **MODULE** for my project.

A **MODULE** = a folder containing:

- one or more `.mjs` files (each file may contain one or more functions)  
- an orchestrator `.mjs` file (separate from the numbered files)  
- a `.env` template file for environment variables  
- a README snippet for the module  

**No API routes, no docker-compose updates, no patch file.**  
API wiring will be handled later in a second script.

The script itself must follow my **Coding Policy and Procedures for One Way Path Communities**.

---

## 📥 INPUTS  
Inputs are simple variables defined at the **top of the script** (NO CLI, NO JSON):

### 1. Project plan name / location
- A short name for the module (e.g., `"Newsletter Module"`).  
- A file path to a **project plan/spec**.  
- The project plan contains:
  - A numbered list of files to create (`1`, `2`, `3`, …)  
  - Requirements for each file  
  - A separate description for the **orchestrator file**

The script reads the **entire** project plan text and passes it to ChatGPT for context.  
The script does *not* parse the plan beyond loading it as a string.

### 2. Coding policy
- A file path to `coding-policy-and-procedures.md`.  
- The script loads the full text and includes it in the **system/developer prompt** for every API call.

### 3. Module folder location / name
- The base folder where the module will be created (e.g., `"./newsletterJS"`).  
- Folders for JS modules **must end in `JS`**.  
- The script must **create this folder** if it does not exist.

### 4. Number of FILES (excluding orchestrator)
- An integer (`NUM_FILES`) defining how many numbered files to generate.  
- Example: `NUM_FILES = 3` → generate **file 1**, **file 2**, **file 3**, then generate the orchestrator separately.  
- The orchestrator is *not* counted here.

---

## 📤 OUTPUTS  
The script must generate the following inside the module folder:

---

### 1. Numbered `.mjs` files (files `1..NUM_FILES`)
Each generated `.mjs` file must:

- Follow the coding policy:
  - **One main responsibility per file**
  - Clear, well-named functions
  - JSDoc for every exported function (summary, `@param`, `@returns`, example usage)
  - Embedded self-test block:
    ```js
    if (import.meta.url === `file://${process.argv[1]}`) {
      ...
    }
    ```
- Use naming conventions:
  - Functions: `camelCase`
  - File names: exactly as defined by the plan or by the script’s naming rule

---

### 2. Orchestrator `.mjs` file
- A separate `.mjs` file described in the project plan  
- Follows all coding policy rules (single responsibility, JSDoc, self-test)  
- Does **not** contain Express or API wiring at this stage  

---

### 3. `.env` template
- File name: `.env.<moduleName>.example`  
- Contains only environment variables the module needs  
- Placeholder values only  
- Format:
  ```
  KEY=value   # comment
  ```

---

### 4. README snippet
A README file (e.g., `README.<moduleName>.md`) or snippet containing:

- Module purpose  
- What each `.mjs` file does (including orchestrator)  
- Required environment variables  
- How to run module self-tests (`node filename.mjs`)  
- Reminder to tag a stable release version (`v1.0.0`)  

---

## 🤖 OPENAI USAGE

The module-builder script must:

- Load the **full coding policy** and include it in **every** system/developer prompt  
- Load the **full project plan** and include it in **every** user prompt  

### For each numbered file (i = 1..NUM_FILES):

Send:

- full coding policy  
- full project plan  

With instruction:

> “Build **file i** as described in the plan.  
> Return **only** the complete `.mjs` file content for file i.”

### For the orchestrator:

> “Build the **orchestrator file** described in the plan.  
> Return **only** the complete `.mjs` file content for the orchestrator.”

### For the `.env` file:

> “Return **only** the `.env.<moduleName>.example` content.”

### For the README:

> “Return **only** the README markdown for this module.”

### All calls run in **parallel**

- Create one promise per call  
- Use `Promise.all` to wait for all responses  
- Write results to files when resolved  

---

## 🛠 SCRIPT REQUIREMENTS

The script must:

- Be written in Node.js ES modules (`.mjs`)  
- Use `fs/promises`  
- Use the official OpenAI client  
- Read API key from `process.env.OPENAI_API_KEY`  
- Define all inputs as constants at the top (plan path, policy path, module folder, module name, NUM_FILES)  
- Create the module folder if needed  
- Build a **promise list**:
  - 1 promise per numbered file  
  - 1 for orchestrator  
  - 1 for `.env`  
  - 1 for README  
- Save all results when promises resolve  
- Log filenames created  

