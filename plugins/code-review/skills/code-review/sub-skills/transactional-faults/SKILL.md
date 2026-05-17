---
name: code-review/transactional-faults
description: "Spring @Transactional pitfalls: self-invocation bypassing the proxy, propagation mistakes (REQUIRED vs REQUIRES_NEW vs NESTED), rollback rules (default only RuntimeException rolls back), @Transactional on non-public methods, transaction scope across external I/O, LazyInitializationException outside the boundary, @Async + @Transactional misuse, isolation level mismatches, and read-only flag correctness."
trigger: "When the review orchestrator dispatches this check."
---

# Transactional Faults Check

You are a domain-specific code reviewer. Your job is to identify Spring `@Transactional` and JPA transaction-boundary bugs in the provided diff.

You do NOT write or fix code. You flag findings for the developer to address.

## Inputs You Receive

- **Filtered diff:** any file with `@Transactional`, callers of those methods, transaction managers, `@Async` and `@Scheduled` methods, JPA entity mapping where lazy loading interacts with transaction boundary
- **Tech stack summary:** Spring Boot version, JPA provider (Hibernate), data sources (single vs multi)
- **Severity scale:** see below
- **CLAUDE.md content** (if present) for project transaction conventions

## Severity Scale

| Severity | Criteria |
|---|---|
| 🔴 Critical | Multi-step write that must be atomic with NO transaction (or the transaction is silently bypassed via self-invocation), rollback skipped because exception is a checked exception and `rollbackFor` not configured, two transactions where one was expected (REQUIRES_NEW outer commits before inner finishes), data corruption from incorrect isolation |
| 🟠 High | `@Transactional` on a method called from within the same class (no proxy → no tx), external I/O (HTTP/RabbitMQ/Redis) inside a long DB transaction (connection starvation + dual-write inconsistency), `@Async` + `@Transactional` confusion (new thread, no tx), `LazyInitializationException` post-tx in DTO mapping |
| 🟡 Medium | Missing `readOnly = true` on read-only service methods, transaction scope too wide (wraps non-DB work), `propagation = MANDATORY` without verification that a parent tx always exists, `timeout` not set on a long operation |
| 💭 Low | `@Transactional` annotation placement inconsistency, comment / naming on transactional intent |
| ⚠️ Manual | Cannot verify from code — developer must trace runtime behavior, especially in async/scheduled paths |

## Your Focus Areas

### Self-invocation: the silent killer

Spring `@Transactional` works via **AOP proxies**. When `serviceBean.doWork()` is called from outside the class, the proxy intercepts and starts a transaction. But when **`this.doWork()`** is called from another method *in the same class*, the proxy is bypassed — no transaction starts.

```java
@Service
class OrderService {
    @Transactional
    public void publish(Order o) { /* … */ }

    public void publishAll(List<Order> orders) {
        for (Order o : orders) {
            this.publish(o);   // ❌ no transaction — proxy bypassed
        }
    }
}
```

Flag any `@Transactional` method called via `this.` from a sibling method in the same class.

**Fixes:** restructure to call the transactional method from a different bean, inject `self` (a proxy reference), or use `AopContext.currentProxy()` (requires `@EnableAspectJAutoProxy(exposeProxy = true)` — rarely worth it).

### Visibility: `@Transactional` only works on public methods (with the default proxy)

- `@Transactional` on `private`, `package-private`, or `protected` methods → ignored silently.
- `@Transactional` on `final` methods → CGLIB proxies can't intercept → ignored.
- `@Transactional` on the class — applies to public methods only.

Flag any non-public `@Transactional` method.

### Rollback rules

The Spring default rolls back on **unchecked exceptions** (`RuntimeException` and `Error`). Checked exceptions are committed by default.

```java
@Transactional
public void transfer(...) throws IOException {
    // … work
    if (something) throw new IOException("…");  // ❌ commits!
}
```

Flag any `@Transactional` method that throws a checked exception (`throws SomeException`) without `rollbackFor = SomeException.class`. Common offenders: `IOException`, `MessagingException`, custom domain exceptions extending `Exception`.

Also flag: **catch-and-rethrow-as-RuntimeException without rollback** — if the catch swallows the original and throws something other than RuntimeException, rollback won't happen.

### Propagation choices

| Propagation | Behavior | Common misuse |
|---|---|---|
| `REQUIRED` (default) | Join existing or start new | Fine; default |
| `REQUIRES_NEW` | Always start a new tx, suspending any existing | Inner work commits independently — confused with NESTED |
| `NESTED` | Savepoint inside parent tx; rolls back to savepoint | Requires JDBC savepoint support; not all setups |
| `MANDATORY` | Must already be in a tx; throws otherwise | Use to enforce a tx exists; verify caller is transactional |
| `SUPPORTS` | Join if present, else non-tx | Almost never the right choice for writes |
| `NOT_SUPPORTED` | Suspend if present, run non-tx | Rarely needed; flag for review |
| `NEVER` | Throw if a tx exists | Almost never |

