# Login & Registration — Architecture & Design Specification

## 1. Document Overview

### 1.1 Purpose

This document defines the architecture and technical design for a secure Login & Registration application using:

- **Angular** — Frontend
- **Spring Boot** — Backend REST API
- **MySQL** — Relational database
- **Spring Security** — Authentication and authorization
- **JWT** — Access-token based authentication
- **Redis** — Optional session, token, rate-limit, and temporary-state store
- **Email Service** — Email verification and password reset
- **Docker** — Application packaging
- **Nginx / API Gateway** — Edge routing and TLS termination

### 1.2 Architecture Goals

- Secure authentication
- Clear separation of frontend, backend, and persistence layers
- RESTful API design
- Horizontal scalability
- Maintainability and testability
- Centralized security controls
- Traceability from requirements to implementation and tests
- Cloud-agnostic deployment

---

# 2. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | Angular | Web application/UI |
| Frontend Language | TypeScript | Application development |
| UI | Angular Material / CSS | User interface |
| Backend | Spring Boot | REST APIs |
| Security | Spring Security | Authentication/authorization |
| Authentication | JWT | Access token |
| ORM | Spring Data JPA | Database access |
| Database | MySQL | Persistent user data |
| Migration | Flyway | Database schema migration |
| Cache | Redis | Optional caching/rate limiting |
| Email | SMTP/Email Provider | Verification/reset emails |
| API Documentation | OpenAPI / Swagger | API documentation |
| Testing | JUnit + Mockito | Backend testing |
| Testing | Angular Test Framework | Frontend testing |
| E2E | Playwright/Cypress | End-to-end testing |
| Security Testing | OWASP ZAP | DAST |
| SAST | Semgrep / CodeQL | Source security scanning |
| SCA | Trivy | Dependency/container scanning |
| Container | Docker | Packaging |
| CI/CD | Jenkins/GitHub Actions/GitLab CI | Automation |

---

# 3. High-Level Architecture

```text
                         Internet / Users
                                |
                                v
                     +----------------------+
                     |   Load Balancer /    |
                     |   Nginx / Gateway    |
                     +----------+-----------+
                                |
                +---------------+---------------+
                |                               |
                v                               v
       +------------------+             +------------------+
       | Angular Frontend |             | Static Assets    |
       | Web Application  |             | / CDN            |
       +--------+---------+             +------------------+
                |
                | HTTPS / REST / JSON
                v
       +----------------------+
       | Spring Boot Backend  |
       | REST API             |
       +----------+-----------+
                  |
       +----------+----------+----------------+
       |                     |                |
       v                     v                v
+--------------+      +-------------+   +-------------+
| Spring       |      | Redis       |   | Email       |
| Security/JWT |      | Optional    |   | Service     |
+------+-------+      +-------------+   +-------------+
       |
       v
+----------------------+
| Service Layer        |
| RegistrationService  |
| AuthenticationService|
| PasswordService      |
+----------+-----------+
           |
           v
+----------------------+
| Repository Layer     |
| Spring Data JPA      |
+----------+-----------+
           |
           v
+----------------------+
|       MySQL          |
| Users / Tokens /     |
| Audit / Verification |
+----------------------+
```

---

# 4. Logical Architecture

The backend shall follow a layered architecture.

```text
Angular
   |
   v
Controller Layer
   |
   v
Security Layer
   |
   v
Service Layer
   |
   v
Repository Layer
   |
   v
MySQL
```

Supporting components:

```text
Service Layer
    |
    +---- Email Service
    |
    +---- Token Service
    |
    +---- Audit Service
    |
    +---- Redis
```

---

# 5. Frontend Architecture — Angular

## 5.1 Angular Structure

```text
src/
├── app/
│   ├── core/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── services/
│   │   └── models/
│   │
│   ├── features/
│   │   ├── auth/
│   │   │   ├── login/
│   │   │   ├── register/
│   │   │   ├── forgot-password/
│   │   │   ├── reset-password/
│   │   │   └── verify-email/
│   │   │
│   │   └── dashboard/
│   │
│   ├── shared/
│   │   ├── components/
│   │   ├── directives/
│   │   └── pipes/
│   │
│   ├── app.routes.ts
│   └── app.component.ts
│
└── assets/
```

## 5.2 Angular Responsibilities

Angular shall:

- Display login and registration forms.
- Perform client-side validation.
- Call backend REST APIs.
- Display validation and authentication errors.
- Maintain authentication state.
- Protect authenticated routes.
- Handle token/session expiration.
- Redirect unauthenticated users to login.
- Provide logout functionality.

---

# 6. Angular Authentication Flow

