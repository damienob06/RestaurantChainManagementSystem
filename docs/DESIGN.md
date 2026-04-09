# Design Specification

## Restaurant Chain Management System

**Version:** 1.0
**Date:** 2026-04-09

---

## 1. Entity Design

This section specifies the JPA entity classes, their fields, annotations, and relationships in full detail. All entities use Lombok `@Data`/`@NoArgsConstructor`/`@AllArgsConstructor` for brevity, but manual getters/setters are equally valid.

### 1.1 Role Enum

```java
public enum Role {
    ADMIN,
    MANAGER
}
```

Stored as a string in the database via `@Enumerated(EnumType.STRING)`.

### 1.2 User Entity

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false)
    private String password;  // bcrypt hash — never stored in plaintext

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Role role;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }
}
```

**Notes:**
- `password` stores a bcrypt hash (60 characters). Column length should be at least 255 to accommodate future algorithm changes.
- The `User` entity does NOT have a direct `@OneToOne` back-reference to `Restaurant`. The relationship is owned by the `Restaurant` side to keep the user entity clean and avoid circular dependencies in serialisation.

### 1.3 Restaurant Entity

```java
@Entity
@Table(name = "restaurants")
public class Restaurant {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false)
    private String address;

    @OneToOne
    @JoinColumn(name = "manager_id", unique = true)
    private User manager;

    @OneToMany(mappedBy = "restaurant", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Menu> menus = new ArrayList<>();

    @OneToMany(mappedBy = "restaurant", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Discount> discounts = new ArrayList<>();

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }
}
```

**Cascade behaviour:** When a restaurant is deleted, all its menus (and their items) and discounts are deleted automatically via `CascadeType.ALL` + `orphanRemoval = true`.

### 1.4 Menu Entity

```java
@Entity
@Table(name = "menus")
public class Menu {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "restaurant_id", nullable = false)
    private Restaurant restaurant;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false)
    private Boolean active = true;

    @OneToMany(mappedBy = "menu", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<MenuItem> items = new ArrayList<>();

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }
}
```

**Notes:**
- `FetchType.LAZY` on the `restaurant` association prevents loading the full restaurant (and all its other menus) every time a single menu is fetched.
- Deleting a menu cascades to all its items.

### 1.5 MenuItem Entity

```java
@Entity
@Table(name = "menu_items")
public class MenuItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "menu_id", nullable = false)
    private Menu menu;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false, length = 50)
    private String category;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }
}
```

**Critical:** `price` is `BigDecimal`, not `double` or `float`. Floating-point types introduce rounding errors with currency. `BigDecimal` with `precision = 10, scale = 2` stores values like `12.99` exactly.

### 1.6 Discount Entity

```java
@Entity
@Table(name = "discounts")
public class Discount {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "restaurant_id", nullable = false)
    private Restaurant restaurant;

    @Column(nullable = false)
    private String description;

    @Column(nullable = false, precision = 5, scale = 2)
    private BigDecimal percentage;  // 1.00 to 100.00

    @Column(name = "start_date", nullable = false)
    private LocalDate startDate;

    @Column(name = "end_date", nullable = false)
    private LocalDate endDate;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }
}
```

---

## 2. DTO Design

DTOs (Data Transfer Objects) separate the API contract from the internal entity structure. This prevents exposing internal fields (like password hashes) and allows the API shape to evolve independently of the database schema.

### 2.1 Naming Convention

- `*Request` — incoming data from the client (used in POST/PUT bodies)
- `*Response` — outgoing data to the client (returned in responses)

### 2.2 Auth DTOs

```java
// --- LoginRequest ---
public class LoginRequest {
    @NotBlank
    private String username;

    @NotBlank
    private String password;
}

// --- LoginResponse ---
public class LoginResponse {
    private String token;
    private String role;
    private Long restaurantId;  // null for ADMIN
}

// --- RegisterRequest ---
public class RegisterRequest {
    @NotBlank @Size(min = 3, max = 50)
    private String username;

