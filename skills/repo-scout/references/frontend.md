# Frontend standard — React · Next.js · TypeScript

Used by `/pr-review` when a repo's `review-rules.md` says `Standards: frontend`.
Section 3 (Next.js) applies only if the repo uses Next.js; the rest applies to any React app.
Project rules in `review-rules.md` are added on top and win if they conflict.

## How to use these rules
- Only flag a rule when you can point to a **specific line** in the diff.
- Cite the rule ID in the comment title, e.g. `C2 · issue (blocking) [FE-SEC-01] — ...`.
  The `/review-summary` skill counts findings by rule ID.
- Severity: **B** = blocking, **N** = non-blocking. You may raise N to B if the
  impact in this PR is serious; say why.
- Formatting and anything ESLint / Prettier / `tsc` already reports are **not**
  review comments. If the PR disables a lint rule (`eslint-disable`, `@ts-ignore`),
  that **is** a comment (FE-TS-03).
- For UI/CSS bugs, find the real cause in the diff before proposing a fix, and
  propose the smallest change.

---

## 1. Security (OWASP)

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-SEC-01 | No `dangerouslySetInnerHTML` with untrusted data unless sanitized (e.g. DOMPurify). No `eval`, `new Function`, or building `href` from user input without checking it isn't `javascript:`. | B | Search diff for these APIs. |
| FE-SEC-02 | Auth tokens are not stored in `localStorage`/`sessionStorage`; prefer `HttpOnly` cookies or the auth library's secure storage. | B | `localStorage.setItem` with tokens. |
| FE-SEC-03 | No secrets in frontend code. Only truly public values use `NEXT_PUBLIC_` / `VITE_` env vars — they ship to the browser. | B | New public env vars; API keys in source. |
| FE-SEC-04 | Hiding a button is not security: every action is also protected on the server/API. | B | New admin-only UI with no matching server check in this PR or the API. |
| FE-SEC-05 | No personal data or tokens in `console.log`, URLs/query strings, or error-tracking payloads. | B | Diff. |
| FE-SEC-06 | New dependencies exist, are maintained, and are needed (watch for AI-invented package names). | B if unknown/unused | `package.json` diff; check the npm page. |

## 2. React (react.dev — Rules of React)

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-RCT-01 | Components and hooks are pure: no side effects during render (no API calls, no mutating outside variables, no `Math.random`/`Date.now` driving output). | B | Code in the component body outside hooks/handlers. |
| FE-RCT-02 | Rules of Hooks: hooks only at the top level of components/custom hooks — never in conditions, loops, or after an early `return`. | B | Hook calls inside `if`/loops/after returns. |
| FE-RCT-03 | State and props are never mutated; update with new objects/arrays. | B | `state.x = `, `arr.push` on state, then `setState(arr)`. |
| FE-RCT-04 | No unnecessary `useEffect`: values computed from props/state are calculated during render (or `useMemo`), events are handled in event handlers, not effects. | N | Effects that only `setState` from other state/props. |
| FE-RCT-05 | Effects that subscribe, start timers, or fetch have cleanup (unsubscribe, clear timer, ignore/abort stale requests). Dependency arrays are complete (no silenced `exhaustive-deps`). | B | `useEffect` without return cleanup; `eslint-disable` for deps. |
| FE-RCT-06 | List `key`s are stable IDs from the data, not array index or random values, when items can be added, removed or reordered. | N (B if items have state/inputs) | `key={index}`, `key={Math.random()}`. |
| FE-RCT-07 | Every data fetch shows loading, error, and empty states. | N | New fetching code without these branches. |
| FE-RCT-08 | Components do one job; large components (~200+ lines) or repeated JSX blocks are split into components. | N | Long new components; copy-pasted JSX. |
| FE-RCT-09 | DRY: existing shared components, hooks, and utilities are reused, not re-created. | N | Search repo for similar components/hooks. |