```text
User
 |
 v
Angular Login Page
 |
 v
AuthService.login()
 |
 v
POST /api/v1/auth/login
 |
 v
Spring Boot
 |
 v
AuthenticationManager
 |
 v
MySQL User Lookup
 |
 v
Password Verification
 |
 v
JWT Generated
 |
 v
Angular Receives Authentication Result
 |
 v
Authenticated Application
```

---

# 7. Angular Route Protection

Protected routes shall use an Angular authentication guard.

```text
User requests /dashboard
        |
        v
Authentication Guard
        |
    +---+---+
    |       |
Authenticated  Not Authenticated
    |               |
    v               v
Dashboard          /login
```

Example route model:

```typescript
{
  path: 'dashboard',
  canActivate: [authGuard],
  loadComponent: () => import('./dashboard/dashboard.component')
}
```

---

# 8. HTTP Interceptor

The Angular HTTP interceptor shall:

1. Add the access token where the selected token strategy requires it.
2. Handle HTTP 401 responses.
3. Attempt token refresh when enabled.
4. Redirect to login when the session is no longer valid.
5. Avoid attaching authentication credentials to public endpoints.

Example:

```text
Angular HTTP Request
        |
        v
HTTP Interceptor
        |
        v
Add Authorization Header
        |
        v
Spring Boot API
```

---

# 9. Backend Architecture — Spring Boot

## 9.1 Package Structure

```text
src/main/java/com/example/auth/
│
├── config/
│   ├── SecurityConfig.java
│   ├── OpenApiConfig.java
│   └── CorsConfig.java
│
├── controller/
│   └── AuthController.java
│
├── service/
│   ├── AuthenticationService.java
│   ├── RegistrationService.java
│   ├── PasswordService.java
│   ├── TokenService.java
│   └── EmailService.java
│
├── repository/
│   ├── UserRepository.java
│   ├── RefreshTokenRepository.java
│   └── AuditEventRepository.java
│
├── entity/
│   ├── User.java
│   ├── RefreshToken.java
│   ├── EmailVerificationToken.java
│   └── AuditEvent.java
│
├── dto/
│   ├── LoginRequest.java
│   ├── LoginResponse.java
│   ├── RegistrationRequest.java
│   └── RegistrationResponse.java
│
├── security/
│   ├── JwtAuthenticationFilter.java
│   ├── JwtTokenService.java
│   └── CustomUserDetailsService.java
│
├── exception/
│   ├── GlobalExceptionHandler.java
│   └── AuthenticationException.java
│
└── util/
```

---

# 10. Backend Request Flow

```text
HTTP Request
    |
    v
Spring Security Filter Chain
    |
    v
JWT Validation
    |
    v
Controller
    |
    v
Service
    |
    v
Repository
    |
    v
MySQL
```

For login:

```text
POST /auth/login
      |
      v
AuthController
      |
      v
AuthenticationService
      |
      v
AuthenticationManager
      |
      v
UserDetailsService
      |
      v
UserRepository
      |
      v
MySQL
      |
      v
PasswordEncoder
      |
      v
JWT Token Service
      |
      v
LoginResponse
```

---

# 11. Spring Security Architecture

```text
Client
 |
 | Authorization: Bearer <JWT>
 v
Spring Security Filter Chain
 |
 v
JwtAuthenticationFilter
 |
 +---- Invalid Token ----> 401
 |
 v
Token Validation
 |
 v
SecurityContext
 |
 v
Controller
```

The backend shall not trust client-provided identity information without validating the authentication token.

---

# 12. JWT Design

## 12.1 Access Token

The access token shall contain only the minimum required claims.

Example conceptual payload:

```json
{
  "sub": "USR-10001",
  "roles": ["USER"],
  "iat": 1755000000,
  "exp": 1755003600
}
```

Recommended properties:

- Short lifetime
- Signed using a strong signing key
- No password or sensitive information
- Issuer validation
- Audience validation where applicable

## 12.2 Refresh Token

Refresh tokens should be:

- Long-lived relative to access tokens
- Random and unpredictable
- Stored securely
- Rotated where supported
- Revocable
- Associated with a user/session

---

# 13. MySQL Database Architecture

## 13.1 Core Tables

```text
users
email_verification_tokens
refresh_tokens
password_reset_tokens
audit_events
```

## 13.2 Entity Relationship

