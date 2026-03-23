# Trace Interaction -- Cross-Component Scenario Tracing Skill for Claude Code

A structured methodology for tracing user actions and system events across component boundaries to find integration bugs. Built for use as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash command.

## What It Does

Trace Interaction follows a scenario (e.g., "user changes API key at runtime") through every file and function it touches, tracking state at each hop. It answers the question: **does this action correctly propagate to ALL downstream consumers?**

Most bugs in multi-service systems live at integration boundaries, not inside individual files. A function can be correct in isolation but broken in context because:

- A downstream consumer is never notified of the state change
- Data crosses a boundary with a type or unit mismatch
- An async gap allows stale state to be read
- One platform handles an edge case that another does not

## The Methodology

1. **Define the scenario** -- a concrete user action or system event
2. **Identify the entry point** -- which file and function handles this first?
3. **Trace the call chain** -- for each function call, read the actual code and note:
   - What state is modified (variables, maps, configs, stores)
   - What downstream consumers depend on this state
   - Whether each consumer is notified or updated
4. **Track data at each hop** -- what type, format, and unit is the data in? Does it change?
5. **Check for gaps** at each step:
   - Is there a component that uses this value but is not updated here?
   - Is there a type or unit mismatch between what is passed and what is expected?
   - Is there an async gap where state could be stale?
   - Does error handling in the middle of the chain silently swallow failures?
6. **Report** -- for each broken path, document exactly where propagation stops

## Real Results

Developed and refined over 1,300 cycles of code review on a multi-platform system. Bugs found by this methodology include:

- **Timestamp unit mismatch**: one component sent seconds, another expected milliseconds -- silent corruption until expiry math went wrong
- **State propagation failure**: config change updated the verifier but not the relay connection, causing stale authentication
- **Type confusion**: a lane ID of `0` was valid data on one platform but treated as "unassigned" on another
- **Silent swallow**: error in middleware returned early without propagating the failure to the caller, producing a false "success"

## Installation

Copy `SKILL.md` to your Claude Code commands directory:

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
```

## License

MIT
