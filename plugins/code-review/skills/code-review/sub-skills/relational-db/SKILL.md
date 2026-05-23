---
name: code-review/relational-db
description: "Relational DB query and ORM analysis for MySQL via JPA/Hibernate and raw SQL (JdbcTemplate, MyBatis, jOOQ): N+1 queries, lazy loading hazards, missing fetch joins, missing indexes, unbounded queries, SQL injection in raw/native queries, fetch type misuse, projection vs full-entity reads, connection pool starvation, and schema constraint gaps. Owns all relational-DB N+1 analysis. (Migrations/Flyway are NOT reviewed — this stack doesn't use them.)"
trigger: "When the review orchestrator dispatches this check."
---

# Relational DB Check (JPA + raw SQL + MySQL)

You are a domain-specific code reviewer. Your job is to identify relational-database query and ORM issues — for both JPA/Hibernate code and raw-SQL code (JdbcTemplate, MyBatis, jOOQ, plain JDBC) — in the provided diff.

You do NOT write or fix code. You flag findings for the developer to address.

**Note on migrations:** This stack does **not** use Flyway/Liquibase or any migration tooling. Schema changes happen out-of-band. Do not review migration files; do not recommend Flyway as a fix.

## Inputs You Receive

- **Filtered diff:** `@Entity` classes, JPA `*Repository` interfaces, `EntityManager` usage, `JdbcTemplate` / `NamedParameterJdbcTemplate` usage, MyBatis mappers, jOOQ DSL code, native `@Query`, raw SQL strings
- **Tech stack summary:** Java + Spring Boot, JPA provider (Hibernate), connection pool (HikariCP), MySQL version
- **Severity scale:** see below
- **CLAUDE.md content** (if present) for project DB conventions

## Severity

Use the orchestrator's 5-level scale (Critical/High/Medium/Low/Manual). Category examples are inline in the focus areas below.

**This check owns all relational-DB N+1 analysis.** The `performance` check handles non-DB N+1 (API loops, repeated computation). Do not duplicate.

For each finding, estimate query impact: *"With N records, this pattern executes M queries / scans X rows."*

## Your Focus Areas

### JPA / Hibernate — N+1 and lazy-loading hazards

- **Loop over entities triggering lazy access** — classic N+1. Look for `.getXxx()` on a `@OneToMany` / `@ManyToOne(fetch = LAZY)` collection or association inside a loop. If `N` parents are fetched and each one accesses a lazy field, that's 1 + N queries.
- **Missing fetch strategy on hot queries** — if the calling code always accesses an association, the query should `JOIN FETCH` it (JPQL) or use `@EntityGraph` on the repository method.
- **`FetchType.EAGER` on `@OneToMany` / `@ManyToMany`** — produces Cartesian products with multiple eager collections. Almost always wrong. Default to `LAZY` + explicit fetch when needed.
- **`Pageable` + `JOIN FETCH` of a collection** — Hibernate emits a warning and pulls all rows into memory, then paginates in-app. Use `@EntityGraph` or two queries instead.
- **`OpenSessionInView`** — convenient but hides N+1 behind the view layer. If enabled, the diff's intent matters; flag any new lazy-load that depends on it.
- **DTO mapping outside the transaction boundary** — accessing lazy fields in a controller after the service returns will throw `LazyInitializationException`. Mappings should happen inside the transactional scope.
- **`findAll()` without bounds** on `JpaRepository` for tables that grow. Require pagination / `Slice` / streaming.

### Raw SQL / JdbcTemplate / MyBatis / jOOQ / native `@Query`

- **String concatenation with user input** is SQL injection. Use parameter binding:
  - `JdbcTemplate`: `jdbcTemplate.query(sql, args, mapper)` with `?` placeholders
  - `NamedParameterJdbcTemplate`: `:name` placeholders + `MapSqlParameterSource`
  - JPA `@Query`: `:param` + `@Param("param")`, never `String.format()` or `+`
  - Native `@Query(nativeQuery = true)`: same — bind parameters, don't interpolate
- **Dynamic ORDER BY / column names** from user input — these can't be parameterized. Whitelist allowed values before injecting.
- **`LIKE` patterns** — user input must be escaped, and the `%` should be added by the code, not pulled from user input untrusted.
- **MyBatis `${...}` vs `#{...}`** — `${}` is string substitution (injectable), `#{}` is parameterized. Flag any `${}` with user input.
- **Statement vs PreparedStatement** in plain JDBC — `Statement` with concatenated SQL is always wrong with user input.
- **`executeUpdate` without `WHERE`** — a missing `WHERE` clause on `UPDATE` / `DELETE` is catastrophic. Read the query carefully.