```text
+----------------+
| users          |
+----------------+
| id PK          |
| email UNIQUE   |
| password_hash  |
| first_name     |
| last_name      |
| status         |
| email_verified |
| created_at     |
| updated_at     |
+-------+--------+
        |
        | 1:N
        |
+-------v--------+
| refresh_tokens |
+----------------+
| id PK          |
| user_id FK     |
| token_hash     |
| expires_at     |
| revoked_at     |
| created_at     |
+----------------+

        users
          |
          | 1:N
          v
+--------------------------+
| email_verification_tokens|
+--------------------------+

        users
          |
          | 1:N
          v
+--------------------------+
| password_reset_tokens    |
+--------------------------+

        users
          |
          | 1:N
          v
+--------------------------+
| audit_events             |
+--------------------------+
```

---

# 14. MySQL Schema

## 14.1 Users

```sql
CREATE TABLE users (
    id BINARY(16) PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    status VARCHAR(30) NOT NULL,
    email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    failed_login_attempts INT NOT NULL DEFAULT 0,
    account_locked_until TIMESTAMP NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## 14.2 Refresh Tokens

```sql
CREATE TABLE refresh_tokens (
    id BINARY(16) PRIMARY KEY,
    user_id BINARY(16) NOT NULL,
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    revoked_at TIMESTAMP NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_refresh_user
        FOREIGN KEY (user_id)
        REFERENCES users(id)
);
```

Production implementation should use Flyway migrations rather than executing schema manually during application startup.

---

# 15. Registration Architecture

```text
Angular Register
       |
       v
POST /api/v1/auth/register
       |
       v
RegistrationController
       |
       v
RegistrationService
       |
       +--> Validate Request
       |
       +--> Check Existing Email
       |
       +--> PasswordEncoder
       |
       +--> Create User
       |
       +--> Generate Verification Token
       |
       +--> EmailService
       |
       v
201 Created
```

The complete operation shall use appropriate transaction boundaries.

---

# 16. Login Architecture

```text
Angular Login
      |
      v
POST /api/v1/auth/login
      |
      v
AuthenticationController
      |
      v
AuthenticationService
      |
      +--> Find User
      |
      +--> Check Account Status
      |
      +--> Verify Password
      |
      +--> Check Email Verification
      |
      +--> Reset Failed Attempts
      |
      +--> Generate Access Token
      |
      +--> Generate Refresh Token
      |
      +--> Audit LOGIN_SUCCESS
      |
      v
200 OK
```

Failed authentication:

```text
Invalid Credentials
       |
       v
Increment Failed Attempts
       |
       +---- Threshold reached?
                 |
              +--+--+
              |     |
             No    Yes
              |     |
              v     v
           401    LOCKED
```

---

# 17. Registration API

## Endpoint

```http
POST /api/v1/auth/register
```

## Request

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "password": "Secure@123",
  "confirmPassword": "Secure@123"
}
```

## Response

```json
{
  "userId": "USR-10001",
  "message": "Registration successful. Please verify your email."
}
```

---

# 18. Login API

## Endpoint

```http
POST /api/v1/auth/login
```

## Request

```json
{
  "email": "john.doe@example.com",
  "password": "Secure@123"
}
```

## Response

```json
{
  "accessToken": "<access-token>",
  "refreshToken": "<refresh-token>",
  "expiresIn": 3600,
  "user": {
    "userId": "USR-10001",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com"
  }
}
```

---

# 19. Email Verification

```text
Registration
    |
    v
Generate Random Verification Token
    |
    v
Store Token Hash + Expiration
    |
    v
Send Email
    |
    v
User Clicks Link
    |
    v
GET /api/v1/auth/verify-email?token=...
    |
    v
Validate Token
    |
    v
Mark email_verified = TRUE
    |
    v
Invalidate Token
```

Verification tokens shall have a short configurable lifetime and shall be single-use.

---

# 20. Password Reset

```text
Forgot Password
      |
      v
POST /auth/forgot-password
      |
      v
Generate Reset Token
      |
      v
Send Email
      |
      v
User Opens Reset Link
      |
      v
POST /auth/reset-password
      |
      v
Validate Token
      |
      v
Hash New Password
      |
      v
Update Password
      |
      v
Invalidate Existing Sessions/Tokens
```

The forgot-password endpoint should return a generic response regardless of whether the email exists.

---

# 21. Security Architecture

## Security Controls

```text
                    Security Controls
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
   Authentication     Authorization       Protection
        |                  |                  |
        v                  v                  v
      JWT               Roles            Rate Limiting
   Password Hashing     Permissions      CSRF where applicable
   Refresh Tokens       API Access       Input Validation
        |                                     |
        +------------------+------------------+
                           |
                           v
                    Audit Logging
```

## Required Controls