    @NotBlank @Size(min = 8)
    private String password;

    @NotNull
    private Role role;
}
```

### 2.3 Restaurant DTOs

```java
// --- RestaurantRequest ---
public class RestaurantRequest {
    @NotBlank @Size(max = 100)
    private String name;

    @NotBlank @Size(max = 255)
    private String address;

    @NotNull
    private Long managerId;
}

// --- RestaurantResponse ---
public class RestaurantResponse {
    private Long id;
    private String name;
    private String address;
    private Long managerId;
    private String managerUsername;
    private LocalDateTime createdAt;
}
```

### 2.4 Menu DTOs

```java
// --- MenuRequest ---
public class MenuRequest {
    @NotBlank @Size(max = 100)
    private String name;

    private Boolean active = true;  // defaults to true on creation
}

// --- MenuResponse ---
public class MenuResponse {
    private Long id;
    private Long restaurantId;
    private String name;
    private Boolean active;
    private List<MenuItemResponse> items;  // included when fetching a single menu
    private LocalDateTime createdAt;
}
```

### 2.5 MenuItem DTOs

```java
// --- MenuItemRequest ---
public class MenuItemRequest {
    @NotBlank @Size(max = 100)
    private String name;

    private String description;

    @NotNull @DecimalMin("0.00")
    private BigDecimal price;

    @NotBlank @Size(max = 50)
    private String category;
}

// --- MenuItemResponse ---
public class MenuItemResponse {
    private Long id;
    private Long menuId;
    private String name;
    private String description;
    private BigDecimal price;
    private String category;
    private LocalDateTime createdAt;
}
```

### 2.6 Discount DTOs

```java
// --- DiscountRequest ---
public class DiscountRequest {
    @NotBlank @Size(max = 255)
    private String description;

    @NotNull @DecimalMin("1.00") @DecimalMax("100.00")
    private BigDecimal percentage;

    @NotNull
    private LocalDate startDate;

    @NotNull
    private LocalDate endDate;
}

// --- DiscountResponse ---
public class DiscountResponse {
    private Long id;
    private Long restaurantId;
    private String description;
    private BigDecimal percentage;
    private LocalDate startDate;
    private LocalDate endDate;
    private Boolean expired;      // derived: endDate < today
    private LocalDateTime createdAt;
}
```

### 2.7 Error Response DTO

```java
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private LocalDateTime timestamp;
}
```

---

## 3. Service Layer Design

Each service class encapsulates the business logic for its domain. Services are the **only** layer that enforces business rules — controllers delegate to services, and repositories just handle persistence.

### 3.1 AuthService

```
AuthService
├── login(LoginRequest) → LoginResponse
│   ├── Find user by username (404 if not found)
│   ├── Verify password with BCryptPasswordEncoder.matches()
│   ├── Generate JWT token with claims (role, restaurantId)
│   └── Return token + role + restaurantId
│
└── register(RegisterRequest) → void
    ├── Check username uniqueness (409 Conflict if taken)
    ├── Hash password with BCryptPasswordEncoder.encode()
    └── Save new User entity
```

### 3.2 RestaurantService

```
RestaurantService
├── getAllRestaurants() → List<RestaurantResponse>
│   └── Admin only — returns all restaurants
│
├── getRestaurantById(Long id, User currentUser) → RestaurantResponse
│   ├── If MANAGER: verify id matches their assigned restaurant (403 if not)
│   └── Return restaurant details
│
├── createRestaurant(RestaurantRequest) → RestaurantResponse
│   ├── Admin only
│   ├── Verify manager exists and has MANAGER role
│   ├── Verify manager is not already assigned to another restaurant
│   └── Create and save restaurant
│
├── updateRestaurant(Long id, RestaurantRequest) → RestaurantResponse
│   ├── Admin only
│   ├── Find restaurant (404 if not found)
│   ├── Apply updates
│   └── Save and return
│
└── deleteRestaurant(Long id) → void
    ├── Admin only
    ├── Find restaurant (404 if not found)
    └── Delete (cascades to menus, items, discounts)
