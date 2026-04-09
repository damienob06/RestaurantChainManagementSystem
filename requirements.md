# Restaurant Chain Management System — Requirements

## 1. Tech Stack

| Layer        | Technology                      |
|--------------|---------------------------------|
| Language     | Java 21                         |
| Framework    | Spring Boot 4.0.5               |
| Build Tool   | Gradle                          |
| Database     | PostgreSQL (hosted on Railway)   |
| Testing      | JUnit 5 (via Spring Boot Test)   |

## 2. Testing Frameworks

- **Unit Testing:** JUnit 5 — included through `spring-boot-starter-test`.
- **Integration Testing:** Spring Boot Test (`@SpringBootTest`) — runs tests against the full application context, including the database layer.

## 3. Executable Binary

The final deliverable is a **`.jar` file** (Spring Boot fat jar), built with:

```bash
./gradlew bootJar
```

The resulting jar in `build/libs/` can be executed directly:

```bash
java -jar demo-0.0.1-SNAPSHOT.jar
```

## 4. Pipeline Technology

**GitHub Actions** — the repository is hosted on GitHub, so GitHub Actions is the natural CI/CD choice.

A pipeline should:

1. Run unit and integration tests on push/PR.
2. Build the `.jar` artifact.
3. (Optionally) deploy to Railway.

## 5. `.gitignore`

The `.gitignore` is built around **Gradle + Java + Spring Boot**.

### What to ignore

- `.gradle/`, `build/` — Gradle build outputs
- `.idea/`, `*.iml`, `*.iws`, `*.ipr` — IntelliJ IDEA files
- `.vscode/` — VS Code settings
- `.apt_generated`, `.classpath`, `.factorypath`, `.project`, `.settings`, `.springBeans`, `.sts4-cache` — Eclipse/STS files
- `.env` — environment secrets (see section 6)
- `HELP.md` — Spring Initializr boilerplate

### What can be removed

The root `.gitignore` still contains **Maven-specific entries** from before the switch to Gradle. These can be removed since Maven is no longer used:

- `target/` — Maven build output (Gradle uses `build/`)
- `.mvn/wrapper/maven-wrapper.jar` — Maven wrapper jar
- `!**/src/main/**/target/` and `!**/src/test/**/target/` — Maven target exclusions

### What should be added

- `.env` — must never be committed
- `.gradle/` — Gradle cache directory (present in `demo/.gitignore` but missing from root)

## 6. `.env` — Sensitive Configuration

A hypothetical `.env` file holds secrets such as:

- `DB_URL` — PostgreSQL connection string (Railway)
- `DB_USERNAME` — database username
- `DB_PASSWORD` — database password
- `JWT_SECRET` — token signing key (if auth is added)

### Workaround

Since `.env` cannot be committed:

1. **`.env.example`** — a committed template with placeholder values so other developers know which variables are required.
2. **`application.properties`** — references the variables using `${DB_URL}` syntax so Spring resolves them from the environment at runtime.
3. **CI/CD secrets** — in GitHub Actions, store the real values as repository secrets and inject them during the pipeline run.

## 7. Data Handling

Data is stored in **PostgreSQL on Railway**.

- Spring Data JPA / Hibernate manages the ORM layer.
- Entities map to tables (e.g., `Restaurant`, `MenuItem`, `Order`, `Employee`).
- Migrations can be handled with Flyway or Liquibase for schema versioning.

### Potential Flaws — Scaling and Data Management

| Flaw | Detail |
|------|--------|
| **Single database instance** | Railway's free/hobby tier is a single node — no built-in replication or failover. If it goes down, the entire chain is offline. |
| **Network latency** | The database is remote (Railway), so every query crosses the network. High-traffic periods could bottleneck on connection pool limits. |
| **No caching layer** | Without Redis or similar, repeated reads (e.g., menu lookups) always hit the database. |
| **Schema migrations at scale** | Altering large tables (adding columns, indexes) can lock tables and cause downtime without careful migration strategies. |
| **Multi-tenancy** | If each restaurant branch shares the same tables, queries must always filter by branch — a missing filter leaks data across locations. |
| **Backups** | Railway provides some backup capability, but a self-managed backup strategy (e.g., `pg_dump` on a schedule) adds resilience. |

## 8. Project Folder Structure (MVC)

```
RestaurantChainManagementSystem/
├── demo/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/example/demo/
│   │   │   │   ├── controller/    # REST controllers (API endpoints)
│   │   │   │   ├── service/       # Business logic
│   │   │   │   ├── repository/    # Data access (Spring Data JPA)
│   │   │   │   ├── model/         # Entity classes (JPA entities)
│   │   │   │   ├── dto/           # Data transfer objects
│   │   │   │   ├── config/        # Spring configuration classes
│   │   │   │   └── DemoApplication.java
│   │   │   └── resources/
│   │   │       ├── application.properties
│   │   │       └── db/migration/  # Flyway migration scripts (if used)
│   │   └── test/
│   │       └── java/com/example/demo/
│   │           ├── controller/    # Controller tests
│   │           ├── service/       # Service unit tests
│   │           └── repository/    # Repository integration tests
│   ├── build.gradle
│   └── settings.gradle
├── .env.example
├── .gitignore
└── requirements.md
```