- HTTPS/TLS
- Secure password hashing
- Strong JWT signing
- Token expiration
- Refresh-token revocation
- Rate limiting
- Brute-force protection
- Input validation
- Secure HTTP headers
- CORS restrictions
- Generic authentication errors
- Audit logging
- Secret management
- Dependency vulnerability scanning

---

# 22. CORS

CORS shall be explicitly configured.

Example conceptual policy:

```text
Allowed Origin:
https://app.example.com

Allowed Methods:
GET
POST
PUT
DELETE
OPTIONS

Allowed Headers:
Authorization
Content-Type
```

Do not use unrestricted `*` origins in production when credentials are involved.

---

# 23. Error Handling

Spring Boot shall use centralized exception handling.

```text
Controller
    |
    v
Exception
    |
    v
GlobalExceptionHandler
    |
    v
Standard Error Response
```

Example:

```json
{
  "timestamp": "2026-08-13T10:00:00Z",
  "status": 401,
  "code": "AUTHENTICATION_FAILED",
  "message": "Invalid email or password.",
  "correlationId": "abc-123"
}
```

---

# 24. Observability

The application shall provide:

- Structured application logs
- Correlation IDs
- Authentication event logs
- API latency metrics
- Error metrics
- Database health metrics
- JVM metrics
- Health/readiness endpoints

Recommended endpoints:

```text
/actuator/health
/actuator/metrics
```

Sensitive information such as passwords, tokens, and secrets must never be logged.

---

# 25. Deployment Architecture

```text
                    Internet
                       |
                       v
              +----------------+
              | Load Balancer  |
              +-------+--------+
                      |
          +-----------+-----------+
          |                       |
          v                       v
+-------------------+   +-------------------+
| Angular / Nginx   |   | Angular / Nginx   |
| Instance 1        |   | Instance 2        |
+---------+---------+   +---------+---------+
          |                       |
          +-----------+-----------+
                      |
                      v
             +-------------------+
             | Spring Boot       |
             | API Instances     |
             +---------+---------+
                       |
          +------------+------------+
          |            |            |
          v            v            v
       MySQL        Redis        Email
       Primary      Cluster      Provider
```

Spring Boot instances should remain stateless where possible to support horizontal scaling.

---

# 26. Docker Architecture

## Frontend

```text
Angular Build
     |
     v
Static Files
     |
     v
Nginx Container
```

## Backend

```text
Spring Boot
     |
     v
JAR
     |
     v
Java Runtime Container
```

Example deployment:

```text
docker-compose
   |
   +-- frontend
   +-- backend
   +-- mysql
   +-- redis
```

For production, managed database and cache services may be preferred over running stateful services directly in application containers.

---

# 27. Configuration Management

Environment-specific configuration shall not be hardcoded.

Example:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

security:
  jwt:
    secret: ${JWT_SECRET}
    access-token-expiration: ${JWT_ACCESS_EXPIRATION}

email:
  provider: ${EMAIL_PROVIDER}
```

Secrets shall be stored using a dedicated secret-management solution.

---

# 28. Database Migration

Flyway shall manage database schema versions.

```text
V1__create_users.sql
V2__create_refresh_tokens.sql
V3__create_verification_tokens.sql
V4__create_password_reset_tokens.sql
V5__create_audit_events.sql
```

Deployment flow:

```text
Application Deployment
        |
        v
Flyway Migration
        |
        v
Validate Schema
        |
        v
Start Application
```

---

# 29. API Versioning

APIs shall use explicit versioning.

```text
/api/v1/auth/register
/api/v1/auth/login
/api/v1/auth/logout
/api/v1/auth/refresh
/api/v1/auth/forgot-password
/api/v1/auth/reset-password
/api/v1/auth/verify-email
```

Breaking API changes should use a new API version.

---

# 30. Testing Architecture

```text
                    Test Pyramid
                         |
              +----------+----------+
              |                     |
              v                     v
         Unit Tests            Component Tests
              |                     |
              +----------+----------+
                         |
                         v
                 Integration Tests
                         |
                         v
                    API Tests
                         |
                         v
                   E2E Tests
                         |
                         v
                  Security Tests
```

## Backend

- JUnit
- Mockito
- Spring Boot Test
- Testcontainers
- REST API tests

## Frontend

- Angular unit tests
- Component tests
- E2E tests

## Security

- Semgrep / CodeQL
- Trivy
- OWASP ZAP
- Dependency scanning
- Secret scanning

---

# 31. CI/CD Pipeline

```text
Developer
    |
    v
Git Push / Pull Request
    |
    v
Build
    |
    +---- Angular Build
    |
    +---- Spring Boot Build
    |
    v
Unit Tests
    |
    v
Integration Tests
    |
    v
SAST
    |
    v
