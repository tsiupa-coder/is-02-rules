# Claude Code Rules for Excalidraw Monorepo

This file defines the standards Claude must follow when working in this repository.

---

## Rule 1: Always Use Type-Only Imports for Type Annotations

When importing TypeScript types or interfaces, always use the `import type` syntax, never mix type and value imports in the same statement.

**Correct:**
```ts
import type { ExcalidrawElement } from "@excalidraw/element/types";
import { arrayToMap } from "@excalidraw/common";
```

**Incorrect:**
```ts
import { ExcalidrawElement, arrayToMap } from "@excalidraw/common";
```

**How to verify:**
Run `yarn test:code` and look for ESLint errors from the `@typescript-eslint/consistent-type-imports` rule. Zero errors means this rule is satisfied. You can also run `npx eslint --rule '{"@typescript-eslint/consistent-type-imports": "error"}' path/to/file.ts` on any individual file.

---

## Rule 2: Never Import from Package Barrel Files Within `packages/excalidraw`

Non-test source files inside `packages/excalidraw/` must never import from their own package barrel (`index.tsx` or via `@excalidraw/excalidraw`). Always use direct relative imports to the specific module.

**Correct (inside packages/excalidraw):**
```ts
import { mutateElement } from "../element/mutateElement";
```

**Incorrect:**
```ts
import { mutateElement } from "@excalidraw/excalidraw";
import { mutateElement } from "../../index";
```

**How to verify:**
Run `yarn test:code`. The ESLint rule `@typescript-eslint/no-restricted-imports` (configured in `.eslintrc.json` under the `packages/excalidraw/**` override) will flag any such imports with the message: *"Do not import from the barrel 'index.tsx' files."*

---

## Rule 3: Never Import Directly from `jotai`

All state atoms must be accessed through the repo's own Jotai wrappers: `editor-jotai` (for the editor package) or `app-jotai` (for the app). Direct imports from the `jotai` package are forbidden everywhere.

**Correct:**
```ts
import { atom } from "../editor-jotai";
```

**Incorrect:**
```ts
import { atom } from "jotai";
```

**How to verify:**
Run `yarn test:code`. The ESLint `no-restricted-imports` rule (configured in `.eslintrc.json`) will report the error: *"Do not import from \"jotai\" directly."* You can also `grep -r "from \"jotai\"" packages/ excalidraw-app/` — any match is a violation.

---

## Rule 4: Maintain Import Group Order with Newlines Between Groups

All imports must be ordered in this exact sequence, with a blank line separating each group:
1. `builtin` (Node.js built-ins)
2. `external` (third-party packages, then `@excalidraw/*` packages after)
3. `internal`
4. `parent` (relative paths going up: `../`)
5. `sibling` (relative paths at same level: `./`)
6. `index`
7. `object`
8. `type` (type-only imports last)

**How to verify:**
Run `yarn test:code` and look for warnings from the `import/order` ESLint rule. Any file with mis-ordered or missing newlines between groups will be reported. To auto-fix: `yarn fix:code`.

---

## Rule 5: All Tests Must Pass the Coverage Thresholds

When adding new code, ensure the overall test suite does not drop below these thresholds (defined in `vitest.config.mts`):

| Metric     | Minimum |
|------------|---------|
| Lines      | 60%     |
| Branches   | 70%     |
| Functions  | 63%     |
| Statements | 60%     |

New logic — especially utility functions and data transformations — must have accompanying tests in a `*.test.ts` or `*.test.tsx` file co-located with the source file.

**How to verify:**
Run `yarn test:coverage --watch=false`. If any threshold is breached, Vitest will exit with a non-zero code and list the offending metrics. A clean run prints all thresholds as green.

---

## Rule 6: Commit Messages Must Follow Conventional Commits Format

Every commit message must use the Conventional Commits specification:

```
<type>(<optional scope>): <short description>

[optional body]
```

Valid types: `feat`, `fix`, `chore`, `refactor`, `test`, `docs`, `style`, `perf`, `ci`.

**Examples:**
```
feat(clipboard): support pasting mixed image+text content
fix(renderer): correct bounding box for rotated elements
chore(deps): bump vitest to 3.0.7
test(element): add coverage for group selection edge cases
```

Rules:
- Description is lowercase, imperative mood, no trailing period.
- Scope is the package or major subsystem (e.g., `clipboard`, `element`, `app`, `collab`).
- Body wraps at 72 characters.

**How to verify:**
Run `git log --oneline -10` and inspect the most recent commits. Each line must match the pattern `type(scope): description`. You can also run:
```bash
git log --format="%s" -20 | grep -vP "^(feat|fix|chore|refactor|test|docs|style|perf|ci)(\(.+\))?: .+"
```
Any output indicates a non-compliant commit message.

---

## Rule 7: All Formatting Must Pass Prettier

