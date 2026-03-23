---
description: "Trace a cross-component interaction scenario through actual code. Follows call chains across files, tracking state propagation. Use when checking if a user action (key change, config update, reconnect, etc.) correctly updates ALL downstream consumers."
---

# Interaction Scenario Trace

You are tracing a cross-component interaction scenario through the codebase. Your goal is to follow data and state from the entry point through every downstream consumer, checking for propagation gaps, type mismatches, and stale state.

## Input
The user provides a scenario like: "user changes API key at runtime" or "WebSocket reconnects after token refresh"

## Method

### Step 1: Identify the Entry Point
- Which file and function handles this action first?
- Read the actual code, not just the function signature.
- Note the initial state: what data is available, what type/format is it in?

### Step 2: Follow the Call Chain
For each function call or state mutation in the chain, read the actual code and record:
- **What state is modified**: variables, maps, caches, config objects, database records, files
- **What downstream consumers depend on this state**: other modules, services, UI components, caches, connections
- **Whether each consumer is notified/updated**: direct call, event emission, polling, or nothing at all
- **Data format at this hop**: type, unit (seconds vs ms), encoding (hex vs base64), nullability

### Step 3: Check for Gaps
At each step in the chain, ask these questions:

1. **Missing consumer**: Is there a component that reads this value but is not updated here? Search for all imports/references to the changed state.
2. **Type/unit mismatch**: Does the data cross a boundary where the type or unit changes? (e.g., seconds to milliseconds, string to number, nullable to non-nullable)
3. **Async gap**: Is there a window where a concurrent reader could see stale state? (e.g., cache updated after response sent, event emitted before store committed)
4. **Error swallowing**: Does a try/catch or `.catch()` in the middle of the chain silently eat failures, causing downstream consumers to see success when there was a failure?
5. **Partial update**: Are multiple pieces of state updated non-atomically? Could a crash or error between updates leave the system in an inconsistent state?
6. **Missing return value consumption**: Does a function return important data (error, status, updated value) that the caller ignores?

### Step 4: Cross-Platform Check
If the scenario spans multiple platforms or languages:
- Does each platform handle the same edge cases?
- Are default values consistent?
- Are error codes and their meanings the same?
- Do data formats agree at serialization boundaries (JSON, protobuf, query params)?

### Step 5: Report Findings

For each broken propagation path:
```
Scenario: [description]
Entry: [file:line]
Chain: [step1] -> [step2] -> ... -> [broken step]
Bug: [file:line] -- [severity] -- [description]
Fix: [what should happen instead]
```

Severity levels:
- **P1**: Data loss, security bypass, or silent corruption
- **P2**: Incorrect behavior visible to users
- **P3**: State inconsistency that self-heals or is unlikely to trigger
- **P4**: Code quality issue, missing error handling for impossible cases

If no bugs are found:
```
Chain traced clean through [N] steps across [M] files.
```

## Common Interaction Patterns to Trace

These patterns are frequent sources of propagation bugs in multi-component systems:

| Pattern | What breaks |
|---------|------------|
| Credential/key rotation at runtime | Connections, caches, and relay rooms using stale credentials |
| Config change propagation | Some consumers updated, others still on old values |
| Successful operation broadcasting | Stats, audit log, cache, sync -- one is missed |
| Threshold/policy change | Some enforcement points updated, others not |
| Sync message across network boundary | Auth verification, replay guards, peer state -- type or format mismatch |
| Reconnect after failure | Old state reused instead of re-initialized |
| TTL expiry and cleanup | Resources not released, watchers not cancelled |

## Tips

- **Read the code, not the function name.** A function named `updateAllConsumers` might not actually update all consumers.
- **Search for all references** to any state that is modified. The call chain only shows explicit calls; implicit consumers (polling, watching, importing the module) are where bugs hide.
- **Check return values.** A common pattern is `store.update()` returning an error or previous value that the caller discards.
- **Follow the data format.** If a timestamp enters as ISO 8601, is converted to epoch seconds in one place and epoch milliseconds in another, that is a bug waiting to happen.
