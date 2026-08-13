# Architecture Design — Login & Registration Module

## 1. Architecture Overview

The Login & Registration module follows a **clean, layered architecture** with clear separation of concerns across three primary layers: Presentation (Angular), Application (Spring Boot), and Persistence (MySQL). This design enables horizontal scalability, independent deployment, and comprehensive testability.

```
┌─────────────────────────────────────────────────────────────┐
│                     Browser / Client                        │
│                    (Angular SPA)                            │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS / REST / JSON
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              API Gateway / Load Balancer                     │
│            (Nginx / AWS ALB)                                │
│            - TLS Termination                                │
│            - Rate Limiting                                  │
│            - Request Routing                                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         Spring Boot Application Tier                         │
│         (Stateless, Horizontally Scalable)                  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Controller Layer (REST Endpoints)                  │  │
│  │  - POST /api/v1/auth/register                       │  │
│  │  - POST /api/v1/auth/login                          │  │
│  │  - POST /api/v1/auth/logout                         │  │
│  │  - POST /api/v1/auth/refresh                        │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │                                      │
│  ┌──────────────────▼──────────────────────────────────┐  │
│  │  Security & Filter Chain (Spring Security)          │  │
│  │  - JwtAuthenticationFilter                          │  │
│  │  - CORS Configuration                               │  │
│  │  - HTTP Security Headers                            │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │                                      │
│  ┌──────────────────▼──────────────────────────────────┐  │
│  │  Service Layer (Business Logic)                     │  │
│  │  - AuthenticationService                            │  │
│  │  - RegistrationService                              │  │
│  │  - PasswordService                                  │  │
│  │  - TokenService                                     │  │
│  │  - EmailService                                     │  │
│  │  - AuditService                                     │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │                                      │
│  ┌──────────────────▼──────────────────────────────────┐  │
│  │  Repository Layer (Data Access)                     │  │
│  │  - UserRepository                                   │  │
│  │  - RefreshTokenRepository                           │  │
│  │  - AuditEventRepository                             │  │
│  └──────────────────┬──────────────────────────────────┘  │
└────────────────────┼────────────────────────────────────────┘
                     │
        ┌────────────┼────────────┬──────────────┐
        ▼            ▼            ▼              ▼
   ┌─────────┐  ┌────────┐  ┌─────────┐   ┌──────────┐
   │ MySQL   │  │ Redis  │  │ Email   │   │ Audit    │
   │ (Users, │  │(Rate   │  │Service  │   │ Service  │
   │ Tokens) │  │Limiting)│  │(SMTP)   │   │(Logging) │
   └─────────┘  └────────┘  └─────────┘   └──────────┘
```

---

## 2. Core Layers

### 2.1 Presentation Layer (Angular)

**Responsibility:** User interface, form validation, route protection, token management.

#### Components

```
AuthModule
├── LoginComponent
│   ├── Template: login form
│   ├── Logic: form submission, error display
│   └── Services: AuthService, HTTP interception
│
├── RegisterComponent
│   ├── Template: registration form
│   ├── Logic: form validation, submission
│   └── Services: AuthService, validation
│
├── ForgotPasswordComponent
│   ├── Template: email input form
│   └── Services: AuthService
│
├── ResetPasswordComponent
│   ├── Template: new password form
│   └── Services: AuthService
│
├── VerifyEmailComponent
│   └── Services: AuthService
│
└── DashboardComponent (Protected)
    ├── Template: authenticated content
    └── Guard: AuthGuard
```

#### Key Services

- **AuthService** — Manages authentication state, API calls, token storage
- **TokenService** — Token extraction, validation, expiration checks
- **AuthGuard** — Route protection, redirect unauthenticated users

#### HTTP Interceptor

```typescript
// Adds Authorization header to all requests
// Handles 401 responses
// Triggers token refresh on expiration
// Redirects to login if session invalid
```

#### Routing

