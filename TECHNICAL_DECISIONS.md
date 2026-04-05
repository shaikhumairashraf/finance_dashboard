# Technical Decisions & Tradeoffs

---

## 1. H2 In-Memory Database
**Decision**: Use H2 instead of PostgreSQL/MySQL.

| Pro | Con |
|---|---|
| Zero setup, runs instantly | Data lost on every restart |
| Perfect for demos/assessment | Not suitable for real deployment |
| Built-in web console at `/h2-console` | No concurrent multi-node support |

**Tradeoff**: Speed of setup over persistence. Swapping to PostgreSQL requires only changing 2 lines in `application.properties` and one `pom.xml` dependency — the JPA layer is fully database-agnostic.

---

## 2. JWT Stateless Authentication (No Refresh Tokens)
**Decision**: Single access token, 24h expiry, no refresh token.

| Pro | Con |
|---|---|
| Stateless — no session store needed | Can't revoke a token mid-session |
| Scales horizontally without sticky sessions | Deactivating a user doesn't instantly block active tokens |
| Simple to implement and test | Longer expiry = longer exposure window if token is stolen |

**Tradeoff**: Simplicity over security granularity. A production system would add a refresh token + blacklist table to handle revocation.

---

## 3. First-User-Becomes-Admin Pattern
**Decision**: `userRepository.count() == 0` → assign ADMIN, else VIEWER.

| Pro | Con |
|---|---|
| No separate admin bootstrap script needed | Fragile in concurrent startup scenarios (race condition) |
| Works seamlessly in demo/test environments | Not suitable for multi-tenant or cloud deployments |

**Tradeoff**: Simplicity over robustness. A production system would use environment-variable-seeded admin credentials or a dedicated setup endpoint locked behind a one-time bootstrap token.

---

## 4. Soft Delete on Financial Records
**Decision**: `deleted = true` flag instead of `DELETE FROM financial_records`.

| Pro | Con |
|---|---|
| Preserves audit trail | All queries must include `WHERE deleted = false` |
| Allows recovery if mistakenly deleted | Table grows indefinitely without a purge strategy |
| Consistent with finance domain (records shouldn't disappear) | `findWithFilters` query is more complex |

**Tradeoff**: Data integrity and auditability over query simplicity. Worth it in a finance context — you never want to lose a transaction record.

---

## 5. JPQL `findWithFilters` Instead of Specifications
**Decision**: Single `@Query` method with nullable params instead of `JpaSpecificationExecutor`.

| Pro | Con |
|---|---|
| One method, readable JPQL | Long query string harder to read at a glance |
| No extra Specification boilerplate classes | Adding new filter params requires editing the JPQL string |
| Performs well with H2 and indexed columns | Enum params need careful null handling in some JPA providers |

**Tradeoff**: Less code over flexibility. `Specification<T>` would be better for a large, evolving filter surface area.

---

## 6. URL-Level RBAC in SecurityConfig vs Method-Level `@PreAuthorize`
**Decision**: Enforce access control via `HttpSecurity` URL rules, not `@PreAuthorize` annotations.

| Pro | Con |
|---|---|
| All security rules in one place (`SecurityConfig`) — easy to audit | Less granular — can't do row-level security per user |
| No annotations scattered across controller methods | Controller methods look the same regardless of role |
| Fails fast before the controller even executes | Business rule exceptions (e.g., last admin protection) still need service-layer checks |

**Tradeoff**: Centralized security policy over fine-grained flexibility. `@EnableMethodSecurity` is enabled so `@PreAuthorize` could be added later if per-record ownership checks are needed.

---

## 7. DTO Layer — No Entity Exposure
**Decision**: All APIs use dedicated DTOs (request/response), never expose JPA entities directly.

| Pro | Con |
|---|---|
| Passwords never leak in responses | More boilerplate classes to write |
| API contract decoupled from DB schema | Manual mapping (no MapStruct) means more `toResponse()` methods |
| Can evolve DB schema without breaking API clients | |

**Tradeoff**: Safety and maintainability over brevity. The right call — exposing entities directly is a common security and coupling mistake.

---

## 8. No MapStruct / No ModelMapper
**Decision**: Manual mapping in service `toResponse()` methods.

| Pro | Con |
|---|---|
| No additional dependency | Verbose — every field mapped explicitly |
| Mapping logic is explicit and debuggable | Easy to forget a new field when entity is updated |

**Tradeoff**: Transparency over convenience. For an assessment this is fine; MapStruct would be the right call at production scale.

---

## 9. DataInitializer Excluded During Tests via `@Profile`
**Decision**: `@Profile("!test")` on `DataInitializer`, `@ActiveProfiles("test")` on test classes.

| Pro | Con |
|---|---|
| Tests start with a clean, predictable DB state | Slightly more annotation boilerplate on tests |
| First-registered user in each test becomes ADMIN correctly | Developer must remember to set profile if adding new tests |
| Avoids test flakiness from seeded data interference | |

**Tradeoff**: This was a correctness fix — without it, the `DataInitializer` runs first, making no test user the first to register, which breaks the admin-role assumption in all tests.

---

## Summary: What Would Change for Production

| Area | Assessment Choice | Production Choice |
|---|---|---|
| Database | H2 in-memory | PostgreSQL / MySQL |
| Auth | Single JWT, 24h | JWT + refresh token + revocation list |
| Secrets | Hardcoded JWT secret in properties | Vault / environment injection |
| Mapping | Manual DTOs | MapStruct |
| Soft delete purge | None | Scheduled archival job |
| Rate limiting | None | Bucket4j / API Gateway |
| Caching | None | Cache dashboard aggregates (Redis) |
| Logging | SLF4J INFO | Structured JSON logs + audit trail |
