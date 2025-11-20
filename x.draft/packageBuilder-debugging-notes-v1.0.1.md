# Package Builder Debugging Notes (v1.0.0)

This file is meant to travel *with* the project plan and coding policy so the package builder can remind ChatGPT of common pitfalls when generating ES modules.

## 1. ES Module `import` Statements Must Be Top-Level

**Rule**

In ES modules, all `import` statements must live at the **top level** of the file. You **cannot** put `import` inside:

- `if` statements
- functions
- loops
- or any other block

**What to do instead**

- Use **static imports** at the top of the file:

  ```js
  import fs from "node:fs/promises";
  import path from "node:path";
  ```

- If you absolutely need to load something only in a self-test block or at runtime, use **dynamic import**:

  ```js
  if (import.meta.url === `file://${process.argv[1]}`) {
    (async () => {
      const dotenvModule = await import("dotenv");
      dotenvModule.config();
    })();
  }
  ```

**Common symptom**

Errors like:

```
SyntaxError: Unexpected token '*'
```

---

## 2. Every Module Must Export At Least One Symbol

Each module must define and export at least one named symbol. Example:

```js
export async function myFunction() {}
```

Common failure:

```
SyntaxError: The requested module './fileScanner.mjs' does not provide an export named 'fileScanner'
```

---

## 3. `.env` Files Are **Not** Loaded Automatically (Use `dotenv`)

Node does **not** read a `.env` file automatically. Any module that depends on env vars must ensure `.env` is loaded.

### If module has a self-test:

Use:

```js
import "dotenv/config";
```

OR instruct the user to run:

```bash
node -r dotenv/config myFile.mjs
```

### If `.env` is loaded by orchestrator:

- Do **not** load `dotenv` again in modules.
- README must specify how `.env` is loaded.

Common symptom:

- Code works under main app but fails standalone.
- Errors like “password authentication failed” because `process.env` values were missing.

---

## 4. Usage

When generating code from package builder:

- Provide coding policy + project plan + this file.
- Instruct ChatGPT to:
  - Keep imports top-level.
  - Ensure each module exports correctly.
  - Load `.env` explicitly when needed.