```typescript
const routes: Routes = [
  { path: 'login', component: LoginComponent },
  { path: 'register', component: RegisterComponent },
  { path: 'forgot-password', component: ForgotPasswordComponent },
  { path: 'reset-password', component: ResetPasswordComponent },
  { path: 'verify-email', component: VerifyEmailComponent },
  { 
    path: 'dashboard', 
    component: DashboardComponent,
    canActivate: [AuthGuard]
  }
];
```

### 2.2 Application Layer (Spring Boot)

**Responsibility:** Business logic, security decisions, token generation, password hashing, audit logging.

#### Sub-Layers

##### 2.2a Controller Layer

```
AuthController
├── POST /api/v1/auth/register
├── POST /api/v1/auth/login
├── POST /api/v1/auth/logout
├── POST /api/v1/auth/refresh
├── POST /api/v1/auth/forgot-password
├── POST /api/v1/auth/reset-password
└── GET  /api/v1/auth/verify-email
```

**Responsibilities:**
- Request validation (input constraints)
- Response mapping (DTOs)
- HTTP status codes
- Error handling delegation

##### 2.2b Service Layer

**Core Services:**

1. **AuthenticationService**
   - Validates credentials
   - Retrieves user from repository
   - Verifies password hash
   - Checks email verification status
   - Manages failed login attempts
   - Handles account lockout
   - Generates tokens
   - Triggers audit logging

2. **RegistrationService**
   - Validates registration input
   - Checks for duplicate emails
   - Encodes password
   - Creates user record
   - Generates verification token
   - Triggers email verification

3. **PasswordService**
   - Validates password policy
   - Generates reset tokens
   - Validates reset tokens
   - Updates password hash
   - Invalidates refresh tokens on password change

4. **TokenService**
   - Generates JWT access tokens
   - Generates refresh tokens
   - Validates tokens
   - Handles token expiration
   - Manages refresh token storage

5. **EmailService**
   - Sends verification emails
   - Sends password reset emails
   - Uses Freemarker templates

6. **AuditService**
   - Logs security events
   - Records user actions
   - Includes correlation IDs
   - Tracks IP addresses (if permitted)

##### 2.2c Security Components

```
Spring Security Configuration
├── SecurityConfig
│   ├── Disables CSRF (stateless API)
│   ├── Configures CORS
│   ├── Sets HTTP security headers
│   ├── Defines authentication manager
│   └── Configures JWT filter
│
├── JwtAuthenticationFilter
│   ├── Extracts JWT from request
│   ├── Validates token
│   ├── Populates SecurityContext
│   └── Handles exceptions
│
├── JwtTokenService
│   ├── Creates tokens
│   ├── Signs tokens
│   ├── Extracts claims
│   └── Validates signature
│
└── CustomUserDetailsService
    ├── Loads user by username
    └── Returns UserDetails
```

##### 2.2d Exception Handling

```
GlobalExceptionHandler
├── Catches all exceptions
├── Maps to standard error response
├── Includes correlation ID
├── Returns appropriate HTTP status
├── Logs for debugging
└── Prevents information leakage
```

#### Service Interactions (Example: Login Flow)

```
User Submits Credentials
           │
           ▼
AuthController.login()
           │
           ▼
AuthenticationService.authenticate()
           │
           ├─ UserRepository.findByEmail()
           │
           ├─ PasswordEncoder.matches()
           │
           ├─ Check email_verified
           │
           ├─ Reset failed_login_attempts
           │
           ├─ TokenService.generateAccessToken()
           │
           ├─ TokenService.generateRefreshToken()
           │
           ├─ RefreshTokenRepository.save()
           │
           ├─ AuditService.logLoginSuccess()
           │
           └─ Return LoginResponse
           
           (JWT tokens included in response)
```

### 2.3 Persistence Layer (Spring Data JPA)

**Responsibility:** Data access, database abstraction, transaction management.

#### Repositories

