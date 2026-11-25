# state-audit

An agent skill for finding explicit and implicit state machines in a codebase, mapping their states, invariants, and transition tables in one file, tracing reachable invalid states across retries and concurrency, and fixing confirmed bugs one machine per change.

## What it catches

**Before: scattered flags and a stale async race (TypeScript).**

```ts
let loading = false;
let error: string | null = null;
let data: User | null = null;

async function fetchUser(id: string) {
  loading = true;
  error = null;
  try {
    data = await api.getUser(id); // late reply from request A overwrites request B
  } catch (e) {
    error = String(e);            // data from previous user stays non-null alongside error
  } finally {
    loading = false;
  }
}
```

**After (Pass 1 traces the race; Pass 2 confirms it with a test and applies the smallest fix):**

```ts
type UserState =
  | { kind: "idle" }
  | { kind: "loading"; attemptId: number }
  | { kind: "ready"; user: User }
  | { kind: "error"; message: string };
```

Late completions check `state.kind === "loading" && state.attemptId === id` before writing, and `data` cannot coexist with `error`.

## What makes it low-noise

- **Mechanical admission gate:** admits a candidate only when 2+ fields jointly encode one lifecycle, 2+ storage writes mutate one status field, a persisted status crosses a process/network boundary or triggers external effects, or async/resource ownership depends on an implicit mode. Drops independent toggles and orthogonal flags.
- **Strict evidence split:** separates `Confirmed` (reproduced by a failing test), `Traced` (`file:line` path with overlap rationale), `Hypothesis` (unranked), `Inspected` (read-only), and `Checked` (exercised by a named test or model).
- **Two passes, small diffs:** Pass 1 edits no product code and writes `STATE_MACHINES.md`. Pass 2 fixes one approved machine at a time starting from rung 0 (add the missing guard/generation check) before reaching for unions, reducers, or libraries.
- **Diagrams and formal checks on request:** text and tables by default; Mermaid `stateDiagram-v2` on request, or handoff to installed statechart/model-checking skills when a traced concurrency bug warrants TLC or Lean.

## Install and use

```bash
npx skills add https://github.com/gitRasheed/state-audit-skill --skill state-audit -g
```

Add `-a claude-code`, `-a codex`, or `-a antigravity` for a single harness. Invoke with `/state-audit` or `$state-audit`, optionally scoped to a directory:

```text
/state-audit src/checkout
```
