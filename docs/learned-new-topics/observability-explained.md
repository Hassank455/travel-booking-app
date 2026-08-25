# Observability Explained

## What is Observability?

Observability is the ability to understand what is happening inside a system by collecting and analyzing operational data.

When something goes wrong in production, observability helps answer questions like:

- What happened?
- Why did it happen?
- Where did it happen?
- Which component is responsible?
- Is this an isolated issue or a system-wide problem?

Unlike debugging during development, observability is designed for understanding systems running in production.

---

# The Three Pillars of Observability

Observability is built on three core pillars.

## 1. Logs

Logs record individual events that happen inside the application.

They answer:

> **What happened?**

Examples:

- User logged in.
- Flight search started.
- Booking confirmed.
- Payment failed.
- Provider timeout.
- Notification sent.

Example:

```json
{
  "level": "info",
  "event": "booking_confirmed",
  "bookingId": 1001,
  "provider": "Duffel"
}
```

Logs are chronological records of important events.

---

## 2. Metrics

Metrics are numerical measurements collected over time.

They answer:

> **Is something becoming unhealthy?**

Examples:

- Search requests per minute.
- Booking success rate.
- Payment failure rate.
- Redis cache hit ratio.
- API latency.
- Worker retry count.

Examples:

```
booking_success_total

search_requests_total

provider_latency_ms

redis_cache_hit_ratio
```

Metrics are used to build dashboards and trigger alerts.

---

## 3. Traces

A trace represents the complete journey of a request through the system.

They answer:

> **Where did the request spend its time?**

Example:

```
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
Customer
```

A trace is made up of multiple **spans**.

Example:

```
HTTP Request
├── Validate Request      2 ms
├── Redis Lookup          6 ms
├── Duffel API          420 ms
├── Normalize            18 ms
└── Response              4 ms
```

This immediately shows where time was spent.

---

# Correlation ID

A Correlation ID uniquely identifies one business request.

Example:

```
POST /flight-bookings

Correlation ID

req_8f93ab12
```

Every log generated while processing that request includes the same Correlation ID.

Example:

```
Booking Started

↓

Payment Authorized

↓

Provider Booking

↓

Booking Confirmed

↓

Notification Sent
```

All of these logs share:

```
correlationId = req_8f93ab12
```

This makes troubleshooting significantly easier.

---

# How a Request Flows

Example:

Customer searches for flights.

```
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

During this flow:

Logs record important events.

Metrics measure performance.

Traces record the complete execution path.

---

# Booking Flow Example

```
Customer

↓

Booking Module

↓

Transaction Created

↓

Duffel Booking

↓

Database Updated

↓

Message Broker

↓

Background Worker

↓

Notification Sent
```

The original HTTP request may finish before the notification is sent.

The Background Worker should continue using the same Correlation ID so the entire business operation can still be traced.

---

# Structured Logging

Instead of writing:

```
Booking failed.
```

Prefer structured logs.

Example:

```json
{
  "event": "booking_provider_failure",
  "provider": "Duffel",
  "bookingId": 1001,
  "errorType": "TIMEOUT",
  "durationMs": 5000,
  "correlationId": "req_123"
}
```

Structured logs are machine-readable and easy to search.

---

# Metrics Examples

Examples of useful metrics:

API

- Requests per second
- Error rate
- Response time

Search

- Cache hit ratio
- Provider latency
- Search duration

Booking

- Booking success rate
- Booking failures

Payments

- Payment success rate
- Payment latency

Workers

- Queue size
- Retry count
- Failed jobs

Infrastructure

- PostgreSQL connections
- Redis memory
- CPU usage
- Memory usage

---

# Tracing Example

Imagine this trace:

```
POST /flight-bookings

├── Validate Request           4 ms
├── Payment Authorization    350 ms
├── Duffel Booking          5200 ms
├── Save Booking              20 ms
└── Publish Event              5 ms
```

Immediately you know that the provider call is the bottleneck.

---

# Health Checks

Typical health endpoints:

```
/health

/liveness

/readiness
```

### Liveness

Answers:

> Is the application process running?

### Readiness

Answers:

> Is the application ready to receive traffic?

For example:

```
Backend Running

Database Down

Redis Up
```

Result:

```
Liveness

OK

Readiness

FAILED
```

The application is alive but should not receive new requests.

---

# Dashboards

Dashboards visualize the health of the platform.

Typical dashboards include:

- API latency
- Error rate
- Booking success rate
- Search performance
- Redis cache hit ratio
- Worker queue size
- Provider response time

Dashboards help engineers understand system health at a glance.

---

# Alerts

Alerts notify engineers when something important happens.

Examples:

Critical

- Database unavailable
- Backend unavailable
- Redis unavailable

High

- Booking failure rate increases
- Provider timeout rate increases

Medium

- Slow searches
- Queue backlog

Alerts should always be actionable.

---

# Incident Investigation Example

A customer reports:

> My booking is stuck.

Investigation might look like:

Database

↓

Booking Status History

↓

Application Logs

↓

Trace

↓

Metrics

Example:

```
Booking

↓

PROCESSING

↓

Duffel Timeout

↓

Worker Retry

↓

Still Processing
```

Metrics reveal:

```
Duffel Timeout Rate = 27%
```

This indicates a provider-wide issue rather than a single failed booking.

---

# Security Considerations

Observability must never expose sensitive information.

Never log:

- Passwords
- JWT tokens
- API secrets
- CVV
- Card numbers
- Full passport numbers

Sensitive values should be masked.

Example:

```
Authorization

Bearer ********
```

```
Passport

******678
```

---

# Typical Observability Stack

Although technologies may change, a common implementation looks like:

```
Application

↓

Instrumentation

↓

Logs
Metrics
Traces

↓

Observability Platform

↓

Dashboards

↓

Alerts
```

Possible tools include:

- OpenTelemetry
- Prometheus
- Grafana
- Jaeger
- Tempo
- Loki
- Elasticsearch
- Pino
- Winston

The tools are implementation details.

The architecture and concepts remain the same.

---

# Key Takeaways

- Logs tell you **what happened**.
- Metrics tell you **whether something is wrong**.
- Traces tell you **where it happened**.
- Correlation IDs connect everything belonging to one business operation.
- Observability is essential for understanding production systems.
- Good observability dramatically reduces debugging and incident resolution time.