```
UserRepository
├── findByEmail(email: String)
├── existsByEmail(email: String)
└── Custom queries

RefreshTokenRepository
├── findByToken(token: String)
├── deleteByUser(user: User)
└── findByUserAndNotRevoked()

EmailVerificationTokenRepository
├── findByToken(token: String)
├── deleteExpiredTokens()

PasswordResetTokenRepository
├── findByToken(token: String)
├── deleteExpiredTokens()

AuditEventRepository
├── findByUser(user: User)
├── findByEventType(eventType: String)
└── Custom date range queries
```

#### Transactions

- **Registration:** `@Transactional` wraps user creation, token generation, and audit logging
- **Login:** `@Transactional` wraps credential verification and token generation
- **Password Reset:** `@Transactional` wraps password update and token invalidation
- **Logout:** `@Transactional` wraps token revocation

---

## 3. Data Flow Diagrams

### 3.1 Registration Flow

```
User
 │
 ├─ Enters: firstName, lastName, email, password, confirmPassword
 │
 ▼
Angular RegisterComponent
 │
 ├─ Client-side validation (required fields, password match)
 │
 ├─ POST /api/v1/auth/register
 │
 ▼
Spring Boot AuthController
 │
 ├─ Validates input (Bean Validation)
 │
 ▼
RegistrationService
 │
 ├─ findByEmail(email) [Repository]
 │     │
 │     └─ If exists: throw DuplicateEmailException
 │
 ├─ validatePasswordPolicy(password)
 │     │
 │     └─ If invalid: throw PasswordPolicyException
 │
 ├─ PasswordEncoder.encode(password)
 │
 ├─ Create User entity
 │     │
 │     └─ Set status: PENDING_VERIFICATION
 │
 ├─ userRepository.save(user) [Transaction]
 │
 ├─ TokenService.generateVerificationToken()
 │
 ├─ emailVerificationTokenRepository.save(token)
 │
 ├─ EmailService.sendVerificationEmail(email, token)
 │
 ├─ AuditService.logUserRegistered(userId)
 │
 ▼
Return 201 Created
{
  "userId": "USR-10001",
  "message": "Registration successful. Please verify your email."
}

User Receives Email
 │
 ├─ Clicks verification link
 │
 ▼
Browser navigates to
/verify-email?token=<verification-token>
 │
 ▼
Angular VerifyEmailComponent
 │
 ├─ Extracts token from query param
 │
 ├─ GET /api/v1/auth/verify-email?token=...
 │
 ▼
Spring Boot AuthController
 │
 ▼
RegistrationService.verifyEmail(token)
 │
 ├─ emailVerificationTokenRepository.findByToken(token)
 │
 ├─ Check token expiration
 │
 ├─ Get associated User
 │
 ├─ Update user.email_verified = TRUE
 │
 ├─ Update user.status = ACTIVE
 │
 ├─ userRepository.save(user)
 │
 ├─ emailVerificationTokenRepository.delete(token)
 │
 ├─ AuditService.logEmailVerified(userId)
 │
 ▼
Return 200 OK
{
  "message": "Email verified successfully. You can now log in."
}

Angular redirects to /login
```

### 3.2 Login Flow

