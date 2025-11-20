# Package Builder Debugging Notes (v1.0.5)

This file is meant to travel *with* the project plan and coding policy so the package builder can remind ChatGPT of **non-negotiable rules** when generating ES modules.

> **Purpose:** These notes are to be read and applied **before any code is generated or tested**, so that the resulting modules do **not** hit these known failure modes.

When you invoke ChatGPT via the package builder, you must instruct it to:

- Read this file fully.
- Treat each rule as a **hard constraint**, not a suggestion.
- Apply the “Pre-generation checklist” at the end to its own output.

---

## 1. ES Module `import` Statements Must Be Top-Level

### Rule

In ES modules, all `import` statements must live at the **top level** of the file. You **cannot** put `import` inside:

- `if` statements  
- functions  
- loops  
- or any other block

### What to do instead

- Use **static imports** at the top of the file:

  ```js
  import fs from "node:fs/promises";
  import path from "node:path";
  ```

- If you need to load something only during a self-test or at runtime, use **dynamic import** *inside* the block:

  ```js
  if (import.meta.url === `file://${process.argv[1]}`) {
    (async () => {
      const dotenvModule = await import("dotenv");
      dotenvModule.config();
    })();
  }
  ```

### Common symptom

Errors like:

```txt
SyntaxError: Unexpected token '*'
```

> **Important:** The same error can also come from **JSDoc comments that accidentally contain `*/`**.  
> See Rule 5 (JSDoc) below.

---

## 2. Every Module Must Export At Least One Symbol

Each module must define and export at least one named symbol.

Example (good):

```js
export async function myFunction() {}
```

Common failure:

```txt
SyntaxError: The requested module './fileScanner.mjs' does not provide an export named 'fileScanner'
```

This usually means:

- The file forgot to export anything, or  
- The export name doesn’t match what the orchestrator or other modules import.

**Prevention when generating code:**

- For each `.mjs` file, ensure there is at least one `export` that matches the project plan.
- If the plan says “exported function name: `fileScanner`”, the module must include:

  ```js
  export async function fileScanner(...) { ... }
  ```

---

## 3. `.env` Files Are **Not** Loaded Automatically (Use `dotenv`)

Node does **not** read a `.env` file automatically. Any module that depends on env vars must ensure `.env` is loaded.

### If module has a self-test

The module itself must load the environment once at the **top of the file**:

```js
import "dotenv/config";
```

OR the README may instruct:

```bash
node -r dotenv/config myFile.mjs
```

…but for your system:

> **Rule:** Every module that has a self-test MUST include  
> `import "dotenv/config";` at the top of the file, so that running:
>
> ```bash
> node module.mjs
> ```
> works with no extra flags.

### If `.env` is loaded by orchestrator

Even if the orchestrator also loads `.env`:

- Each module with a self-test **STILL** needs:

  ```js
  import "dotenv/config";
  ```

- The README must state clearly how `.env` is handled.

### Common symptom

- Code works under the main app / orchestrator but fails standalone.
- Errors like “password authentication failed” because `process.env` values were missing.

---

## 4. CRITICAL RULE: Imports (including dotenv) MUST Be Top-Level

This rule overrides any ambiguity in earlier text. The package builder and all generated modules MUST follow it:

1. **All `import` statements MUST appear at the very top of the file**, before any other code (except comments).
2. **Never put `import` anywhere else** — not inside a self-test block, not in a function, not in an `if` statement, not in a loop.
3. This includes the dotenv import:

   ```js
   import "dotenv/config"; // ✅ allowed ONLY at top level
   ```

   ❌ Illegal patterns that WILL cause errors:

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

If there is any conflict between this rule and other wording, **this rule wins**.  
All generated code MUST treat imports as top-level only, including dotenv.

---

## 5. JSDoc Comments Must Never Contain the Literal `*/` Inside the Text

Inside a JSDoc block (`/** ... */`), the sequence `*/` **always** closes the comment.

If you include `*/` in the description text, the comment will end early and the next line (often starting with `* @returns`) will be parsed as real JavaScript code.

This commonly causes errors like:

```txt
SyntaxError: Unexpected token '*'
```

### BAD example (do NOT generate this)

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

### GOOD example (what ChatGPT must generate)

Rephrase the description so it does **not** contain `*/` literally:

```js
/**
 * Parses the JSDoc comment block string.
 *
 * @param {string} comment The full JSDoc comment block, including the opening and closing markers.
 * @returns {{ ... }}
 */
```

### Mandatory prevention rules for ChatGPT

When generating any `.mjs` file:

1. **Never** output the literal text `*/` inside a JSDoc block comment.
2. If you need to talk about comment delimiters, do one of the following instead:
   - Use words only, e.g.  
     “the opening and closing JSDoc comment markers”
   - Or break the sequence, e.g.  
     `'* /'`, `'*\/''`, or similar, so that the raw `*/` never appears.
3. Do **not** include raw `*/` in:
   - descriptions,
   - `@param` text,
   - `@returns` text,
   - examples inside JSDoc comments.

> **Rule of thumb:** if you need to refer to the comment delimiters, describe them verbally or escape them. Never write `*/` literally inside `/** ... */`.

---

## 6. Usage: How the Package Builder Must Apply These Notes

When generating code from the package builder:

1. Provide:
   - Coding policy  
   - Project plan  
   - **This debugging notes file (v1.0.5)**

2. Explicitly instruct ChatGPT to:
   - Read the debugging notes fully before generating any code.
   - Treat every rule as a **hard requirement**, not a suggestion.
   - Apply the **pre-generation checklist** (below) to every file it outputs.

---

## 7. Pre-Generation Checklist (ChatGPT Must Self-Check)

For **each module file** (`*.mjs`) that ChatGPT produces, it must mentally verify:

1. **Imports**
   - [ ] All `import` statements are at the very top of the file (after comments only).
   - [ ] No `import` appears inside functions, self-test blocks, conditionals, or loops.
   - [ ] `import "dotenv/config";` (if present) is also at top level.

2. **Exports**
   - [ ] The file exports at least one symbol (`export function ...`, `export const ...`, etc.).
   - [ ] Export names match what the project plan and orchestrator expect.

3. **Environment**
   - [ ] Any module with a self-test includes `import "dotenv/config";` at the top.
   - [ ] Assumptions about `.env` loading are clearly consistent with the README requirements.

4. **JSDoc**
   - [ ] No JSDoc comment contains the literal `*/` in its text.
   - [ ] Any mention of comment delimiters is paraphrased or escaped.
   - [ ] `@param` / `@returns` tags are syntactically valid and not accidentally closing the comment.

If any of these checks fail, ChatGPT must adjust the code **before** presenting it as final output.

## 8. No Duplicate Exported Declarations for the Same Name

### Rule

Within a single `.mjs` file, a given exported name must be declared **only once**.

Bad pattern (will cause a syntax error):

```js
export async function parseFiles(filePaths) {
  // old stub implementation
}

// ... later in the same file ...

export async function parseFiles(filePaths) {
  // real implementation
}

## 9. Syntax check: no stray quotes or unbalanced parentheses

### Rule

All generated `.mjs` files (including patch scripts) **must be valid JavaScript syntax** and should pass:

```bash
node --check <filename>.mjs

---

