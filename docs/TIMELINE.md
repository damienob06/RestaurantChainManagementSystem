# Development Timeline

## Restaurant Chain Management System

**Version:** 1.0
**Date:** 2026-04-09
**Team Size:** 7 developers

---

## Team Roster

| ID | Role | Primary Responsibility |
|----|------|----------------------|
| P1 | Project Setup & DevOps | Project skeleton, build config, CI/CD pipeline, deployment |
| P2 | Database & Entities | JPA entities, repositories, database schema, seed data |
| P3 | Authentication & Security | Spring Security, JWT, login/register, auth filter |
| P4 | Restaurant Management | Restaurant CRUD endpoints, admin features |
| P5 | Menu & Item Management | Menu and menu item CRUD endpoints |
| P6 | Discount Management | Discount CRUD endpoints, date validation |
| P7 | Testing & Quality | Unit tests, integration tests, test infrastructure |

---

## Phase Overview

```
Phase 1: Foundation     ████████░░░░░░░░░░░░░░░░░░░░░░  (Week 1)
Phase 2: Core Features  ░░░░░░░░████████████████░░░░░░░░  (Weeks 2-3)
Phase 3: Integration    ░░░░░░░░░░░░░░░░░░░░░░░░████████  (Week 4)
```

---

## Phase 1 — Foundation (Week 1)

**Goal:** Working project skeleton with database, security, and CI/CD. By the end of this phase, every team member can clone the repo, run the app, and hit a test endpoint.

### P1 — Project Setup & DevOps

| Day | Task | Deliverable |
|-----|------|-------------|
| 1 | Initialise Spring Boot project with Gradle | `build.gradle`, `settings.gradle`, wrapper scripts, `RcmsApplication.java` |
| 1 | Create the folder structure (config, controller, service, repository, model, dto, exception) | Empty packages with placeholder classes |
| 1 | Create `.env.example` with all required variables | `.env.example` committed |
| 1 | Add `.env` to `.gitignore`, clean up `.gitignore` for Gradle + Java + IDE files | Updated `.gitignore` |
| 2 | Configure `application.properties` to read from environment variables | Database URL, credentials, JWT secret all from env |
| 2 | Add `dotenv-java` dependency for local `.env` loading | Working local config |
| 2 | Set up GitHub Actions CI workflow (`.github/workflows/ci.yml`) | Pipeline runs on push: checkout → Java 17 → `gradle build` |
| 3 | Add all Spring Boot dependencies to `build.gradle` (web, data-jpa, security, validation, jwt, postgres) | Complete dependency list |
| 3 | Create a health-check endpoint (`GET /api/health` → 200 OK) for testing | Verifiable deployment |
| 3 | Write `README.md` with setup instructions for the team | Team can clone and run |

**Blocked by:** Nothing — P1 starts immediately.
**Blocks:** Everyone (P2–P7 need the skeleton to start).

### P2 — Database & Entities

| Day | Task | Deliverable |
|-----|------|-------------|
| 2 | Set up Railway PostgreSQL instance, get connection string | Working remote DB |
| 2 | Design final SQL schema (tables, columns, types, constraints, indexes) | `schema.sql` or equivalent documentation |
| 3 | Create all 5 JPA entity classes: `User`, `Restaurant`, `Menu`, `MenuItem`, `Discount` | Entity files with annotations |
| 3 | Create `Role` enum | `Role.java` |
| 3 | Create all 5 repository interfaces (extend `JpaRepository`) | Repository interfaces |
| 4 | Verify Hibernate auto-generates tables correctly by running the app | Tables created in Railway PostgreSQL |
| 4 | Create `data.sql` with seed data (1 admin user, 2 managers, 2 restaurants, sample menus/items/discounts) | Seed data for development |
| 5 | Test repository methods with a simple integration test | Verified DB layer |

**Blocked by:** P1 (project skeleton + DB config).
**Blocks:** P3, P4, P5, P6 (all need entities and repositories).

### P3 — Authentication & Security (starts mid-week)

| Day | Task | Deliverable |
|-----|------|-------------|
| 3 | Add JWT dependencies, create `JwtUtil` class (generate, parse, validate tokens) | JWT utility |
| 4 | Create `JwtAuthFilter` (extract token from header, validate, set security context) | Auth filter |
| 4 | Create `SecurityConfig` (filter chain, permit login, protect other routes) | Security config |
| 5 | Create `AuthController` with `POST /api/auth/login` | Working login endpoint |
| 5 | Create `AuthService` with login logic (find user, verify password, return token) | Auth service |
| 5 | Create `LoginRequest`, `LoginResponse`, `RegisterRequest` DTOs | Auth DTOs |

