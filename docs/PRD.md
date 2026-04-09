# Product Requirements Document (PRD)

## Restaurant Chain Management System

**Version:** 1.0
**Date:** 2026-04-09
**Team Size:** 7 developers

---

## 1. Product Overview

### 1.1 Purpose

The Restaurant Chain Management System (RCMS) is a centralised web-based application that enables a restaurant franchise to manage multiple restaurant locations from a single platform. The system provides a chain-level administrator with full oversight of all restaurants, while granting individual franchise managers autonomy over their own restaurant's menu, pricing, and promotional discounts.

### 1.2 Problem Statement

Managing a growing restaurant chain requires coordinating menus, pricing, and promotions across multiple locations — each of which may have regional variations. Without a centralised system, franchise operators rely on ad-hoc communication (spreadsheets, emails, phone calls), which leads to:

- Inconsistent menus across locations
- No audit trail when items or prices change
- No central visibility for the chain administrator
- Difficulty rolling out or revoking promotions
- No secure, role-separated access for managers vs. administrators

### 1.3 Product Goals

| # | Goal | Success Measure |
|---|------|-----------------|
| G1 | Centralised restaurant oversight | Admin can view and manage all restaurants from one dashboard |
| G2 | Per-restaurant menu autonomy | Each manager can independently maintain their restaurant's menu |
| G3 | Location-specific discounts | Managers can create, edit, and expire discounts scoped to their restaurant |
| G4 | Secure role-based access | Admins and managers authenticate separately and see only what their role permits |
| G5 | Executable deliverable | The system ships as a runnable `.jar` binary |

---

## 2. User Personas

### 2.1 Chain Administrator (Admin)

- **Role:** Oversees the entire restaurant chain
- **Goals:**
  - View all restaurants and their current status
  - Create new restaurant locations and assign franchise managers
  - Edit or remove any restaurant, menu, item, or discount across the chain
  - Monitor consistency and compliance across locations
- **Access level:** Unrestricted — full CRUD on all resources
- **Typical actions:**
  - Onboard a new franchise location
  - Reassign a franchise manager
  - Audit a restaurant's menu for brand compliance
  - Remove a discontinued item chain-wide

### 2.2 Franchise Manager (Manager)

- **Role:** Manages a single restaurant location day-to-day
- **Goals:**
  - Maintain the restaurant's menu (add, edit, remove items)
  - Set item prices appropriate for the location
  - Create and manage promotional discounts exclusive to their restaurant
  - View their own restaurant's data
- **Access level:** Restricted to their assigned restaurant only
- **Typical actions:**
  - Add a seasonal menu item
  - Change the price of an existing item
  - Create a "20% off appetisers" discount for the weekend
  - Deactivate a menu that is no longer in use

---

## 3. Functional Requirements

### 3.1 Authentication & Authorisation

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-AUTH-01 | Users must authenticate with a username and password | Must |
| FR-AUTH-02 | The system must issue a JWT token upon successful login | Must |
| FR-AUTH-03 | All API endpoints (except login/register) must require a valid JWT | Must |
| FR-AUTH-04 | The system must enforce two roles: `ADMIN` and `MANAGER` | Must |
| FR-AUTH-05 | Admins can access all resources across all restaurants | Must |
| FR-AUTH-06 | Managers can only access resources belonging to their assigned restaurant | Must |
| FR-AUTH-07 | A manager attempting to access another restaurant's data must receive a 403 Forbidden | Must |
| FR-AUTH-08 | Only admins can register new users | Should |

### 3.2 Restaurant Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-REST-01 | Admins can create a new restaurant with a name, address, and assigned manager | Must |
| FR-REST-02 | Admins can view a list of all restaurants | Must |
| FR-REST-03 | Admins can view the details of any single restaurant | Must |
| FR-REST-04 | Admins can update a restaurant's details (name, address, manager) | Must |
| FR-REST-05 | Admins can delete a restaurant (cascading to its menus, items, and discounts) | Must |
| FR-REST-06 | Managers can view the details of their own restaurant | Must |
| FR-REST-07 | Each restaurant must have a unique ID, a name, an address, and a reference to its franchise manager | Must |

