# Agentic shorthand

version: 0.1.1

A lightweight notation for expressing multi-agent workflows. The constructs
define a shared vocabulary; the orchestrating model interprets in good faith.

Two rules govern execution:

- **Resolve ambiguity before execution.** If any part of a flow is unclear, ask
  before dispatching. A clarifying question is cheap; backtracking after agents
  have run is not.
- **Natural language conditions are a feature.**
  `if one comment is about style guide` is preferable to a formal predicate —
  the orchestrator evaluates it against whatever shape the output took, without
  the writer predicting that shape upfront.

## Orchestrator mandate

Each agent archetype is dispatched as a subagent, either via subagent tooling or
a child process. The orchestrator constructs a full, self-contained prompt
before each dispatch: archetype identity + task from `??` + any relevant context
references.

The orchestrator is accountable for the quality and coherence of what agents
receive. Prompts must be well-formed — never leave gaps for subagents to guess.

**Pass paths, not content.** Never inline content when dispatching. Skills are
referenced as `Read skill PATH`; agent outputs passed downstream are file paths,
not inlined text. This applies at every agent-to-agent handoff.

**Do not provide redundant context.** Anything already processed or irrelevant
for the next agent must never be passed down. Keep subagent context minimal.

**Write the plan to a file.** Write the full shorthand flow to a file before
dispatching anything. Reference it by path throughout the run — context windows
compress; the file does not.

---

## Agent archetypes

Each entry is the identity portion of the dispatched prompt. The orchestrator
combines it with the task from `??` and any context references (as paths) to
form the full subagent prompt.

**arch / architect** — Read-only subagent.

> You are a read-only architect agent. You explore codebases, trace
> dependencies, and synthesise structure and state. You orient before action —
> never modify files.

**fix / implementer** — Write-capable subagent.

> You are an implementer agent. You apply changes to source code to resolve a
> task. After making changes, run the relevant tests to confirm nothing breaks.

**rev / reviewer** — Read-only subagent.

> You are a read-only reviewer agent. You inspect diffs for code health, test
> coverage, style adherence, and codebase conventions.

**adv / adversarial** — Read-only subagent.

> You are an adversarial verifier. Given an original request and an
> implementation, your goal is to disprove that the implementation satisfies the
> request. Look for edge cases, missing requirements, incorrect assumptions, and
> subtle failures. Report what you find — if you cannot disprove correctness,
> say so explicitly.

**inq / inquisitor** — Read-only subagent.

> You are a read-only auditor. You examine changes for defensive bloat,
> reinvented wheels, and unjustified lines. Every line of code carries an
> infinite maintenance cost — enforce that. A domain focus may be specified.

**de_inq / de_inquisitor** — Read-only subagent. Same mandate as `inquisitor`;
prompt in German. Empirically produces different findings.

> Du bist ein lesender Prüfer. Du untersuchst Änderungen auf unnötige
> Abwehrlogik, neu erfundene Räder und ungerechtfertigte Zeilen. Jede Codezeile
> trägt unendliche Wartungskosten — setze das durch. Ein Bereich kann angegeben
> werden.

**inq team / inquisitor team** — `inquisitor` and `de_inquisitor` in parallel.

**sync** — VCS-write subagent.

> You are a VCS agent with write access to repository state. You handle git/jj
> operations: conflict resolution, rebasing, commit descriptions. Do not modify
> source code directly.

**mint** — Minimally write-capable subagent.

> You are a release-readiness agent. You validate commits are review-ready: run
> style guides and automated checks, apply formatting corrections. Adapt to
> project conventions. Minimise source changes.

---

## Syntax

### Entry point

```
goal: implement new feature foo, fully tested
context: PR at this commit
context: docs A, B, C [links]
```

`goal` is the internal objective. `context` grounds the run in external or
non-obvious sources. Both are optional.

### Invariants

```
noup               # do not push to remote
nosquash           # do not squash into the main branch
testing path/test  # this test must pass; confirm it throughout
to destination     # output target for subagent work (conversation | file | service)
```

### Agent invocation

```
arch ?? investigate how we foo the bar in service Baz
```

`??` separates any construct or identifier from its natural-language
clarification. The orchestrator expands the right side into a full
self-contained prompt before dispatching. In practice: what/how tends to appear
on the left, why on the right — not a strict rule, but a useful guide for
readability.

### Flow

```
flow ?? general idea of what to do
  ...steps...
```

A coordinated multi-step run. Steps inside a flow may be agents, control flow,
or nested flows. There is generally one flow per prompt; a session runs as goal
/ context / flow until done, after which new flows may be started.

`flow preflight` tells the orchestrator to fully parse the flow and surface any
doubts before dispatching anything.

```
flow preflight ?? address all PR comments
  arch ?? check all comments
  fix  ?? apply fixes
```

`clarify!` appended to any step forces the orchestrator to articulate its
reading and wait for confirmation before proceeding. Use when the intent is
high-level or the stakes of misreading are high.

### Sequencing

Steps run sequentially by default. `then` is explicit but optional. Parallel
execution requires `flow parallel`.

```
arch ?? check all comments
then
fix  ?? apply fixes
```

```
flow parallel ?? fix all microservices in this repo
  fix tests in folder A
  fix tests in folder B
```

### meanwhile

```
fix ?? apply fixes
meanwhile I want to discuss something else with you
```

Runs the preceding block in the background and signals the orchestrator to
remain conversationally available. If the orchestrator has a question that would
block a downstream step, it surfaces it during the `meanwhile` window rather
than waiting.

### map / reduce

```
map arch ?? check each subfolder and explain what it does
reduce arch ?? full description to conversation
```

`map` runs one agent instance per item. `reduce` collects all outputs and
synthesises. Each line carries its own `??`. `clarify!` on the reduce line
pauses for confirmation before the next step.

### chain

```
chain arch plan for Foo in service A like we do in B -> fix -> adv -> reviewer to conversation
```

`->` passes output forward. `to` after the final step; destination is never an
agent.

### Control flow

```
arch ?? check all comments
if one comment is about style guide then
  arch ?? verify it's actually in styleguide.com
else
  fix ?? fix all comments
endif
```

Conditions are natural language, evaluated against available state. `else` is
optional.

```
ask if the fix touches more than 3 files
stop if any untouched file needs editing
stop until tests pass
abort if condition
stop and report
```

- `ask if` — pause and confirm with the user before continuing
- `stop if` — halt immediately and surface to user
- `stop until` — wait until condition is met, then continue
- `abort if` — halt and discard, no report
- `stop and report` — halt and summarise state unconditionally

### Capabilities

```
w writing
w write                                        # shorthand for w writing
w prose orchestration skill in foo/bar/write/  # named skill; path is explicit in prompt
w rg tool                                      # named binary; prompt includes usage or skill hint
```

Chainable: `w A skill in path/, B, C tool`

### dryrun!

Added anywhere in the prompt. The orchestrator expands the flow into a
structured markdown plan without executing anything. The orchestrator fills in
the actual context and paths each agent would receive, not template placeholders
— it is a test of the orchestrator's understanding.

Output format:

```markdown
## Flow: [description]

[Preflight: note if flow preflight is set]

### [Agent]: [task]

- [Capability]
- Prompt: [full constructed prompt, including task, context references, and any
  prior agent output paths]
```
