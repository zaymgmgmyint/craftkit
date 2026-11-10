# Backend standard — Java 21 · Spring Boot 3.x · JPA/Hibernate · PostgreSQL

Used by `/pr-review` when a repo's `review-rules.md` says `Standards: backend-java`.
Project rules in `review-rules.md` are added on top and win if they conflict.

## How to use these rules
- Only flag a rule when you can point to a **specific line** in the diff.
- Cite the rule ID in the comment title, e.g. `C1 · issue (blocking) [BE-SEC-02] — ...`.
  The `/review-summary` skill counts findings by rule ID.
- Severity: **B** = blocking, **N** = non-blocking. You may raise N to B if the
  impact in this PR is serious; say why.
- Formatting, import order and brace style belong to the formatter/linter
  (Checkstyle, Spotless). Do **not** comment on them.
- Rules apply to changed code only. Problems in untouched code go under
  "Pre-existing (not blocking)".

---

## 1. Security (OWASP)

| ID | Rule | Sev | How to check |
|---|---|---|---|
| BE-SEC-01 | Every new or changed endpoint has an authentication and permission check (`@PreAuthorize`, security config, or explicit check). | B | Find each new `@*Mapping`; trace whether access is restricted. |
| BE-SEC-02 | Loading a record by ID also checks the caller may access **that** record (owner / tenant / role). No `findById(id)` straight from a request parameter. | B | Path/query IDs → repository call → is there an ownership or scope filter? (IDOR) |
| BE-SEC-03 | SQL and JPQL use bind parameters only. No string concatenation or `String.format` of user input into queries, including `ORDER BY` / sort fields (allow-list those). | B | Search diff for `+` / `format(` / `formatted(` near `createQuery`, `createNativeQuery`, `@Query`, `JdbcTemplate`. |
| BE-SEC-04 | Request bodies and parameters are validated (`@Valid` + Bean Validation constraints on DTOs). | B | New DTOs without constraints; controller params without `@Valid`. |
| BE-SEC-05 | No secrets in code or committed config (passwords, keys, tokens). Use environment variables / secret store. | B | Literal credentials in `.java`, `application*.yml`, test files. |
| BE-SEC-06 | Logs never contain passwords, tokens, full personal data, or raw request bodies with personal data. | B | New `log.*` calls with DTOs, `toString()` of entities, headers. |
| BE-SEC-07 | Error responses never expose stack traces, SQL or internal class names to the client. | B | Exception handlers returning `e.getMessage()` / stack traces. |
| BE-SEC-08 | CORS is not opened to `*` with credentials; CSRF is only disabled for stateless token APIs, with a reason. | B | Changes in `SecurityFilterChain` / `CorsConfiguration`. |

## 2. Spring Boot

| ID | Rule | Sev | How to check |
|---|---|---|---|
| BE-SPR-01 | Use constructor injection with `final` fields. No field `@Autowired`. | N | `@Autowired` on fields; non-final injected fields. |
| BE-SPR-02 | Controllers are thin: no business logic or repository calls. Logic lives in services. | N | Repository injected into a controller; `if`/loops of business rules in controller. |
| BE-SPR-03 | `@Transactional` is on the service layer. Read-only service methods use `@Transactional(readOnly = true)`. | N | Transactions on controllers; reads without `readOnly`. |
| BE-SPR-04 | No call from one method to a `@Transactional` method **in the same class** expecting a new/different transaction — the proxy is bypassed and it silently doesn't apply. | B | `this.otherMethod()` where `otherMethod` has its own `@Transactional` settings. |
| BE-SPR-05 | No slow external calls (HTTP, MQTT, devices, email) inside a DB transaction. | N | Remote client calls inside `@Transactional` methods. |
| BE-SPR-06 | Configuration is read through typed `@ConfigurationProperties` (records are fine), not scattered `@Value` strings. | N | New `@Value("${...}")` in several classes for one feature. |
| BE-SPR-07 | Errors map to correct HTTP status: client mistakes → 4xx, not found → 404, conflicts → 409, server faults → 5xx. Use one global `@RestControllerAdvice`, ideally returning `ProblemDetail`. | B if a client error returns 500; else N | Thrown exceptions with no handler; generic `catch (Exception)` returning 500. |
| BE-SPR-08 | Outgoing HTTP clients have connect and read timeouts. | B | New `RestClient` / `WebClient` / `RestTemplate` without timeouts. |