### 3.3 Menu Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-MENU-01 | A restaurant can have one or more menus (e.g., "Breakfast", "Dinner", "Seasonal") | Must |
| FR-MENU-02 | Managers can create a new menu for their restaurant | Must |
| FR-MENU-03 | Managers can update a menu's name or active status | Must |
| FR-MENU-04 | Managers can delete a menu (cascading to its items) | Must |
| FR-MENU-05 | Managers can view all menus belonging to their restaurant | Must |
| FR-MENU-06 | Admins can perform all menu operations on any restaurant | Must |
| FR-MENU-07 | Each menu must have a unique ID, a name, an active/inactive flag, and belong to exactly one restaurant | Must |

### 3.4 Menu Item Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-ITEM-01 | Each menu contains zero or more menu items | Must |
| FR-ITEM-02 | Managers can add an item to one of their restaurant's menus | Must |
| FR-ITEM-03 | Managers can update an item's name, description, price, or category | Must |
| FR-ITEM-04 | Managers can remove an item from a menu | Must |
| FR-ITEM-05 | Managers can view all items on a given menu | Must |
| FR-ITEM-06 | Admins can perform all item operations on any restaurant's menus | Must |
| FR-ITEM-07 | Each item must have a unique ID, a name, a description, a price (≥ 0), a category, and belong to exactly one menu | Must |
| FR-ITEM-08 | Item prices must be stored with two decimal places of precision | Should |

### 3.5 Discount Management

| ID | Requirement | Priority |
|----|-------------|----------|
| FR-DISC-01 | Managers can create a discount exclusive to their restaurant | Must |
| FR-DISC-02 | A discount must have a description, a percentage (1–100), a start date, and an end date | Must |
| FR-DISC-03 | Managers can update a discount's details | Must |
| FR-DISC-04 | Managers can delete a discount | Must |
| FR-DISC-05 | Managers can view all discounts for their restaurant | Must |
| FR-DISC-06 | Admins can perform all discount operations on any restaurant | Must |
| FR-DISC-07 | The system should not automatically apply discounts to item prices — discounts are informational records that can be displayed alongside items | Should |
| FR-DISC-08 | Expired discounts (end date in the past) should still be queryable but clearly marked as inactive | Should |

---

## 4. Non-Functional Requirements

| ID | Requirement | Category | Target |
|----|-------------|----------|--------|
| NFR-01 | The system must respond to API requests within 500ms under normal load | Performance | p95 < 500ms |
| NFR-02 | The system must handle at least 50 concurrent users without degradation | Scalability | 50 concurrent sessions |
| NFR-03 | Passwords must be hashed using bcrypt before storage | Security | bcrypt with strength ≥ 10 |
| NFR-04 | JWT tokens must expire after a configurable duration (default: 24 hours) | Security | Configurable TTL |
| NFR-05 | No secrets (database credentials, JWT signing keys) may be committed to version control | Security | `.env` excluded via `.gitignore` |
| NFR-06 | The application must be packaged as an executable `.jar` file | Deployment | `java -jar app.jar` |
| NFR-07 | The project must include unit tests for service-layer logic | Quality | JUnit 5 |
| NFR-08 | The project must include integration tests for controller endpoints | Quality | MockMvc / SpringBootTest |
| NFR-09 | A CI/CD pipeline must run tests automatically on every push | Quality | GitHub Actions |
| NFR-10 | The system must use PostgreSQL as its persistent data store | Infrastructure | PostgreSQL on Railway |

---

## 5. User Stories

### 5.1 Admin Stories

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-A01 | As an admin, I want to log in so that I can access the management system | Given valid credentials, when I POST to `/api/auth/login`, then I receive a JWT token with role `ADMIN` |
| US-A02 | As an admin, I want to see all restaurants so that I can oversee the chain | Given I am authenticated as admin, when I GET `/api/restaurants`, then I receive a list of all restaurants with their manager info |
| US-A03 | As an admin, I want to create a new restaurant so that I can onboard a new franchise location | Given I provide a name, address, and manager ID, when I POST to `/api/restaurants`, then the restaurant is created and returned with a generated ID |
| US-A04 | As an admin, I want to update a restaurant's details so that I can correct information or reassign managers | Given a valid restaurant ID and updated fields, when I PUT to `/api/restaurants/{id}`, then the restaurant is updated |
| US-A05 | As an admin, I want to delete a restaurant so that I can remove closed locations | Given a valid restaurant ID, when I DELETE `/api/restaurants/{id}`, then the restaurant and all its menus, items, and discounts are removed |
| US-A06 | As an admin, I want to register new manager accounts so that I can grant access to franchise managers | Given a username, password, and role, when I POST to `/api/auth/register`, then a new user account is created |
| US-A07 | As an admin, I want to edit any restaurant's menu, items, or discounts so that I can enforce brand standards | Given I am authenticated as admin, when I operate on any restaurant's sub-resources, then the operation succeeds regardless of restaurant ownership |

