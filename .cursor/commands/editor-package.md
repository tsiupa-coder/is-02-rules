You are about to edit code inside one of the `packages/` workspaces (excalidraw, element, common, math, or utils). Follow this checklist before and after making changes.

## Pre-Edit Checklist

1. **Identify the correct package** — read the task description and confirm the change belongs in:
   - `packages/excalidraw/` — editor UI, actions, hooks, scene, renderer
   - `packages/element/` — element types, mutation logic, geometry, bounding boxes
   - `packages/common/` — shared constants, utility functions, types used across packages
   - `packages/math/` — vector/matrix math, point types (`Point` from `types.ts`)
   - `packages/utils/` — general-purpose helpers with no Excalidraw-specific knowledge

2. **Check the package boundary** — run:
   ```bash
   grep -r "firebase\|socket\.io\|import\.meta\.env" packages/excalidraw/ packages/element/ packages/common/
   ```
   Any match means app-layer code has leaked into a library package — do not add more.

3. **Read the relevant module** before editing. Use the Read tool on the target file and its co-located test file (`*.test.ts` / `*.test.tsx`).

4. **Check existing exports** — if adding a new export to a package, verify it isn't already exported somewhere else:
   ```bash
   grep -rn "export.*<FunctionOrTypeName>" packages/
   ```

## Editing Standards (enforced by CLAUDE.md)

- Use `import type` for all type-only imports.
- No imports from barrel files inside `packages/excalidraw/` (no `@excalidraw/excalidraw` self-imports in non-test files).
- No direct `jotai` imports — use `editor-jotai` or `app-jotai`.
- Catch clauses: `catch (error: unknown)`, never `catch (error: any)`.
- For math code: always import `Point` from `packages/math/src/types.ts`, never use `{ x, y }` inline object shapes.

## Post-Edit Checklist

1. **Run type checking on the affected package:**
   ```bash
   yarn test:typecheck
   ```

2. **Run lint on the changed files:**
   ```bash
   yarn test:code
   ```

3. **Run the test suite:**
   ```bash
   yarn test:app --watch=false
   ```

4. **If you added or changed a public export**, update the package's `index.ts` / `index.tsx` barrel if the export should be part of the public API. If it should be internal only, do NOT add it to the barrel.

5. **Check if a test exists** for the changed logic. If not, note it explicitly so the user knows coverage may have dropped.

## Output Format

After completing edits, report:
```
## Editor Package Change Summary
- Package: @excalidraw/<name>
- Files changed: [list]
- New exports: [list or "none"]
- Tests updated: yes / no / n/a
- Type check: PASS / FAIL
- Lint: PASS / FAIL (N errors, N warnings)
- Test suite: PASS / FAIL
- Notes: [anything unusual]
```
