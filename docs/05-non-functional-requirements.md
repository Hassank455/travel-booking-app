# Non-Functional Requirements

## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Requirement Format](#2-requirement-format)
- [3. Performance](#3-performance)
- [4. Availability](#4-availability)
- [5. Reliability and Resilience](#5-reliability-and-resilience)
- [6. Scalability](#6-scalability)
- [7. Security and Privacy](#7-security-and-privacy)
- [8. Data Integrity](#8-data-integrity)
- [9. Maintainability and Extensibility](#9-maintainability-and-extensibility)
- [10. Testability](#10-testability)
- [11. Observability and Auditability](#11-observability-and-auditability)
- [12. Usability and Accessibility](#12-usability-and-accessibility)
- [13. Backup and Recovery](#13-backup-and-recovery)
- [14. MVP Quality Targets](#14-mvp-quality-targets)
- [15. Related Documents](#15-related-documents)
- [16. Summary](#16-summary)

---

## 1. Purpose

This document defines the main non-functional requirements of the travel booking platform.

Non-functional requirements describe how the platform must operate rather than which business features it provides.

They cover:

- Performance.
- Availability.
- Reliability.
- Scalability.
- Security.
- Data protection.
- Maintainability.
- Testability.
- Observability.
- Recovery.
- Usability.

Functional behavior is defined in [Functional Requirements](04-functional-requirements.md).

Detailed implementation decisions are documented in the relevant architecture and operational documents.

---

## 2. Requirement Format

Each requirement uses the following format:

```text
NFR-{CATEGORY}-{NUMBER}
```

Examples:

```text
NFR-PERF-001
NFR-SEC-001
NFR-REL-001
```

The word **shall** indicates a mandatory requirement.

The word **should** indicates an important target that may be refined during implementation or performance testing.

---

# 3. Performance

### NFR-PERF-001 — Response-Time Targets

The platform should target the following response times under normal operating conditions:

- Internal API operations: `95%` within `500 ms`.
- Cached search results: `95%` within `1 second`.
- Live flight or hotel searches: `95%` within `8 seconds`.
- Overall live-search timeout: no more than `12 seconds`.
- Booking-status retrieval: `95%` within `500 ms`.

Live search targets include external provider latency.

---

### NFR-PERF-002 — Bounded External Calls

Every external provider request shall use a defined timeout.

The platform shall not wait indefinitely for:

- Flight providers.
- Hotel providers.
- Payment gateways.
- Notification providers.

Detailed timeout behavior will be defined in [Search Architecture](08-search-architecture.md) and [Failure Scenarios](13-failure-scenarios.md).

---

### NFR-PERF-003 — Partial Search Results

The platform shall return successful results when one or more providers respond successfully, even if another provider fails or times out.

The system shall not delay the entire search indefinitely while waiting for an unhealthy provider.

---

### NFR-PERF-004 — Pagination and Result Limits

Endpoints that may return large datasets shall support pagination or bounded result sizes.

Examples include:

- Booking history.
- Administration booking lists.
- User lists.
- Payment records.
- Audit records.

---

### NFR-PERF-005 — Query Efficiency

Frequently executed database queries shall avoid:

- Unbounded table scans.
- N+1 query patterns.
- Loading unnecessary columns.
- Loading unnecessary relations.
- Unbounded sorting.
- Returning unnecessarily large payloads.

---

# 4. Availability

### NFR-AVAIL-001 — Availability Target

The MVP should target:

```text
99.5% monthly availability
```

This target applies to platform-controlled services and excludes approved maintenance periods and external provider outages where appropriate.

---

### NFR-AVAIL-002 — Provider Failure Isolation

The failure of one external provider shall not make the entire platform unavailable.

For example:

- A failed flight provider shall not stop other flight providers.
- A hotel-provider failure shall not stop flight search.
- A notification failure shall not invalidate a confirmed booking.
- A payment-gateway outage shall not prevent access to existing bookings.

---

### NFR-AVAIL-003 — Graceful Degradation

When a non-critical dependency is unavailable, the platform should continue with reduced functionality where safe.

Examples include:

- Returning results from available providers.
- Serving cached search data.
- Delaying notifications.
- Disabling an unhealthy provider temporarily.
- Moving unresolved operations to manual review.

---

### NFR-AVAIL-004 — Health Checks

The platform shall expose health and readiness checks suitable for:

- Deployment validation.
- Load-balancer routing.
- Dependency monitoring.
- Operational alerts.

Detailed health-check behavior will be defined in [Observability](15-observability.md).

---

# 5. Reliability and Resilience

### NFR-REL-001 — Duplicate Prevention

The platform shall prevent duplicate business operations caused by:

- Customer double-clicks.
- Client retries.
- Network retries.
- Duplicate webhooks.
- Background-job retries.
- Repeated provider callbacks.

Critical operations include:

- Booking creation.
- Payment processing.
- Cancellation.
- Refunds.

---

### NFR-REL-002 — Idempotent Processing

Critical write operations shall support idempotency where applicable.

The system shall safely handle the same request or event more than once without creating duplicate business effects.

---

### NFR-REL-003 — Controlled Retries

Retries shall use:

- A limited retry count.
- Appropriate delays.
- Operation-specific rules.
- Idempotency protection.
- Failure recording.

Unsafe booking or payment operations shall not be retried automatically without sufficient protection.

---

### NFR-REL-004 — Unknown External Results

A timeout shall not automatically be treated as proof that an external operation failed.

When the final result is unknown, the operation shall be:

- Recorded.
- Reconciled.
- Retried only when safe.
- Escalated to manual review when necessary.

Detailed recovery behavior will be documented in [Booking Orchestration](09-booking-orchestration.md) and [Failure Scenarios](13-failure-scenarios.md).

---

### NFR-REL-005 — Safe State Transitions

Booking, payment, cancellation, and refund states shall change only through valid business transitions.

These states shall remain separate.

For example, a successful payment shall not automatically mean that the provider confirmed the booking.

---

### NFR-REL-006 — Background Processing Safety

Restarting or repeating a background job shall not cause:

- Duplicate bookings.
- Duplicate charges.
- Duplicate refunds.
- Invalid repeated state transitions.
- Loss of workflow state.

---

# 6. Scalability

### NFR-SCALE-001 — Horizontal Scaling

Application and background-worker instances should support horizontal scaling without changes to core business logic.

Shared state shall not depend only on local process memory.

---

### NFR-SCALE-002 — Shared State

Persistent or shared state shall be stored in appropriate systems such as:

- PostgreSQL.
- Redis.
- A durable queue.
- Object storage.

---

### NFR-SCALE-003 — Bounded Concurrency

The platform shall control concurrency when calling external providers.

Traffic spikes shall not create unlimited provider requests or exhaust application resources.

---

### NFR-SCALE-004 — Provider Rate Limits

The platform shall respect provider-specific rate limits.

Rate limits shall be handled independently for each provider.

---

### NFR-SCALE-005 — Capacity Monitoring

The platform should collect enough metrics to estimate:

- Requests per second.
- Concurrent searches.
- Provider request volume.
- Booking creation rate.
- Queue depth.
- Database growth.
- Redis memory usage.

---

# 7. Security and Privacy

Detailed security controls will be documented in [Security](14-security.md).

### NFR-SEC-001 — Transport Security

All production communication over public or untrusted networks shall use encrypted transport such as HTTPS with TLS.

---

### NFR-SEC-002 — Password Protection

Passwords shall be stored only as strong salted hashes.

Plain-text or reversibly encrypted passwords shall not be stored.

---

### NFR-SEC-003 — Backend Authorization

Authorization shall be enforced by the backend for every protected operation.

Frontend visibility shall not be considered an authorization control.

---

### NFR-SEC-004 — Least Privilege

Customers, internal users, services, and workers shall receive only the permissions required for their responsibilities.

---

### NFR-SEC-005 — Secret Management

Secrets shall not be stored in source code.

This includes:

- Database credentials.
- Redis credentials.
- Provider keys.
- Payment-gateway secrets.
- Webhook secrets.
- Encryption keys.

---

### NFR-SEC-006 — Sensitive Logging

Logs shall not contain:

- Passwords.
- Authentication tokens.
- Raw payment-card data.
- Provider secrets.
- Private keys.
- Unmasked sensitive identity data.

---

### NFR-SEC-007 — Input Validation

All external input shall be validated before processing.

This includes input from:

- Customers.
- Internal users.
- Providers.
- Payment gateways.
- Webhooks.
- Background jobs.

---

### NFR-SEC-008 — Rate Limiting

Sensitive and public endpoints shall use appropriate rate limiting.

Examples include:

- Login.
- Registration.
- Password recovery.
- Search.
- Booking creation.
- Payment operations.
- Webhook endpoints.

---

### NFR-PRIV-001 — Data Minimization

The platform shall collect only the personal data required for:

- Account management.
- Search.
- Booking.
- Payment coordination.
- Customer support.
- Legal or operational obligations.

---

### NFR-PRIV-002 — Traveler Data Protection

Traveler and hotel-guest data shall be protected even when the person does not own a platform account.

Passport and identity-document data shall receive stronger access and storage controls.

---

### NFR-PRIV-003 — Payment Card Data

The platform shall not store raw payment-card data.

Payment data should be handled through secure, gateway-hosted, or tokenized mechanisms.

---

### NFR-PRIV-004 — Data Retention

The platform shall define retention rules for:

- User profiles.
- Traveler profiles.
- Bookings.
- Payments.
- Refunds.
- Audit records.
- Logs.
- Provider payloads.
- Temporary search data.

---

# 8. Data Integrity

### NFR-DATA-001 — Permanent Data Storage

Confirmed bookings, payments, refunds, and audit records shall be stored in durable persistent storage.

They shall not depend only on Redis or process memory.

---

### NFR-DATA-002 — Cache Independence

Loss of Redis data may remove temporary data such as:

- Search results.
- Offer mappings.
- Temporary locks.
- Temporary sessions.

It shall not remove permanent booking or financial records.

---

### NFR-DATA-003 — Transactional Integrity

Related local database operations shall use transactions when partial updates could create invalid business state.

---

### NFR-DATA-004 — Historical Snapshots

A confirmed booking shall retain the passenger, guest, offer, and pricing information used at booking time.

Later changes to a saved traveler profile shall not modify previous bookings.

---

### NFR-DATA-005 — Financial Traceability

Historical payment and refund records shall not be silently overwritten.

Corrections shall use new auditable operations or valid state transitions.

---

### NFR-DATA-006 — Database Constraints

Database constraints shall reinforce application rules where appropriate.

Examples include:

- Unique idempotency keys.
- Unique processed webhook identifiers.
- Unique external references where guaranteed.
- Required foreign-key relationships.

---

# 9. Maintainability and Extensibility

### NFR-MAINT-001 — Modular Architecture

The initial implementation shall use clear module boundaries within a Modular Monolith.

Expected modules may include:

- Identity.
- Search.
- Offers.
- Providers.
- Booking.
- Payments.
- Notifications.
- Administration.

---

### NFR-MAINT-002 — Separation of Concerns

Business logic shall remain separated from:

- HTTP transport.
- Database implementation.
- Provider SDKs.
- Payment-gateway SDKs.
- Notification delivery.
- Framework-specific behavior.

---

### NFR-MAINT-003 — Provider Isolation

Provider-specific models, errors, authentication, and API behavior shall remain inside provider adapters.

Core business logic shall not depend directly on provider response formats.

---

### NFR-MAINT-004 — Configuration Management

Environment-specific values shall be stored in configuration and shall not be hard-coded into application logic.

---

### NFR-MAINT-005 — Database Migrations

Database schema changes shall be managed through version-controlled migrations.

---

### NFR-MAINT-006 — Consistent Engineering Standards

The project shall use consistent:

- Naming conventions.
- Formatting.
- Linting.
- Error handling.
- Logging.
- Testing practices.

---

### NFR-EXT-001 — New Provider Integration

Adding a new flight or hotel provider should not require significant changes to core search or booking logic.

---

### NFR-EXT-002 — Capability Differences

The platform shall support providers with different capabilities.

For example, a provider may support:

- Search only.
- Search and booking.
- Cancellation.
- Status retrieval.
- Ticket retrieval.
- Voucher retrieval.

---

### NFR-EXT-003 — Payment and Notification Adapters

Payment-gateway and notification-provider behavior should be isolated to allow future replacement or expansion.

---

# 10. Testability

### NFR-TEST-001 — Unit Testing

Core business rules shall be testable without requiring live external providers.

---

### NFR-TEST-002 — Integration Testing

The platform shall include integration tests for critical infrastructure and adapters, including:

- PostgreSQL.
- Redis.
- Provider integrations.
- Payment webhooks.
- Background jobs.

---

### NFR-TEST-003 — End-to-End Testing

Critical user workflows shall have end-to-end coverage.

At minimum:

- Registration and login.
- Flight search.
- Hotel search.
- Flight booking.
- Hotel booking.
- Payment confirmation.
- Booking retrieval.
- Cancellation.
- Refund tracking.

---

### NFR-TEST-004 — Failure Testing

Tests shall cover important failure scenarios such as:

- Provider timeout.
- Provider error.
- Expired offer.
- Price change.
- Duplicate request.
- Payment failure.
- Duplicate webhook.
- Redis unavailability.
- Notification failure.
- Unknown booking result.

---

### NFR-TEST-005 — Load Testing

Core workflows shall be load-tested before production release.

Performance targets shall be validated using percentile-based measurements rather than averages only.

---

# 11. Observability and Auditability

Detailed observability requirements will be documented in [Observability](15-observability.md).

### NFR-OBS-001 — Structured Logging

Application logs shall use structured fields.

Logs should include relevant identifiers such as:

- Request identifier.
- Search identifier.
- Booking identifier.
- Payment identifier.
- Provider identifier.
- Background-job identifier.

---

### NFR-OBS-002 — Metrics

The platform shall collect metrics for:

- Request count.
- Error rate.
- Response latency.
- Provider success and failure rates.
- Provider timeouts.
- Queue depth.
- Worker failures.
- Database health.
- Redis health.

---

### NFR-OBS-003 — Alerts

Operational alerts shall be configured for critical failures, including:

- High application error rates.
- High provider timeout rates.
- Database unavailability.
- Payment webhook failures.
- Queue backlog.
- Stuck bookings.
- Reconciliation failures.

---

### NFR-AUDIT-001 — Sensitive Action Audit

Sensitive internal actions shall generate audit records.

Audit records shall identify:

- The actor.
- The action.
- The affected entity.
- The time.
- The result.
- The reason where required.

---

### NFR-AUDIT-002 — Audit Integrity

Audit records shall not be editable through ordinary application workflows.

Only authorized internal roles shall be able to access them.

---

# 12. Usability and Accessibility

### NFR-UX-001 — Clear Validation

Users shall receive clear validation messages for invalid:

- Search criteria.
- Passenger data.
- Guest data.
- Payment data.
- Booking requests.

---

### NFR-UX-002 — Clear Statuses

Customer-facing booking, payment, cancellation, and refund statuses shall use understandable language.

Internal workflow codes shall not be exposed directly.

---

### NFR-UX-003 — Clear Pricing

The platform shall clearly show:

- Base amount where available.
- Taxes.
- Fees.
- Total amount.
- Currency.

---

### NFR-UX-004 — Price Changes

If an offer price changes during revalidation, the platform shall show the new price and require explicit customer acceptance before continuing.

---

### NFR-UX-005 — Processing Feedback

The user interface shall clearly indicate when a booking is:

- Processing.
- Waiting for confirmation.
- Confirmed.
- Failed.
- Under review.

---

### NFR-UX-006 — Accessibility

Major customer workflows should support:

- Keyboard navigation on web.
- Screen-reader labels.
- Readable text and contrast.
- Validation messages associated with fields.
- Status communication that does not depend only on color.

---

# 13. Backup and Recovery

### NFR-RECOVERY-001 — Automated Backups

Production persistent data shall be backed up automatically.

Backups containing sensitive data shall be encrypted.

---

### NFR-RECOVERY-002 — Restore Testing

Backup restoration shall be tested periodically.

A successfully created backup shall not be considered reliable until restoration is verified.

---

### NFR-RECOVERY-003 — Recovery Point Objective

The MVP should target:

```text
Recovery Point Objective: 15 minutes or less
```

for critical persistent business data where supported by the selected infrastructure.

---

### NFR-RECOVERY-004 — Recovery Time Objective

The MVP should target:

```text
Recovery Time Objective: 4 hours or less
```

for restoration of the core platform after a major recoverable failure.

---

### NFR-RECOVERY-005 — Recovery Procedures

Recovery procedures shall be documented for:

- Database failure.
- Redis failure.
- Queue failure.
- Provider outage.
- Payment-gateway outage.
- Failed deployment.
- Secret compromise.

---

# 14. MVP Quality Targets

| Area | Initial MVP Target |
|---|---|
| Application availability | 99.5% monthly |
| Internal API latency | 95% within 500 ms |
| Cached search latency | 95% within 1 second |
| Live search latency | 95% within 8 seconds |
| Overall search timeout | Maximum 12 seconds |
| Booking-status retrieval | 95% within 500 ms |
| Duplicate bookings | Prevented through idempotency |
| Duplicate charges | Prevented through idempotency |
| Critical data RPO | 15 minutes or less |
| Core platform RTO | 4 hours or less |
| Transport encryption | Required |
| Raw payment-card storage | Prohibited |
| External-call timeout | Required |
| Structured logging | Required |
| Sensitive-action audit | Required |
| Provider failure isolation | Required |
| Critical workflow testing | Required |
| Pagination for large datasets | Required |

These targets are initial values.

They may be refined after:

- Provider selection.
- Infrastructure selection.
- Load testing.
- Production measurements.
- Security review.
- Legal review.
- Business agreement review.

---

# 15. Related Documents

- [Product Overview](01-product-overview.md)
- [Project Scope](02-scope.md)
- [User Roles](03-user-roles.md)
- [Functional Requirements](04-functional-requirements.md)
- [Business Rules](06-business-rules.md)
- [Domain Model](07-domain-model.md)
- [Search Architecture](08-search-architecture.md)
- [Booking Orchestration](09-booking-orchestration.md)
- [Caching Strategy](10-caching-strategy.md)
- [Database Design](11-database-design.md)
- [API Design](12-api-design.md)
- [Failure Scenarios](13-failure-scenarios.md)
- [Security](14-security.md)
- [Observability](15-observability.md)
- [Implementation Roadmap](16-implementation-roadmap.md)

---

# 16. Summary

The travel booking platform must operate safely and predictably under real-world conditions.

The platform must:

- Meet measurable response-time targets.
- Isolate external provider failures.
- Prevent duplicate bookings and charges.
- Handle unknown external-operation results safely.
- Scale application and worker instances.
- Protect customer and traveler data.
- Preserve permanent and auditable business records.
- Support adding providers without rewriting core logic.
- Provide sufficient testing, logging, metrics, and alerts.
- Support backup and recovery.
- Present clear statuses, prices, and errors to users.

Detailed implementation mechanisms belong in the specialized architecture, security, failure-handling, and observability documents.