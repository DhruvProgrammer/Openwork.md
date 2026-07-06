```markdown
# SOA Architecture for an E‑Commerce Website

A concise, implementation‑oriented guide covering **frontend**, **backend**, and **security** only.

---

## 1. High‑Level Architecture

- **Frontend**: Web/Mobile apps call APIs.
- **API Gateway**: Single entry point for all clients.
- **Backend (SOA)**: Independent services for key e‑commerce capabilities.
- **Infrastructure**: CI/CD, monitoring, logging, container orchestration.

```text
Clients (Web, Mobile)
        │
        ▼
   API Gateway
        │
        ▼
 ┌───────────────┬───────────────┬───────────────┐
 │  Catalog Svc  │  Cart Svc     │  Order Svc    │ ...
 └───────────────┴───────────────┴───────────────┘
```

---

## 2. Backend: Core Services (SOA)

Design services around business capabilities; each is independent, with a clear contract (API).

### 2.1 Core Services

| Service                | Responsibilities                                              | Typical APIs (examples)                            |
|------------------------|---------------------------------------------------------------|----------------------------------------------------|
| **User/Auth Service**  | Registration, login, profiles, sessions                      | `POST /users`, `POST /auth/login`, `GET /me`       |
| **Catalog Service**    | Products, categories, prices, search                         | `GET /products`, `GET /products/{id}`              |
| **Inventory Service**  | Stock levels, reservations                                  | `GET /stock/{sku}`, `POST /stock/reserve`          |
| **Cart Service**       | Shopping cart per user, promo codes                          | `GET /cart`, `POST /cart/items`, `DELETE /cart`    |
| **Order Service**      | Order creation, status, history                              | `POST /orders`, `GET /orders/{id}`                 |
| **Payment Service**    | Payment intents, captures, refunds (integrates with PSPs)    | `POST /payments`, `POST /payments/{id}/refund`     |
| **Shipping Service**   | Shipping options, rates, labels, tracking                    | `POST /shipping/quote`, `GET /shipping/{id}`       |
| **Notification Service** | Emails/SMS/push (order, shipping, password reset)         | `POST /notifications/email`                        |
| **Recommendation Service (optional)** | Personalized recommendations                  | `GET /recommendations?userId=...`                  |

### 2.2 Communication Patterns

- **Synchronous (request/response)**: REST/HTTP or gRPC between frontend ↔ gateway ↔ services.
- **Asynchronous (events/queues)**:
  - Order created → send email, update analytics, notify warehouse.
  - Payment succeeded → finalize order, confirm inventory.

Use a message broker (e.g., Kafka, RabbitMQ) for async workflows.

---

## 3. Frontend Architecture

Frontend is a client of the SOA backend, consuming only **well‑designed APIs** and not touching internal databases.

### 3.1 Web Frontend (e.g., React/Vue/Next.js)

- **Key responsibilities**:
  - Render product lists, details, search results.
  - Manage cart UI (add/remove items, apply coupons).
  - Handle checkout flow and address/payment forms.
  - Display order history and tracking.

- **Patterns**:
  - Use a **BFF (Backend for Frontend)** if needed to aggregate multiple backend calls into one optimized response per page (e.g., product page gets catalog + inventory + recommendations in one call).
  - Use caching (CDN) for static assets and product listing pages.

### 3.2 Mobile Frontend

- Same API contracts as web.
- Consider:
  - Offline caching for catalog and cart.
  - JWT or OAuth tokens stored in secure storage (Keychain/Keystore).

---

## 4. API Gateway

API Gateway sits between clients and services.

**Responsibilities:**

- Route requests to appropriate services.
- Terminate TLS (HTTPS).
- Apply **authentication/authorization**.
- Rate limiting, quota, and throttling.
- Request/response transformation (if needed).
- Basic caching for common reads (e.g., product details).

---

## 5. Security

Focus on **auth**, **API protection**, **payment safety**, and **data protection**.

### 5.1 Authentication & Authorization

- **Protocols**:
  - OAuth 2.0 / OpenID Connect for user login and SSO.
  - JWT access tokens for APIs (short‑lived).
- **Flows**:
  - Web: Authorization Code Flow (with PKCE if SPA).
  - Mobile: Authorization Code Flow with PKCE.

**Best practices:**

- Store only hashed passwords (e.g., bcrypt/Argon2) in User Service.
- Use refresh tokens (httpOnly, secure cookies or secure storage).
- Role‑based access control:
  - Roles: `CUSTOMER`, `ADMIN`, possibly `SUPPORT`.
  - Check roles per endpoint in gateway and/or services.

### 5.2 API & Transport Security

- **HTTPS everywhere** (no HTTP for production).
- Validate and sanitize all inputs:
  - Prevent SQL injection, XSS, CSRF.
  - Use parameterized queries and ORM.
- Enable **CORS** policy to allow only trusted origins.
- Implement **rate limiting** and **IP blocking** for brute‑force and abuse.
- Use **mTLS or signed tokens** for service‑to‑service communication in internal network if threat model requires it.

### 5.3 Payment Security

- Use a reputable **Payment Service Provider (PSP)** (Stripe, Adyen, etc.).
- **Never store raw card numbers or CVV** on your servers:
  - Use PSP tokens or hosted payment pages.
- Aim for **PCI DSS compliance** appropriate to your integration model:
  - Let PSP handle as much card data as possible to reduce scope.
- Log only non‑sensitive payment identifiers (payment intent ID, status).

### 5.4 Data Protection & Privacy

- Encrypt sensitive data **at rest**:
  - Use database encryption, KMS for key management.
- Encrypt **in transit** (TLS).
- Restrict access to PII via fine‑grained permissions.
- Implement **data retention** and **deletion** policies:
  - Allow account deletion, export of user data (GDPR‑like).
- Audit logging:
  - Log access to sensitive data (profile changes, password reset, refunds).
  - Centralized logging and alerting.

### 5.5 Service Hardening

- Each service:
  - Has minimal permissions to its own database.
  - Exposes only necessary endpoints.
  - Implements input/output validation.
- Regular:
  - Security scans (SAST/DAST).
  - Dependency vulnerability checks.
  - Backup and disaster recovery tests.

---

## 6. Operational Essentials (Minimal)

- **Deployment**:
  - Containerize services (Docker).
  - Orchestrate with Kubernetes or cloud equivalents.
- **CI/CD**:
  - Per‑service pipelines with automated tests.
- **Monitoring & Observability**:
  - Metrics (latency, error rate, throughput).
  - Centralized logs per service.
  - Distributed tracing for cross‑service calls.

---

## 7. Implementation Checklist (Condensed)

- [ ] Define APIs for: User/Auth, Catalog, Cart, Order, Payment, Inventory, Shipping, Notification.
- [ ] Implement API Gateway for routing, TLS, auth, rate limiting.
- [ ] Build web frontend (and optionally mobile) consuming only gateway APIs.
- [ ] Use OAuth2/OIDC + JWT for user login and API access.
- [ ] Integrate with PSP; avoid storing card data; follow PCI DSS guidance.
- [ ] Encrypt data in transit and at rest; set up role‑based access and audit logs.
- [ ] Establish logging, monitoring, and CI/CD.

---

This markdown can be used directly as a “.md agent” spec to guide design, documentation, or automated tooling around an SOA‑based e‑commerce system.
```