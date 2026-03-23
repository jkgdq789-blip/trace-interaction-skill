[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Works with Claude Code](https://img.shields.io/badge/Works_with-Claude_Code-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)
[![GitHub stars](https://img.shields.io/github/stars/jkgdq789-blip/trace-interaction-skill?style=social)](https://github.com/jkgdq789-blip/trace-interaction-skill)

# Trace Interaction

Cross-component interaction tracing skill for Claude Code -- finds integration bugs between services.

**Most bugs in microservices and multi-component systems are at integration boundaries, not inside individual files.** A function can be correct in isolation but broken in context.

## Quick Start

```bash
cp SKILL.md ~/.claude/commands/trace-interaction.md     # install once
# then in Claude Code:
/trace-interaction user changes API key at runtime      # trace any scenario
```

## Demo

```
$ /trace-interaction WebSocket reconnect after token refresh

  Tracing scenario through 8 files across 3 services...

  Entry: src/auth/token-manager.ts:refreshToken()

  CHAIN:
  [1] token-manager.ts:42  -- refreshes token, stores in memory
  [2] api-client.ts:118    -- picks up new token via getter    OK
  [3] ws-relay.ts:67       -- holds old token in closure       BUG
  [4] gate-service.ts:34   -- validates token from ws header   OK
      (but receives STALE token from step 3)

  +------------------------------------------------------------------+
  | P1 | ws-relay.ts:67                                              |
  |    | Chain: token-manager -> api-client -> ws-relay -> gate       |
  |    | Bug: WebSocket connection captures token at connect time     |
  |    |      in a closure. Token refresh updates the manager but     |
  |    |      never triggers ws-relay reconnect. All messages after   |
  |    |      refresh are sent with the expired token.                |
  |    | Fix: Subscribe ws-relay to token change events, reconnect    |
  |    |      on refresh.                                             |
  +------------------------------------------------------------------+

  Summary: 1 broken chain, propagation stops at step 3 of 4
```

## What It Does

Trace Interaction follows a scenario (e.g., "user changes API key at runtime") through every file and function it touches, tracking state at each hop. It answers the question: **does this action correctly propagate to ALL downstream consumers?**

### The Methodology

1. **Define the scenario** -- a concrete user action or system event
2. **Identify the entry point** -- which file and function handles this first?
3. **Trace the call chain** -- for each function call, read the actual code and note what state is modified and who depends on it
4. **Track data at each hop** -- what type, format, and unit is the data in? Does it change?
5. **Check for gaps** -- missing consumers, type/unit mismatches, async stale reads, swallowed errors, partial updates
6. **Report** -- for each broken path, document exactly where propagation stops

## Real Bug Found

A gate node sent timestamps in seconds but a gate web component expected milliseconds. Every cross-gate sync message was silently dropped as "stale" because the receiving side interpreted a seconds-epoch value as a milliseconds-epoch value -- placing every message roughly 50 years in the past. The system reported zero errors. Only tracing the full chain from sender through the network boundary to receiver revealed the unit mismatch.

## Common Patterns It Catches

| Pattern | What breaks |
|---------|------------|
| Credential/key rotation at runtime | Connections and caches using stale credentials |
| Config change propagation | Some consumers updated, others still on old values |
| Sync messages across network boundaries | Type or unit mismatch at serialization layer |
| Reconnect after failure | Old state reused instead of re-initialized |
| TTL expiry and cleanup | Resources not released, watchers not cancelled |

## How It Was Developed

Refined over **1,300 cycles** of code review on a multi-platform system. Bugs found include timestamp unit mismatches, state propagation failures, type confusion across platforms, and silently swallowed errors in middleware chains.

## Installation

```bash
# As a project command (scoped to one repo)
mkdir -p .claude/commands
cp SKILL.md .claude/commands/trace-interaction.md

# As a user command (available in all projects)
mkdir -p ~/.claude/commands
cp SKILL.md ~/.claude/commands/trace-interaction.md
```

## Usage

Inside Claude Code, invoke the skill with:

```
/trace-interaction <scenario description>
```

Examples:

```
/trace-interaction user changes API key at runtime
/trace-interaction WebSocket reconnect after token refresh
/trace-interaction config update propagation from admin panel to all workers
/trace-interaction file upload from client through gateway to storage service
/trace-interaction user session expiry and cleanup across microservices
```

## Output

For clean chains:
```
Chain traced clean through N steps across M files.
```

For broken chains:
```
Scenario: [description]
Entry: [file:line]
Chain: [step1] -> [step2] -> ... -> [broken step]
Bug: [file:line] -- [severity] -- [description]
Fix: [what should happen instead]
```

## Compatible With

- **Claude Code** -- install as a slash command
- **Any language** -- JavaScript, TypeScript, Python, Go, Rust, Swift, Kotlin, Java, and more
- **Any multi-file codebase** -- monorepos, microservices, multi-platform projects

## License

[MIT](LICENSE)
