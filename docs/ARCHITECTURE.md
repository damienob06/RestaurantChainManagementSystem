# Architecture Specification

## Restaurant Chain Management System

**Version:** 1.0
**Date:** 2026-04-09

---

## 1. System Overview

The Restaurant Chain Management System (RCMS) is a server-side Java application built on Spring Boot. It exposes a RESTful JSON API consumed by HTTP clients (Postman, curl, or a future frontend). The system follows a **layered MVC architecture** with clear separation between the HTTP layer (controllers), business logic (services), and data access (repositories).

```
┌─────────────────────────────────────────────────────────┐
│                      HTTP Clients                       │
│              (Postman / curl / future UI)                │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS / JSON
                         ▼
┌─────────────────────────────────────────────────────────┐
│                   Spring Boot Application                │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Security Filter Chain                 │  │
│  │         (JWT Authentication + Role Check)          │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │                  Controllers                       │  │
│  │    (REST endpoints — request/response handling)    │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │                   Services                         │  │
│  │        (Business logic, validation, rules)         │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │                 Repositories                       │  │
│  │          (Spring Data JPA interfaces)              │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │              JPA / Hibernate ORM                   │  │
│  └───────────────────────┬───────────────────────────┘  │
└──────────────────────────┼──────────────────────────────┘
                           │ JDBC
                           ▼
              ┌─────────────────────────┐
              │   PostgreSQL (Railway)   │
              └─────────────────────────┘
```

---

## 2. Architecture Style

### 2.1 Layered MVC

The application uses a **four-layer architecture**:

| Layer | Responsibility | Spring Stereotype |
|-------|---------------|-------------------|
| **Controller** | Accepts HTTP requests, validates input shape, delegates to services, returns HTTP responses | `@RestController` |
| **Service** | Contains all business logic, enforces rules (e.g., ownership checks), orchestrates repository calls | `@Service` |
| **Repository** | Data access — translates between JPA entities and database rows | `@Repository` (via Spring Data JPA) |
| **Model/Entity** | Defines the data structures that map to database tables | `@Entity` |

### 2.2 Why Layered MVC

- **Simplicity:** Straightforward for a team of 7 with varying experience levels
- **Testability:** Each layer can be tested in isolation (mock the layer below)
- **Separation of concerns:** Controllers never touch the database; repositories never contain business logic
- **Spring Boot convention:** This is the standard, well-documented pattern for Spring — maximum community support and tutorials

### 2.3 Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Monolith vs. Microservices | **Monolith** | Team of 7 on a university project — microservices add deployment complexity with no real scaling benefit here |
| REST vs. GraphQL | **REST** | Simpler, well-understood by the team, sufficient for CRUD operations |
| Server-rendered vs. API-only | **API-only** | Decouples backend from any future frontend; API can be tested with Postman |
| SQL vs. NoSQL | **SQL (PostgreSQL)** | Data is relational (restaurants → menus → items); foreign keys enforce integrity; joins make querying straightforward |
| ORM vs. raw SQL | **ORM (Hibernate/JPA)** | Reduces boilerplate; Spring Data JPA generates queries from method names; team doesn't need to write SQL for basic CRUD |
| Session vs. Token auth | **JWT tokens** | Stateless — no server-side session storage; scales horizontally if needed; standard for REST APIs |

---

## 3. Technology Stack

### 3.1 Core

| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Java | 17 (LTS) |
| Framework | Spring Boot | 3.x |
| Build tool | Gradle | 8.x |
| ORM | Spring Data JPA + Hibernate | (bundled with Spring Boot) |
| Database | PostgreSQL | 15+ (Railway-managed) |
| Authentication | Spring Security + JWT (`io.jsonwebtoken:jjwt`) | 0.12.x |
| Password hashing | bcrypt (via Spring Security) | — |

### 3.2 Testing

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Unit testing | JUnit 5 | Service-layer logic |
| Mocking | Mockito | Mock repositories in service tests |
| Integration testing | Spring Boot Test + MockMvc | Controller endpoint tests against embedded context |
| Assertions | AssertJ (optional) | Fluent, readable assertions |

