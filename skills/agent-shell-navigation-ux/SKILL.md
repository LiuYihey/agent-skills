---
name: agent-shell-navigation-ux
description: >-
  Preserve agent transcript scroll across app page changes, and give async
  actions instant press feedback plus in-flight state. Use when building or
  fixing product shells with a side chat plus main content, when navigation
  resets conversation position, or when launch/submit clicks feel dead during
  latency.
---

# Agent Shell Navigation UX

Engineering points for apps with a **persistent agent conversation** beside a
**main content area** that navigates independently.

## When to apply

- Main page / route changes while chat stays visible (or remounts for the same session)
- Transcript jumps to top/bottom after navigate, rehydrate, or keep-alive hide
- Async CTAs (launch, submit, dispatch) feel ignored until the network returns
- Designing ownership for chat viewport vs page shell state

## 1. Page navigation ↔ conversation place

### Rule

**Transcript scroll belongs to session identity, not to the React instance or the current main page.**

Reset (or auto-scroll to end) only when the **active session id** changes
(new chat, history pick, task→linked session). Do **not** reset because:

- Main content navigated (list ↔ detail ↔ wizard)
- Chat remounted for layout (inline ↔ sidebar)
- Keep-alive toggled `display: none`
- Same-session server rehydrate replaced the message array

### Prefer

- **Session-keyed scroll memory** (module store or equivalent) shared across mounts
- Capture while the scroll container is **visible** — before navigate / before hide
- On mount or same-session list replace: **restore** remembered offset (retry after layout)
- Treat hidden containers as unreliable (`scrollTop` reads as `0`) — never write that back

### Avoid

- Binding scroll to page/`route` effects (“on page change, scroll chat to bottom”)
- First-paint “scroll to end” on every remount of the same session
- Auto-scroll driven only by message-array length after a bulk replace
- Saving scroll from a `display:none` panel (false zero)

### Ownership checklist

1. Who owns transcript viewport? → session store / chat shell (not the page router)
2. Who may clear it? → session switch only
3. When does main navigate? → capture first if chat may remount or hide
4. After rehydrate of the **same** session? → preserve, do not follow as “new messages”

## 2. Click feel + user mental model (async actions)

### Rule

**Press must confirm registration before the network or navigation completes.**

Async work has latency. If the UI stays idle until success, users re-click,
abandon, or assume the app is broken — even when the request is in flight.

### Prefer

- **Immediate affordance** on pointer down / click: pressed style, disable duplicate submit
- **In-flight ownership** keyed by the entity (card id, form id) — sync ref + UI state before `await`
- Distinct **busy label** (e.g. Launching…) while the request runs
- Success path may navigate afterward; failure clears busy and restores the CTA
- One owner for post-success navigation (app shell), not timers cancelled by remount

### Avoid

- Waiting for HTTP success before any visual change
- Busy flag only in React state if a re-render can clear it before the request starts
- Trusting prose / toast alone without a disabled in-flight control
- Double-submit because the button looked idle during latency

### Mental model

| Moment | User should believe |
|--------|---------------------|
| Click | “That registered” |
| Waiting | “Work is in progress” |
| Done | “Result / next page” |
| Error | “I can try again” |

## Anti-patterns (both)

- Symptom patches: keyword routes, forced extra scrolls, “scroll after 300ms if on history page”
- Splitting “feel” and “navigate” across unrelated timeouts that remounts cancel
- Letting page shell or rehydrate **own** conversation viewport as a side effect

## Validation

- [ ] Launch/submit → navigate: chat stays at the reading position (same session)
- [ ] Remount sidebar/inline for the same session: scroll restored
- [ ] Switching sessions still scrolls appropriately (end or remembered for that id)
- [ ] Click shows busy before response; second click does not double-fire
- [ ] Failure returns an actionable idle CTA