SCA / Trivy
    |
    v
Secret Scan
    |
    v
Docker Build
    |
    v
Container Scan
    |
    v
Deploy to Test
    |
    v
E2E Tests
    |
    v
DAST / ZAP
    |
    v
Approval
    |
    v
Production
```

---

# 32. AI-SDLC Traceability

The architecture shall support traceability across the development lifecycle.

```text
SPECIFICATION
     |
     +--> Requirement ID
              |
              v
        Architecture
              |
              v
        API Contract
              |
              v
       Implementation
              |
              v
          Unit Test
              |
              v
      Integration Test
              |
              v
       Security Test
              |
              v
      Acceptance Test
              |
              v
          Release
```

Example:

```text
FR-007 Login
   |
   +--> Architecture: AuthenticationService
   |
   +--> API: POST /api/v1/auth/login
   |
   +--> Code:
   |       AuthenticationService.login()
   |
   +--> Unit Test:
   |       AUTH-LOGIN-UNIT-001
   |
   +--> API Test:
   |       AUTH-LOGIN-API-001
   |
   +--> Security Test:
   |       SEC-AUTH-001
   |
   +--> E2E:
   |       AUTH-LOGIN-E2E-001
   |
   +--> Acceptance:
           AC-LOGIN-001
```

---

# 33. Recommended Repository Structure

```text
login-registration/
│
├── frontend/
│   ├── src/
│   ├── angular.json
│   ├── package.json
│   └── Dockerfile
│
├── backend/
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
│
├── database/
│   └── migration/
│       ├── V1__create_users.sql
│       └── V2__create_refresh_tokens.sql
│
├── tests/
│   ├── api/
│   └── e2e/
│
├── security/
│   ├── semgrep/
│   └── zap/
│
├── docs/
│   ├── architecture.md
│   └── api/
│
├── docker-compose.yml
└── README.md
```

---

# 34. Key Architecture Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Frontend | Angular | Enterprise web application framework |
| Backend | Spring Boot | Mature enterprise Java ecosystem |
| API | REST | Simple, interoperable integration |
| Database | MySQL | Reliable relational persistence |
| ORM | Spring Data JPA | Productivity and maintainability |
| Authentication | Spring Security | Enterprise security framework |
| Token | JWT | Stateless API authentication |
| Migration | Flyway | Controlled schema evolution |
| Cache | Redis | Optional scalability and rate limiting |
| API Contract | OpenAPI | Consumer/developer clarity |
| Container | Docker | Portable deployment |
| SAST | Semgrep/CodeQL | Source security analysis |
| SCA | Trivy | Dependency/container security |
| DAST | OWASP ZAP | Runtime security testing |

---

# 35. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Login API should normally respond within 2 seconds |
| Availability | Target 99.9% availability |
| Scalability | Backend must support horizontal scaling |
| Security | HTTPS required for production |
| Maintainability | Layered architecture with clear separation |
| Observability | Logs, metrics, health checks, correlation IDs |
| Reliability | Database transactions for critical operations |
| Compatibility | Current supported Chrome, Edge, Firefox, Safari |
| Recovery | Database backup and recovery strategy required |
| Auditability | Security events must be traceable |

---

# 36. Open Design Decisions

1. JWT storage strategy: HttpOnly secure cookie vs in-memory/browser storage.
2. Access-token lifetime.
3. Refresh-token rotation strategy.
4. Redis requirement for the first production release.
5. MFA implementation timeline.
6. SSO/SAML/OIDC integration.
7. Email provider selection.
8. Password policy configuration.
9. Account lockout threshold.
10. Multi-tenant architecture requirements.

---

# 37. Definition of Done

The Login & Registration feature is complete when:

- [ ] Angular registration page is implemented.
- [ ] Angular login page is implemented.
- [ ] Angular route protection is implemented.
- [ ] Spring Boot authentication APIs are implemented.
- [ ] Spring Security is configured.
- [ ] Password hashing is implemented.
- [ ] JWT access-token handling is implemented.
- [ ] Refresh-token strategy is implemented.
- [ ] MySQL schema is created through Flyway.
- [ ] Email verification is implemented.
- [ ] Password reset is implemented.
- [ ] Rate limiting is implemented.
- [ ] Audit events are implemented.
- [ ] Unit tests pass.
- [ ] Integration tests pass.
- [ ] E2E tests pass.
- [ ] SAST scan passes configured quality gates.
- [ ] Trivy/SCA scan passes configured quality gates.
- [ ] DAST scan passes configured quality gates.
- [ ] API documentation is published.
- [ ] Deployment configuration is validated.
- [ ] Requirements-to-test traceability is complete.