### 3.3 DevOps & Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Version control | Git + GitHub | Source code management |
| CI/CD | GitHub Actions | Automated build, test, and artifact packaging |
| Database hosting | Railway | Managed PostgreSQL instance |
| Secret management | `.env` file (local) + GitHub Secrets (CI) | Keep credentials out of version control |

### 3.4 Dependencies (build.gradle)

```groovy
dependencies {
    // Spring Boot starters
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'org.springframework.boot:spring-boot-starter-validation'

    // JWT
    implementation 'io.jsonwebtoken:jjwt-api:0.12.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.5'
    runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.5'

    // Database
    runtimeOnly 'org.postgresql:postgresql'

    // Dotenv for local development
    implementation 'io.github.cdimascio:dotenv-java:3.0.0'

    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

---

## 4. Database Design

### 4.1 Entity-Relationship Diagram

```
┌──────────────┐       1:1        ┌──────────────────┐
│    users     │◄─────────────────│   restaurants     │
├──────────────┤                  ├──────────────────┤
│ id (PK)      │                  │ id (PK)          │
│ username     │                  │ name             │
│ password     │                  │ address          │
│ role         │                  │ manager_id (FK)  │
│ created_at   │                  │ created_at       │
└──────────────┘                  └────────┬─────────┘
                                           │
                              ┌────────────┼────────────┐
                              │ 1:N                     │ 1:N
                              ▼                         ▼
                    ┌──────────────────┐     ┌──────────────────┐
                    │     menus        │     │    discounts     │
                    ├──────────────────┤     ├──────────────────┤
                    │ id (PK)          │     │ id (PK)          │
                    │ restaurant_id(FK)│     │ restaurant_id(FK)│
                    │ name             │     │ description      │
                    │ active           │     │ percentage       │
                    │ created_at       │     │ start_date       │
                    └────────┬─────────┘     │ end_date         │
                             │               │ created_at       │
                             │ 1:N           └──────────────────┘
                             ▼
                    ┌──────────────────┐
                    │   menu_items     │
                    ├──────────────────┤
                    │ id (PK)          │
                    │ menu_id (FK)     │
                    │ name             │
                    │ description      │
                    │ price            │
                    │ category         │
                    │ created_at       │
                    └──────────────────┘