### 5.2 Manager Stories

| ID | Story | Acceptance Criteria |
|----|-------|---------------------|
| US-M01 | As a manager, I want to log in so that I can manage my restaurant | Given valid credentials, when I POST to `/api/auth/login`, then I receive a JWT token with role `MANAGER` and my assigned restaurant ID |
| US-M02 | As a manager, I want to view my restaurant's details so that I can confirm my location's information | Given I am authenticated, when I GET `/api/restaurants/{myId}`, then I receive my restaurant's details |
| US-M03 | As a manager, I want to create a menu so that I can organise my restaurant's offerings | Given a menu name, when I POST to `/api/restaurants/{myId}/menus`, then a new menu is created under my restaurant |
| US-M04 | As a manager, I want to add items to a menu so that customers can see what is available | Given item details (name, description, price, category), when I POST to `/api/menus/{menuId}/items`, then the item is added to the menu |
| US-M05 | As a manager, I want to update an item's price so that I can adjust for local costs | Given a valid item ID and new price, when I PUT to `/api/items/{id}`, then the item price is updated |
| US-M06 | As a manager, I want to create a discount for my restaurant so that I can run promotions | Given discount details (description, percentage, start date, end date), when I POST to `/api/restaurants/{myId}/discounts`, then the discount is created |
| US-M07 | As a manager, I want to deactivate a menu so that it is no longer visible without deleting it | Given a valid menu ID, when I PATCH `/api/menus/{id}` with `active: false`, then the menu is marked inactive |
| US-M08 | As a manager, I must not be able to access another restaurant's data so that franchise boundaries are enforced | Given I am authenticated, when I attempt to access a resource belonging to another restaurant, then I receive 403 Forbidden |

---

## 6. Scope

### 6.1 In Scope

- RESTful API backend (no frontend/UI in v1)
- JWT-based authentication and role-based authorisation
- Full CRUD for restaurants, menus, menu items, and discounts
- PostgreSQL database with JPA/Hibernate ORM
- Unit and integration test suites
- CI/CD pipeline via GitHub Actions
- Executable `.jar` packaging
- `.env`-based secret management with `.env.example` template

### 6.2 Out of Scope (v1)

- Frontend / web UI (API-only for this release)
- Customer-facing ordering or payment
- Real-time notifications or WebSocket support
- Multi-tenancy or white-labelling
- Reporting, analytics, or dashboards
- Image/photo uploads for menu items
- Chain-wide discounts that apply to all restaurants simultaneously
- Email notifications or password reset flows
- Rate limiting or API throttling
- Internationalisation (i18n)

---

## 7. Assumptions and Constraints

### 7.1 Assumptions

1. Each franchise manager manages exactly one restaurant (1:1 relationship)
2. The admin account is pre-seeded or created via a protected registration endpoint
3. All communication with the API happens over HTTPS in production
4. The team has access to a shared GitHub repository
5. Railway provides a managed PostgreSQL instance accessible via connection string in `.env`
6. Menu items belong to exactly one menu (no sharing items across menus)

### 7.2 Constraints

1. **Team size:** 7 developers with varying experience levels
2. **Language:** Java (primary), Spring Boot framework
3. **Database:** PostgreSQL hosted on Railway
4. **Build tool:** Gradle
5. **Testing:** JUnit 5 (mandated by assignment)
6. **Secrets:** Must not be committed — `.env` pattern with `.env.example`
7. **Deliverable:** Must produce a runnable `.jar` file

---

## 8. Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Database schema changes mid-development break other team members' work | High | High | Finalise entity design early (Person 2 completes first); use Flyway or manual SQL migrations |
| Merge conflicts from 7 people working in parallel | High | Medium | Clear folder ownership per person; short-lived feature branches; frequent merges to main |
| Railway free tier has connection limits or downtime | Medium | High | Each developer can run PostgreSQL locally via Docker for development; Railway used for shared/staging only |
| JWT implementation has security flaws | Medium | High | Use well-tested libraries (`jjwt`); follow OWASP guidelines; Person 3 focuses exclusively on this |
| Scope creep (adding frontend, ordering, etc.) | Medium | Medium | Strict adherence to "Out of Scope" list above |
| Team members blocked waiting for foundational work | High | Medium | Persons 1 and 2 start immediately; others can scaffold their layers with stub data while waiting |
