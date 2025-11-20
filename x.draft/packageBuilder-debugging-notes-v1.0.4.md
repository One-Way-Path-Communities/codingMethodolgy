# Package Builder Debugging Notes (v1.0.4)

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

**Each module that has a self-test MUST load `.env` on its own using:**

```
import "dotenv/config";
```

This ensures self-tests work when modules are run directly via `node module.mjs`.

The orchestrator MAY also load `.env`, but that does **not** remove the need for modules to load it independently.
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

## 4. CRITICAL RULE: Imports (including dotenv) MUST be top-level

This rule overrides any ambiguity in earlier text. The package builder and all generated modules MUST follow it:

1. **All `import` statements MUST appear at the very top of the file**, before any other code (except comments).
2. **Never put `import` inside a self-test block, function, if-statement, or any other block.**
3. This includes the dotenv line:

   ```js
   import "dotenv/config"; // ✅ allowed ONLY at top level
   ```

   ❌ The following patterns are illegal and WILL cause `SyntaxError: Unexpected string`:

   ```js
   // BAD: inside self-test
   if (import.meta.url === `file://${process.argv[1]}`) {
     import "dotenv/config";
   }

   // BAD: inside a function
   async function selfTest() {
     import "dotenv/config";
   }
   ```

4. For modules with self-tests, do it this way instead:

   ```js
   // Top of file
   import "dotenv/config";
   import fs from "node:fs/promises";
   // ... other imports ...

   // Later, self-test block (NO imports here)
   if (import.meta.url === `file://${process.argv[1]}`) {
     (async () => {
       const root = process.env.CODE_TRACKER_TEST_SCANNER_ROOT || "./test-data/fileScanner";
       // self-test logic...
     })();
   }
   ```

If there is any conflict between this rule and other wording, **this rule wins**. All generated code MUST treat imports as top-level only, including dotenv.

## 5. JSDoc comments must not contain a literal `*/` inside the text

Inside a JSDoc block (`/** ... */`), the sequence `*/` **always** closes the comment.
If you include `*/` in the description text, the comment will end early and the next
line (often starting with `* @returns`) will be parsed as real JavaScript code.

This commonly causes errors like:

```txt
SyntaxError: Unexpected token '*'
```

### BAD example

```js
/**
 * Parses the JSDoc comment block string.
 *
 * @param {string} comment The full JSDoc comment block including /** and */
 * @returns {{ ... }}
 */
```

In the line:

```js
* @param {string} comment The full JSDoc comment block including /** and */
```

the `*/` at the end closes the comment, so `* @returns` on the next line is treated as code.

### GOOD example

Rephrase the description so it does **not** contain `*/` literally:

```js
/**
 * Parses the JSDoc comment block string.
 *
 * @param {string} comment The full JSDoc comment block, including the opening and closing markers.
 * @returns {{ ... }}
 */
```

Rule of thumb: if you need to talk about the comment delimiters, describe them in words instead of writing `*/` literally inside the JSDoc block.