```
User
 │
 ├─ Enters: email, password
 │
 ▼
Angular LoginComponent
 │
 ├─ Client-side validation (required fields, email format)
 │
 ├─ POST /api/v1/auth/login
 │     {
 │       "email": "user@example.com",
 │       "password": "Secure@123"
 │     }
 │
 ▼
Spring Boot AuthController
 │
 ├─ Validates request body
 │
 ▼
AuthenticationService.authenticate(email, password)
 │
 ├─ userRepository.findByEmail(email)
 │     │
 │     └─ If not found: 
 │        ├─ Record failed attempt (IP-based, not user)
 │        └─ Throw InvalidCredentialsException
 │
 ├─ Check user.status = ACTIVE
 │     │
 │     └─ If LOCKED:
 │        ├─ Check if lock expired
 │        │  └─ If not: throw AccountLockedException
 │        └─ If expired: reset failed_login_attempts, unlock
 │
 ├─ Check user.email_verified = TRUE
 │     │
 │     └─ If FALSE: throw EmailNotVerifiedException
 │
 ├─ PasswordEncoder.matches(password, user.password_hash)
 │     │
 │     └─ If no match:
 │        ├─ Increment user.failed_login_attempts
 │        ├─ If >= threshold (e.g., 5):
 │        │  └─ Set user.account_locked_until = now + duration
 │        ├─ userRepository.save(user)
 │        ├─ AuditService.logLoginFailed(email)
 │        └─ Throw InvalidCredentialsException
 │
 ├─ Reset failed_login_attempts = 0
 │
 ├─ TokenService.generateAccessToken(userId, roles)
 │     │
 │     └─ Create JWT payload {sub, roles, iat, exp}
 │        └─ Sign with private key
 │
 ├─ TokenService.generateRefreshToken(userId)
 │     │
 │     ├─ Generate cryptographically random token
 │     ├─ Hash the token (bcrypt)
 │     └─ Return plaintext for client storage
 │
 ├─ refreshTokenRepository.save(RefreshToken)
 │     │
 │     └─ Store: {userId, token_hash, expires_at, revoked_at}
 │
 ├─ userRepository.save(user) [Reset failed attempts]
 │
 ├─ AuditService.logLoginSuccess(userId, ipAddress)
 │
 ▼
Return 200 OK
{
  "accessToken": "<JWT>",
  "refreshToken": "<refresh-token>",
  "expiresIn": 3600,
  "user": {
    "userId": "USR-10001",
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com"
  }
}

Angular TokenService
 │
 ├─ Extracts token from response
 │
 ├─ Stores in memory (or HttpOnly cookie)
 │
 ├─ Sets token expiration timer
 │
 └─ Redirects to /dashboard

User navigates to /dashboard
 │
 ▼
AuthGuard.canActivate()
 │
 ├─ Checks if token exists in memory
 │
 └─ If valid: Allow navigation
    Else: Redirect to /login
```

### 3.3 Protected Request Flow

```
User clicks protected resource
 │
 ▼
Angular component calls API
 │
 │ GET /api/v1/user/profile
 │
 ▼
HTTP Interceptor
 │
 ├─ TokenService.getAccessToken()
 │
 ├─ Adds header: Authorization: Bearer <JWT>
 │
 ▼
Spring Boot JwtAuthenticationFilter
 │
 ├─ Extracts Bearer token from header
 │
 ├─ JwtTokenService.validateToken(token)
 │     │
 │     ├─ Check signature
 │     ├─ Check expiration (iat + exp)
 │     └─ Extract claims (sub, roles)
 │
 ├─ Create Authentication object
 │
 ├─ SecurityContext.setAuthentication(auth)
 │
 ▼
Controller/Service can access
 │
 ├─ SecurityContextHolder.getAuthentication()
 │
 └─ @AuthenticationPrincipal annotation

Response sent back to Angular
 │
 ▼
HTTP Interceptor
 │
 ├─ Check response status
 │
 ├─ If 401 (token expired):
 │  ├─ POST /api/v1/auth/refresh
 │  ├─ Get new accessToken
 │  ├─ Retry original request
 │  └─ Continue
 │
 └─ Else: Return response to component
```

### 3.4 Logout Flow

```
User clicks logout
 │
 ▼
Angular LogoutComponent
 │
 ├─ POST /api/v1/auth/logout
 │     {
 │       "refreshToken": "<refresh-token>"
 │     }
 │
 ▼
Spring Boot AuthController
 │
 ▼
AuthenticationService.logout(refreshToken)
 │
 ├─ refreshTokenRepository.findByToken(token)
 │     │
 │     └─ If not found: throw InvalidTokenException
 │
 ├─ Mark token as revoked: revoked_at = now
 │
 ├─ refreshTokenRepository.save(token)
 │
 ├─ AuditService.logLogout(userId)
 │
 ▼
Return 200 OK
{
  "message": "Logout successful"
}

Angular TokenService
 │
 ├─ Clear accessToken from storage
 │
 ├─ Clear refreshToken from storage
 │
 ├─ Clear authentication state
 │
 ▼
Redirect to /login
```