```

### 3.3 MenuService

```
MenuService
├── getMenusByRestaurant(Long restaurantId, User currentUser) → List<MenuResponse>
│   ├── Verify ownership or admin role
│   └── Return menus (without nested items for list view)
│
├── getMenuById(Long menuId, User currentUser) → MenuResponse
│   ├── Find menu (404 if not found)
│   ├── Verify the menu's restaurant matches the user's access
│   └── Return menu with nested items
│
├── createMenu(Long restaurantId, MenuRequest, User currentUser) → MenuResponse
│   ├── Verify ownership or admin role
│   ├── Find restaurant (404 if not found)
│   └── Create and save menu
│
├── updateMenu(Long menuId, MenuRequest, User currentUser) → MenuResponse
│   ├── Find menu (404)
│   ├── Verify ownership or admin
│   ├── Apply updates (name, active flag)
│   └── Save and return
│
└── deleteMenu(Long menuId, User currentUser) → void
    ├── Find menu (404)
    ├── Verify ownership or admin
    └── Delete (cascades to items)
```

### 3.4 MenuItemService

```
MenuItemService
├── addItem(Long menuId, MenuItemRequest, User currentUser) → MenuItemResponse
│   ├── Find menu (404)
│   ├── Verify menu's restaurant ownership or admin
│   └── Create and save item
│
├── getItem(Long itemId, User currentUser) → MenuItemResponse
│   ├── Find item (404)
│   ├── Verify access via item → menu → restaurant chain
│   └── Return item
│
├── updateItem(Long itemId, MenuItemRequest, User currentUser) → MenuItemResponse
│   ├── Find item (404)
│   ├── Verify access
│   ├── Apply updates
│   └── Save and return
│
└── deleteItem(Long itemId, User currentUser) → void
    ├── Find item (404)
    ├── Verify access
    └── Delete
```

### 3.5 DiscountService

```
DiscountService
├── getDiscountsByRestaurant(Long restaurantId, User currentUser) → List<DiscountResponse>
│   ├── Verify ownership or admin
│   └── Return discounts with computed `expired` flag
│
├── getDiscount(Long discountId, User currentUser) → DiscountResponse
│   ├── Find discount (404)
│   ├── Verify access
│   └── Return with `expired` flag
│
├── createDiscount(Long restaurantId, DiscountRequest, User currentUser) → DiscountResponse
│   ├── Verify ownership or admin
│   ├── Validate: endDate >= startDate (400 if not)
│   ├── Validate: percentage between 1 and 100 (400 if not)
│   └── Create and save
│
├── updateDiscount(Long discountId, DiscountRequest, User currentUser) → DiscountResponse
│   ├── Find discount (404)
│   ├── Verify access
│   ├── Re-validate business rules
│   ├── Apply updates
│   └── Save and return
│
└── deleteDiscount(Long discountId, User currentUser) → void
    ├── Find discount (404)
    ├── Verify access
    └── Delete
```

### 3.6 Ownership Verification Pattern

Every service method that accesses a scoped resource follows this pattern:

```java
private void verifyAccess(Restaurant restaurant, User currentUser) {
    if (currentUser.getRole() == Role.ADMIN) {
        return;  // Admins can access everything
    }
    // For managers: the restaurant's manager must be the current user
    if (!restaurant.getManager().getId().equals(currentUser.getId())) {
        throw new UnauthorisedException("You do not have access to this restaurant");
    }
}
```

This check is performed in every service method that is not admin-only. The check is in the **service layer**, not the controller, because:
- Controllers shouldn't contain business logic
- Multiple controllers might need the same check
- It's easier to unit test in the service layer

---

## 4. Controller Design

### 4.1 Principles

- Controllers are thin — they parse the HTTP request, call the service, and return the HTTP response
- All validation of request bodies is handled by `@Valid` annotations on DTOs
- The currently authenticated user is injected via `@AuthenticationPrincipal`
- Response codes follow REST conventions:
  - `200 OK` — successful read or update
  - `201 Created` — successful creation
  - `204 No Content` — successful deletion
  - `400 Bad Request` — validation failure
  - `401 Unauthorized` — missing or invalid JWT
  - `403 Forbidden` — valid JWT but insufficient permissions
  - `404 Not Found` — resource does not exist
  - `409 Conflict` — duplicate (e.g., username already taken)

### 4.2 AuthController

```
POST /api/auth/login
  Request:  LoginRequest { username, password }
  Response: 200 → LoginResponse { token, role, restaurantId }
            401 → ErrorResponse (invalid credentials)

