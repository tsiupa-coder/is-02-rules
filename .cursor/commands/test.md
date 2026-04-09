Run the project's test suite and provide a structured summary of results.

Steps:
1. Run `yarn test:typecheck 2>&1` to check TypeScript compilation. Report any type errors.
2. Run `yarn test:code 2>&1` to run ESLint. Report the count of errors and warnings; list any errors.
3. Run `yarn test:other 2>&1` to check Prettier formatting. List any files that need reformatting.
4. Run `yarn test:app --watch=false --reporter=verbose 2>&1` to execute unit tests. Capture output.
5. Summarize all results in this format:

---
## Test Run Summary

### TypeScript (`yarn test:typecheck`)
- Status: PASS / FAIL
- Errors: [count] — [list up to 5 error messages]

### Lint (`yarn test:code`)
- Status: PASS / FAIL
- Errors: [count], Warnings: [count]
- Top issues: [list up to 5]

### Formatting (`yarn test:other`)
- Status: PASS / FAIL
- Files needing formatting: [list]

### Unit Tests (`yarn test:app`)
- Status: PASS / FAIL
- Tests passed: [N] / [total]
- Test suites: [passed] / [total]
- Failed tests: [list test names and error snippets]
- Duration: [seconds]

### Overall: PASS / FAIL
[One sentence describing the most critical issue, or "All checks passed."]
---

If any step takes longer than 3 minutes, cancel it and note "timed out" in the summary.
