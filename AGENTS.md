# AGENTS.md — AI Agent Guide for the Excalidraw Monorepo

This file documents how AI agents (Claude Code and similar) should understand, navigate,
and contribute to the Excalidraw monorepo.

---

## Project Overview

**Excalidraw** is an open-source, browser-based virtual whiteboard application for creating
hand-drawn style diagrams. It is a TypeScript/React monorepo managed with Yarn Workspaces.

- **Live app:** https://excalidraw.com
- **npm package:** `@excalidraw/excalidraw` (the embeddable editor component)
- **License:** MIT
- **Stack:** TypeScript, React 19, Vite, Vitest, Jotai, jsdom

The repo serves two audiences simultaneously:
1. End users of the hosted app (`excalidraw-app/`)
2. Developers who embed the editor via the npm package (`packages/excalidraw/`)

---

## Agent Architecture

This repo does not ship autonomous runtime agents. Instead, the "agent architecture" describes
how AI coding assistants (Claude Code) are structured to work with the codebase:

```
┌─────────────────────────────────────────────┐
│              Claude Code Session             │
│                                              │
│  ┌──────────┐   ┌──────────┐   ┌─────────┐  │
│  │ /review  │   │  /test   │   │/explain │  │
│  │  agent   │   │  agent   │   │  agent  │  │
│  └────┬─────┘   └────┬─────┘   └────┬────┘  │
│       │              │              │        │
│       └──────────────┼──────────────┘        │
│                      │                       │
│            ┌─────────▼──────────┐            │
│            │   CLAUDE.md rules  │            │
│            │  (enforced on all  │            │
│            │   agent outputs)   │            │
│            └────────────────────┘            │
└─────────────────────────────────────────────┘
```

Each slash command in `.claude/commands/` is a **task agent**: a focused, stateless prompt
that Claude Code executes in the context of the current working directory. Task agents are:
- **Stateless** — each invocation starts fresh from the current git state
- **Rule-bound** — all outputs must comply with CLAUDE.md
- **Tool-enabled** — agents can read files, run shell commands, and produce structured output

```
.claude/commands/
├── review.md          → /review         Code review of changed files
├── test.md            → /test           Full test suite runner + summary
├── explain.md         → /explain        Codebase architecture overview
├── editor-package.md  → /editor-package Pre/post checklist for package-level edits
└── pr-checks.md       → /pr-checks      Full pre-PR validation checklist
```

---

## Available Agents

### `/review` — Code Review Agent

**File:** `.claude/commands/review.md`  
**Invocation:** Type `/review` in the Claude Code prompt  
**Purpose:** Reviews all files changed since the last commit against the rules in CLAUDE.md.

What it does:
1. Identifies changed files via `git diff HEAD --name-only`
2. Reads each changed file and its diff
3. Checks: import types, barrel imports, jotai usage, import order, test coverage, security
4. Produces a per-file PASS/FAIL report
5. Issues an overall APPROVED / NEEDS CHANGES verdict

**Best used when:** Before committing, to catch rule violations and bugs early.

---

### `/test` — Test Runner Agent

**File:** `.claude/commands/test.md`  
**Invocation:** Type `/test` in the Claude Code prompt  
**Purpose:** Runs the full test suite and produces a human-readable summary.

What it does:
1. `yarn test:typecheck` — TypeScript compilation check
2. `yarn test:code` — ESLint with zero-warning policy
3. `yarn test:other` — Prettier formatting check
4. `yarn test:app --watch=false` — Vitest unit tests
5. Summarizes each step's result with counts and failure details

**Best used when:** After making changes, to verify nothing is broken before pushing.

---

### `/explain` — Codebase Explainer Agent

**File:** `.claude/commands/explain.md`  
**Invocation:** Type `/explain` in the Claude Code prompt  
**Purpose:** Produces a structured architectural overview of the monorepo.

What it does:
1. Reads workspace manifests to identify all packages
2. Explains each package's role and its relationship to others
3. Describes key patterns: state management, testing, build, data flow
4. Lists entry points for the app and the npm package

**Best used when:** Onboarding to the codebase, or before making cross-package changes.

---

### `/editor-package` — Package Edit Guide

**File:** `.claude/commands/editor-package.md`  
**Invocation:** Type `/editor-package` in the Claude Code prompt  
**Purpose:** Step-by-step checklist for safely editing code inside any `packages/*` workspace.

What it does:
1. Confirms the correct package for the change
2. Checks package boundary constraints (no Firebase/env-var leakage into library packages)
3. Reads the target file and its test before editing
4. Verifies no existing export is being duplicated
5. Runs typecheck, lint, and tests after the edit
6. Reports a structured change summary

**Best used when:** Making changes to `packages/excalidraw`, `packages/element`, `packages/common`, etc.

---

### `/pr-checks` — Pre-PR Validation Agent

**File:** `.claude/commands/pr-checks.md`  
**Invocation:** Type `/pr-checks` in the Claude Code prompt  
**Purpose:** Runs the full set of checks a reviewer would perform before approving a PR.

What it does:
1. Summarizes commits and diff stats for the branch
2. Audits all commit messages for Conventional Commits format
3. Validates file placement against CLAUDE.md Rule 8 (file structure)
4. Runs `yarn test:typecheck`, `yarn test:code`, `yarn test:other`
5. Runs a security scan for: `dangerouslySetInnerHTML`, unsafe navigation, `Math.random`, `catch (any)`
6. Runs the full test suite
7. Checks coverage thresholds (for PRs with significant new logic)
8. Produces a PASS/FAIL table + APPROVED / NEEDS CHANGES verdict

**Best used when:** Before opening a PR or requesting review.

---

## How to Run

