# 15. Security

## 1. Purpose

This document defines the main security principles and controls of the Travel Booking Platform.

The platform handles sensitive data such as:

- Customer accounts.
- Traveler and guest information.
- Travel documents.
- Booking information.
- Payment transactions.
- External provider credentials.

Security controls must protect this data while allowing the platform to communicate safely with external providers such as flight, hotel, payment, and notification providers.

---

## 2. Security Goals

The platform security design aims to provide:

- Confidentiality of sensitive customer data.
- Integrity of bookings and transactions.
- Strong authentication and authorization.
- Protection against common API attacks.
- Secure communication with external providers.
- Secure handling of credentials and secrets.
- Traceability of sensitive operations.
- Minimal exposure of personally identifiable information.

---

## 3. Security Principles

The platform follows these principles:

### Least Privilege

Users, administrators, services, and external integrations should receive only the permissions required to perform their responsibilities.

---

### Defense in Depth

Security should not depend on a single control.

Multiple protection layers may include:

```text
Authentication
      ↓
Authorization
      ↓
Input Validation
      ↓
Business Validation
      ↓
Database Constraints
      ↓
Audit Logging
```

---

### Never Trust Client Input

All data received from:

- Web clients.
- Mobile clients.
- External providers.
- Webhooks.

must be validated before being trusted.

---

### Secure by Default

Sensitive operations should be denied unless explicitly allowed.

---

### Minimize Sensitive Data

The platform should only collect and retain information required by the business.

Sensitive data should not be copied unnecessarily across:

- Database records.
- Logs.
- Cache.
- Events.
- Notifications.

---

## 4. Authentication

Authentication verifies the identity of the user accessing the platform.

The platform may support:

- Email and password.
- Social authentication.
- Other identity providers in the future.

After successful authentication, the backend issues an authenticated session or token used for subsequent API requests.

Conceptually:

```text
Customer
   ↓
Login
   ↓
Identity Verification
   ↓
Authenticated Session / Token
   ↓
Protected API
```

Passwords must never be stored in plain text.

Only a secure password hash should be persisted.

---

## 5. Authorization

Authorization determines what an authenticated user is allowed to do.

The platform uses role-based authorization.

Initial roles may include:

```text
CUSTOMER
ADMIN
SUPPORT
FINANCE
```

Examples:

```text
Customer
→ View their own bookings.

Customer
→ Cannot view another customer's booking.

Support Agent
→ May inspect bookings for support purposes.

Finance
→ May inspect transaction information.

Admin
→ May manage provider configuration.
```

Authorization must be enforced in the Backend Application.

Client-side restrictions are not considered security controls.

---

## 6. Resource Ownership

Authorization must verify both:

```text
Role
+
Resource Ownership
```

For example:

```text
GET /flight-bookings/1001
```

must not succeed only because the caller has the `CUSTOMER` role.

The system must also verify:

```text
booking.customer_id == authenticated_customer.id
```

This prevents insecure direct object reference attacks.

Administrative roles may bypass ownership rules only when explicitly authorized.

---

## 7. Sensitive Data Protection

The platform handles sensitive customer and traveler information such as:

- Email addresses.
- Phone numbers.
- Dates of birth.
- Passport numbers.
- Nationality.
- Travel documents.

Sensitive information should be protected both:

```text
In Transit
```

and:

```text
At Rest
```

All communication with clients and external providers must use encrypted transport such as HTTPS/TLS.

Highly sensitive fields may require additional encryption at the application or database level depending on regulatory and business requirements.

---

## 8. Travel Document Data

Traveler profiles and booking passenger snapshots may contain:

```text
passport number
document type
document expiry
nationality
date of birth
```

This information must be treated as sensitive PII.

The platform should:

- Limit access to authorized modules and roles.
- Avoid including travel-document data in logs.
- Avoid storing document information in Redis unless explicitly necessary.
- Avoid including full document values in monitoring events.
- Retain document information only as long as required.

For operational interfaces, document numbers should normally be masked.

Example:

```text
P12345678
```

may be displayed as:

```text
******678
```

when the complete number is unnecessary.

---

## 9. Payment Security

The platform should minimize its exposure to payment-card data.

The backend should not store:

```text
Full card number
CVV
Raw payment credentials
```

unless explicitly required and the platform is designed for the corresponding compliance scope.

Payment processing should preferably be delegated to a trusted payment provider.

The platform persists only necessary references such as:

```text
transaction.id
provider_transaction_reference
payment_method
amount
currency
status
```

Example:

```text
provider_transaction_reference = pi_123...
```

rather than storing actual payment credentials.

---

## 10. External Provider Credentials

The platform integrates with providers such as:

```text
Duffel
LiteAPI
Payment Provider
Notification Provider
```

Provider credentials may include:

- API keys.
- Client secrets.
- Signing secrets.
- Authentication tokens.

These secrets must not be stored directly inside:

```text
providers
```

or committed to the source-code repository.

Secrets should be stored using an appropriate secure secret-management mechanism and provided to the application at runtime.

Conceptually:

```text
Secret Manager / Secure Environment
            ↓
Backend Application
            ↓
Provider Integration
```

---

## 11. External Provider Communication

All communication with external providers must:

- Use encrypted connections.
- Apply connection and request timeouts.
- Validate provider responses.
- Avoid trusting provider data blindly.
- Prevent provider error messages from leaking internal information to customers.

Provider responses must pass through the Provider Integration Layer before entering the core domain.

Conceptually:

```text
External Provider
       ↓
Provider Adapter
       ↓
Validation
       ↓
Normalization
       ↓
Business Domain
```

---

## 12. Webhook Security

