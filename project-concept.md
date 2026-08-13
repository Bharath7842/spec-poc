# Project Concept — Login & Registration

## What This Project Is

A secure, production-ready Login & Registration module built as a full-stack web application. It covers the complete user identity lifecycle: account creation, email verification, authentication, session management, and password recovery.

This project serves as a reference implementation for an AI-SDLC workflow, demonstrating full traceability from requirements through design, implementation, testing, and release.

---

## The Problem It Solves

Most applications need a reliable, secure authentication layer before any real functionality can be built. This project delivers that foundation — a self-contained auth module that teams can adopt as-is or extend with features like MFA, SSO, and social login in future phases.

---

## Core Capabilities

| Capability | Description |
|---|---|
| Registration | New users sign up with name, email, and password |
| Email Verification | Accounts are verified via a single-use tokenized email link |
| Login | Authenticated sessions issued via JWT access + refresh tokens |
| Logout | Active tokens are invalidated on logout |
| Password Reset | Self-service reset flow via verified email |
| Account Lockout | Automatic temporary lockout after repeated failed attempts |
| Audit Trail | All security events are logged with correlation IDs |

---

## Technology Stack

| Layer | Choice |
|---|---|
| Frontend | Angular (TypeScript) |
| UI Library | Angular Material |
| Backend | Spring Boot (Java) |
| Security | Spring Security + JWT |
| Database | MySQL with Flyway migrations |
| Cache / Rate Limiting | Redis (optional) |
| Email | SMTP / Email Provider |
| Containers | Docker + Docker Compose |
| Edge Routing | Nginx / API Gateway |

---

## Architecture at a Glance

The system follows a clean, layered architecture:

```
Browser (Angular)
      |
      | HTTPS / REST / JSON
      v
Nginx / Load Balancer
      |
      v
Spring Boot API
      |
  +---+---+---+
  |       |   |
JWT    Redis  Email
  |
  v
Service Layer
  |
  v
Spring Data JPA
  |
  v
MySQL
```

Angular handles the UI, route protection, token management, and HTTP interception. Spring Boot owns all auth logic, password hashing, token generation, and audit logging. MySQL persists users, tokens, and events. Redis optionally handles rate limiting and temporary state.

---

## Key Design Decisions

- **JWT for authentication** — stateless access tokens keep backend instances horizontally scalable.
- **Argon2id / bcrypt for passwords** — industry-standard hashing; plaintext passwords are never stored or logged.
- **Refresh token rotation** — short-lived access tokens with a separate, revocable refresh token for session continuity.
- **Generic auth error messages** — errors never reveal whether an email account exists.
- **Flyway for schema management** — all schema changes are versioned and auditable.
- **OpenAPI / Swagger** — API contract is published and discoverable.
- **Centralized exception handling** — all errors return a consistent structured response with correlation IDs.

---

## Security Posture

The module is designed with defense-in-depth:

- HTTPS/TLS enforced at the edge
- Rate limiting on all auth endpoints
- Brute-force protection via account lockout
- Input validation at both client and server
- CORS explicitly configured (no wildcard origins in production)
- Secure HTTP headers
- Token expiration and revocation
- SAST (Semgrep / CodeQL), SCA (Trivy), and DAST (OWASP ZAP) in the CI/CD pipeline
- Secrets managed via environment variables / secret store — never hardcoded

---

## What's Out of Scope (Phase 1)

The following are intentionally deferred:

- Multi-factor authentication (MFA)
- Social login (Google, GitHub, etc.)
- SSO / SAML / OIDC
- Biometric authentication
- Multi-tenant / organization management

These are natural Phase 2 extensions once the core auth foundation is stable.

---

## API Surface

All endpoints are versioned under `/api/v1/auth/`:

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/register` | Create a new account |
| POST | `/login` | Authenticate and receive tokens |
| POST | `/logout` | Invalidate active session |
| POST | `/forgot-password` | Request a password reset email |
| POST | `/reset-password` | Complete password reset with token |
| GET  | `/verify-email` | Confirm email ownership |
| POST | `/refresh` | Obtain a new access token |

---

## Data Model Summary

The core data model revolves around the `users` table, with supporting tables for tokens and audit events:

```
users
  ├── email_verification_tokens (1:N)
  ├── refresh_tokens (1:N)
  ├── password_reset_tokens (1:N)
  └── audit_events (1:N)
```

---

## Non-Functional Targets

| Concern | Target |
|---|---|
| Login response time | < 2 seconds |
| Registration response time | < 3 seconds |
| Availability | 99.9% |
| Scalability | Horizontally scalable backend |
| Auditability | All security events logged with correlation IDs |
| Browser support | Chrome, Edge, Firefox, Safari (current) |

---

## CI/CD & Quality Gates

The pipeline runs on every push/PR:

1. Build (Angular + Spring Boot)
2. Unit tests
3. Integration tests
4. SAST scan
5. Dependency / container scan (Trivy)
6. Secret scan
7. Docker build + container scan
8. E2E tests (Playwright / Cypress)
9. DAST (OWASP ZAP)
10. Manual approval → Production

---

## Definition of Done

The feature is shippable when all registration, login, verification, and password reset flows are implemented and tested, Spring Security and JWT are fully configured, Flyway migrations are in place, all test tiers pass, SAST/SCA/DAST scans meet quality gates, API docs are published, and requirements-to-test traceability is complete.

---

## Open Questions

1. JWT storage strategy — HttpOnly cookie vs. in-memory/browser storage?
2. Access token lifetime — how short is short enough?
3. Refresh token rotation — enabled from day one?
4. Is Redis required for the initial production release, or optional?
5. MFA — target phase and approach?
6. Email provider — which vendor?
7. Account lockout threshold — how many failed attempts?
8. Multi-tenancy — is it on the roadmap at all?
