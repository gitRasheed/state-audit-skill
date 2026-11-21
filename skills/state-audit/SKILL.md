---
name: state-audit
description: "Audit a codebase or module for explicit and implicit state machines, document states, invariants, and transition tables, trace reachable invalid states across retries and concurrency, and fix confirmed bugs one machine per change. Makes no product-code edits by default. Use for /state-audit, 'find implicit state machines', or lifecycle, retry, and concurrency bug hunts. Not for trivial boolean toggles, pure derived values, or speculative framework rewrites."
---

# State Audit

Find the explicit and implicit state machines in scope, map their states, invariants, and transitions, and prove which invalid states or transitions are actually reachable, including through retries and concurrency.

Pass 1 edits no product code: write one untracked `STATE_MACHINES.md` (or the path the user names) and stop. Pass 2 fixes one approved machine at a time behind tests.

## Pass 1: Audit

### Admission gate
Admit a candidate **only** if at least one holds:
- 2+ fields jointly encode one lifecycle (booleans, enums, nullable timestamps/errors/payloads, or external DB/queue/file/lease state).
- 2+ direct storage-write sites (including in-memory state flushed later) mutate one status/stage field without one owning transition function.
- A persisted status drives an external side effect or crosses a process, network, or privilege boundary (exclude single-pass file writes with no stages).
- Resource ownership or async ordering depends on an implicit mode (timers, sockets, locks, child processes, retries, or `await` followed by a state write without a sequence/generation guard).

Drop immediately: single independent UI toggles, orthogonal product types where all combinations are legal (check sum vs product **per axis** with `file:line`), seq/generation-fenced async writes, pure derived values, static startup config, and behavior already enforced upstream or documented as an accepted limit in repo docs. Empty inventory is a valid stop.

### Trace and check
Trace real reads and writes; never infer from names. Before calling anything unguarded or torn, check DB constraints/indexes/triggers, upstream parsers, ownership/`Drop`/RAII sites, filesystem commit markers (atomic rename, `fsync`, lockfiles, manifests), and repo docs. Cluster repeated patterns into one row.

Watch specifically for hazards models routinely miss:
- Non-terminal states with no exit under crash, cancel, timeout, lease expiry, or restart, and self-blocking borrows/locks that block the consumer from draining capacity.
- Stale async completions and stale view/ID pointers that reset only on exact match (missing ancestor deletion).
- Orphaned child/grandchild processes, inherited FDs blocking EOF, temp dirs deleted under a live child, weak senders dropped before first poll, and detach conflated with cancel.
- Partial local side effects (e.g. `remove_dir_all`) failing before state updates; DB conditional writes paired with non-atomic external calls (needs outbox or claim-then-recover + idempotency key).
- Lease expiry without monotonic fencing tokens, at-least-once redelivery, wall-clock TTLs, rolling-deploy enum skew, and authority leaks (e.g. public `setStatus` skipping move authorization).

### Evidence and risk
Every claim cites `file:line` and one tag:
- **Confirmed:** reproduced by a failing test or script (Pass 2).
- **Traced:** concrete step-by-step path; for concurrency, retry, or scheduling bugs, cite why actors overlap or the valid single-thread executor schedule.
- **Hypothesis:** plausible unguarded gap without a complete reachable path (or crossing out of scope/target). Keep unranked.
- **No traced hazard in scope:** inspected within coverage with no traced flaw found.

Rate **Risk (Likelihood)**: **High** (irrecoverable production/user data loss, money, auth bypass, stuck irreversible workflow; rerunnable dev/benchmark output is Medium), **Medium** (broken user flow, rerunnable data loss, duplicated non-financial effect, leaked process/resource), **Low** (local UI glitch, redundant state); Likelihood `Common` / `Edge` / `Extreme`.

### Report (`STATE_MACHINES.md`)
1. **Scope, exclusions, and dropped candidates** (date, commit SHA if in git, 1-phrase drop reasons).
2. **Summary table:** `# | Machine | Axes (Sum/Product) | Evidence | Top Risk (Likelihood) | Key files | Fix rung`.
3. **Detailed sections (`Traced` only):** states & external storage, transition table `(from, trigger, who, guard [enforced/none/unknown], to, side effects, file:line)`, invariants & exits, trigger path, refactor sketch.
4. **Unranked hypotheses.**
5. **Up to 3 next steps** ranked by Risk and Likelihood.

Stop and ask which single machine (if any) to fix first.

## Pass 2: Reproduce and fix (on approval)

Fix **one machine per change** (one representative at a time for clusters). Do not add new test or state-machine dependencies unless already in the repo.
1. **Confirm:** write a failing test/repro for each `Traced` bug on the approved machine (`Confirmed`). If unrunnable on this host (e.g. another OS), keep `Traced (unrunnable on host)`; if runnable and it does not fail, downgrade to `Hypothesis`.
2. **Pin:** test the transitions the fix touches plus illegal combinations.
3. **Smallest fix rung:** `0` add missing guard/generation check/conditional write in place -> `1` delete or derive redundant state -> `2` collapse mutually exclusive flags into one enum/union owning its data (keep non-serializable handles like `AbortController` in the runner, not state data; avoid `mem::replace` placeholders and catch-all wildcards) -> `3` centralize transitions (plus outbox/claim-then-recover for DB + external effects) -> `4` state-machine library only for 4+ states with nested/parallel async transitions.
4. **Preserve contracts:** keep public APIs, ordering, and error messages intact unless approved; state the smallest input whose behavior changes for any non-equivalent refactor.

## Diagrams and formal checks (on request)

Pass 1 is text and tables only. When the user asks for visuals or formal checks:
- **Model first, bugs as overlay:** draw the intended machine (states, normal transitions, exits) as the base graph, even for clean machines. Layer traced hazards on top as annotations (Mermaid) or toggleable overlays/tabs (HTML if explicitly requested).
- **Match format to shape:** Mermaid `stateDiagram-v2` for lifecycles, `sequenceDiagram` for concurrency/borrow interleavings, State × Event matrix for 5+ states or cross-cutting events.
- **Installed skills & formal escalation:** hand off to an installed statechart or formal-verification skill if present. Suggest a model checker/prover only for a `Traced` concurrency, retry, or invariant hazard tests cannot cover, and replay any counterexample against real code before calling it a bug.