POST /api/auth/register        [Admin only]
  Request:  RegisterRequest { username, password, role }
  Response: 201 → (empty body, or the created user's id)
            409 → ErrorResponse (username already exists)
```

### 4.3 RestaurantController

```
GET /api/restaurants            [Admin only]
  Response: 200 → List<RestaurantResponse>

POST /api/restaurants           [Admin only]
  Request:  RestaurantRequest { name, address, managerId }
  Response: 201 → RestaurantResponse

GET /api/restaurants/{id}       [Admin or owning Manager]
  Response: 200 → RestaurantResponse
            403 → ErrorResponse (not your restaurant)
            404 → ErrorResponse

PUT /api/restaurants/{id}       [Admin only]
  Request:  RestaurantRequest { name, address, managerId }
  Response: 200 → RestaurantResponse
            404 → ErrorResponse

DELETE /api/restaurants/{id}    [Admin only]
  Response: 204 → (no body)
            404 → ErrorResponse
```

### 4.4 MenuController

```
GET /api/restaurants/{restaurantId}/menus    [Admin or owning Manager]
  Response: 200 → List<MenuResponse>  (without nested items)

POST /api/restaurants/{restaurantId}/menus   [Admin or owning Manager]
  Request:  MenuRequest { name, active }
  Response: 201 → MenuResponse

GET /api/menus/{id}                          [Admin or owning Manager]
  Response: 200 → MenuResponse (with nested items)
            404 → ErrorResponse

PUT /api/menus/{id}                          [Admin or owning Manager]
  Request:  MenuRequest { name, active }
  Response: 200 → MenuResponse

DELETE /api/menus/{id}                       [Admin or owning Manager]
  Response: 204 → (no body)
```

### 4.5 MenuItemController

```
POST /api/menus/{menuId}/items               [Admin or owning Manager]
  Request:  MenuItemRequest { name, description, price, category }
  Response: 201 → MenuItemResponse

GET /api/items/{id}                          [Admin or owning Manager]
  Response: 200 → MenuItemResponse

PUT /api/items/{id}                          [Admin or owning Manager]
  Request:  MenuItemRequest { name, description, price, category }
  Response: 200 → MenuItemResponse

DELETE /api/items/{id}                       [Admin or owning Manager]
  Response: 204 → (no body)
```

### 4.6 DiscountController

```
GET /api/restaurants/{restaurantId}/discounts [Admin or owning Manager]
  Response: 200 → List<DiscountResponse>

POST /api/restaurants/{restaurantId}/discounts [Admin or owning Manager]
  Request:  DiscountRequest { description, percentage, startDate, endDate }
  Response: 201 → DiscountResponse

GET /api/discounts/{id}                       [Admin or owning Manager]
  Response: 200 → DiscountResponse

PUT /api/discounts/{id}                       [Admin or owning Manager]
  Request:  DiscountRequest { description, percentage, startDate, endDate }
  Response: 200 → DiscountResponse

DELETE /api/discounts/{id}                    [Admin or owning Manager]
  Response: 204 → (no body)
```

---

## 5. Security Design

### 5.1 JWT Token Structure

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "sub": "johndoe",
  "role": "MANAGER",
  "restaurantId": 3,
  "iat": 1744185600,
  "exp": 1744272000
}
```

- `sub` — the username (used to load the full user from the database)
- `role` — `ADMIN` or `MANAGER` (used for quick role checks without a DB call)
- `restaurantId` — the restaurant the manager is assigned to (null for admins; used for quick ownership checks)
- `iat` — issued at (epoch seconds)
- `exp` — expiration (epoch seconds, default 24h after iat)

**Signature:** HMAC-SHA256 using the `JWT_SECRET` from `.env`.

### 5.2 JWT Authentication Filter

The `JwtAuthFilter` is a `OncePerRequestFilter` that runs before Spring Security's default authentication:

```
Incoming request
     │
     ▼
Extract "Authorization" header
     │
     ├── Missing or doesn't start with "Bearer " → skip filter, continue chain
     │                                              (Spring Security will reject
     │                                               if endpoint requires auth)
     ▼
Extract token string (after "Bearer ")
     │
     ▼
Parse and validate token (signature + expiry)
     │
     ├── Invalid or expired → skip filter (request will fail as 401)
     │
     ▼
Extract username from "sub" claim
     │
     ▼
Load UserDetails from database (or build from claims)
     │
     ▼
Create UsernamePasswordAuthenticationToken
Set in SecurityContextHolder
     │
     ▼
Continue filter chain (request is now authenticated)
```

### 5.3 SecurityConfig

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http,
                                           JwtAuthFilter jwtFilter) throws Exception {
        http
            .csrf(csrf -> csrf.disable())                  // Stateless API — no CSRF
            .sessionManagement(sm ->
                sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/login").permitAll()
                .requestMatchers("/api/auth/register").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

**Notes:**
- CSRF is disabled because the API is stateless (no cookies, no sessions). JWT in the `Authorization` header is not vulnerable to CSRF.
- Session creation is set to `STATELESS` to prevent Spring from creating HTTP sessions.
- The `/api/auth/login` endpoint is public. All other endpoints require a valid JWT.
- The `/api/auth/register` endpoint requires the `ADMIN` role (checked at the Spring Security level).
- Fine-grained ownership checks (manager can only access their own restaurant) are in the service layer, not here.

---

## 6. Error Handling Design

### 6.1 GlobalExceptionHandler

A `@ControllerAdvice` class catches exceptions thrown by controllers and services and converts them to consistent `ErrorResponse` JSON:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return buildResponse(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(UnauthorisedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorised(UnauthorisedException ex) {
        return buildResponse(HttpStatus.FORBIDDEN, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return buildResponse(HttpStatus.BAD_REQUEST, message);
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleConflict(DataIntegrityViolationException ex) {
        return buildResponse(HttpStatus.CONFLICT, "A resource with that identifier already exists");
    }

    private ResponseEntity<ErrorResponse> buildResponse(HttpStatus status, String message) {
        ErrorResponse body = new ErrorResponse(
            status.value(), status.getReasonPhrase(), message, LocalDateTime.now()
        );
        return ResponseEntity.status(status).body(body);
    }
}
```

### 6.2 Custom Exceptions

| Exception | HTTP Status | When Thrown |
|-----------|-------------|-------------|
| `ResourceNotFoundException` | 404 | Entity not found by ID |
| `UnauthorisedException` | 403 | Manager tries to access another restaurant's resource |
| `MethodArgumentNotValidException` | 400 | `@Valid` fails on a request DTO (handled by Spring) |
| `DataIntegrityViolationException` | 409 | Unique constraint violation (e.g., duplicate username) |

---

## 7. Validation Rules

### 7.1 Request Validation (via `@Valid` + Jakarta annotations)

| DTO | Field | Rule |
|-----|-------|------|
| LoginRequest | username | `@NotBlank` |
| LoginRequest | password | `@NotBlank` |
| RegisterRequest | username | `@NotBlank`, `@Size(min=3, max=50)` |
| RegisterRequest | password | `@NotBlank`, `@Size(min=8)` |
| RegisterRequest | role | `@NotNull` |
| RestaurantRequest | name | `@NotBlank`, `@Size(max=100)` |
| RestaurantRequest | address | `@NotBlank`, `@Size(max=255)` |
| RestaurantRequest | managerId | `@NotNull` |
| MenuRequest | name | `@NotBlank`, `@Size(max=100)` |
| MenuItemRequest | name | `@NotBlank`, `@Size(max=100)` |
| MenuItemRequest | price | `@NotNull`, `@DecimalMin("0.00")` |
| MenuItemRequest | category | `@NotBlank`, `@Size(max=50)` |
| DiscountRequest | description | `@NotBlank`, `@Size(max=255)` |
| DiscountRequest | percentage | `@NotNull`, `@DecimalMin("1.00")`, `@DecimalMax("100.00")` |
| DiscountRequest | startDate | `@NotNull` |
| DiscountRequest | endDate | `@NotNull` |

### 7.2 Business Validation (in service layer)

| Rule | Where | Error |
|------|-------|-------|
| Discount `endDate` must be >= `startDate` | DiscountService.create/update | 400 Bad Request |
| Manager must exist and have `MANAGER` role | RestaurantService.create | 400 Bad Request |
| Manager must not be assigned to another restaurant | RestaurantService.create | 409 Conflict |
| Username must be unique | AuthService.register | 409 Conflict |
| Password must match stored hash | AuthService.login | 401 Unauthorized |

---

## 8. Testing Strategy

### 8.1 Unit Tests (Service Layer)

Unit tests test service methods in isolation by mocking the repository layer.

**What to test:**
- Happy path for each CRUD operation
- Ownership verification (manager accessing own resource = OK, other restaurant = exception)
- Admin bypass (admin accessing any resource = OK)
- Not found scenarios (invalid ID throws `ResourceNotFoundException`)
- Validation logic (discount dates, duplicate usernames)

**Example pattern:**

```java
@ExtendWith(MockitoExtension.class)
class RestaurantServiceTest {

    @Mock
    private RestaurantRepository restaurantRepository;

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private RestaurantService restaurantService;

    @Test
    void createRestaurant_withValidRequest_shouldReturnRestaurantResponse() {
        // Arrange: mock userRepository.findById to return a MANAGER user
        // Arrange: mock restaurantRepository.save to return the saved entity
        // Act: call restaurantService.createRestaurant(request)
        // Assert: verify the response fields match
    }

    @Test
    void getRestaurantById_asManagerOfDifferentRestaurant_shouldThrowUnauthorised() {
        // Arrange: create a manager user assigned to restaurant 1
        // Arrange: mock restaurantRepository.findById(2) to return restaurant 2
        // Act + Assert: verify UnauthorisedException is thrown
    }
}
```

### 8.2 Integration Tests (Controller Layer)

Integration tests boot the Spring context and test the full request/response cycle using `MockMvc`.

**What to test:**
- HTTP status codes for each endpoint
- JSON response structure
- Authentication enforcement (missing token = 401)
- Role enforcement (manager calling admin-only endpoint = 403)
- Validation error responses (invalid body = 400 with error details)

**Example pattern:**

```java
@SpringBootTest
@AutoConfigureMockMvc
class RestaurantControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getRestaurants_withoutToken_shouldReturn401() throws Exception {
        mockMvc.perform(get("/api/restaurants"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void getRestaurants_asAdmin_shouldReturn200() throws Exception {
        String token = obtainAdminToken();
        mockMvc.perform(get("/api/restaurants")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$").isArray());
    }

    @Test
    void createRestaurant_asManager_shouldReturn403() throws Exception {
        String token = obtainManagerToken();
        mockMvc.perform(post("/api/restaurants")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"Test\",\"address\":\"123 St\",\"managerId\":1}"))
            .andExpect(status().isForbidden());
    }
}
```

### 8.3 Test Coverage Targets

| Layer | Target | Rationale |
|-------|--------|-----------|
| Service | 80%+ line coverage | This is where business logic lives — the highest-risk code |
| Controller | Key paths covered | Focus on auth enforcement and error responses, not JSON serialisation details |
| Repository | Not directly tested | Spring Data generates implementations — testing these tests the framework, not our code |
| Model/DTO | Not directly tested | Simple data carriers with no logic |
