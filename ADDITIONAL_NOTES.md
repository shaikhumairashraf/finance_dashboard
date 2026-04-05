# Additional Notes

---

## Assumptions Made

1. **Role model is fixed at three levels** — VIEWER, ANALYST, ADMIN. No custom or composite roles were modeled since the requirement listed these three explicitly.

2. **Financial records belong to the system, not a user** — any ADMIN can update or delete any record regardless of who created it. The `createdBy` field is stored for audit purposes only, not for ownership enforcement.

3. **Analyst cannot create/update/delete records** — the requirement stated "Analyst: Can view records and access insights." This was interpreted strictly; Analysts are read-only on records but have access to the full dashboard/analytics layer.

4. **Registration is open** — any user can self-register. Role assignment is automatic (first=ADMIN, rest=VIEWER). In a real system, registration would likely be invite-only or admin-initiated.

5. **Amount is always positive** — the `type` field (INCOME / EXPENSE) carries the sign semantics. An amount of `1200.00` with type `EXPENSE` represents a negative cash flow. Storing negative amounts was avoided to keep aggregation queries clean.

6. **No multi-currency support** — all amounts are treated as the same currency. Adding a `currency` field and exchange rate conversion would be a straightforward extension.

7. **Category is a free-text string** — not a foreign key to a categories table. This keeps setup simple but means typos create new logical categories (e.g., "Salery" vs "Salary"). A production system would normalize this.

8. **Pagination is zero-based** — Spring Data's default. Page 0 = first page. Default page size is 20, max enforced at 50 for recent activity.

---

## What Was Implemented That Was Not Explicitly Required

- **Soft delete** — records are flagged `deleted=true` rather than physically removed, preserving audit trails.
- **Pagination and sorting** on record listing with configurable `page`, `size`, `sort` parameters.
- **Search** on description field via `?search=keyword` query parameter.
- **Last-admin protection** — the system refuses to deactivate or delete the last active ADMIN user.
- **Swagger UI** — full interactive API documentation at `/swagger-ui.html` with Bearer auth support.
- **H2 Console** — database inspector at `/h2-console` for direct SQL queries during development.
- **DataInitializer** — seeds 3 users (admin/analyst/viewer) and 15 financial records across 3 months for immediate demo use.
- **Integration tests** — 20 tests covering auth, RBAC enforcement, CRUD correctness, soft delete, validation, and dashboard endpoints.
- **OpenAPI annotations** — `@Tag`, `@Operation`, `@Parameter`, `@SecurityRequirement` on all controllers for clean Swagger documentation.

---

## How to Extend This Project

### Switch to PostgreSQL
1. Replace in `pom.xml`:
```xml
<!-- Remove H2, add: -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```
2. Update `application.properties`:
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/financedb
spring.datasource.username=your_user
spring.datasource.password=your_password
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=update
```

### Add a New Role (e.g., MANAGER)
1. Add `MANAGER` to `Role.java` enum.
2. Add URL rules in `SecurityConfig.java` for what MANAGER can access.
3. Update `DataInitializer` to seed a manager user if needed.

### Add Per-Record Ownership (User can only edit their own records)
Enable method security already present (`@EnableMethodSecurity`) and add to service:
```java
@PreAuthorize("#record.createdBy.username == authentication.name or hasRole('ADMIN')")
```

### Add Refresh Tokens
1. Create a `RefreshToken` entity with a UUID token, userId, and expiry.
2. Add `POST /api/auth/refresh` endpoint that validates the refresh token and issues a new access token.
3. Add `POST /api/auth/logout` that deletes the refresh token.

### Add Rate Limiting
Add Bucket4j dependency and a filter:
```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.7.0</version>
</dependency>
```

---

## Running Commands Reference

### Start the application
```powershell
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.10.7-hotspot"
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"
$env:M2_HOME = "C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2025.2\plugins\maven\lib\maven3"
cd "C:\Users\sashra10\Documents\projects\finance-dashboard"
& "$env:M2_HOME\bin\mvn.cmd" spring-boot:run
```

### Run tests
```powershell
& "$env:M2_HOME\bin\mvn.cmd" test
```

### Build JAR
```powershell
& "$env:M2_HOME\bin\mvn.cmd" clean package -DskipTests
# JAR will be at: target/finance-dashboard-1.0.0.jar
```

### Run JAR
```powershell
& "$env:JAVA_HOME\bin\java" -jar target/finance-dashboard-1.0.0.jar
```

---

## Default Credentials (Seeded on Startup)

| Username | Password   | Role    | Can Do |
|----------|-----------|---------|--------|
| admin    | admin123  | ADMIN   | Everything |
| analyst  | analyst123| ANALYST | Read records + full dashboard |
| viewer   | viewer123 | VIEWER  | Read records only |

---

## Key URLs

| URL | Description |
|-----|-------------|
| http://localhost:8080/swagger-ui.html | Interactive API documentation |
| http://localhost:8080/h2-console | Database browser (JDBC URL: `jdbc:h2:mem:financedb`, User: `sa`) |
| http://localhost:8080/api-docs | Raw OpenAPI JSON spec |
| http://localhost:8080/api/auth/login | Login endpoint |
| http://localhost:8080/api/dashboard/summary | Dashboard summary |

---

## Test Structure

```
src/test/java/com/finance/dashboard/
├── FinanceDashboardApplicationTests.java    — context loads smoke test
└── controller/
    ├── AuthControllerTest.java              — 7 tests: register, login, validation, auth enforcement
    ├── FinancialRecordControllerTest.java   — 7 tests: CRUD, RBAC, soft delete, filters, validation
    └── DashboardControllerTest.java         — 5 tests: summary, categories, trends, activity, viewer blocked
```

All tests use:
- `@SpringBootTest` — full application context
- `@AutoConfigureMockMvc` — real HTTP processing without a live server
- `@DirtiesContext` — fresh database per test class
- `@ActiveProfiles("test")` — skips DataInitializer so test data is fully controlled