---

## 4. Cross-Cutting Concerns

### 4.1 Security

```
Security Stack
├── HTTPS/TLS
│   └── Nginx terminates TLS
│
├── Authentication
│   ├── JWT access tokens (short-lived: 1 hour)
│   ├── Refresh tokens (long-lived: 7 days)
│   └── Token validation on every protected request
│
├── Authorization
│   ├── Role-based access control (RBAC)
│   ├── Roles embedded in JWT
│   └── Checked on every endpoint
│
├── Password Security
│   ├── Argon2id hashing (Spring Security default)
│   ├── Never logged
│   ├── Strong policy enforcement
│   └── No plaintext storage
│
├── Account Protection
│   ├── Failed login tracking
│   ├── Account lockout (e.g., 5 failed attempts)
│   ├── Automatic unlock (e.g., 30 minutes)
│   └── Generic error messages (no email existence leaking)
│
├── Attack Prevention
│   ├── Rate limiting (e.g., 10 requests/minute/IP)
│   ├── CSRF protection (stateless API = not needed)
│   ├── XSS protection (Content-Security-Policy header)
│   ├── SQL injection prevention (Parameterized queries via JPA)
│   ├── Brute-force protection (Account lockout)
│   └── Credential stuffing prevention (Rate limiting + lockout)
│
├── HTTP Security Headers
│   ├── Strict-Transport-Security (HSTS)
│   ├── X-Frame-Options (DENY)
│   ├── X-Content-Type-Options (nosniff)
│   ├── Content-Security-Policy
│   └── Referrer-Policy
│
├── CORS Configuration
│   ├── Explicit allowed origins
│   ├── Allowed methods (GET, POST, PUT, DELETE)
│   ├── Allowed headers (Authorization, Content-Type)
│   └── Credentials flag (true only for same-origin)
│
└── Secrets Management
    ├── JWT signing key from environment
    ├── Database credentials from environment
    ├── Email credentials from environment
    └── Never hardcoded in source
```

### 4.2 Error Handling

```
Error Handling Chain
├── Controller Layer
│   └── @RequestBody validation → 400 Bad Request
│
├── Service Layer
│   ├── DuplicateEmailException → 409 Conflict
│   ├── PasswordPolicyException → 400 Bad Request
│   ├── InvalidCredentialsException → 401 Unauthorized
│   ├── EmailNotVerifiedException → 403 Forbidden
│   ├── AccountLockedException → 423 Locked
│   ├── InvalidTokenException → 401 Unauthorized
│   └── Other exceptions → 500 Internal Server Error
│
├── GlobalExceptionHandler
│   ├── Catches all exceptions
│   ├── Maps to standard error response
│   ├── Includes correlation ID
│   ├── Logs exception
│   └── Never leaks stack traces
│
└── Standard Error Response
    {
      "timestamp": "2026-08-13T10:00:00Z",
      "status": 401,
      "code": "AUTHENTICATION_FAILED",
      "message": "Invalid email or password.",
      "correlationId": "abc-123"
    }
```

### 4.3 Logging & Audit

