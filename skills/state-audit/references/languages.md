# Language Idioms for Explicit State

Match the surrounding repo's style and keep new state types module-private unless callers need them. Do not add new test or state-machine dependencies unless they are already in the repo or approved by the user.

## Rust
- **Representation:** Replace correlated `bool`/`Option<T>` fields with an `enum` whose variants own their state-specific data (`enum Job { Queued, Running { started_at: Instant }, Failed { reason: String } }`).
- **Transitions:** Consuming `self` (typestate) prevents reusing an old state at compile time for owned values, but does not fit cleanly behind `Arc`/`Mutex` or `&mut self` containers. For in-place `&mut self` transitions, note that `mem::replace` requires a placeholder variant (which can itself become a new invalid state); prefer returning the next state from a pure function or keeping transitions atomic.
- **Guards:** Avoid wildcard `_ =>` arms on state enums so adding a new state forces compile-time handling of every transition.
- **Verification:** Transition matrix unit tests; add `proptest` event-sequence tests or `loom` concurrency checks only if already in the repo.

## Python
- **Representation:** Closed `Enum` or `Literal` union, or tagged frozen `@dataclass(slots=True)` variants with an explicit discriminator field (e.g. `kind: Literal["running"] = "running"`). Note `frozen=True` is shallow: mutable fields (`list`, `dict`) remain mutable, so use `tuple` or `MappingProxyType` for immutable state payloads.
- **Transitions:** Route mutations through one pure `transition(state, event) -> tuple[State, list[Effect]]` function or an explicit `ALLOWED_TRANSITIONS` table that raises on illegal moves.
- **Guards:** Python does not enforce exhaustiveness at runtime; use `typing.assert_never(state)` with a static checker (`pyright`/`mypy`) and cover touched transitions in tests.
- **Verification:** Transition unit tests; use `hypothesis.stateful.RuleBasedStateMachine` only if `hypothesis` is already in the repo.

## Java (21+)
- **Representation:** `sealed interface` with `record` variants per state plus exhaustive `switch` expressions (no `default` branch so new states fail compilation).
- **Persistence and side effects:** Guard persisted transitions with optimistic locking (`version`) or conditional SQL (`UPDATE jobs SET status = 'running' WHERE id = ? AND status = 'queued'`) and check affected row count (`== 1`). Because a DB transaction and an external network call are not atomic together, use a transactional outbox or claim-then-recover with an idempotency key for external side effects.
- **Verification:** JUnit transition tests plus `CountDownLatch`/`ExecutorService` tests exercising competing workers on the same row.

## C++ (17/20+)
- **Representation:** `std::variant<StateA, StateB, ...>` of small state structs visited via `std::visit` with explicit per-state handlers (avoid a catch-all generic `[](const auto&)` in `overloaded`, which silently defeats compile-time exhaustiveness).
- **Gotchas:**
  - `std::variant` can become `valueless_by_exception` if a throwing assignment/emplace fails; keep state moves `noexcept`.
  - C++ sum types are not affine: `&&`-qualified transition methods do not prevent callers from touching a moved-from object. Check with `clang-tidy` (`bugprone-use-after-move`) and tie resource handles to RAII types owned by the active state.

## TypeScript
- **Representation:** Discriminated union (`type State = { kind: "idle" } | { kind: "loading"; attemptId: number } | { kind: "error"; message: string }`) with `readonly` fields. Validate untrusted JSON/DB inputs at the boundary into this type; TypeScript types vanish at runtime.
- **Transitions and handles:** Pure `transition(state: State, event: Event): { state: State; effects: Effect[] }` with `switch (state.kind)` ending in an exhaustive `never` check. Keep non-serializable handles (`AbortController`, timers, sockets) in the effect runner keyed by `attemptId` rather than inside serializable state data.
- **Library rule:** Plain discriminated unions + pure reducer cover most flows. Reach for a state-machine library (such as XState) only for 4+ states with nested/parallel async states where a union + switch breaks down.
