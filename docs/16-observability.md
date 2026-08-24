# 16. Observability

## 1. Purpose

This document defines the observability strategy for the Travel Booking Platform.

Observability enables the platform to understand the internal state of the system by collecting operational information from applications, infrastructure, background workers, databases, caches, and external providers.

The primary objective is to enable engineers to quickly answer questions such as:

- What happened?
- Why did it happen?
- Where did it happen?
- Is the system healthy?
- Which component is responsible?

This document focuses on architectural principles rather than implementation technologies.

The following diagram illustrates how observability is implemented across the platform, from the client request through the modular monolith, background workers, data stores, and external providers, and how every component contributes logs, metrics, and traces.

**Diagram**

[Observability Architecture](./diagrams/observability/observability-architecture.mmd)

---

## 2. Observability Goals

The platform observability strategy aims to provide:

- Early detection of failures.
- Faster incident investigation.
- Visibility into customer requests.
- Visibility into background processing.
- Performance monitoring.
- External provider monitoring.
- Capacity planning.
- Operational insight without exposing sensitive information.

---

## 3. Observability Pillars

The platform follows the three fundamental pillars of observability.

### Logs

Logs record discrete events that occurred inside the platform.

They answer:

> What happened?

Examples include:

- Customer login.
- Flight search request.
- Booking confirmation.
- Provider timeout.
- Payment callback.
- Worker retry.

---

### Metrics

Metrics measure numerical system behavior over time.

They answer:

> Is something becoming unhealthy?

Examples include:

- Search requests per minute.
- Booking success rate.
- Provider latency.
- Redis cache hit ratio.
- Worker queue length.

---

### Traces

Traces represent the complete journey of a request across the platform.

They answer:

> Where did the request spend its time?

Examples include:

```text
Customer
      ↓
Backend
      ↓
Search Module
      ↓
Redis
      ↓
Duffel
      ↓
Normalize Results
      ↓
Response
```

Traces are particularly valuable for requests that interact with multiple internal modules and external providers.

---

## 4. Correlation IDs

Every incoming request should receive a unique Correlation ID.

The Correlation ID should accompany the request throughout its entire lifecycle.

Conceptually:

```text
Client Request
      ↓
Correlation ID Generated
      ↓
Backend
      ↓
Redis
      ↓
Database
      ↓
Background Worker
      ↓
Provider
      ↓
Response
```

This enables engineers to reconstruct the complete request path using a single identifier.

Background jobs should inherit or create an appropriate correlation identifier when continuing asynchronous work.

---

## 5. Logging Strategy

Application logs should describe meaningful business and technical events.

Examples include:

- Authentication.
- Flight search.
- Hotel search.
- Booking creation.
- Booking cancellation.
- Payment processing.
- Notification delivery.
- Worker execution.
- Provider communication.

Logs should provide sufficient operational context without exposing sensitive information.

The platform should avoid logging:

- Passwords.
- Authentication tokens.
- API secrets.
- Full passport numbers.
- Payment credentials.

Sensitive information should be masked or removed before being written to logs.

Detailed security requirements are documented in:

`15-security.md`

---

## 6. Metrics Strategy

Metrics provide a quantitative view of platform health.

Important platform metrics include:

### API Metrics

- Request rate.
- Request latency.
- Error rate.
- Success rate.

---

### Search Metrics

- Flight searches.
- Hotel searches.
- Average provider response time.
- Cache hit ratio.
- Cache miss ratio.

---

### Booking Metrics

- Booking requests.
- Successful bookings.
- Failed bookings.
- Booking confirmation time.

---

### Payment Metrics

- Payment success rate.
- Payment failure rate.
- Payment processing latency.

---

### Worker Metrics

- Queue length.
- Processing time.
- Retry count.
- Failed jobs.

---

### Infrastructure Metrics

- PostgreSQL connections.
- Redis memory usage.
- Redis hit ratio.
- Broker queue depth.
- CPU utilization.
- Memory utilization.

These metrics help identify performance degradation before customers experience failures.

---

## 7. Distributed Tracing

A single customer request may travel through multiple architectural components.

For example:

```text
Customer
      ↓
Backend Application
      ↓
Search Module
      ↓
Redis
      ↓
Duffel
      ↓
LiteAPI
      ↓
Normalization
      ↓
Customer
```