## 3. JPA / Hibernate

| ID | Rule | Sev | How to check |
|---|---|---|---|
| BE-JPA-01 | No N+1 queries: a loop or mapping over a list must not trigger a lazy load per item. Use fetch join, `@EntityGraph`, or a DTO projection. | B on list endpoints; else N | Loops/streams touching associations after a `findAll`/list query. |
| BE-JPA-02 | `@ManyToOne` and `@OneToOne` are `fetch = FetchType.LAZY` (JPA default is EAGER). | N | New associations without explicit LAZY. |
| BE-JPA-03 | Controllers never return entities. Return DTOs/records. | B | `ResponseEntity<SomeEntity>` / entity as return type. |
| BE-JPA-04 | List endpoints are paginated (`Pageable` / limit). No unbounded `findAll()` on tables that grow. | B | New list endpoints without paging. |
| BE-JPA-05 | Don't combine collection fetch join with pagination (Hibernate paginates in memory, warning HHH90003004). | B | `JOIN FETCH` of a collection + `Pageable`. |
| BE-JPA-06 | `spring.jpa.open-in-view` is `false`; don't rely on lazy loading in the web layer. | N | Config changes; `LazyInitializationException` workarounds. |
| BE-JPA-07 | Entities don't use Lombok `@Data` / generated `equals`/`hashCode` over all fields or lazy associations. | N | `@Data`, `@EqualsAndHashCode` on `@Entity`. |
| BE-JPA-08 | Use `save` / dirty checking intentionally; no `save()` inside loops for large batches without batching. | N | `repository.save` in a loop over many rows. |

## 4. PostgreSQL & migrations

| ID | Rule | Sev | How to check |
|---|---|---|---|
| BE-SQL-01 | Every schema change is a new migration file (Flyway/Liquibase). Never edit a migration that has already run. | B | Modified existing `V*__*.sql`. |
| BE-SQL-02 | Columns used in new `WHERE`, `JOIN` or `ORDER BY` have an index. Foreign key columns are indexed (PostgreSQL does not do this automatically). | N (B for large tables) | New FK / query columns vs. index statements. |
| BE-SQL-03 | Uniqueness the business needs is enforced by a DB constraint, not only by a service check. | B | "must be unique" requirement with no `UNIQUE` constraint. |
| BE-SQL-04 | Times use `timestamptz` (Java `Instant` / `OffsetDateTime`), not `timestamp` without time zone. | N | New `timestamp` columns; `LocalDateTime` for event times. |
| BE-SQL-05 | IDs use `generated ... as identity` (or UUID), not `serial`. | N | `serial` / `bigserial` in new tables. |
| BE-SQL-06 | Use `text` (or `varchar` without length) unless a real length limit exists; never `char(n)`, `money`. | N | Column types in migrations. |
| BE-SQL-07 | Avoid `NOT IN (subquery)` (wrong results with NULLs) — use `NOT EXISTS`. Avoid `BETWEEN` on timestamps — use `>= start AND < end`. | B if results can be wrong | Query text. |
| BE-SQL-08 | Table and column names are lower_snake_case. | N | Quoted/upper-case identifiers. |
| BE-SQL-09 | Risky changes on big tables (index creation, type changes, NOT NULL without default) are planned to avoid long locks (e.g. `CREATE INDEX CONCURRENTLY`). | B for large tables | Migrations touching large tables. |

## 5. Java language & clean code