**Blocked by:** P2 (User entity + UserRepository).
**Blocks:** P4, P5, P6 (need auth working to test protected endpoints).

### P7 — Testing Infrastructure (starts mid-week)

| Day | Task | Deliverable |
|-----|------|-------------|
| 3 | Set up test configuration (`application-test.properties` with H2 or test DB) | Test environment config |
| 4 | Create test utility class (helper methods to generate test JWTs, create test users) | `TestUtils.java` |
| 5 | Write initial auth tests: login success, login failure, missing token | First test suite |

**Blocked by:** P1 (skeleton), P3 (auth endpoints).

---

## Phase 2 — Core Features (Weeks 2–3)

**Goal:** All CRUD endpoints are implemented and individually working. P4, P5, P6 work in parallel on their respective domains.

### P3 — Authentication (completes)

| Day | Task | Deliverable |
|-----|------|-------------|
| W2 D1 | Implement `POST /api/auth/register` (admin-only) | Register endpoint |
| W2 D1 | Add `BCryptPasswordEncoder` bean and wire into register flow | Secure password storage |
| W2 D2 | Implement the ownership verification helper method in a shared base/utility | `verifyAccess()` pattern available for other services |
| W2 D2 | Test register endpoint, verify admin-only enforcement | Auth fully complete |

### P4 — Restaurant Management

| Day | Task | Deliverable |
|-----|------|-------------|
| W2 D1 | Create `RestaurantRequest` and `RestaurantResponse` DTOs | DTOs |
| W2 D1 | Create `RestaurantService` with `getAllRestaurants()` and `getRestaurantById()` | Read operations |
| W2 D2 | Add `createRestaurant()` with manager validation | Create operation |
| W2 D3 | Add `updateRestaurant()` and `deleteRestaurant()` | Update + delete |
| W2 D3 | Create `RestaurantController` wiring all endpoints | REST endpoints |
| W2 D4 | Add ownership check — managers can only view their own restaurant | Access control |
| W2 D5 | Test all endpoints manually with Postman | Verified restaurant CRUD |

### P5 — Menu & Item Management

