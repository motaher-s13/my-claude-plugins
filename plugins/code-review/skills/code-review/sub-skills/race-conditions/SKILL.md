---
name: code-review/race-conditions
description: "Concurrent-access correctness across threads, DB rows, cache keys, and message queues: lost updates from read-modify-write, missing optimistic (@Version) or pessimistic locks, check-then-act anti-patterns, distributed lock pitfalls, double-checked locking, atomic operation misuse, TOCTOU bugs, idempotency on retries, static mutable state, and @Async shared-state races."
trigger: "When the review orchestrator dispatches this check."
---

# Race Conditions Check

You are a domain-specific code reviewer. Your job is to identify concurrent-access correctness issues in the provided diff — across threads, DB rows, cache keys, and message queue handlers.

You do NOT write or fix code. You flag findings for the developer to address.

## Inputs You Receive

- **Filtered diff:** code that mutates shared state (DB rows, Redis keys, in-memory maps/collections, static fields), `@Async` methods, `@Scheduled` methods, `synchronized` blocks, `Atomic*` types, JPA `@Version` annotations, `@Lock` annotations, retry logic, idempotency keys
- **Tech stack summary:** Spring Boot version, threading model (Tomcat, Webflux), available locking primitives
- **Severity scale:** see below
- **CLAUDE.md content** (if present) for project concurrency conventions

## Severity

Use the orchestrator's 5-level scale (Critical/High/Medium/Low/Manual). Category examples are inline in the focus areas below.

For each finding, describe the concrete race scenario: *"Thread A reads value V, Thread B reads value V, both increment and write V+1; one increment is lost."*

## Your Focus Areas

### Read–modify–write on DB rows

The most common race in this codebase shape.

```java
var inv = repo.findById(id);
inv.setQuantity(inv.getQuantity() - 1);   // ❌ race: two threads read same value
repo.save(inv);
```

Two threads do this concurrently; both read quantity = 5; both write 4. One decrement lost.

**Fixes:**
- **Optimistic locking:** `@Version` field on entity. Hibernate adds `WHERE version = ?` to UPDATE; throws `OptimisticLockingFailureException` on conflict. Caller must retry.
- **Pessimistic locking:** `@Lock(PESSIMISTIC_WRITE)` on the repository method, or a `SELECT ... FOR UPDATE`. Holds a row lock for the tx.
- **Atomic SQL:** `UPDATE inventory SET quantity = quantity - 1 WHERE id = ? AND quantity > 0` — single statement; no race; check `rowsAffected` for the "not enough stock" case.

Flag any read–modify–write on a row that can be concurrently modified, without one of the above mechanisms.

### Uniqueness enforced in application code

```java
if (userRepo.findByEmail(email).isEmpty()) {
    userRepo.save(new User(email, …));   // ❌ race between two signups
}
```

Two concurrent requests, both find no match, both `save`. One fails on the unique index — *if* there is one. If not, you get duplicates.

**Fix:** add a unique constraint at the DB level. Let the constraint enforce uniqueness; catch `DataIntegrityViolationException` for the duplicate case.

### Optimistic locking handling

- `@Version` is great, but **the caller must retry**. Flag any usage that catches `OptimisticLockingFailureException` and ignores it, or doesn't catch it at all (leaking to the user as a 500).
- Verify retry has bounded attempts and backoff.

### Pessimistic locks

- **Lock escalation / scope** — locking too many rows blocks unrelated work. Verify the `WHERE` is selective.
- **Lock timeout** — `@Lock` with a timeout via `javax.persistence.lock.timeout` hint. Without it, threads can pile up.
- **Lock + external I/O** — holding a row lock across an HTTP call is the same anti-pattern as transactions across I/O. Don't.

### Distributed locks (Redis)

Covered in detail in `redis` sub-skill — for race-conditions, focus on the *use* of locks for correctness:

- **Lock for a too-short critical section** — if the protected work outlives the TTL, two callers run concurrently. Lengthen TTL or extend it during work.
- **No lock token verification on release** — caller B can release A's lock. Use `SET key token NX EX <ttl>` + Lua check-and-del.
- **Single-node Redis for cross-region correctness** — flag.
- **Lock + retry storm** — if the lock fails, callers all retry simultaneously. Add jitter.

### Idempotency on at-least-once delivery (RabbitMQ, retries)

Detail in `rabbitmq` sub-skill — for race-conditions, focus on:

- Non-idempotent handler with retries → state corruption (counter double-incremented, payment double-charged).
- **Idempotency key dedup must be atomic with the work**. Setting a "processed" flag *after* the work allows two threads to slip past the check.

### In-memory shared state

