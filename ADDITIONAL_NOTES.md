# Additional Notes

## Assumptions Made

1. **Roles are fixed** at VIEWER, ANALYST, ADMIN — no composite roles.
2. **Records belong to the system** — any ADMIN can edit/delete any record. `createdBy` is for audit only, not ownership.
3. **Analyst is read-only on records** — "view records and access insights" was interpreted as read + dashboard, no write.
4. **Registration is open** — first user → ADMIN, rest → VIEWER automatically.
5. **Amount is always positive** — `type` (INCOME/EXPENSE) carries the sign semantics, keeping aggregation queries clean.
6. **Category is free-text** — not normalized to a lookup table. Simpler for this scope; a production system would use a categories table.
7. **Pagination is zero-based** — Spring Data default. Page size defaults to 20; recent activity capped at 50.

---

## Extra Features (Beyond Requirements)

- **Soft delete** — `deleted=true` flag preserves audit trail instead of physical removal
- **Pagination + sorting + search** — `?type=INCOME&category=Salary&startDate=2026-01-01&page=0&size=10&sort=date,desc`
- **Last-admin protection** — system rejects deactivating or deleting the only remaining ADMIN
- **Swagger UI** — interactive docs at `/swagger-ui.html` with Bearer auth built in
- **H2 Console** — live DB inspector at `/h2-console`
- **DataInitializer** — seeds 3 users + 15 records across 3 months on startup
- **20 integration tests** — auth, RBAC, CRUD, soft delete, validation, dashboard

---

## How to Extend

**Switch to PostgreSQL** — change `pom.xml` (replace H2 with `postgresql` driver) and update `application.properties`:
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/financedb
spring.datasource.username=your_user
spring.datasource.password=your_password
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
```

**Add a new role** — add to `Role.java` enum, add URL rules in `SecurityConfig`, done.

**Per-record ownership** — `@EnableMethodSecurity` is already on. Add `@PreAuthorize` to service methods.

**Refresh tokens** — create a `RefreshToken` entity, add `POST /api/auth/refresh` and `POST /api/auth/logout` endpoints.

---

## Running Commands

```powershell
# Set environment (run once per terminal session)
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.10.7-hotspot"
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"
$env:M2_HOME = "C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2025.2\plugins\maven\lib\maven3"

# Start app
& "$env:M2_HOME\bin\mvn.cmd" spring-boot:run

# Run tests
& "$env:M2_HOME\bin\mvn.cmd" test

# Build JAR
& "$env:M2_HOME\bin\mvn.cmd" clean package -DskipTests
# Output: target/finance-dashboard-1.0.0.jar

# Run JAR directly
& "$env:JAVA_HOME\bin\java" -jar target/finance-dashboard-1.0.0.jar
```

---

## Default Users (Seeded on Startup)

| Username | Password    | Role    | Access |
|----------|-------------|---------|--------|
| admin    | admin123    | ADMIN   | Full access |
| analyst  | analyst123  | ANALYST | Read records + dashboard |
| viewer   | viewer123   | VIEWER  | Read records only |

---

## Key URLs

| URL | Description |
|-----|-------------|
| http://localhost:8080/swagger-ui.html | Interactive API docs |
| http://localhost:8080/h2-console | DB browser (`jdbc:h2:mem:financedb`, user: `sa`) |
| http://localhost:8080/api-docs | OpenAPI JSON spec |

---

## Test Structure

```
src/test/
├── FinanceDashboardApplicationTests   — context loads (1 test)
└── controller/
    ├── AuthControllerTest             — 7 tests: register, login, validation, token auth
    ├── FinancialRecordControllerTest  — 7 tests: CRUD, RBAC, soft delete, filters, validation
    └── DashboardControllerTest        — 5 tests: summary, categories, trends, activity, viewer blocked
```

All test classes use `@ActiveProfiles("test")` to skip `DataInitializer` and `@DirtiesContext` for a clean DB per class.