CSS, SCSS, JSON, Markdown, HTML, and YAML files must be formatted with the project's Prettier configuration (`@excalidraw/prettier-config`). Never hand-format these files.

**How to verify:**
Run `yarn test:other` (which runs `prettier --list-different` across all target files). A clean run produces no output. Any listed file has formatting that diverges from Prettier's output. To auto-fix: `yarn fix:other`.

---

## Rule 8: File Structure — New Files Must Follow Package Conventions

All new source files must be placed in the correct workspace and follow co-location rules:

| What you're adding | Where it goes |
|--------------------|---------------|
| Element geometry / mutation logic | `packages/element/src/` |
| Shared utilities / constants / types | `packages/common/src/` |
| Math primitives (vectors, matrices) | `packages/math/src/` |
| Editor UI components / hooks | `packages/excalidraw/` |
| App-level concerns (Firebase, collab, sharing) | `excalidraw-app/` |
| Tests for a module `foo.ts` | Co-located as `foo.test.ts` in the same directory |

Additional rules:
- Component files use `.tsx`; pure logic files use `.ts`.
- CSS for a component lives alongside it as `ComponentName.scss` (CSS modules).
- Never add application-layer code (Firebase, socket.io) to `packages/excalidraw/` — that package must remain embeddable without app-layer dependencies.
- New packages under `packages/` require a `package.json`, `tsconfig.json`, and `README.md`.

**How to verify:**
1. Check that the file path matches the table above.
2. Run `yarn test:typecheck` — misplaced imports that violate the workspace boundaries will produce type errors.
3. Grep for app-layer imports in the core package: `grep -r "firebase\|socket.io" packages/excalidraw/` — any match is a violation.

---

## Rule 9: Error Handling — Use `unknown`, Preserve Cause, No Silent Swallowing

All error handling must follow these standards:

**Catch clauses must use `unknown`, not `any`:**
```ts
// CORRECT
try { ... } catch (error: unknown) {
  throw new Error("Operation failed", { cause: error });
}

// INCORRECT
try { ... } catch (error: any) {
  throw new Error(error.message);   // discards stack trace, unsafe cast
}
```

**Never silently swallow errors** (catch and ignore):
```ts
// INCORRECT
try { riskyOperation(); } catch (_) { /* ignore */ }

// CORRECT — at minimum log with context
try { riskyOperation(); } catch (error: unknown) {
  console.error("riskyOperation failed:", error);
}
```

**Async functions must have error boundaries** — any `async` function that can throw must either propagate the error or handle it explicitly. Do not leave unhandled promise rejections.

**How to verify:**
1. `grep -rn "catch (error: any\|catch (err: any\|catch (e: any" packages/ excalidraw-app/` — any match is a violation.
2. `grep -rn "catch.*{[\s]*}" packages/ excalidraw-app/` — empty catch blocks are violations.
3. TypeScript with `useUnknownInCatchVariables: true` (planned for a future `tsconfig.json` update) will flag `catch (e: any)` automatically.

---

## Rule 10: Security — Validate External Input, No Unsafe DOM APIs, Use Web Crypto Only

Security rules that apply to all new code in this repo:

**1. Never use `dangerouslySetInnerHTML`** unless the input has been explicitly sanitized with an approved library (e.g., DOMPurify). The app processes user-generated content; raw HTML injection creates XSS vectors.
```tsx
// INCORRECT
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// CORRECT
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

**2. Always validate URL protocol before navigation or fetch:**
```ts
// INCORRECT
window.open(userProvidedUrl);

// CORRECT
const safe = new URL(userProvidedUrl);
if (!["https:", "http:"].includes(safe.protocol)) {
  throw new Error(`Blocked unsafe URL protocol: ${safe.protocol}`);
}
window.open(safe.href);
```

**3. Use only the Web Crypto API (`window.crypto.subtle`) for cryptographic operations** — never implement custom encryption, roll your own random, or use `Math.random()` for anything security-sensitive. AES-GCM with 128-bit keys is the project standard (see `packages/excalidraw/data/encryption.ts`).

**4. Environment secrets belong in `.env.*` files, never in source code.** Firebase config, API keys, and similar values must be accessed via `import.meta.env.VITE_APP_*`.

**How to verify:**
1. `grep -rn "dangerouslySetInnerHTML" packages/ excalidraw-app/` — any new occurrence without a DOMPurify call nearby is a violation.
2. `grep -rn "window\.open\|location\.href\|location\.assign" packages/ excalidraw-app/` — review each call to confirm URL validation is present.
3. `grep -rn "Math\.random" packages/ excalidraw-app/` — any use in security-sensitive paths (key generation, token creation) is a violation.
4. `grep -rn "VITE_APP_" packages/ excalidraw-app/` — all secret references should use `import.meta.env`; hardcoded strings are violations.