```

### 4.2 Table Definitions

#### `users`
| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `BIGSERIAL` | PRIMARY KEY |
| `username` | `VARCHAR(50)` | NOT NULL, UNIQUE |
| `password` | `VARCHAR(255)` | NOT NULL (bcrypt hash) |
| `role` | `VARCHAR(20)` | NOT NULL, CHECK (`ADMIN` or `MANAGER`) |
| `created_at` | `TIMESTAMP` | NOT NULL, DEFAULT NOW() |

#### `restaurants`
| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `BIGSERIAL` | PRIMARY KEY |
| `name` | `VARCHAR(100)` | NOT NULL |
| `address` | `VARCHAR(255)` | NOT NULL |
| `manager_id` | `BIGINT` | FOREIGN KEY → `users(id)`, UNIQUE |
| `created_at` | `TIMESTAMP` | NOT NULL, DEFAULT NOW() |

#### `menus`
| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `BIGSERIAL` | PRIMARY KEY |
| `restaurant_id` | `BIGINT` | FOREIGN KEY → `restaurants(id)` ON DELETE CASCADE, NOT NULL |
| `name` | `VARCHAR(100)` | NOT NULL |
| `active` | `BOOLEAN` | NOT NULL, DEFAULT TRUE |
| `created_at` | `TIMESTAMP` | NOT NULL, DEFAULT NOW() |

#### `menu_items`
| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `BIGSERIAL` | PRIMARY KEY |
| `menu_id` | `BIGINT` | FOREIGN KEY → `menus(id)` ON DELETE CASCADE, NOT NULL |
| `name` | `VARCHAR(100)` | NOT NULL |
| `description` | `TEXT` | |
| `price` | `DECIMAL(10,2)` | NOT NULL, CHECK (>= 0) |
| `category` | `VARCHAR(50)` | NOT NULL |
| `created_at` | `TIMESTAMP` | NOT NULL, DEFAULT NOW() |

#### `discounts`
| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `BIGSERIAL` | PRIMARY KEY |
| `restaurant_id` | `BIGINT` | FOREIGN KEY → `restaurants(id)` ON DELETE CASCADE, NOT NULL |
| `description` | `VARCHAR(255)` | NOT NULL |
| `percentage` | `DECIMAL(5,2)` | NOT NULL, CHECK (> 0 AND <= 100) |
| `start_date` | `DATE` | NOT NULL |
| `end_date` | `DATE` | NOT NULL, CHECK (end_date >= start_date) |
| `created_at` | `TIMESTAMP` | NOT NULL, DEFAULT NOW() |

### 4.3 Indexes

| Table | Index | Purpose |
|-------|-------|---------|
| `restaurants` | `manager_id` (UNIQUE) | Fast lookup by manager; enforces 1:1 |
| `menus` | `restaurant_id` | Fast lookup of menus by restaurant |
| `menu_items` | `menu_id` | Fast lookup of items by menu |
| `discounts` | `restaurant_id` | Fast lookup of discounts by restaurant |
| `users` | `username` (UNIQUE) | Fast login lookup |

### 4.4 Scaling Flaws & Data Management Concerns

| Concern | Description | Potential Mitigation (future) |
|---------|-------------|-------------------------------|
| **Single database instance** | Railway provides one PostgreSQL instance — if it goes down, the entire system is unavailable | Add read replicas; use connection pooling (PgBouncer) |
| **No caching** | Every API call hits the database directly, even for data that rarely changes (e.g., menus) | Add Redis or in-memory caching for read-heavy endpoints |
| **Cascade deletes** | Deleting a restaurant wipes all its menus, items, and discounts instantly — no undo | Implement soft deletes (`deleted_at` column) instead of hard deletes |
| **No audit trail** | No record of who changed what and when | Add an `audit_log` table or use Hibernate Envers |
| **1:1 manager constraint** | A manager can only manage one restaurant — if a franchise owner manages multiple locations, this model breaks | Change to a many-to-many join table (`restaurant_managers`) |
| **No pagination** | Listing all items or all restaurants returns the entire dataset in one response | Add paginated queries (`Pageable` in Spring Data) |
| **No connection pooling config** | Default HikariCP settings may not match Railway's connection limits | Tune `spring.datasource.hikari.maximum-pool-size` in properties |
| **Price stored as DECIMAL** | Correct for storage, but floating-point math in Java (if using `double`) can introduce rounding errors | Always use `BigDecimal` in Java entities, never `double` for money |

---

## 5. API Design

### 5.1 Base URL

```
http://localhost:8080/api
```

### 5.2 Endpoint Summary

| Method | Endpoint | Role | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/login` | Public | Authenticate and receive JWT |
| POST | `/api/auth/register` | Admin | Create a new user account |
| GET | `/api/restaurants` | Admin | List all restaurants |
| POST | `/api/restaurants` | Admin | Create a restaurant |
| GET | `/api/restaurants/{id}` | Admin/Owner | Get restaurant details |
| PUT | `/api/restaurants/{id}` | Admin | Update a restaurant |
| DELETE | `/api/restaurants/{id}` | Admin | Delete a restaurant |
| GET | `/api/restaurants/{id}/menus` | Admin/Owner | List menus for a restaurant |
| POST | `/api/restaurants/{id}/menus` | Admin/Owner | Create a menu |
| GET | `/api/menus/{id}` | Admin/Owner | Get a menu with its items |
| PUT | `/api/menus/{id}` | Admin/Owner | Update a menu |
| DELETE | `/api/menus/{id}` | Admin/Owner | Delete a menu |
| POST | `/api/menus/{id}/items` | Admin/Owner | Add an item to a menu |
| GET | `/api/items/{id}` | Admin/Owner | Get an item |
| PUT | `/api/items/{id}` | Admin/Owner | Update an item |
| DELETE | `/api/items/{id}` | Admin/Owner | Delete an item |
| GET | `/api/restaurants/{id}/discounts` | Admin/Owner | List discounts for a restaurant |
| POST | `/api/restaurants/{id}/discounts` | Admin/Owner | Create a discount |
| GET | `/api/discounts/{id}` | Admin/Owner | Get a discount |
| PUT | `/api/discounts/{id}` | Admin/Owner | Update a discount |
| DELETE | `/api/discounts/{id}` | Admin/Owner | Delete a discount |

