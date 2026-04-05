# Technical Decisions & Tradeoffs

## 1. H2 In-Memory Database
Chosen for zero-config setup. Data resets on restart — acceptable for an assessment. Switching to PostgreSQL needs only 2 property changes; the JPA layer is fully database-agnostic.

## 2. JWT — Single Token, No Refresh
Stateless, 24h expiry. No refresh token keeping things simple. Tradeoff: a deactivated user's existing token stays valid until it expires. Production fix: add a refresh token table + revocation list.

## 3. First-User-Becomes-Admin
`userRepository.count() == 0` → ADMIN, else VIEWER on registration. Simple for demos. Fragile under concurrent startup — a production system would seed admin credentials via environment variables.

## 4. Soft Delete on Financial Records
`deleted = true` flag instead of physical delete. Preserves audit trail (critical in finance). Tradeoff: every query must filter `deleted = false` and the table grows without a purge strategy.

## 5. JPQL `findWithFilters` Instead of Specifications
Single `@Query` with nullable params covers all filter combinations without extra boilerplate. Tradeoff: adding new filter fields requires editing the JPQL string. `JpaSpecificationExecutor` would scale better but adds more classes.

## 6. URL-Level RBAC over `@PreAuthorize`
All role rules are centralized in `SecurityConfig` — easy to audit in one place. Tradeoff: can't do row-level ownership checks at the URL layer. `@EnableMethodSecurity` is already enabled so `@PreAuthorize` can be layered on later.

## 7. Manual DTOs — No MapStruct
All entities are mapped to DTOs manually in service `toResponse()` methods. Prevents password/internals leaking in responses and decouples API contract from the DB schema. Verbose but explicit and debuggable. MapStruct would be the right call at scale.

## 8. `@Profile("!test")` on DataInitializer
Prevents seeded data from interfering with tests. Without this, the DataInitializer runs first, meaning no test user is ever the first registrant, which breaks the admin-role logic across all 20 tests.

---

## Production vs Assessment Choices

| Area | This Project | Production |
|---|---|---|
| Database | H2 in-memory | PostgreSQL / MySQL |
| Auth | Single JWT, 24h | JWT + refresh token + revocation |
| Secrets | Properties file | Vault / env injection |
| Mapping | Manual DTOs | MapStruct |
| Soft delete purge | None | Scheduled archival job |
| Rate limiting | None | Bucket4j / API Gateway |
| Caching | None | Redis for dashboard aggregates |
| Logging | SLF4J INFO | Structured JSON + audit trail |