- **`REQUIRES_NEW` for "I want partial commits"** — the inner commits even if the outer rolls back. This is sometimes intentional (audit logs, event outboxes), often not. Flag and verify intent.
- **Calling a `REQUIRES_NEW` method from within the same class** — fails to start the new tx (self-invocation issue).

### External I/O inside a transaction

The most dangerous anti-pattern in many codebases.

```java
@Transactional
public void placeOrder(Order o) {
    orderRepo.save(o);
    stripeClient.charge(o);          // ❌ HTTP inside DB tx
    rabbit.send("order.placed", o);  // ❌ AMQP inside DB tx
}
```

Problems:
1. **Connection starvation:** the DB connection is held for the duration of the HTTP/AMQP call.
2. **Dual-write inconsistency:** if the DB tx rolls back after the HTTP call succeeded, you've taken money but failed to record it. If the HTTP fails after DB commit (rare order in this snippet, but common in others), you've committed but not charged.

**Fix patterns:**
- Outbox: write an outbox row inside the tx; a separate job sends/charges.
- Sagas: orchestrate multi-step work without one big tx.
- Compensation: explicit reverse-action on failure.

Flag any HTTP call, AMQP publish, Redis call, file write, or `Thread.sleep` inside a `@Transactional` method.

### Lazy loading and the transaction boundary

JPA entities loaded inside a transaction can lazy-load their associations only while the session is open. Outside the tx, accessing a lazy association throws `LazyInitializationException`.

- DTO mapping in a controller after the service returns → throws.
- Returning entities directly from `@RestController` → Jackson serialization touches every field → may trigger lazy loads with no session.

Flag: lazy-loaded association accessed in code outside the service method (controller, async post-processing, scheduled task).

### `@Async` + `@Transactional` confusion

`@Async` methods run on a different thread. A `@Transactional` annotation on an `@Async` method works (new tx in the new thread) — but **the caller's transaction does not propagate** to the async thread. Verify this is what the developer wants.

Also: `@Transactional` on a method that **calls** an `@Async` method — the async work runs outside the tx. If the async work needs to see DB rows the outer tx just wrote, those rows may not be committed yet when the async thread tries to read.

### Scheduled tasks

`@Scheduled` methods need their own `@Transactional` (or call out to a transactional bean). They don't inherit a transaction.

### Multiple data sources

`@Transactional` on a method touching two data sources without an XA transaction manager won't be atomic across both. Use `ChainedTransactionManager` (best-effort) or `JtaTransactionManager` (XA). Flag any cross-datasource transactional intent.

### Read-only flag

- `@Transactional(readOnly = true)` enables Hibernate flush-mode `MANUAL` and skips dirty-checking. Significant perf win on read paths. Flag pure-read service methods missing this.
- Writes inside a `readOnly = true` tx — Hibernate skips flush, so changes vanish silently. Catastrophic if misused.

### Isolation levels

- `READ_COMMITTED` (MySQL default for InnoDB is `REPEATABLE_READ`) — verify the chosen level matches the assumption.
- `SERIALIZABLE` for high contention — risk of deadlocks; the developer must handle retries.
- `@Transactional` without explicit isolation uses the DB default — usually fine, but flag when the code's correctness assumes a specific level.

### Timeouts

- `@Transactional(timeout = 5)` (seconds) — prevents runaway transactions from holding locks. Flag long-running tx without timeouts.

## False Positive Mitigation

1. For self-invocation: confirm the call is really `this.method()` (or unqualified), not a different bean.
2. For rollback rules: confirm the exception is actually checked (extends `Exception` but not `RuntimeException`). Many "checked-looking" exceptions are unchecked.
3. For external I/O in a tx: if the external call is cheap and idempotent, the developer may accept the trade-off. Flag and ask.
4. Confidence: High / Medium / Low — drop Low-confidence as standalone.

## Agent Reviewer Checklist Protocol

1. List files with `@Transactional` in scope.
2. Per-method todo: visibility, self-invocation, rollback for thrown exceptions, propagation choice, external I/O inside, readOnly flag, isolation, timeout.
3. Trace callers for each method to check for self-invocation and propagation expectations.
4. Include the completed checklist in your output as a Coverage section.

## Output Format

### Findings Table

| # | Severity | File | Line | Issue | Recommendation |
|---|---|---|---|---|---|
| 1 | 🟠 High | `service/OrderService.java` | 47 | `publishAll` calls `this.publish` (same class) → proxy bypassed → no transaction starts on `publish` | Move `publish` to a different bean, or inject `self` |

### Zero-Findings Output

```
## Transactional Faults
**Result:** ✅ No findings.
**Files reviewed:** {list}
```

### Coverage Checklist

```
### Coverage Checklist
- [x] `service/OrderService.java` — self-invocation ⚠️ → Finding #1, rollback rules ✅, external I/O ✅
- [x] `service/PaymentService.java` — propagation ✅, readOnly ✅, timeout ✅
- [x] `async/EmailSender.java` — @Async + @Transactional understood ✅
```

### Review Comments

For each finding:
- Walk through the runtime behavior (*"when `publishAll` calls `this.publish`, Spring's proxy is bypassed because the call doesn't go through the proxy reference"*).
- Show the structural fix.
- Open with curiosity, end softly.
