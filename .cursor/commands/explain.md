Explain the structure of the Excalidraw monorepo to the user. Tailor the depth to what they ask; default to a high-level architectural overview.

Steps:
1. Read `package.json` at the root to identify workspaces.
2. List the top-level directories and briefly state each one's role.
3. For each workspace package under `packages/`, read its `package.json` and `README.md` (if present) to understand its purpose.
4. Read `vitest.config.mts` and `.eslintrc.json` to understand testing and linting setup.
5. Check `excalidraw-app/` for the main application entry point.

Then produce a structured explanation:

---
## Excalidraw Monorepo — Architecture Overview

### Workspace Layout
| Workspace | Path | Purpose |
|-----------|------|---------|
| excalidraw-app | `/excalidraw-app` | The deployed web application at excalidraw.com |
| @excalidraw/excalidraw | `/packages/excalidraw` | Core editor React component (the npm-published package) |
| @excalidraw/element | `/packages/element` | Element types, mutations, geometry |
| @excalidraw/common | `/packages/common` | Shared utilities, constants, types |
| @excalidraw/math | `/packages/math` | Vector/matrix math primitives |
| @excalidraw/utils | `/packages/utils` | General-purpose helpers |

### Key Architectural Patterns
- **State management:** Jotai atoms, accessed only via `editor-jotai` / `app-jotai` wrappers
- **Testing:** Vitest + jsdom, co-located test files (`*.test.ts` / `*.test.tsx`)
- **Linting:** ESLint with `@excalidraw/eslint-config` + stricter rules for the core package
- **Formatting:** Prettier via `@excalidraw/prettier-config`
- **Build:** Vite for the app, esbuild for packages

### Data Flow (simplified)
User interaction → React event handlers in `App.tsx` → Action system (`packages/excalidraw/actions/`) → State mutations via `mutateElement` (`packages/element`) → Re-render

### Entry Points
- App: `excalidraw-app/index.tsx`
- Core package: `packages/excalidraw/index.tsx`
- Tests: configured in `vitest.config.mts`, setup in `setupTests.ts`
---

If the user asks about a specific subsystem, read the relevant files and provide a deeper explanation of that area.