```
Logging Strategy
├── Application Logs
│   ├── Structured JSON format
│   ├── Correlation IDs on every log entry
│   ├── Correlation ID passed through call chain
│   ├── Never log passwords, tokens, secrets
│   ├── Levels: DEBUG, INFO, WARN, ERROR
│   └── SLF4J + Logback configuration
│
├── Audit Logs
│   ├── Security events table
│   ├── Events: USER_REGISTERED, LOGIN_SUCCESS, LOGIN_FAILED,
│   │          ACCOUNT_LOCKED, LOGOUT, PASSWORD_RESET, etc.
│   ├── Fields: userId, eventType, timestamp, ipAddress, 
│   │          clientInfo, correlationId
│   └── Queried for compliance and investigation
│
├── Metrics
│   ├── Micrometer metrics
│   ├── Request latency (histograms)
│   ├── Error rates (counters)
│   ├── Token generation time
│   ├── Database query times
│   └── Cache hit/miss rates
│
└── Log Destinations
    ├── Console (development)
    ├── File (production)
    ├── ELK Stack (Elasticsearch, Logstash, Kibana)
    └── Prometheus / Grafana (metrics)
```

### 4.4 Configuration Management

```
Configuration Hierarchy
├── application.properties / application.yml (defaults)
│
├── application-dev.properties (development overrides)
├── application-staging.properties (staging overrides)
├── application-prod.properties (production overrides)
│
└── Environment Variables (highest priority)
    ├── DB_URL
    ├── DB_USERNAME
    ├── DB_PASSWORD
    ├── JWT_SECRET
    ├── JWT_ACCESS_EXPIRATION
    ├── JWT_REFRESH_EXPIRATION
    ├── EMAIL_HOST
    ├── EMAIL_PORT
    ├── EMAIL_USERNAME
    ├── EMAIL_PASSWORD
    ├── CORS_ALLOWED_ORIGINS
    └── ... (others)
```

---

## 5. Database Schema Architecture

### 5.1 Core Tables

```sql
-- Users table (core entity)
users
├── id (UUID, PK)
├── email (VARCHAR, UNIQUE)
├── password_hash (VARCHAR)
├── first_name (VARCHAR)
├── last_name (VARCHAR)
├── status (ENUM: ACTIVE, INACTIVE, LOCKED, PENDING_VERIFICATION)
├── email_verified (BOOLEAN)
├── failed_login_attempts (INT)
├── account_locked_until (TIMESTAMP, nullable)
├── created_at (TIMESTAMP)
└── updated_at (TIMESTAMP)

-- Refresh tokens (1:N relationship with users)
refresh_tokens
├── id (UUID, PK)
├── user_id (UUID, FK → users.id)
├── token_hash (VARCHAR, UNIQUE)
├── expires_at (TIMESTAMP)
├── revoked_at (TIMESTAMP, nullable)
└── created_at (TIMESTAMP)

-- Email verification tokens (1:N relationship with users)
email_verification_tokens
├── id (UUID, PK)
├── user_id (UUID, FK → users.id)
├── token_hash (VARCHAR, UNIQUE)
├── expires_at (TIMESTAMP)
└── created_at (TIMESTAMP)

-- Password reset tokens (1:N relationship with users)
password_reset_tokens
├── id (UUID, PK)
├── user_id (UUID, FK → users.id)
├── token_hash (VARCHAR, UNIQUE)
├── expires_at (TIMESTAMP)
└── created_at (TIMESTAMP)

-- Audit events (1:N relationship with users)
audit_events
├── id (UUID, PK)
├── user_id (UUID, FK → users.id, nullable for failed login events)
├── event_type (VARCHAR)
├── ip_address (VARCHAR, nullable)
├── user_agent (VARCHAR, nullable)
├── correlation_id (VARCHAR)
├── result (VARCHAR: SUCCESS, FAILURE)
├── created_at (TIMESTAMP)
└── details (JSON, nullable)
```

### 5.2 Indexes

```sql
-- For frequent queries
users: INDEX (email)
refresh_tokens: INDEX (user_id), INDEX (token_hash)
email_verification_tokens: INDEX (user_id), INDEX (token_hash)
password_reset_tokens: INDEX (user_id), INDEX (token_hash)
audit_events: INDEX (user_id), INDEX (event_type), INDEX (created_at)
```

---

## 6. Deployment Architecture

### 6.1 Local Development

