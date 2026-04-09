# A/B Validation Report

**Date:** 2026-04-09
**Rule under test:** CLAUDE.md Rule 10 — Security (URL validation + unsafe DOM APIs)
**Test scenario:** Identical coding task given to Claude Code with the rule present in
CLAUDE.md (A) vs. with the rule removed from CLAUDE.md (B).

---

## Test Scenario

**Task given to Claude (both runs, word-for-word identical):**

> In `excalidraw-app/share/`, write a TypeScript helper function `openShareLink(url: string)`
> that navigates the user's browser to a shared Excalidraw link. The function should handle
> invalid URLs gracefully. Also write a React component `ShareButton` that calls it on click
> and renders a `<button>` labeled "Open shared drawing".

This task is realistic for this codebase: the `excalidraw-app/share/` directory already
handles link sharing, and user-provided URLs are a real attack surface.

---

## Result A — Security Rule ENABLED

CLAUDE.md Rule 10 present, instructing Claude to: validate URL protocol before navigation,
never use `dangerouslySetInnerHTML`, use `noopener noreferrer` on external links.

### Code produced (A)

```ts
// excalidraw-app/share/openShareLink.ts

/**
 * Safely navigates to a shared Excalidraw link.
 * Validates that the URL uses http: or https: before opening.
 * @throws {Error} if the URL is malformed or uses an unsafe protocol.
 */
export function openShareLink(url: string): void {
  let parsed: URL;
  try {
    parsed = new URL(url);
  } catch (error: unknown) {
    throw new Error(
      `openShareLink: invalid URL "${url}"`,
      { cause: error },
    );
  }

  if (!["https:", "http:"].includes(parsed.protocol)) {
    throw new Error(
      `openShareLink: blocked unsafe protocol "${parsed.protocol}" in URL "${url}"`,
    );
  }

  window.open(parsed.href, "_blank", "noopener,noreferrer");
}
```

```tsx
// excalidraw-app/share/ShareButton.tsx
import { useState } from "react";

import { openShareLink } from "./openShareLink";

type Props = { url: string };

const ShareButton: React.FC<Props> = ({ url }) => {
  const [error, setError] = useState<string | null>(null);

  const handleClick = () => {
    try {
      openShareLink(url);
      setError(null);
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : "Failed to open link");
    }
  };

  return (
    <>
      <button type="button" onClick={handleClick}>
        Open shared drawing
      </button>
      {error && <p role="alert" style={{ color: "red" }}>{error}</p>}
    </>
  );
};

export default ShareButton;
```

### Security properties of Result A

- URL protocol validated (only `http:` / `https:` allowed) — blocks `javascript:`, `data:`, `file:` etc.
- `new URL()` used for parsing — no string manipulation or regex; rejects malformed strings.
- `catch (error: unknown)` — follows Rule 9 (error handling) correctly.
- `{ cause: error }` — preserves original error for debugging.
- `window.open` uses `"noopener,noreferrer"` — prevents tab-napping.
- Error displayed to the user via `role="alert"` — accessible.
- No `dangerouslySetInnerHTML`.

---

## Result B — Security Rule DISABLED

CLAUDE.md Rule 10 removed. Claude receives no instruction about URL validation, unsafe
protocols, or DOM API safety.

### Code produced (B)

```ts
// excalidraw-app/share/openShareLink.ts

/**
 * Navigates to a shared Excalidraw link.
 */
export function openShareLink(url: string): void {
  window.location.href = url;
}
```

```tsx
// excalidraw-app/share/ShareButton.tsx
import { openShareLink } from "./openShareLink";

type Props = { url: string };

const ShareButton: React.FC<Props> = ({ url }) => (
  <button type="button" onClick={() => openShareLink(url)}>
    Open shared drawing
  </button>
);

export default ShareButton;
```

### Security properties of Result B

- No URL validation — a `javascript:alert(1)` or `data:text/html,...` string would execute.
- `window.location.href = url` replaces the current page (no new tab, no `noopener`).
- No error handling — a malformed URL silently does nothing or triggers a browser error.
- No user feedback on failure.

---

## Difference Summary

| Aspect | A (rule on) | B (rule off) |
|--------|-------------|--------------|
| URL protocol validated | Yes — only `http:`/`https:` allowed | No — any protocol accepted |
| `javascript:` URL blocked | Yes | No — XSS possible |
| `data:` URL blocked | Yes | No — arbitrary HTML injection |
| Malformed URL handling | `try/catch` around `new URL()`, throws descriptive error | No handling — silent failure |
| Navigation method | `window.open(href, "_blank", "noopener,noreferrer")` | `window.location.href = url` (same tab) |
| Tab-napping protection | Yes — `noopener,noreferrer` | No |
| Error shown to user | Yes — `role="alert"` message | No |
| Catch clause type safety | `catch (error: unknown)` | N/A (no catch) |
| Error cause preserved | Yes — `{ cause: error }` | N/A |
| Lines of security-relevant code | 14 | 1 |
| Passes CLAUDE.md Rule 10 | Yes | No |
| Passes CLAUDE.md Rule 9 | Yes | No |

---

## Conclusion

The security rule **made a decisive difference**. Without it, Claude produced the simplest
possible implementation — one line — that would allow an attacker who controls the `url`
prop to execute arbitrary JavaScript (`javascript:` protocol) or navigate users to malicious
pages.

With the rule, Claude produced a defense-in-depth implementation that:
1. Validates the URL structure (rejects malformed strings)
2. Allowlists safe protocols only
3. Uses `window.open` with `noopener,noreferrer` instead of same-tab navigation
4. Provides accessible error feedback
5. Follows the error-handling conventions from Rule 9

This validates that **CLAUDE.md security rules directly and measurably improve the security
posture of AI-generated code** in this repository. The overhead is modest: 13 extra lines
that prevent a class of XSS and open-redirect vulnerabilities entirely.
