---
name: state-audit
description: "Audit a codebase or module for explicit and implicit state machines, document states, invariants, and transition tables, trace reachable invalid states across retries and concurrency, and fix confirmed bugs one machine per change. Makes no product-code edits by default. Use for /state-audit, 'find implicit state machines', or lifecycle, retry, and concurrency bug hunts. Not for trivial boolean toggles, pure derived values, or speculative framework rewrites."
---

# State Audit

Find the state machines a codebase already implements (explicit or implicit), map their states, invariants, and transitions, and prove which invalid states or transitions the code can actually reach, including through retries and concurrency.

Pass 1 makes **no product-code edits**: it writes one untracked report file (`STATE_MACHINES.md`, or a path the user specifies) and stops. Pass 2 fixes one approved machine at a time behind behavior-pinning tests.

Read `references/patterns.md` when auditing async, concurrent, persisted, or resource-owning code. Read `references/languages.md` during Pass 2 before picking a refactored type shape.

## Pass 1: Audit (no product-code edits)

### 1. Set scope and exclusions
If the user named a directory, module, or bug, stay inside it. For a whole-repo request, map entrypoints, schemas, workers, and UI state first; if the repo is too large for one pass, cover the highest-risk domains and list what was skipped.

### 2. Mechanical admission gate
Admit a candidate lifecycle **only** if at least one holds:
- **Two or more fields** jointly encode one lifecycle (booleans, enums, nullable timestamps/errors/payloads, or external DB/queue/lease fields).
- **Two or more direct storage-write sites** mutate one status/stage field without delegating to a single owning transition function.
- **A persisted status** crosses a process/network boundary or drives an external side effect (catches single-writer authority leaks like a generic `setStatus`).
- **Resource ownership or async ordering** depends on an implicit mode (timers, sockets, locks, child processes, retries, or `await` followed by a state write with no generation/attempt check).

**Not a finding (exclude immediately):**
- Single independent UI toggles (`isMenuOpen`, `isDarkMode`, `isTooltipVisible`) that do not constrain other fields or trigger multi-step flows.
- Orthogonal product types where every combination is legal (e.g. `isMuted` and `isPinned`). Declare sum vs product **per axis** with `file:line`; if all axes are orthogonal with no cross-axis constraint, drop the candidate.
- Pure derived values computed via selectors, getters, or views.
- Static config or feature flags read at startup and never transitioned at runtime.
- Missing checks already enforced upstream by a parser, type system, DB constraint, or unique index.

An empty inventory is a valid result. If little or nothing clears the gate, say so and stop.

### 3. Reconstruct machines from real code
Trace actual reads and writes; never infer transitions from field names alone. Before calling any transition "unguarded", run a negative search across schemas, migrations, unique indexes, DB triggers, and upstream parsers.

Cluster repeated patterns (e.g. 20 identical hooks) into one row with a count and representative examples.

Write a full reconstruction only for machines with at least one `Traced` or `Hypothesis` finding (keep clean machines to one summary-table row):
- **States and storage:** owning type/table/module, effective states, initial and terminal states, and any external state (DB rows, queues, files, leases, remote acks).
- **Transition table:** `(from, event/trigger, who can trigger, guard [enforced / none / unknown], to, side effects, file:line)`. Flag when a public entrypoint wraps an internal mutation.
- **Invariants and exits:** cross-field invariants, and whether every non-terminal state has a guaranteed exit (failure, cancel, timeout, lease expiry, or restart recovery).
- **Reachable hazards:** impossible field combinations, unguarded transitions, dead-end states, non-atomic check-then-act, stale async completions, duplicate side effects on retry, fencing/lease races, and concurrent writers.

### 4. Evidence and risk taxonomy
Every claim cites `file:line` and uses exactly one evidence tag:
- **Confirmed:** reproduced by a failing test or runnable script (Pass 2).
- **Traced:** concrete step-by-step code path reaching the invalid state or bad transition; for concurrency/retry hazards, cite why both actors or replays can actually overlap.
- **Hypothesis:** the representation allows the flaw and no guard was found, but a full trigger path was not traced. Keep hypotheses in an unranked section; never mix them into headline findings.
- **No traced hazard in scope:** inspected within stated coverage with no traced flaw found. Absence of a writer in inspected files is not proof none exists elsewhere.

Risk scale:
- **High:** data loss or corruption, financial/billing error, auth bypass, or permanently stuck irreversible workflow.
- **Medium:** broken user-facing flow, duplicated non-financial side effect, or leaked resource under failure/retry.
- **Low:** local UI glitch or redundant state without external impact.

### 5. Write `STATE_MACHINES.md` and stop
Write one canonical report (default untracked `STATE_MACHINES.md`, or a path the user names; do not commit it into a fix PR unless asked):
1. **Scope & coverage:** date, commit SHA (if in git), directories inspected, and exclusions.
2. **Summary table:** `# | Machine | Axes (Sum / Product) | Evidence | Risk | Key files | Suggested fix rung`.
3. **Detailed sections (`Traced` findings only):** states, transition table, invariants, step-by-step trigger path, and concrete refactor sketch.
4. **Unranked hypotheses:** plausible gaps not fully traced.
5. **Up to 3 next steps:** ranked by risk and reachability.

End Pass 1 by asking which single machine (if any) to fix first. Do not edit product code.

## Pass 2: Reproduce and fix (only after user approval)

Fix **one machine per change/PR**. For a clustered pattern (e.g. 20 hooks), fix one representative at a time unless the user asks for a batch.

1. **Confirm first:** write a failing test or runnable repro for each `Traced` bug on the approved machine to promote it to **Confirmed**. If you cannot reproduce it, downgrade it to `Hypothesis` and explain why.
2. **Pin behavior:** write tests covering the transitions the fix touches plus the illegal combinations before refactoring.
3. **Apply the smallest fix on the ladder** (stop at the lowest rung that closes the bug class):
   0. Add the missing guard, generation/attempt check, or conditional write at the existing site.
   1. Delete redundant state or derive it from existing source state.
   2. Collapse mutually exclusive flags into one typed enum or discriminated union whose variants own their state-specific data.
   3. Centralize transitions in one pure function, method set, or table that rejects illegal moves (for persisted state plus external side effects, pair conditional DB writes with a transactional outbox or claim-then-recover + idempotency key).
   4. Introduce a state-machine library **only** for 4+ states with nested/parallel async transitions where a typed union + switch breaks down.
4. **Preserve contracts:** keep public APIs, ordering, and error messages intact unless the user approved a breaking change. For any non-equivalent refactor, state the smallest input whose behavior changes.
5. **Report the fix:** state which machine moved from `Traced` to `Confirmed` and which tests guard it; update `STATE_MACHINES.md` only if the user keeps it in the repo.

## Diagrams and formal checks (on request)

Pass 1 is text and tables only. Add visuals or formal checks only when the user asks:
- **Mermaid diagram:** emit one Mermaid `stateDiagram-v2` block per requested machine inside `STATE_MACHINES.md`, annotating traced illegal paths and dead ends as numbered notes with `file:line` below. Emit a single-page HTML diagram only if the user explicitly asks for HTML.
- **Installed skills first:** if the environment already lists an installed statechart or formal-verification skill, offer to hand off one named machine to it rather than duplicating it here.
- **Formal escalation:** suggest a model checker or prover (TLA+/TLC, Lean, etc.) only for a `Traced` concurrency, retry, or invariant hazard that tests cannot cover, and always replay any counterexample against the real code before calling it a bug.