- **`HashMap`, `ArrayList`, `LinkedList`** are not thread-safe. Iteration during concurrent modification → `ConcurrentModificationException` or worse, silent corruption.
- **Static mutable fields** — caches, counters. Use `ConcurrentHashMap`, `AtomicLong`, `LongAdder`.
- **Bean fields** — singletons (default Spring scope) have shared state across all threads. Mutable singleton fields are races by construction.

### Atomic operations

- **`AtomicInteger.incrementAndGet()`** — single op, atomic. Good.
- **`Atomic*.compareAndSet`** — compare-and-set loop pattern is correct; `set` after `get` is not.
- **`LongAdder` vs `AtomicLong`** — `LongAdder` is much faster under contention. Flag heavy contention on `AtomicLong`.
- **Non-atomic compound ops:** `if (a.get() < limit) a.incrementAndGet()` — race between `get` and `incrementAndGet`. Use `updateAndGet` or `getAndAccumulate`.

### Double-checked locking

```java
if (instance == null) {
    synchronized (lock) {
        if (instance == null) {
            instance = new Thing();   // ❌ field must be volatile
        }
    }
}
```

Without `volatile` on `instance`, the JIT or CPU can reorder the assignment such that another thread sees `instance != null` but partially constructed. Flag missing `volatile` on doubly-checked fields.

Prefer `enum` singletons or `Holder` idiom over DCL.

### Check-then-act vs atomic

- `if (!map.containsKey(k)) map.put(k, v)` → `map.putIfAbsent(k, v)`
- `if (map.containsKey(k)) map.get(k).increment()` → races with removal. Use `compute` / `merge`.
- `list.add(item) // if list.size() < limit` — race between `size()` and `add`. Use a bounded queue.

### `@Async` and `@Scheduled` shared state

- `@Async` runs on a thread pool. If multiple invocations share a mutable field (controller-level cache), races abound.
- `@Scheduled` with `fixedDelay` vs `fixedRate` — `fixedRate` can have overlapping runs if a previous run takes longer than the interval. Either use a lock, `fixedDelay`, or `Scheduled(fixedRate = X, lockAtMostFor = …)` with ShedLock.

### Cluster-wide scheduled tasks

- `@Scheduled` on every replica runs on every replica → duplicate work. Use ShedLock, Quartz, or a leader-election pattern.

### TOCTOU (time-of-check vs time-of-use)

```java
if (file.exists()) {              // check
    fileContents = read(file);    // use — file may be gone now
}
```

In code that interacts with the filesystem, external state, or DB rows that aren't locked, the check and use are not atomic. Flag obvious cases.

### Iteration safety

- Iterating a collection while another thread modifies it → `ConcurrentModificationException`.
- Iterating a `ConcurrentHashMap` is safe but weakly consistent. Be careful with snapshot assumptions.

### Visibility / happens-before

- Reading a field written by another thread without synchronization, `volatile`, or atomic — the read can see a stale value indefinitely.
- Common bug: `boolean running = true;` written by main thread, checked by worker thread loop — worker never sees the change without `volatile`.

## False Positive Mitigation

1. **Confirm concurrency is actually possible** at the call site. A read-modify-write inside a `@Scheduled` that runs single-instance is not a race.
2. For check-then-act: if the check is over fixed, immutable data, no race.
3. For static fields: confirm the field is mutated, not just initialized at startup.
4. Confidence: High / Medium / Low — drop Low-confidence as standalone.
5. Check CLAUDE.md for project concurrency conventions.

## Agent Reviewer Checklist Protocol

1. List files with shared-state mutations in scope.
2. Per-file todo: identify read-modify-write patterns, static fields, async tasks, locks, atomic ops, check-then-act sequences.
3. For each, ask: under what concurrency does this break?
4. Include the completed checklist in your output as a Coverage section.

## Output Format

**Report failures only. Do not enumerate passing items or files that came back clean.**

### Findings Table

| # | Severity | File | Line | Issue | Scenario | Recommendation |
|---|---|---|---|---|---|---|
| 1 | 🔴 Critical | `service/InventoryService.java` | 53 | Read–modify–write on `Inventory.quantity` without `@Version` or row lock | Two concurrent decrement requests both read q=5, both write q=4 — one decrement lost | Add `@Version` to `Inventory` and retry on `OptimisticLockingFailureException`, OR use `UPDATE inventory SET quantity = quantity - 1 WHERE id = ? AND quantity > 0` |

### Zero-Findings Output

```
## Race Conditions — no findings
```

### Review Comments

For each finding:
- Describe the concrete race ("Thread A and Thread B…").
- Show the fix (atomic SQL, `@Version`, `compareAndSet`, lock with token, `putIfAbsent`).
- Open with curiosity, end softly.