External systems may asynchronously notify the platform about:

- Payment status.
- Booking updates.
- Provider changes.

Webhook endpoints must not trust incoming requests based only on their URL.

When supported by the provider, incoming webhooks should be verified using mechanisms such as:

- Cryptographic signatures.
- Shared webhook secrets.
- Timestamps.
- Replay protection.

Conceptually:

```text
Webhook Request
       ↓
Signature Verification
       ↓
Replay Validation
       ↓
Payload Validation
       ↓
Process Event
```

Invalid webhook requests must be rejected before changing business state.

---

## 13. Input Validation

Every API request must be validated before reaching business logic.

Validation may include:

- Required fields.
- Data types.
- Length limits.
- Date ranges.
- Enumeration values.
- Email format.
- Passenger counts.
- Guest counts.
- Currency codes.
- Airport codes.

Example:

```text
departure_date < current_date
```

should be rejected before executing a provider search.

Input validation reduces both security risk and invalid provider requests.

---

## 14. Database Security

Database access should follow least privilege.

Application database credentials should only have the permissions required by the application.

Important business constraints should also be enforced at the database level where appropriate.

Examples include:

```text
UNIQUE(users.email)

UNIQUE(customers.user_id)

UNIQUE(user_roles.user_id, user_roles.role_id)

FOREIGN KEY relationships

NOT NULL constraints
```

Database constraints provide an additional protection layer even when application validation fails.

---

## 15. Redis Security

Redis is used for temporary data such as:

- Search cache.
- Idempotency records.
- Distributed locks.
- Temporary orchestration state.

Redis must not be treated as a secure persistent store for highly sensitive customer information.

The platform should avoid caching:

- Payment credentials.
- Passwords.
- Authentication secrets.
- Full travel documents.
- Confirmed booking records as authoritative data.

Access to Redis should be restricted to trusted application infrastructure.

---

## 16. API Protection

Public APIs should be protected against abuse.

Potential protections include:

- Authentication where required.
- Authorization.
- Request size limits.
- Rate limiting.
- Input validation.
- Idempotency protection.
- Abuse detection.

Search endpoints require particular attention because they can trigger expensive external provider requests.

Conceptually:

```text
Client
  ↓
Rate Limit
  ↓
Validation
  ↓
Search Cache
  ↓
External Providers
```

This protects both the platform and external provider quotas.

---

## 17. Idempotency Security

Sensitive operations such as booking creation should support idempotency.

For example:

```text
Customer clicks Book twice
```

must not result in:

```text
Two bookings
+
Two transactions
```

An idempotency key can identify the same logical operation across repeated requests.

Idempotency records may be stored temporarily in Redis.

Detailed booking behavior is documented in:

[Booking Orchestration](./09-booking-orchestration.md)

---

## 18. Logging and Sensitive Data

Security requirements directly affect observability.

The platform must never intentionally log:

- Passwords.
- Authentication tokens.
- API secrets.
- Full card information.
- CVV values.
- Full passport numbers.

Sensitive fields should be:

- Removed.
- Masked.
- Redacted.

before being written to logs.

Example:

```text
Authorization: Bearer ********
Passport: ******678
```

The detailed logging policy will be documented in the Observability document.

---

## 19. Audit Logging

Sensitive business and administrative operations should be auditable.

Examples include:

```text
BOOKING_CANCELLED
TRANSACTION_STATUS_CHANGED
CUSTOMER_SUSPENDED
PROVIDER_DISABLED
```

Audit logs should identify:

```text
Who
What
When
Which entity
Relevant reason
```

The persistence design is documented in:

[Database Design](./12-database-design.md)

---

## 20. Error Handling

External API responses must not expose sensitive internal details.

For example, the client should not receive:

```text
Database connection string
Provider credentials
Stack traces
Internal SQL errors
```

Instead, the platform returns a controlled error response while preserving technical details internally for investigation.

Example:

```text
Provider Error
      ↓
Internal Log
      ↓
Normalized Application Error
      ↓
Customer Response
```

---

## 21. Dependency Security

Application dependencies should be:

- Maintained.
- Updated regularly.
- Monitored for known vulnerabilities.
- Removed when no longer required.

Security scanning may later be integrated into CI/CD.

---

## 22. Security Boundaries

The main trust boundaries are:

```text
Customer Applications
        ↓
Backend Application
        ↓
PostgreSQL / Redis
        ↓
External Providers
```

Every transition between trust boundaries requires explicit validation and authorization where applicable.

The backend remains responsible for enforcing business security regardless of client behavior.

---

## 23. Architectural Decisions

The platform currently adopts the following security decisions:

- Authentication identity is separated from customer business data.
- Authorization is enforced by the backend.
- Resource ownership is validated in addition to roles.
- Sensitive customer and traveler information is minimized.
- Passwords are stored only as secure hashes.
- Payment credentials are delegated to payment providers where possible.
- Provider credentials are managed outside the core database.
- External provider responses are validated before entering the domain.
- Webhooks require authenticity verification when supported.
- Booking creation is protected against duplicate submission.
- Redis is not used as an authoritative store for sensitive transactional data.
- Sensitive information is redacted from logs.
- Important administrative and transactional actions are auditable.

---

## 24. Related Documents

- [Backend Component Diagram](./03-backend-component-diagram.md)
- [Domain Model](./07-domain-model.md)
- [Search Architecture](./08-search-architecture.md)
- [Booking Orchestration](./09-booking-orchestration.md)
- [Caching Strategy](./10-caching-strategy.md)
- [Database Design](./12-database-design.md)
- `14-observability.md`
- `15-failure-scenarios.md`