### 5.3 Authentication Flow

```
Client                              Server
  │                                    │
  │  POST /api/auth/login              │
  │  { username, password }            │
  │───────────────────────────────────►│
  │                                    │  Validate credentials
  │                                    │  Generate JWT with:
  │                                    │    - subject: username
  │                                    │    - role: ADMIN|MANAGER
  │                                    │    - restaurantId (if MANAGER)
  │                                    │    - expiry: 24h
  │  200 OK                            │
  │  { token: "eyJhbG..." }           │
  │◄───────────────────────────────────│
  │                                    │
  │  GET /api/restaurants              │
  │  Authorization: Bearer eyJhbG...   │
  │───────────────────────────────────►│
  │                                    │  Extract JWT from header
  │                                    │  Validate signature + expiry
  │                                    │  Extract role → authorise
  │  200 OK                            │
  │  [ { id: 1, name: ... }, ... ]     │
  │◄───────────────────────────────────│
```

### 5.4 Standard Response Format

**Success:**
```json
{
  "id": 1,
  "name": "Downtown Grill",
  "address": "123 Main St",
  "managerId": 5,
  "createdAt": "2026-04-09T10:00:00"
}
```

**Error:**
```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Restaurant with id 99 not found",
  "timestamp": "2026-04-09T10:00:00"
}
```

---

## 6. Security Architecture

### 6.1 Authentication

- Stateless JWT-based authentication
- Tokens signed with HMAC-SHA256 using a secret from `.env`
- Token payload contains: `sub` (username), `role`, `restaurantId` (for managers), `iat`, `exp`
- Tokens expire after 24 hours (configurable via `JWT_EXPIRATION_MS` in `.env`)

### 6.2 Authorisation

The security filter chain intercepts all requests and enforces:

```
/api/auth/**           → permitAll (public)
/api/**                → authenticated (valid JWT required)
```

Within authenticated routes, **service-layer checks** enforce role-based access:

| Action | Admin | Manager (own restaurant) | Manager (other restaurant) |
|--------|-------|--------------------------|----------------------------|
| List all restaurants | Yes | No (sees own only) | No |
| Create restaurant | Yes | No | No |
| Edit restaurant | Yes | No | No |
| Delete restaurant | Yes | No | No |
| CRUD own menus/items | Yes | Yes | No (403) |
| CRUD own discounts | Yes | Yes | No (403) |

### 6.3 Secret Management

```
.env (NEVER committed)          .env.example (committed — safe template)
─────────────────────           ──────────────────────────────────────
DB_URL=jdbc:postgresql://...    DB_URL=jdbc:postgresql://localhost:5432/rcms
DB_USERNAME=real_user           DB_USERNAME=your_db_username
DB_PASSWORD=real_pass           DB_PASSWORD=your_db_password
JWT_SECRET=a1b2c3d4e5...       JWT_SECRET=your_jwt_secret_here
JWT_EXPIRATION_MS=86400000      JWT_EXPIRATION_MS=86400000
```

- `.env` is listed in `.gitignore`
- CI/CD uses GitHub repository secrets (Settings → Secrets → Actions)
- `application.properties` references environment variables: `spring.datasource.url=${DB_URL}`

---

## 7. Deployment Architecture

### 7.1 Local Development

```
Developer machine
├── Java 17 + Gradle
├── Spring Boot (embedded Tomcat)
├── .env file (local secrets)
└── Connects to → Railway PostgreSQL (or local Docker PostgreSQL)
```

### 7.2 CI/CD Pipeline (GitHub Actions)

```
Push to GitHub
     │
     ▼
┌──────────────────────────┐
│  GitHub Actions Workflow  │
│                          │
│  1. Checkout code        │
│  2. Set up Java 17       │
│  3. Inject secrets from  │
│     GitHub Secrets as    │
│     environment vars     │
│  4. Run: gradle build    │
│     (compiles + tests)   │
│  5. Upload .jar artifact │
└──────────────────────────┘
```

