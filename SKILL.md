---
name: browser-visual-ux-qa
description: Simulate a real user in a real browser, capture full-page screenshots across mobile/tablet/desktop and light/dark, audit every screenshot visually and against the source code, fix root causes, redeploy and re-verify with before/after evidence. Use for "QA visual", "revisa la UI/UX", "simula al usuario", "screenshot QA", "fix all buttons/layout", "make it look perfect", or any request to make a web app or PWA look and work right.
---

# Browser Visual UX/UI QA

Distilled from the best prompts in the Notion PROMPTS MASTER HUB: Omniscient UX/UI Audit Agent v4, Code-Native Visual QA Agent, UI Fix + 20 Workflow Validation, Universal UI Problems Detector.

## Prime directives

1. Execute end to end. Never ask to proceed. If blocked, fix the environment (start the server, seed a throwaway user, set the local cookie flag) instead of reporting it.
2. Capture → critique → fix → re-capture. No critique before capture, no "done" without a fresh capture after the fix.
3. Evidence first: every finding names the screenshot file + element, and the cause as `file:line`.
4. Simulate the user, not the code: click by visible text/role, type into labelled fields, wait for what a human would wait for. Never call internal functions to skip a step.
5. Never touch real user data. Use throwaway accounts (`qa+<timestamp>@example.com`) and delete them at the end. Never kill a healthy running server.
6. Same failure three times → write the root cause, mark it BLOCKED, continue with the next item.
7. Forbidden: "etc.", "similar issues elsewhere", "improve spacing" without values, skipping screenshots, sampling.

## Phase 0 — Read the code silently

`package.json`/scripts, router or nav map, global layout/shell, CSS tokens + breakpoints, API client + auth (cookie/token), state shape, README/AGENTS.md. Build internally: view → file map, auth method, endpoints, theme tokens, i18n source.

## Phase 1 — Define the user workflows

From the nav/menu, list 10–20 real jobs a user does, each with ≥6 steps, prioritized by menu prominence. Always include: first run/onboarding, sign up/login/logout, the core daily action, create/edit/delete, share/invite/join (with a second user), settings, empty state, error/offline, reload persistence.

## Phase 2 — Simulate in a browser

Use the available browser (Playwright, the in-app browser, or Chrome). Before writing automation against a library API (Playwright, Puppeteer, framework test utils), check its current docs with Context7 (resolve the library id first, always pass a topic) and match the installed version. Matrix: 390×844 mobile first (PWAs), 768×1024 tablet, 1440×900 desktop; `colorScheme` light and dark. Settle deterministically: `networkidle`, fonts loaded, animations disabled, +500 ms. Screenshot full page after every meaningful step with ordered names `NN-workflow-step-viewport-theme.png`.

Collect automatically on every step:
- console errors, `pageerror`, responses ≥400, requests >1 s
- horizontal overflow: `document.documentElement.scrollWidth > innerWidth`
- clipped text: elements with `scrollWidth > clientWidth + 1` that hold text
- tap targets < 44×44 px on interactive elements
- visible strings matching `undefined|null|NaN|\[object|TODO|lorem`

Minimal probe (inject with `page.evaluate`):

```js
() => {
  const r = { overflowX: document.documentElement.scrollWidth > innerWidth, clipped: [], small: [], bad: [] };
  for (const el of document.querySelectorAll('body *')) {
    const cs = getComputedStyle(el); if (cs.display === 'none' || cs.visibility === 'hidden') continue;
    const b = el.getBoundingClientRect(); if (!b.width) continue;
    if (el.matches('button,a,[role=button],input,select,textarea,label[for]') && (b.width < 44 || b.height < 44)) r.small.push((el.innerText || el.getAttribute('aria-label') || el.tagName).slice(0, 30) + ` ${b.width|0}x${b.height|0}`);
    if (el.children.length === 0 && el.innerText && el.scrollWidth > el.clientWidth + 1 && cs.overflow !== 'visible') r.clipped.push(el.innerText.slice(0, 40));
  }
  const t = document.body.innerText; const m = t.match(/\b(undefined|null|NaN)\b|\[object|lorem/gi); if (m) r.bad = [...new Set(m)];
  return r;
}
```

## Phase 3 — Look at every screenshot

Open each image with the image/vision tool. For each one:
1. Describe it completely: layout, colors, hierarchy, what is above the fold, what the user is supposed to do next.
2. Ask: what looks wrong, broken, cramped, inconsistent, ugly or confusing here?

Every five screenshots, review the last five together; at the end, review the last half together. Compare the same screen across viewports and themes.

Check each category:
- Broken visuals: overflow, clipping, overlap, z-index, misalignment, layout shift, fixed bars covering content, safe-area insets
- Consistency: color, radius, shadow, button variants, icon style, spacing scale, component variants
- Spacing/alignment: uneven margins (give px estimates), cramped rows, orphan elements
- Typography: hierarchy, truncation, body < 14 px, line height, widows
- Interaction states: hover/focus/active/disabled/loading/success/error; every action gives visible feedback; destructive actions confirm
- Accessibility: contrast AA in both themes, 44 px targets, color-only meaning, labels, visible focus
- Empty/loading/error/offline states: present, helpful, with a next action
- Copy: placeholders, raw ids, `undefined`, mixed languages, jargon, wrong tone, information that leaks private data to another user

## Phase 4 — Cross-check with the code

For each visual finding locate the root cause `file:line`. Then run the code-only checks: buttons without a handler, links to missing routes, unhandled empty/error/unauthenticated paths, swallowed errors without UI feedback, data fetched with no loading state, interpolations that can render `undefined`, hard-coded dev URLs, leftover `console.log`, stale caches (service worker, `Cache-Control`).

## Phase 5 — Report with ASCII cards

One card per view, then one tree per issue, then the priority table.

```
┌──────────────────────────────────────────────┐
│ VIEW: Home (390×844 · dark)  STATUS: ⚠️ Partial │
│ FILE: src/pages/Home.tsx                      │
├──────────────────────────────────────────────┤
│ ABOVE FOLD: header, summary card, 3 rows      │
│ EMPTY/ERROR/LOADING: ok / missing / ok        │
│ ISSUES: 🔴0 🟠2 🟡3 🟢1                         │
└──────────────────────────────────────────────┘
#3 UI BUG: stepper buttons 32×32 px
├── Screen: 04-home-mobile-dark.png
├── Category: Touch target
├── Cause: src/styles.css:212 `.step button{width:32px}`
├── Fix: 44×44 px, keep visual size with padding
└── Severity: High · Priority 1
```

| # | Sev | Category | Screen | Issue | Cause file:line | Fix | Status |

Write the report in the user's language.

## Phase 6 — Fix and prove

Smallest root-cause fix, plus high-confidence siblings. No masking with disabled UI or placeholder copy. Run existing tests. Bump asset cache-busters when assets are cached. Commit, push, deploy through the project's normal route, wait until the live asset hash matches. Re-run the same simulation and attach before/after screenshots.

## Completion gate

- [ ] Every workflow ran end to end at mobile light + dark (and tablet/desktop for layout views)
- [ ] Every screenshot was opened and described; no sampling
- [ ] Zero console errors, zero responses ≥400 that the user did not cause, no horizontal overflow
- [ ] All 🔴 and 🟠 issues fixed and re-verified live with after screenshots
- [ ] Throwaway users and test data deleted; real user data untouched
- [ ] Report delivered: cards, issue trees, priority table, fixes with commit hashes, remaining 🟡/🟢
