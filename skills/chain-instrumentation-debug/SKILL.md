---
name: "chain-instrumentation-debug"
description: "Diagnoses full-stack bugs by inserting console.log at critical state-transition points across the request/response chain, then analyzing where values diverge from expectations. Invoke when a bug is hard to pinpoint through static code reading alone."
---

# Chain Instrumentation Debugging

A systematic methodology for diagnosing full-stack bugs by instrumenting the data flow with console output at critical transition points, then reading the logs to pinpoint exactly where state diverges from expectations.

## When to Invoke

- Bug is reproducible but root cause is unclear from code reading
- User reports inconsistent behavior ("sometimes works, sometimes doesn't")
- UI doesn't reflect server state, or vice versa
- State appears to revert silently after an action
- Race conditions or timing-related bugs
- "Works on refresh but not on first interaction"
- Any bug where you need to understand the actual runtime data flow vs. the expected flow

## Methodology

### Step 1: Map the Data Flow

Before writing any logs, trace the complete request/response chain on paper:

```
User Action → Client Handler → Optimistic Update (set) → API Call →
Server Endpoint → DB Mutation → Response Serialization →
Client Response Handler → State Reconciliation (applyState) → UI Re-render
```

Identify every point where a value could change or be lost.

### Step 2: Insert Instrumentation at Every Transition

Add `console.log` at **every state transition** in the chain. Use a consistent prefix tag so logs can be filtered.

**Client-side action (example pattern):**

```typescript
myAction: (id) => {
  console.log('[myAction] start, input:', id, 'current state:', get().myField)
  
  // Optimistic update
  set({ myField: newValue })
  console.log('[myAction] after optimistic set, myField:', get().myField)
  
  // API call
  api.myAction(id).then((response) => {
    console.log('[myAction] API response, myField:', response.myField, 'other:', response.otherField)
    
    // State reconciliation
    applyState(response)
    console.log('[myAction] after applyState, myField:', get().myField)
  }).catch((err) => {
    console.error('[myAction] failed:', err)
  })
},
```

**Server-side endpoint (example pattern):**

```typescript
app.post('/api/myAction', (req, res) => {
  const session = getSession(req.cookies.sid)!
  console.log('[endpoint] request received, session field:', session.myField)
  
  doMutation(session.id, req.body.value)
  console.log('[endpoint] after mutation, session field (in-memory):', session.myField)
  
  const fresh = getSession(req.cookies.sid)!
  console.log('[endpoint] fresh from DB, session field:', fresh.myField)
  
  res.json(buildState(fresh))
})
```

### Step 3: Collect and Analyze Logs

Have the user reproduce the bug and paste the console output. Create a table:

| Log Point | Expected Value | Actual Value | Match? |
|-----------|---------------|-------------|--------|
| start | old value | ? | |
| after optimistic set | new value | ? | |
| API response | new value | ? | |
| after applyState | new value | ? | |

The **first row where Actual ≠ Expected** is the divergence point. Everything upstream of that row is working; everything downstream needs investigation.

### Step 4: Classify the Divergence

Based on where divergence occurs, classify the bug:

| Divergence Point | Likely Cause | Next Step |
|-----------------|-------------|-----------|
| After optimistic set | Zustand/React state not updating | Check selector subscription, check for duplicate store instances |
| API response | Server returning stale/wrong data | Audit server endpoint for stale captured objects, check DB write timing |
| After applyState | applyState overwriting with server's stale value | Check if applyState blindly overwrites locally-managed fields |
| No logs at all | Handler not firing | Check event binding, check for DOM event interception (mousedown vs click), check element visibility |

### Step 5: Fix and Verify

1. Fix the root cause identified in Step 4
2. Keep the instrumentation logs temporarily
3. Have user reproduce — logs should now show all values matching expected
4. Remove or gate the logs behind a debug flag
5. Rebuild and deploy

## Common Patterns Identified by This Method

### Stale Captured Object
Server captures an object at request start, mutates DB, serializes response using the stale in-memory object.
**Fix**: Re-fetch from DB before serializing response.

### Optimistic Update Overwritten
Client sets value optimistically, but a pending API response (from an earlier request) arrives and overwrites it via applyState.
**Fix**: Don't blindly overwrite locally-managed fields from unrelated server responses, or sequence API calls to prevent overlap.

### Event Not Firing
DOM event handler doesn't fire because a `mousedown` or `pointerdown` listener removes the element from the DOM before `click` fires.
**Fix**: Use `click` listener instead of `mousedown` for outside-click detection, or check `contains()` correctly.

### Race Condition
Two concurrent API requests return in unexpected order, the later one carrying stale state overwrites the earlier one's fresh state.
**Fix**: Re-fetch fresh state on the server side, or use request sequencing/cancellation on the client side.

## Log Hygiene

- Use a consistent `[tag]` prefix for easy filtering: `[activate]`, `[briefing]`, `[submit]`
- Log the **input**, the **current state before**, and the **state after** at each transition
- Log object references, not just primitives — `JSON.stringify` if needed
- Remove diagnostic logs after the fix is verified, or gate behind `if (import.meta.env.DEV)`
- Never use `console.log` in production hot paths — only for active debugging