| ID | Rule | Sev | How to check |
|---|---|---|---|
| BE-JAV-01 | DRY: logic that already exists in the codebase is reused, not copied. | N | Search the repo for similar method bodies / utilities. |
| BE-JAV-02 | Methods do one thing; a method over ~40 lines or with deep nesting is split. | N | Long new methods. |
| BE-JAV-03 | Names say what things are/do. No `data`, `temp`, `doProcess`, `handle2`. | N | New identifiers. |
| BE-JAV-04 | No magic numbers/strings: use named constants or enums. | N | Unexplained literals in logic. |
| BE-JAV-05 | Never swallow exceptions: no empty `catch`, no `catch` that only logs and continues as if nothing happened. Don't catch `Exception`/`Throwable` broadly. | B | `catch` blocks in diff. |
| BE-JAV-06 | Resources (streams, connections) use try-with-resources. | B | `new FileInputStream`, manual `close()`. |
| BE-JAV-07 | `Optional` is for return values only — not fields or parameters. Never call `.get()` without a check. | N | `Optional` fields/params; bare `.get()`. |
| BE-JAV-08 | Use `java.time` (not `Date`/`Calendar`); `BigDecimal` for money (not `double`). | B for money; else N | Types in diff. |
| BE-JAV-09 | DTOs and value objects are immutable — prefer `record`. | N | New mutable DTO classes with setters. |
| BE-JAV-10 | Logging uses SLF4J placeholders `log.info("x={}", x)`, not string concatenation; right level (no `info` spam in loops). | N | `log.*("..." + x)`. |
| BE-JAV-11 | Shared mutable state is thread-safe (singleton beans are shared by all requests). No mutable instance fields in `@Service`/`@Component` holding request data. | B | Non-final instance fields in beans written per request. |
| BE-JAV-12 | No dead code, commented-out code, or leftover `System.out` / debug logs. | N | Diff. |

## 6. Tests

| ID | Rule | Sev | How to check |
|---|---|---|---|
| BE-TST-01 | New logic has tests that would **fail** if the logic broke (real assertions, not just "no exception"). | B for business rules | Tests without meaningful assertions; mocks returning the expected value. |
| BE-TST-02 | Tests were not weakened to pass: no changed expected values, deleted assertions, or added `@Disabled` without a stated reason. | B | Compare test files to base branch. |
| BE-TST-03 | Edge cases from the requirements have a test (empty, duplicate, not found, no permission). | N | Requirement list vs. test names. |
| BE-TST-04 | Repository/SQL behaviour is tested against real PostgreSQL (Testcontainers), not H2, when the query uses PostgreSQL features. | N | Native queries tested only on H2. |
| BE-TST-05 | Security rules have a negative test (wrong user/role gets 403/404). | N (B for new endpoints holding personal data) | Tests for new endpoints. |

---

## Sources
- Spring Boot reference — Spring Beans and Dependency Injection (constructor injection): https://docs.spring.io/spring-boot/3.4/reference/using/spring-beans-and-dependency-injection.html
- Spring Framework reference — Using @Transactional (proxy self-invocation, readOnly): https://docs.spring.io/spring/reference/6.2/data-access/transaction/declarative/annotations.html
- Spring Boot reference — Externalized configuration / @ConfigurationProperties
- Hibernate ORM User Guide — fetching strategies; Vlad Mihalcea, N+1 query problem: https://vladmihalcea.com/n-plus-1/
- OWASP SQL Injection Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- OWASP Java Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Java_Security_Cheat_Sheet.html
- OWASP Insecure Direct Object Reference Prevention Cheat Sheet
- OWASP Logging Cheat Sheet
- PostgreSQL wiki — Don't Do This: https://wiki.postgresql.org/wiki/Don%27t_Do_This
- Joshua Bloch, *Effective Java* (3rd ed.) — Optional, try-with-resources, immutability, exceptions
- Google Java Style Guide (formatting — enforced by tools, not review): https://google.github.io/styleguide/javaguide.html
