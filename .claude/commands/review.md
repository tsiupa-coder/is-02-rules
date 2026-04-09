Perform a structured code review of all files changed since the last commit (or staged files if nothing is committed yet).

Steps:
1. Run `git diff HEAD --name-only` to get the list of changed files. If empty, run `git diff --cached --name-only`.
2. For each changed file, read its content and the diff (`git diff HEAD -- <file>`).
3. Review each file against the rules in CLAUDE.md:
   - Are `import type` used for all type-only imports?
   - Are barrel imports avoided inside `packages/excalidraw/**`?
   - Is `jotai` imported directly anywhere (violation)?
   - Are import groups ordered correctly with newlines between them?
   - Are new functions covered by tests?
   - Does any new code introduce security issues (XSS, injection, etc.)?
4. Summarize findings in this format for each file:

   **File:** `path/to/file.ts`
   - PASS / FAIL: Import type correctness
   - PASS / FAIL: No barrel imports
   - PASS / FAIL: No direct jotai imports
   - PASS / FAIL: Import order
   - PASS / FAIL: Test coverage (note if no test file exists for new logic)
   - Issues found: [list any bugs, style problems, or logic errors]
   - Suggestions: [optional improvements]

5. At the end, print an overall verdict: APPROVED / NEEDS CHANGES, with a one-line summary of the most critical issue if any.