Each step should contribute trace information that enables engineers to identify latency bottlenecks.

Similarly, booking traces may include:

```text
Customer
      ↓
Booking Module
      ↓
Payment Provider
      ↓
Booking Persistence
      ↓
Message Broker
      ↓
Notification Worker
```

Tracing should include both synchronous and asynchronous processing.

---

## 8. Health Checks

Every deployable platform component should expose a health status.

Health checks may include:

### Backend

- Running.
- Database connectivity.
- Redis connectivity.
- Broker connectivity.

---

### PostgreSQL

- Accepting connections.
- Read/write availability.

---

### Redis

- Reachable.
- Memory availability.

---

### Message Broker

- Queue availability.
- Consumer availability.

---

### Background Workers

- Running.
- Processing jobs.
- Queue connectivity.

---

### External Providers

Health checks should verify that external integrations are reachable without performing unnecessary business operations.

---

## 9. Monitoring

The platform should continuously monitor:

- Customer-facing APIs.
- Search performance.
- Booking performance.
- Payment processing.
- Background workers.
- Redis.
- PostgreSQL.
- Message broker.
- External providers.

Monitoring should identify abnormal behavior before customers report issues.

---

## 10. Alerting

Important operational events should generate alerts.

Examples include:

### Critical

- Backend unavailable.
- Database unavailable.
- Redis unavailable.
- Message broker unavailable.

---

### High

- Booking failure rate increases.
- Payment failures increase.
- External provider unavailable.

---

### Medium

- Slow search responses.
- Queue backlog.
- Increased retry count.

Alerts should focus on actionable operational issues rather than isolated technical events.

---

## 11. Operational Visibility

Observability should allow engineers to reconstruct important business operations.

For example:

```text
Booking Created
        ↓
Payment Authorized
        ↓
Provider Booking
        ↓
Booking Confirmed
        ↓
Notification Sent
```

Similarly:

```text
Flight Search
        ↓
Cache Miss
        ↓
Duffel Request
        ↓
LiteAPI Request
        ↓
Aggregation
        ↓
Customer Response
```

Operational visibility combines:

- Logs.
- Metrics.
- Traces.
- Audit logs.
- Status history.

Together they provide a complete operational picture.

---

## 12. Background Worker Observability

Background workers should expose visibility into:

- Job execution.
- Retry attempts.
- Processing duration.
- Failure reasons.
- Queue delays.

Worker failures should be observable independently from customer-facing requests.

---

## 13. External Provider Observability

Each provider should be monitored independently.

Examples include:

- Duffel latency.
- LiteAPI latency.
- Payment provider latency.
- Provider error rate.
- Timeout frequency.
- Retry frequency.

Provider metrics help distinguish platform failures from provider failures.

---

## 14. Data Store Observability

The platform should monitor both persistent and temporary storage.

### PostgreSQL

Monitor:

- Connection usage.
- Slow queries.
- Transaction latency.

---

### Redis

Monitor:

- Cache hit ratio.
- Cache misses.
- Memory utilization.
- Key expiration rate.

---

### Message Broker

Monitor:

- Queue depth.
- Consumer lag.
- Failed jobs.
- Dead-letter queues.

---

## 15. Security Considerations

Observability must comply with the platform security policy.

Logs, metrics, and traces must never expose:

- Passwords.
- Authentication tokens.
- Provider secrets.
- Full travel documents.
- Payment credentials.

Sensitive values should be masked before leaving the application.

Detailed requirements are documented in:

`15-security.md`

---

## 16. Architectural Decisions

The platform currently adopts the following observability decisions:

- Logs, metrics, and traces are complementary.
- Every request receives a Correlation ID.
- Background jobs remain observable.
- External providers are monitored independently.
- Infrastructure health is continuously monitored.
- Business operations remain traceable across asynchronous boundaries.
- Sensitive information is excluded from observability data.
- Observability remains independent from any specific monitoring technology.

---

## Related Diagrams

- [Observability Architecture](./diagrams/observability/observability-architecture.mmd)

---

## 17. Related Documents

- `03-backend-component-diagram.md`
- `08-search-architecture.md`
- `09-booking-orchestration.md`
- `10-caching-strategy.md`
- `12-database-design.md`
- `15-security.md`
- `15-failure-scenarios.md`