# Agentic Shorthand

A lightweight notation for multi-agent AI workflows. Structured enough to be
readable at a glance; loose enough that normal writing is a valid condition.

## Why

I find myself wanting to structure the work my agents and subagents do in very
particular ways. First investigate this, then do that if X otherwise do Y. This
is eventually annoyingly verbose. This is an attempt at reducing the amount of
typing, and potentially prove as a useful flow.

**This may be a quirk of how I work, and it may not be how you do it.** That's fine.
Don't use it.

## A bit of the syntax

```
goal ?? implement feature Foo, fully tested

flow ?? address all PR comments
  arch ?? investigate the current approach in service Bar
  fix  ?? apply fixes
  adv  ?? try to disprove the fix satisfies the request
```

Steps run sequentially by default. `flow parallel` for concurrency. `??`
separates subagents or subflows and targets from natural-language clarification.

| Archetype | Role                                                                   |
| --------- | ---------------------------------------------------------------------- |
| `arch`    | Read-only. Explores, traces, synthesises.                              |
| `fix`     | Write-capable. Applies changes, runs tests.                            |
| `rev`     | Read-only. Reviews diffs.                                              |
| `adv`     | Read-only. Tries to disprove correctness.                              |
| `inq`     | Read-only. Audits for bloat and reinvented wheels.                     |
| `de_inq`  | Same as `inq`, prompted in German. Empirically finds different issues. |
| `sync`    | VCS-write only. Git operations, no source changes.                     |
| `mint`    | Minimally write-capable. Release readiness.                            |

```
ask if the fix touches more than 3 files
stop if any untouched file needs editing
stop until tests pass
```

`dryrun!` anywhere expands the flow into a plan without executing.
`flow preflight` surfaces doubts before dispatch.

## A real example

[CORE BREACH](https://github.com/rberenguel/core-breach) had two related bugs in
its telegraph system: dead enemies left markers on the board, and enemies
re-aimed after being pushed. Both lived in `src/combat.js`; neither had an
obvious root cause.

```
goal ?? two telegraph bugs in CORE BREACH — dead enemies leave markers,
         enemies re-aim after being pushed

map arch ?? investigate each bug in src/combat.js
reduce arch ?? synthesise into a fix plan (clarify!)
fix ?? apply the fix
adv ?? kill an enemy and push one — confirm no phantom markers, no re-aim
```

`map arch` runs one agent per bug. `reduce arch` distils the findings;
`clarify!` pauses so the fix plan can be reviewed before any code changes. `adv`
verifies against the original symptoms.

## Status

`shorthand.md` is the working spec. Target: a cross-provider specification for
independent harness implementations.

## Changelog

**0.2.1** — Orchestrator no longer re-writes a user-provided plan file.
**0.2.0** — `goal`/`context` now use `??` syntax. New entry point construct: `fact`. New section "Signaling to the orchestrator": `duck` 🦆, `focus`, `quiz`. New archetypes: `spike`, `scribe`, `self`. Orchestrator mandate extended to cover subagent failure handling.
**0.1.1** — `clarify!` replaces `clarify`; `map`/`reduce` now multi-line.

**0.1.0** — Initial version.
