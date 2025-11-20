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
      // self-test code here
    })();
  }
  ```

**Common symptom**

You see an error like:

```txt
SyntaxError: Unexpected token '*'
```

or

```txt
SyntaxError: Cannot use import statement outside a module
```

when an `import` appears inside a block.

---

## 2. Every Module Must Export at Least One Symbol

**Rule**

Each generated ES module must define and export at least one symbol (usually a named function). The orchestrator and other modules will import these by **name**, so missing or mismatched exports will break the package.

**What this means for generated code**

- At minimum, each module should have:

  ```js
  export async function someModuleFunction(/* args */) {
    // implementation
  }
  ```

- The export name must match how other modules import it, for example:

  ```js
  // fileScanner.mjs
  export async function fileScanner(rootDir) { /* ... */ }

  // orchestrator
  import { fileScanner } from "./fileScanner.mjs";
  ```

**Common symptom**

You see an error like:

```txt
SyntaxError: The requested module './fileScanner.mjs' does not provide an export named 'fileScanner'
```

which means the file either:

- didn’t export anything, or
- used a different export name than the one being imported.

---

## 3. How to Use This Debugging File

When the package builder calls ChatGPT to generate code, it should:

- Provide:
  - the coding policy
  - the project plan
  - **this debugging notes file**
- Explicitly instruct ChatGPT to:
  - Keep imports top-level or use dynamic import in self-tests.
  - Ensure each module exports at least one named symbol that matches the import in other modules.

More common issues can be added as we discover them.