| Day | Task | Deliverable |
|-----|------|-------------|
| W2 D1 | Create `MenuRequest`, `MenuResponse`, `MenuItemRequest`, `MenuItemResponse` DTOs | DTOs |
| W2 D2 | Create `MenuService` with CRUD methods | Menu business logic |
| W2 D3 | Create `MenuController` with all endpoints | Menu REST endpoints |
| W2 D3 | Add ownership verification (manager can only manage own restaurant's menus) | Access control |
| W2 D4 | Create `MenuItemService` with CRUD methods | Item business logic |
| W2 D5 | Create `MenuItemController` with all endpoints | Item REST endpoints |
| W3 D1 | Handle menu activation/deactivation (`active` flag toggle) | Menu on/off feature |
| W3 D2 | Test all menu and item endpoints with Postman | Verified menu + item CRUD |

### P6 — Discount Management

| Day | Task | Deliverable |
|-----|------|-------------|
| W2 D1 | Create `DiscountRequest` and `DiscountResponse` DTOs | DTOs |
| W2 D2 | Create `DiscountService` with CRUD methods | Discount business logic |
| W2 D3 | Add date validation (endDate >= startDate) in service layer | Business rule enforcement |
| W2 D3 | Create `DiscountController` with all endpoints | REST endpoints |
| W2 D4 | Add ownership verification | Access control |
| W2 D4 | Add computed `expired` field in response (endDate < today) | Expiry indicator |
| W2 D5 | Test all discount endpoints with Postman | Verified discount CRUD |

### P7 — Testing (ongoing)

| Day | Task | Deliverable |
|-----|------|-------------|
| W2 D1–D2 | Write `AuthServiceTest` unit tests (login, register, invalid credentials) | Auth test suite |
| W2 D3 | Write `RestaurantServiceTest` unit tests | Restaurant test suite |
| W2 D4 | Write `MenuServiceTest` and `MenuItemServiceTest` unit tests | Menu test suites |
| W2 D5 | Write `DiscountServiceTest` unit tests | Discount test suite |
| W3 D1 | Write `AuthControllerTest` integration tests | Auth integration tests |
| W3 D2 | Write `RestaurantControllerTest` integration tests | Restaurant integration tests |
| W3 D3 | Write `MenuControllerTest` and `MenuItemControllerTest` integration tests | Menu integration tests |
| W3 D4 | Write `DiscountControllerTest` integration tests | Discount integration tests |
| W3 D5 | Run full test suite, identify and report failures to responsible person | Test report |

### P1 — DevOps (ongoing)

| Day | Task | Deliverable |
|-----|------|-------------|
| W2 D1 | Add GitHub Secrets for CI (DB_URL, DB_USERNAME, DB_PASSWORD, JWT_SECRET) | CI can connect to DB |
| W2 D3 | Verify CI pipeline runs tests on PR | Green CI on PRs |
| W3 D1 | Add JAR artifact upload step to CI | Downloadable `.jar` from CI |

---

## Phase 3 — Integration & Polish (Week 4)

**Goal:** Everything works together end-to-end. Bugs are fixed, edge cases are handled, documentation is complete.

### All Team Members

| Day | Task | Owner |
|-----|------|-------|
| W4 D1 | End-to-end testing: full workflow from login → create restaurant → add menu → add items → add discounts | P4, P5, P6 |
| W4 D1 | Fix any cross-feature bugs discovered during integration | Respective owner |
| W4 D2 | Create `GlobalExceptionHandler` if not already done, ensure all errors return consistent JSON | P3 |
| W4 D2 | Review and clean up all controller response codes (201 for creation, 204 for deletion, etc.) | P4, P5, P6 |
| W4 D3 | Final test run — all unit + integration tests must pass | P7 |
| W4 D3 | Fix any remaining test failures | Respective owner |
| W4 D4 | Build final `.jar` file: `./gradlew bootJar` | P1 |
| W4 D4 | Verify `.jar` runs standalone: `java -jar rcms-1.0.0.jar` | P1 |
| W4 D4 | Final CI pipeline run — green build on `main` | P1 |
| W4 D5 | Code review sweep — each person reviews one other person's code | Everyone |
| W4 D5 | Final documentation review | Everyone |

---

## Milestone Summary

| Milestone | Target | Success Criteria |
|-----------|--------|------------------|
| M1 — Skeleton Ready | End of Week 1, Day 3 | App starts, connects to DB, health endpoint returns 200 |
| M2 — Entities & Auth | End of Week 1 | All entities in DB, login returns JWT, protected routes reject unauthenticated requests |
| M3 — CRUD Complete | End of Week 2 | All 20 API endpoints return correct responses for happy-path scenarios |
| M4 — Tests Pass | End of Week 3 | Unit tests pass for all 5 services, integration tests pass for all 5 controllers |
| M5 — Ship | End of Week 4 | `.jar` runs standalone, CI is green, all tests pass, documentation complete |

---

## Dependency Graph

```
Week 1                          Week 2                    Week 3              Week 4
──────                          ──────                    ──────              ──────

P1: Setup ─────┐
               ├──► P2: Entities ──┐
               │                   ├──► P4: Restaurants ──────────────┐
               │                   ├──► P5: Menus & Items ───────────┤
               │                   ├──► P6: Discounts ───────────────┤
               │                   │                                  │
               │    P3: Auth ──────┤                                  ├──► Integration
               │         ▲        │                                  │    & Polish
               │         │        │                                  │
               │         └────────┘                                  │
               │                                                     │
               └──────────────────► P7: Testing ─────────────────────┘
                                        (ongoing from mid-Week 1)

P1: DevOps ──────────────────────────────────────────────────────────┘
                  (ongoing CI maintenance)
```

---

## Risk Mitigation During Development

| Risk | Trigger | Response |
|------|---------|----------|
| P1 or P2 falls behind | M1 not met by end of Day 3 | P3 or P7 assists with setup; others write code locally against H2 in-memory DB |
| Merge conflicts | Multiple PRs touch shared files (entities, config) | Daily pulls from main; small PRs merged frequently; communication in team chat |
| Authentication blocks feature development | P3 not done by start of Week 2 | P4/P5/P6 temporarily disable security in their local branch to develop endpoints; P3 adds security later |
| Tests reveal design flaws | Tests fail because API contract doesn't match service layer | P7 raises issue immediately; fix takes priority over new features |
| Railway DB issues | Connection failures or quota limits | Fall back to local PostgreSQL via Docker (`docker run -p 5432:5432 postgres:15`) |

---

## Git Branching Strategy

```
main (protected — requires PR + passing CI)
 │
 ├── feature/setup-skeleton          (P1)
 ├── feature/entities-and-repos      (P2)
 ├── feature/auth-security           (P3)
 ├── feature/restaurant-crud         (P4)
 ├── feature/menu-item-crud          (P5)
 ├── feature/discount-crud           (P6)
 ├── feature/unit-tests              (P7)
 ├── feature/integration-tests       (P7)
 └── fix/[description]               (anyone — for bug fixes)
```

**Rules:**
1. Never commit directly to `main`
2. Create a feature branch from `main` for each task
3. Open a PR when ready — CI must pass before merge
4. At least one other team member reviews before merge
5. Merge via squash-and-merge to keep `main` history clean
6. Delete the branch after merge