```
Docker Compose (docker-compose.yml)
├── nginx (port 80/443)
├── spring-boot-app (port 8080)
├── mysql (port 3306)
├── redis (port 6379, optional)
└── mailhog (port 1025, 8025, optional for testing email)
```

### 6.2 Production Architecture

```
┌──────────────────────────────────────┐
│        Internet / Users              │
└──────────────┬───────────────────────┘
               │
               ▼
        ┌─────────────┐
        │ AWS Route53 │ (DNS)
        └──────┬──────┘
               │
               ▼
        ┌─────────────┐
        │ AWS ALB     │ (Load Balancer, TLS termination)
        └──────┬──────┘
               │
       ┌───────┴───────┬────────────┐
       │               │            │
       ▼               ▼            ▼
   ┌────────┐   ┌────────┐   ┌────────┐
   │ ECS    │   │ ECS    │   │ ECS    │
   │ Task   │   │ Task   │   │ Task   │
   │(App 1) │   │(App 2) │   │(App 3) │
   └────────┘   └────────┘   └────────┘
       │
       └─────────────┬──────────────────────┐
                     │                      │
                     ▼                      ▼
              ┌─────────────┐        ┌─────────────┐
              │ RDS MySQL   │        │ ElastiCache │
              │ (Primary)   │        │ (Redis)     │
              └─────────────┘        └─────────────┘

Other Services:
├── SNS / SES (Email)
├── CloudWatch (Logs, Metrics)
├── Secrets Manager (Secrets)
└── S3 (Backups, Artifacts)
```

---

## 7. Integration Points

### 7.1 External Services

```
Spring Boot Application
├── Email Service (SMTP or SES)
│   ├── Verify email link
│   ├── Password reset link
│   └── Notifications
│
├── Redis (Optional)
│   ├── Rate limiting
│   ├── Cache
│   └── Session storage
│
├── Secrets Manager (HashiCorp Vault, AWS Secrets Manager)
│   ├── JWT signing key
│   ├── Database credentials
│   └── Email credentials
│
├── Monitoring (Prometheus, Grafana)
│   ├── Metrics export
│   ├── Health checks
│   └── Alerts
│
└── Logging (ELK Stack)
    ├── Log aggregation
    ├── Search & analysis
    └── Alerting
```

---

## 8. API Contract (OpenAPI 3.0)

### Core Endpoints

```yaml
/api/v1/auth:
  /register:
    post:
      summary: Register a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterRequest'
      responses:
        201:
          description: User registered successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RegisterResponse'
        400:
          description: Invalid input
        409:
          description: Email already exists

  /login:
    post:
      summary: Authenticate user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LoginRequest'
      responses:
        200:
          description: Login successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LoginResponse'
        401:
          description: Invalid credentials
        423:
          description: Account locked

  /logout:
    post:
      summary: Logout user
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LogoutRequest'
      responses:
        200:
          description: Logout successful

  /refresh:
    post:
      summary: Refresh access token
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RefreshRequest'
      responses:
        200:
          description: New token issued
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RefreshResponse'
        401:
          description: Invalid refresh token

  /forgot-password:
    post:
      summary: Request password reset
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ForgotPasswordRequest'
      responses:
        200:
          description: Reset email sent (generic response)

  /reset-password:
    post:
      summary: Complete password reset
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ResetPasswordRequest'
      responses:
        200:
          description: Password reset successful

  /verify-email:
    get:
      summary: Verify email address
      parameters:
        - name: token
          in: query
          required: true
          schema:
            type: string
      responses:
        200:
          description: Email verified successfully
        401:
          description: Invalid or expired token
```

---

## 9. Testing Strategy

### 9.1 Test Pyramid

```
                        ▲
                       ╱ ╲
                      ╱   ╲  E2E Tests (5%)
                     ╱─────╲ - Full user flows
                    ╱       ╲ - Playwright
                   ╱─────────╲
                  ╱           ╲ Integration Tests (15%)
                 ╱─────────────╲ - API endpoints
                ╱               ╲ - Database
               ╱─────────────────╲ - Testcontainers
              ╱                   ╲ Unit Tests (80%)
             ╱─────────────────────╲ - Services
            ╱                       ╲ - Controllers
           ╱─────────────────────────╲ - Repositories
          ╱_________________________╲
```

