# State Audit Patterns

Concrete bug classes to look for in async, concurrent, persisted, or resource-owning code.

## 1. Correlated flags stored as a product type
- `isLoading`, `isError`, `isSuccess`, `data`, `error` updated across separate statements or hooks.
- Timestamps acting as implicit states (`paidAt`, `refundedAt`, `cancelledAt`, `deletedAt`) with no mutual-exclusion guard.
- Check: can two mutually exclusive fields be truthy or non-null at the same time? If proving "no" requires tracing multiple files, it belongs in the inventory.

## 2. Non-terminal states with no guaranteed exit
- States like `pending`, `processing`, `uploading`, `reconnecting` that only advance on the happy-path callback.
- Check: what moves the entity out if the worker crashes mid-step, the network drops, the webhook never arrives, or the user cancels? Look for a timeout, sweeper, lease expiry, or crash-recovery path.

## 3. Stale async completions (missing generation / attempt guard)
- Any `await`, timer callback, websocket `onclose`, or background task that writes state after resuming without checking that the current attempt/generation ID still matches.
- Classic failure: request A starts, user cancels or starts request B, request A finishes late and overwrites B's state.

## 4. Orphaned resources with no owning state
- Timers, intervals, sockets, file locks, media handles, or child processes held in module/instance variables outside the state that created them.
- Check every exit path (error, cancel, timeout, reconnect, shutdown): is the handle guaranteed to be cleaned up once?

## 5. Non-atomic check-then-act, retries, and DB/effect divergence
- Read status -> check `if status == "pending"` -> perform external side effect (charge, email, webhook, enqueue) -> write `status = "done"`.
- Even with an atomic `UPDATE ... WHERE id = ? AND status = 'pending'`, checking affected rows before an external call does **not** make the DB commit and external call atomic: a DB rollback after the call, or a crash after commit before the call, still diverges. Look for a **transactional outbox** or **claim-then-recover** pattern with an idempotency key on the external side effect.
- Check DB transaction isolation and row locking (`SELECT ... FOR UPDATE` or optimistic `version` column) when two workers can inspect the same row concurrently.

## 6. Lease, clock, delivery, and deploy-skew races
- **Fencing tokens:** lease expiry alone is unsafe; a paused or GC-stalled lease holder can wake up and overwrite state after another worker took over unless writes check a monotonic fencing token or version column.
- **At-least-once delivery:** queue/webhook redelivery after an ack loss re-runs transitions; verify transitions are idempotent or deduped.
- **Wall-clock vs monotonic TTLs:** comparing `Date.now()` / `time.time()` across hosts or NTP jumps for expiry/backoff can prematurely expire or stall leases.
- **Rolling deploy / schema skew:** persisted rows written by an older deploy or enum values added mid-rollout (expand/contract migrations); check what old readers do when they see a new state value.

## 7. External and cross-layer state drift
- State split across DB columns, caches, job queues, files on disk, and third-party providers.
- Check: if step 2 fails after step 1 commits, which store is authoritative and how does reconciliation recover on restart?

## 8. Authority leaks on transitions
- A generic `setStatus(id, newStatus)` endpoint or public handler wrapping an internal mutation without verifying who is allowed to trigger that move (e.g. a client marking an order `paid` instead of a verified webhook).
