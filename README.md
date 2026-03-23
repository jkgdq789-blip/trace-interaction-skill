[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Works with Claude Code](https://img.shields.io/badge/Works_with-Claude_Code-blueviolet)](https://docs.anthropic.com/en/docs/claude-code)

# Trace Interaction -- Make AI follow call chains across files

AI reviews files one at a time. Bugs live between files. This skill tells Claude to follow the data from A to B to C.

## The problem

Claude reads `token-manager.ts` and says it looks fine. Reads `ws-relay.ts` and says it looks fine. But `ws-relay` captures the token in a closure at connect time, so when `token-manager` refreshes, the WebSocket keeps sending the old token. Neither file is wrong in isolation. The bug is in the handoff.

This skill makes Claude trace the full chain instead of reviewing files one at a time.

## Install in 10 seconds

```bash
cp SKILL.md ~/.claude/commands/trace-interaction.md
```

Then in Claude Code:

```
/trace-interaction user changes API key at runtime
```

## How it works

You describe a scenario in plain English. The skill makes Claude:

1. Find the entry point -- which file handles this first?
2. Follow every call -- read the actual code at each hop, not just the function signature
3. Track the data -- what type, format, and unit is it in? Does it change?
4. Find where it breaks -- missing consumers, stale closures, unit mismatches, swallowed errors

## Example

```
/trace-interaction What happens when the user changes their API key?
  Does the WebSocket reconnect? Does the cache clear?

  Tracing through 8 files across 3 services...

  [1] token-manager.ts:42  -- refreshes token, stores in memory
  [2] api-client.ts:118    -- picks up new token via getter       OK
  [3] ws-relay.ts:67       -- holds old token in closure          BUG
  [4] gate-service.ts:34   -- validates token from ws header      OK
      (but receives STALE token from step 3)

  P1 | ws-relay.ts:67
      WebSocket captures token at connect time. Token refresh
      updates the manager but never triggers reconnect. All messages
      after refresh use the expired token.
```

## What it catches

- **Stale credentials** -- key rotation happens but old connections keep the old key
- **Config drift** -- some consumers pick up the new config, others don't
- **Unit mismatches** -- sender uses seconds, receiver expects milliseconds, messages silently drop
- **Partial updates** -- 3 of 4 downstream systems get the update, the 4th doesn't
- **Swallowed errors** -- a catch block eats the error and the chain silently stops

## Installation

```bash
# Scoped to one repo
mkdir -p .claude/commands
cp SKILL.md .claude/commands/trace-interaction.md

# Available everywhere
mkdir -p ~/.claude/commands
cp SKILL.md ~/.claude/commands/trace-interaction.md
```

## Usage

```
/trace-interaction user changes API key at runtime
/trace-interaction WebSocket reconnect after token refresh
/trace-interaction config update from admin panel to all workers
/trace-interaction file upload from client through gateway to storage
```

Works on any language, any codebase. No dependencies.

## License

[MIT](LICENSE)