### 9.2 Test Coverage Targets

- Unit tests: 80%+ code coverage
- Integration tests: Critical paths (login, registration, password reset)
- E2E tests: Happy path + error scenarios
- Security tests: SAST, SCA, DAST scans

---

## 10. Monitoring & Observability

### 10.1 Metrics to Track

```
Application Metrics
├── Request Metrics
│   ├── Request count (total, by endpoint, by status)
│   ├── Request latency (p50, p95, p99)
│   ├── Error rate (4xx, 5xx)
│   └── Request size
│
├── Authentication Metrics
│   ├── Login attempts (success, failure)
│   ├── Registration attempts
│   ├── Token refresh count
│   ├── Account lockouts
│   └── Password resets
│
├── Database Metrics
│   ├── Query latency
│   ├── Connection pool usage
│   ├── Slow queries
│   └── Query count
│
├── Cache Metrics (Redis)
│   ├── Hit/miss rates
│   ├── Memory usage
│   └── Key evictions
│
└── JVM Metrics
    ├── Heap usage
    ├── GC time
    ├── Thread count
    └── Class loading
```

### 10.2 Health Checks

```
/actuator/health
├── status: UP / DOWN / DEGRADED
├── components:
│   ├── db: MySQL connectivity
│   ├── redis: Redis connectivity (if configured)
│   ├── diskSpace: Disk availability
│   └── email: Email service connectivity
```

---

## 11. Security Design Decisions

### 11.1 Token Storage (Angular)

**Decision:** Store JWT in memory or HttpOnly secure cookie

| Option | Pros | Cons |
|---|---|---|
| **HttpOnly Cookie** | XSS-safe, automatic on every request | CSRF risk (mitigated by SameSite), no custom headers |
| **In-Memory** | Full control, custom headers possible | Vulnerable to XSS, cleared on page reload |
| **Hybrid** | Best of both | More complex |

**Recommended:** HttpOnly secure cookie with SameSite=Strict

### 11.2 Token Expiration

- **Access Token:** 1 hour (short-lived, minimal exposure)
- **Refresh Token:** 7 days (balance between security and UX)
- **Verification Token:** 24 hours (email verification window)
- **Reset Token:** 1 hour (password reset security)

### 11.3 Password Hashing

**Decision:** Argon2id via Spring Security

- Recommended algorithm: Argon2id
- Fallback: bcrypt
- Work factor configurable
- Never use MD5, SHA1, plain bcrypt

---

## 12. Scalability Considerations

### 12.1 Horizontal Scaling

```
Stateless Design
├── Spring Boot instances: Stateless (no session affinity required)
├── Database: Centralized (no replication needed for small scale)
├── Cache: Redis cluster for distributed rate limiting
├── Load Balancer: Distributes traffic across instances
└── Sessions: Token-based (not session storage)
```

### 12.2 Performance Optimization

- Database indexes on frequently queried fields
- Connection pooling (HikariCP)
- Query optimization (avoid N+1 queries)
- Caching with Redis
- HTTP caching headers
- CDN for static assets

---

## 13. Definition of Done

Architecture is complete and validated when:

- [ ] All layers (controller → service → repository) are defined
- [ ] Data flow diagrams show complete request/response cycles
- [ ] Database schema is documented with relationships
- [ ] API contract is specified in OpenAPI 3.0
- [ ] Security controls are specified for each concern
- [ ] Error handling strategy is defined
- [ ] Logging and audit strategy is specified
- [ ] Deployment architecture is documented
- [ ] Monitoring and observability strategy is defined
- [ ] Testing strategy covers unit, integration, E2E, and security
- [ ] Scalability considerations are addressed