### 7.3 Workflow File (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    env:
      DB_URL: ${{ secrets.DB_URL }}
      DB_USERNAME: ${{ secrets.DB_USERNAME }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
      JWT_SECRET: ${{ secrets.JWT_SECRET }}
      JWT_EXPIRATION_MS: 86400000

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build and test
        run: ./gradlew build
      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: rcms-jar
          path: build/libs/*.jar
```

### 7.4 Production Execution

```bash
# The .jar is a self-contained executable (embedded Tomcat)
java -jar restaurant-chain-management-system-1.0.0.jar
```

Environment variables must be set before running (via `.env` loader, system env vars, or a process manager).

---

## 8. Project Structure

```
RestaurantChainManagementSystem/
├── docs/                              # Documentation (PRD, architecture, etc.)
├── .github/
│   └── workflows/
│       └── ci.yml                     # GitHub Actions CI pipeline
├── .env.example                       # Template for environment variables
├── .gitignore                         # Excludes .env, build/, .gradle/, IDE files
├── build.gradle                       # Gradle build configuration
├── settings.gradle                    # Gradle project name
├── gradlew / gradlew.bat             # Gradle wrapper scripts
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
└── src/
    ├── main/
    │   ├── java/com/rcms/
    │   │   ├── RcmsApplication.java           # Spring Boot entry point
    │   │   ├── config/
    │   │   │   ├── SecurityConfig.java        # Spring Security + JWT filter setup
    │   │   │   ├── JwtAuthFilter.java         # JWT token extraction and validation filter
    │   │   │   └── CorsConfig.java            # CORS configuration (if needed)
    │   │   ├── controller/
    │   │   │   ├── AuthController.java        # Login + register endpoints
    │   │   │   ├── RestaurantController.java  # Restaurant CRUD endpoints
    │   │   │   ├── MenuController.java        # Menu CRUD endpoints
    │   │   │   ├── MenuItemController.java    # Menu item CRUD endpoints
    │   │   │   └── DiscountController.java    # Discount CRUD endpoints
    │   │   ├── service/
    │   │   │   ├── AuthService.java
    │   │   │   ├── RestaurantService.java
    │   │   │   ├── MenuService.java
    │   │   │   ├── MenuItemService.java
    │   │   │   └── DiscountService.java
    │   │   ├── repository/
    │   │   │   ├── UserRepository.java
    │   │   │   ├── RestaurantRepository.java
    │   │   │   ├── MenuRepository.java
    │   │   │   ├── MenuItemRepository.java
    │   │   │   └── DiscountRepository.java
    │   │   ├── model/
    │   │   │   ├── User.java
    │   │   │   ├── Restaurant.java
    │   │   │   ├── Menu.java
    │   │   │   ├── MenuItem.java
    │   │   │   ├── Discount.java
    │   │   │   └── Role.java                  # Enum: ADMIN, MANAGER
    │   │   ├── dto/
    │   │   │   ├── LoginRequest.java
    │   │   │   ├── LoginResponse.java
    │   │   │   ├── RegisterRequest.java
    │   │   │   ├── RestaurantRequest.java
    │   │   │   ├── RestaurantResponse.java
    │   │   │   ├── MenuRequest.java
    │   │   │   ├── MenuResponse.java
    │   │   │   ├── MenuItemRequest.java
    │   │   │   ├── MenuItemResponse.java
    │   │   │   ├── DiscountRequest.java
    │   │   │   ├── DiscountResponse.java
    │   │   │   └── ErrorResponse.java
    │   │   └── exception/
    │   │       ├── ResourceNotFoundException.java
    │   │       ├── UnauthorisedException.java
    │   │       └── GlobalExceptionHandler.java  # @ControllerAdvice
    │   └── resources/
    │       ├── application.properties          # Spring config (references env vars)
    │       └── data.sql                        # Seed data (optional)
    └── test/
        └── java/com/rcms/
            ├── controller/
            │   ├── AuthControllerTest.java
            │   ├── RestaurantControllerTest.java
            │   ├── MenuControllerTest.java
            │   ├── MenuItemControllerTest.java
            │   └── DiscountControllerTest.java
            └── service/
                ├── AuthServiceTest.java
                ├── RestaurantServiceTest.java
                ├── MenuServiceTest.java
                ├── MenuItemServiceTest.java
                └── DiscountServiceTest.java
```