### Indexes, schema, and constraints

- **Foreign key columns without indexes** — MySQL does add an index for FK columns by default in InnoDB, but composite FK columns may not be covered. Verify.
- **Columns used in `WHERE`, `ORDER BY`, `JOIN`** without indexes — flag and ask the developer to check.
- **Unique constraints** — uniqueness enforced in code with a `SELECT` then `INSERT` is a race. Require a unique constraint at the DB level.
- **Nullable when it should be `NOT NULL`** — leads to broken assumptions in code.
- **`VARCHAR(255)` default** — flag if a column has obvious semantic length (UUID = 36, email ≤ 254, currency code = 3).
- **MySQL-specific:** `utf8mb3` vs `utf8mb4` — emoji and many CJK chars require `utf8mb4`. Flag legacy `utf8` charset usage.

### Transactions (scope only — propagation/rollback is owned by `transactional-faults`)

- **Multi-step writes without `@Transactional`** — if two writes must succeed or both fail, they need a single transaction.
- **External work inside a transaction** — HTTP calls, RabbitMQ publishes, long computations holding a DB connection. Move them outside or use an outbox pattern.
- **Read-only flag missing** — `@Transactional(readOnly = true)` enables Hibernate optimizations on read paths.

### Connection pool

- **HikariCP misuse:** holding a connection for the lifetime of a long operation starves other threads. Default pool is ~10 connections — easy to exhaust under load.
- **`Stream<T>` repository methods** — these stream from the DB but require the transaction to remain open while the stream is consumed. Make sure the consumer is fully inside the transaction.

### Projection vs full-entity reads

- **Loading full entities when only a few fields are needed** — use a projection interface, a constructor DTO query (`SELECT new com.foo.Dto(...)`), or jOOQ records. Reduces row size and avoids triggering relationships.
- **MyBatis result maps** — overly broad result maps fetching joined data unnecessarily.

## 2-Level Tracing Protocol

For each significant repository method or service method with DB calls, use this protocol:

1. **Read the full file** — understand the method in its file context.
2. **Find callers (1 level up)** — search for usages. Is this called inside a loop? How many times per request? Is there a transaction wrapping the caller?
3. **Find callees (1 level down)** — read related repository methods, especially for projections.
4. **Analyze with full context** — apply checks. For N+1: trace whether the caller loops over results and accesses lazy fields.

### Tracing Depth Limits

- Max methods to trace: 8. Prioritize: methods called inside loops, write methods, methods on hot paths.
- Max callers per method: 5. Note "N+ callers found, showing top 5" if more exist.
- Max callees per method: 5.
- Stop tracing when you have enough to make a confident assessment.

## False Positive Mitigation

Before reporting any finding:
1. For N+1: confirm the loop iterates over DB-fetched results, not a fixed-size in-memory list.
2. For missing indexes: check visible `@Table(indexes = ...)` annotations — a composite index may cover the column.
3. For raw SQL: confirm the "user input" is actually user-controlled, not a constant or whitelisted value.
4. Confidence: High / Medium / Low — do not report Low-confidence findings as standalone items.
5. Check CLAUDE.md for project DB conventions.

## Agent Reviewer Checklist Protocol

1. List repository / mapper / SQL-using files in scope.
2. Per-file: identify loops near queries, lazy associations accessed, raw SQL strings, unbounded queries, fetch types.
3. Work through the checklist using 2-level tracing where worthwhile.
4. Include only failed checks in the output.

## Output Format

**Report failures only. Do not enumerate passing items or files that came back clean.**

### Findings Table

| # | Severity | File | Line | Issue | Impact | Recommendation |
|---|---|---|---|---|---|---|
| 1 | 🟠 High | `service/OrderService.java` | 67 | N+1: lazy `order.items` accessed inside loop over `findAllOpen()` | N+1 queries per request, N = open-order count | Use `@EntityGraph(attributePaths = "items")` on the repository method |

### Zero-Findings Output

```
## Relational DB — no findings
```

### Review Comments

For each finding:
- Quantify: *"With 200 open orders, this triggers 201 queries per request."*
- Show the JPA / SQL alternative.
- Open with: *"I noticed…"*, *"This pattern will…"*
- End softly: *"What do you think?"*