## 3. Next.js (only if the repo uses Next.js)

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-NXT-01 | Server Actions and Route Handlers check authentication **and** authorization (does this user own/may change this record) inside the action — a page-level check does not protect them. | B | `'use server'` functions and `route.ts` without auth + ownership checks. |
| FE-NXT-02 | Server Actions validate all arguments (e.g. zod) — they can be called directly by POST. | B | Actions using arguments without validation. |
| FE-NXT-03 | Data passed to Client Components and returned from Server Actions is a minimal DTO — never raw DB/API records with private fields. | B | Props typed as full entities; actions returning `db.*` results. |
| FE-NXT-04 | Server-only code (DB access, secrets, `process.env`) is in modules marked `import 'server-only'` and not imported by `'use client'` files. | B | Imports from data/DB modules in client files. |
| FE-NXT-05 | Middleware/proxy is not the only auth check; protected data is re-checked where it is read or changed. | B | Auth only in `middleware.ts`/`proxy.ts`. |
| FE-NXT-06 | Route params and `searchParams` are validated; never used to decide permissions. | B | `searchParams.isAdmin`-style logic; unvalidated `[id]`. |
| FE-NXT-07 | No mutations during render (setting cookies, deleting data, revalidating) — use Server Actions. | B | Side effects in page/component bodies. |
| FE-NXT-08 | `'use client'` is placed as low in the tree as possible; pages stay Server Components unless they need interactivity. | N | `'use client'` on pages/layouts without need. |
| FE-NXT-09 | Use framework components where they exist: `next/link` for internal links, `next/image` for images, Metadata API for titles. | N | Raw `<a href="/...">`, `<img>`, manual `<title>`. |
| FE-NXT-10 | Caching doesn't leak per-user or per-tenant data (cache keys include the user/tenant, or the route is dynamic). | B | Cached fetches of personalized data. |

## 4. TypeScript

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-TS-01 | No `any` in new code; use real types or `unknown` + narrowing. | N | `any` in diff. |
| FE-TS-02 | Data from APIs, forms, and storage is validated at runtime (e.g. zod) before being trusted as a typed object; no `as SomeType` on raw JSON. | N (B for data driving permissions/payments) | `as` casts on `fetch().json()`. |
| FE-TS-03 | No `@ts-ignore`, `@ts-expect-error`, `eslint-disable`, or non-null `!` added without a comment explaining why. | N | Diff. |
| FE-TS-04 | API types match the backend contract (DTO field names/types). If the API changed in a related backend PR, the types changed too. | B | Compare with the backend PR / API spec. |

## 5. Accessibility (WCAG 2.2 AA)

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-A11Y-01 | Clickable things are `<button>` or `<a>`, not `<div onClick>`; they work with the keyboard. | N | `onClick` on non-interactive elements. |
| FE-A11Y-02 | Every form input has a visible label (or `aria-label`); errors are shown in text, not only colour. | N | New inputs. |
| FE-A11Y-03 | Images have meaningful `alt` (or `alt=""` if decorative); icon-only buttons have an accessible name. | N | New `<img>`, icon buttons. |
| FE-A11Y-04 | Text meets contrast 4.5:1 in both light and dark themes; use design-system tokens, not hard-coded colours. | N | Hard-coded colour values in diff. |

## 6. Styling & performance

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-STY-01 | Use design-system tokens/components and existing utility classes; no one-off magic pixel values or inline styles for layout. | N | Inline `style={{}}`, arbitrary values. |
| FE-STY-02 | Layout works at mobile width (no horizontal scroll) and at the app's supported breakpoints. | N | Fixed widths in diff. |
| FE-PRF-01 | No large new dependency for a small task; heavy components are lazy-loaded. | N | `package.json` size/necessity. |
| FE-PRF-02 | No expensive work on every render or every keystroke (large sorts/filters without memo, unthrottled API calls on input). | N | Calculations in render; `onChange` → fetch. |

## 7. Tests

| ID | Rule | Sev | How to check |
|---|---|---|---|
| FE-TST-01 | New logic (hooks, utils, form validation) has tests that would fail if it broke. Tests interact like a user (Testing Library queries by role/label), not by implementation details. | N (B for business rules) | Tests vs. new logic. |
| FE-TST-02 | Tests and snapshots were not weakened or regenerated just to pass. | B | Changed expectations / snapshot updates without a reason. |

---

## Sources
- React — Rules of React: https://react.dev/reference/rules
- React — You Might Not Need an Effect: https://react.dev/learn/you-might-not-need-an-effect
- React — Rendering Lists (keys); Updating Objects/Arrays in State
- Next.js — Data Security guide: https://nextjs.org/docs/app/guides/data-security
- Next.js — Authentication guide: https://nextjs.org/docs/app/guides/authentication
- OWASP Next.js Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Nextjs_Security_Cheat_Sheet.html
- OWASP Cross-Site Scripting Prevention Cheat Sheet; HTML5 Security Cheat Sheet (storage)
- TypeScript Handbook (strict mode); typescript-eslint recommended rules
- W3C WCAG 2.2 (Level AA)
- Testing Library — Guiding Principles
