Run the full pre-PR checklist for the current branch. This command verifies everything a reviewer would check before approving a pull request.

## Step 1 — Gather Branch Context

```bash
git log main..HEAD --oneline
git diff main..HEAD --stat
```

Report:
- Number of commits on this branch
- Files changed (count and list)
- Lines added / deleted

## Step 2 — Commit Message Audit

```bash
git log main..HEAD --format="%s"
```

Each commit subject must match: `^(feat|fix|chore|refactor|test|docs|style|perf|ci)(\(.+\))?: .+`

Flag any commit that does not match. Report PASS / FAIL.

## Step 3 — File Structure Check (CLAUDE.md Rule 8)

For every new file in the diff:
- Confirm it is in the correct workspace (see CLAUDE.md Rule 8 table).
- Confirm `.tsx` is used for React components and `.ts` for pure logic.
- Check that no app-layer imports (firebase, socket.io, import.meta.env) appear in `packages/excalidraw/`, `packages/element/`, `packages/common/`, or `packages/math/`.

```bash
grep -rn "firebase\|socket\.io" packages/excalidraw/ packages/element/ packages/common/ packages/math/
```

Report PASS / FAIL + any violations.

## Step 4 — TypeScript Check

```bash
yarn test:typecheck 2>&1
```

Report PASS / FAIL. List up to 10 errors if FAIL.

## Step 5 — Lint Check

```bash
yarn test:code 2>&1
```

Report PASS / FAIL. List all errors (not warnings). Warnings are acceptable but should be noted.

## Step 6 — Formatting Check

```bash
yarn test:other 2>&1
```

Report PASS / FAIL. List any files that need formatting.

## Step 7 — Security Scan (CLAUDE.md Rule 10)

```bash
git diff main..HEAD -- '*.ts' '*.tsx' | grep -E "dangerouslySetInnerHTML|window\.open|location\.href|Math\.random|catch \(.*: any\)"
```

For each match:
- `dangerouslySetInnerHTML` → check if DOMPurify.sanitize() is called on the value.
- `window.open` / `location.href` → check if URL protocol is validated before the call.
- `Math.random` → check if it is used in a security-sensitive context.
- `catch (.*: any)` → flag as a Rule 9 violation.

Report PASS / FAIL + findings.

## Step 8 — Test Suite

```bash
yarn test:app --watch=false 2>&1
```

Report PASS / FAIL. On failure, list failing test names and error messages.

## Step 9 — Coverage (for PRs adding new logic)

If the diff adds more than 50 lines of non-test TypeScript:
```bash
yarn test:coverage --watch=false 2>&1 | tail -20
```

Confirm coverage is still above thresholds: lines ≥ 60%, branches ≥ 70%, functions ≥ 63%, statements ≥ 60%.

## Final Verdict

Output this table, then an overall APPROVED / NEEDS CHANGES verdict:

| Check | Status | Notes |
|-------|--------|-------|
| Commit messages | PASS/FAIL | |
| File structure | PASS/FAIL | |
| TypeScript | PASS/FAIL | N errors |
| Lint | PASS/FAIL | N errors, N warnings |
| Formatting | PASS/FAIL | |
| Security scan | PASS/FAIL | |
| Test suite | PASS/FAIL | N/N tests |
| Coverage | PASS/FAIL/SKIP | |

**Verdict: APPROVED / NEEDS CHANGES**

If NEEDS CHANGES: list the top 3 blocking issues in priority order.