### Prerequisites
```bash
node --version   # must be >= 18.0.0
yarn --version   # 1.22.x (classic)
```

### Install dependencies
```bash
yarn install
```

### Start the development server
```bash
yarn start
# Opens the app at http://localhost:3000
```

### Run all checks (what CI runs)
```bash
yarn test:all
# Equivalent to: test:typecheck && test:code && test:other && test:app --watch=false
```

### Run only unit tests
```bash
yarn test:app --watch=false
```

### Run tests with coverage
```bash
yarn test:coverage
```

### Invoke a slash command agent
In Claude Code, type the command at the prompt:
```
/review          # code review of changed files
/test            # run full test suite with summary
/explain         # codebase architecture overview
/editor-package  # checklist for editing package-level code
/pr-checks       # full pre-PR validation
```

---

## How to Add a New Agent

1. **Create the command file** at `.claude/commands/<name>.md`.
   The filename becomes the slash command: `/name`.

2. **Write a clear task description** at the top — one or two sentences describing what the
   agent does and when to use it.

3. **Write numbered steps** the agent must follow. Be explicit about:
   - Which shell commands to run
   - Which files to read
   - What output format to produce

4. **Define the output format** clearly. Structured output (tables, headers, PASS/FAIL labels)
   is easier to scan than prose.

5. **Reference CLAUDE.md** if the agent makes any code-quality judgements — agents must
   enforce the same rules as human reviewers.

**Example skeleton:**
```markdown
# .claude/commands/my-agent.md

One-sentence description of what this agent does.

Steps:
1. Run `git status` to understand current state.
2. Read the relevant source files.
3. Perform the analysis.
4. Output a summary in this format:

---
## My Agent Output
- Finding 1: ...
- Finding 2: ...
---
```

6. **Test the command** by typing `/my-agent` in a Claude Code session and verifying the
   output matches the defined format.

---

## Dependencies

### Runtime (app)
| Package | Purpose |
|---------|---------|
| `react` / `react-dom` | UI rendering |
| `jotai` | Atomic state management (accessed via `editor-jotai` / `app-jotai` wrappers) |
| `roughjs` | Hand-drawn rendering style |
| `socket.io-client` | Real-time collaboration |
| `firebase` | Persistence for the hosted app |
| `idb-keyval` | IndexedDB wrapper for local storage |
| `i18next` | Internationalization |

### Development / Build
| Package | Purpose |
|---------|---------|
| `typescript` 5.9 | Type checking |
| `vite` 5 | Development server and app bundler |
| `vitest` 3 | Unit test runner |
| `eslint` | Linting (rules in `.eslintrc.json`) |
| `prettier` | Code formatting (`@excalidraw/prettier-config`) |
| `husky` | Git hooks (pre-commit runs lint-staged) |
| `lint-staged` | Runs ESLint + Prettier only on staged files |

### Test Environment
| Package | Purpose |
|---------|---------|
| `jsdom` | DOM simulation for Vitest |
| `vitest-canvas-mock` | Canvas API mock |
| `@testing-library/jest-dom` | Custom DOM matchers |
| `fake-indexeddb` | IndexedDB mock for tests |

---

## Environment Variables

Configured in `.env.development` and `.env.production`. Key variables:

| Variable | Required | Description |
|----------|----------|-------------|
| `VITE_APP_BACKEND_V2_GET_URL` | No | Sharing service GET endpoint |
| `VITE_APP_BACKEND_V2_POST_URL` | No | Sharing service POST endpoint |
| `VITE_APP_WS_SERVER_URL` | No | Collaboration WebSocket server URL |
| `VITE_APP_FIREBASE_CONFIG` | No | Firebase config JSON (for hosted app persistence) |
| `VITE_APP_PLUS_APP` | No | Excalidraw+ app URL |
| `VITE_APP_PLUS_BACKEND` | No | Excalidraw+ backend URL |
| `VITE_APP_AI_BACKEND` | No | AI/Mermaid feature backend URL |

For local development without collaboration or persistence, no environment variables are
required — sensible defaults are provided in `.env.development`.

To override locally, create `.env.local` (gitignored) and set only the variables you need.

---

## Known Limitations

### Agent Limitations

- **No persistent memory across sessions:** Each `/review`, `/test`, or `/explain` invocation
  starts fresh. Agents do not remember previous runs or accumulate context between sessions.

- **Shell command timeout:** The `/test` agent may time out on first run if `node_modules`
  is not installed. Always run `yarn install` before invoking `/test`.

- **Coverage threshold failures are not auto-fixed:** The `/test` agent reports coverage
  shortfalls but cannot generate the missing tests automatically. A human must write them.

- **`/review` only sees committed or staged diffs:** Unstaged changes in working-tree files
  that are not yet staged will not be reviewed. Run `git add` first or adjust the command.

### Codebase Limitations

- **No SSR support:** The core `@excalidraw/excalidraw` package requires a browser
  environment. Next.js and similar frameworks must use dynamic imports with `ssr: false`.

- **Canvas rendering:** Tests run in jsdom which requires `vitest-canvas-mock`. Some visual
  rendering edge cases cannot be caught by unit tests and require manual browser testing.

- **Collaboration server not included:** The real-time collab feature requires a separate
  WebSocket server. The open-source self-hosting setup is documented at
  https://docs.excalidraw.com/docs/introduction/self-hosting but is not part of this repo.

- **TypeScript strict mode is not fully enabled:** Some legacy code uses `any` types.
  New code should avoid `any`; use `unknown` with proper type guards instead.

- **Monorepo build order matters:** When building packages for publishing, the order is:
  `common` → `math` → `element` → `excalidraw`. Running `yarn build:packages` handles
  this automatically.
