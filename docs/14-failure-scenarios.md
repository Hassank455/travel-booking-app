# 14. Failure Scenarios

## 1. Purpose

This document defines the primary failure scenarios that may occur across the Travel Booking Platform and describes the expected system behavior when they happen.

The goal is to ensure that failures are:

- Contained.
- Observable.
- Recoverable where possible.
- Prevented from producing inconsistent booking or transaction states.

This document focuses on architectural behavior rather than provider-specific error codes or implementation details.

---

## 2. Failure Handling Principles

The platform follows these principles when handling failures.

### Failures Should Be Isolated

A failure in one provider or infrastructure component should not unnecessarily affect unrelated operations.

For example:

```text
Flight Provider A fails
        ↓
Provider B still succeeds
        ↓
Partial search results may still be returned
```

---

### Business State Must Remain Consistent

A technical failure must not leave bookings or transactions in an unknown or misleading state.

For example:

```text
Payment succeeded
        ↓
Provider booking timed out
        ↓
Booking must not be marked CONFIRMED
```

The platform should use an explicit intermediate state such as:

```text
PROCESSING
```

or:

```text
MANUAL_REVIEW
```

until the final provider state is known.

---

### Retry Only Safe Operations

Retries must only be used when the operation is:

- Idempotent.
- Protected by an idempotency key.
- Known to be safe to repeat.

Blindly retrying a booking or payment operation may create duplicates.

---

### Prefer Recovery Over Immediate Failure

Temporary infrastructure failures should degrade the system gracefully where possible.

Examples:

- Redis unavailable → continue without cache.
- Notification provider unavailable → retry asynchronously.
- One search provider unavailable → return partial results.

---

### Every Important Failure Must Be Observable

Failures should produce enough information for investigation using:

- Logs.
- Metrics.
- Traces.
- Booking status history.
- Audit logs where appropriate.

Detailed observability requirements are documented in:

`14-observability.md`

---

# 3. Search Failure Scenarios

## 3.1 One Search Provider Fails

Example:

```text
Duffel     → Success
Provider B → Timeout
Provider C → Success
```

Expected behavior:

- The failed provider should be isolated.
- Successful provider responses should still be processed.
- Search may return partial results.
- The failed provider should be recorded in observability data.
- The customer request should not fail unless no usable results remain.

Conceptually:

```text
Search
  ↓
Multiple Providers
  ↓
Some Succeed
Some Fail
  ↓
Aggregate Successful Results
  ↓
Return Partial Results
```

---

## 3.2 All Search Providers Fail

If every provider fails or times out:

```text
Provider A → Failed
Provider B → Failed
Provider C → Failed
```

Expected behavior:

- The search request fails gracefully.
- No stale result should be presented as guaranteed fresh data unless an explicit stale-data strategy exists.
- A normalized platform error should be returned.
- Technical provider errors should remain internal.

---

## 3.3 Provider Timeout

Each provider request should use a bounded timeout.

Expected behavior:

- Provider execution is stopped after the configured timeout.
- Other providers continue independently.
- Timeout metrics are recorded per provider.
- The overall search request should not wait indefinitely.

---

## 3.4 Invalid Provider Response

If a provider returns malformed or unexpected data:

Expected behavior:

- The response should fail validation.
- Invalid offers should not enter the normalized domain.
- Other provider results should continue processing.
- The issue should be logged with provider context.

---

## 3.5 Search Cache Failure

If Redis becomes unavailable:

```text
Search Request
      ↓
Redis unavailable
      ↓
Call Providers directly
      ↓
Return fresh results
```

Expected behavior:

- Search remains functional.
- Performance may degrade.
- Provider traffic may increase.
- Cache failure must not become a complete search outage.

---

# 4. Booking Failure Scenarios

## 4.1 Offer Revalidation Fails

A selected offer may no longer be valid because:

- Price changed.
- Availability disappeared.
- Offer expired.
- Provider rejected the offer.

Expected behavior:

- Booking must not continue using the old offer.
- The customer should receive updated commercial information when possible.
- No confirmed booking should be created.

---

## 4.2 Booking Provider Times Out

The most dangerous scenario occurs when:

```text
Booking Request Sent
        ↓
Provider Timeout
```

because the platform may not know whether the provider actually created the booking.

Expected behavior:

- Do not blindly retry immediately.
- Keep the local booking in an intermediate state such as:

```text
PROCESSING
```

- Trigger reconciliation using the provider reference or request identifier.
- Query the provider to determine whether the booking exists.
- Move the booking to the correct final state only after reconciliation.

---

## 4.3 Provider Booking Fails Before Confirmation

If the provider explicitly rejects the booking:

Expected behavior:

- Booking status becomes `FAILED`.
- Transaction handling follows the payment state.
- The customer receives a controlled failure response.
- Failure reason is recorded for operational investigation.

---

## 4.4 Duplicate Booking Request

Example:

```text
Customer clicks Book twice
```

or the client retries because of a network timeout.

Expected behavior:

- The same logical request must not create duplicate bookings.
- An idempotency key should identify repeated booking attempts.
- The existing operation result should be returned when appropriate.

Detailed idempotency behavior is documented in:

`09-booking-orchestration.md`

---

# 5. Transaction Failure Scenarios

## 5.1 Transaction Fails Before Booking

Example:

```text
Offer Valid
   ↓
Transaction Failed
```

Expected behavior:

- Booking must not be confirmed.
- Booking may remain `PENDING`, become `FAILED`, or expire depending on workflow.
- Provider booking should not proceed if payment is required first.

---

## 5.2 Transaction Succeeds but Booking Fails

This is one of the most important failure cases.

```text
Transaction
SUCCEEDED
      ↓
Provider Booking
FAILED
```

Expected behavior:

- Booking must not be marked `CONFIRMED`.
- The transaction remains financially successful unless compensation is performed.
- The operation enters a compensation or reconciliation flow.
- The booking may enter:

```text
MANUAL_REVIEW
```

if automatic recovery cannot safely resolve the state.

Because refund handling is currently outside the initial product scope, this scenario must still be operationally visible and handled according to the selected payment/provider business policy.

---

## 5.3 Payment Provider Timeout

A timeout does not necessarily mean payment failed.

Expected behavior:

- Do not create another payment blindly.
- Query or reconcile the payment using `provider_transaction_reference`.
- Keep the local transaction in `PENDING` until the external state is known.

---

## 5.4 Duplicate Payment Attempt

Repeated requests must not create duplicate successful charges.

Expected behavior:

- Use idempotency protection.
- Reuse the existing transaction when the same logical payment request is repeated.

---

# 6. Database Failure Scenarios

## 6.1 Database Unavailable

If PostgreSQL becomes unavailable:

Expected behavior:

- Transactional operations such as booking and payment persistence should fail safely.
- The application should not claim success without durable persistence.
- Readiness health checks should fail.
- Alerts should be generated.

---

## 6.2 Database Write Fails After Provider Success

Example:

```text
Provider Booking Confirmed
        ↓
Database Write Failed
```

This is a critical reconciliation scenario.

Expected behavior:

- Do not assume the provider booking can simply be recreated.
- Preserve provider references in available operational context.
- Trigger reconciliation.
- Record the incident for manual review if necessary.

This scenario is one reason provider references and idempotency are important.

---

# 7. Message Broker and Worker Failures

## 7.1 Worker Fails During Processing

A worker may crash while processing:

- Notification delivery.
- Reconciliation.
- Retry tasks.

Expected behavior:

- Work should be retried according to configured policy.
- Jobs should not disappear silently.
- Retry count must be observable.

---

## 7.2 Job Repeated

Message brokers may deliver the same job more than once.

Therefore workers should be designed as idempotently as possible.

Example:

```text
booking.confirmed
```

received twice must not result in two irreversible side effects.

---

## 7.3 Poison Message

A permanently failing message should not retry forever.

Expected behavior:

- Retry a bounded number of times.
- Move the message to a dead-letter or failed-job mechanism.
- Generate operational visibility.
- Allow manual inspection or replay when appropriate.

---

# 8. Notification Failures

Notification delivery is not part of booking correctness.

For example:

```text
Booking = CONFIRMED
Notification = FAILED
```

is valid.

Expected behavior:

- Booking remains confirmed.
- Notification is retried asynchronously.
- Notification failure is recorded.
- The customer-facing booking operation should not be rolled back.

---

# 9. Provider Outage

If Duffel, LiteAPI, or another provider experiences an outage:

Expected behavior depends on the operation.

### Search

- Exclude unavailable provider.
- Return partial results when possible.

### Booking

- Do not automatically switch an already selected provider offer to another provider.
- Keep uncertain operations in a recoverable state.
- Reconcile before retrying irreversible provider actions.

### Monitoring

Provider availability, latency, and errors should be tracked independently.

---

# 10. Failure State Classification

The platform should distinguish between:

### Recoverable Failure

Examples:

- Temporary provider timeout.
- Redis outage.
- Notification failure.

Possible response:

```text
Retry / Fallback / Reconcile
```

---

### Business Failure

Examples:

- Offer expired.
- No availability.
- Payment rejected.

Possible response:

```text
Return controlled business error
```

---

### Unknown Outcome

Examples:

- Booking provider timed out after receiving the request.
- Payment provider timed out during processing.

Possible response:

```text
PROCESSING
        ↓
Reconciliation
```

This category must never be treated automatically as success or failure.

---

# 11. Recovery and Compensation

Failure handling may use:

- Retry.
- Fallback.
- Reconciliation.
- Idempotency.
- Compensation.
- Manual review.

These mechanisms serve different purposes.

### Retry

Repeat a safe temporary operation.

### Fallback

Use an alternative path when correctness allows it.

### Reconciliation

Query an external system to determine the actual final state.

### Compensation

Apply a business action that counteracts an earlier successful step.

### Manual Review

Used when the system cannot safely determine or repair the final state automatically.

---

# 12. Observability Requirements

Every important failure should include operational context such as:

- Correlation ID.
- Booking ID.
- Transaction ID when applicable.
- Provider.
- Operation.
- Failure category.
- Retry count.
- Duration.

Sensitive information must follow the rules defined in:

`13-security.md`

---

# 13. Architectural Decisions

The platform currently adopts the following failure-handling decisions:

- Provider failures are isolated whenever possible.
- Search supports partial provider success.
- Redis failures degrade performance rather than correctness.
- Unknown provider outcomes require reconciliation rather than blind retry.
- Booking and transaction statuses remain independent.
- Duplicate booking and payment requests require idempotency protection.
- Notification failure never invalidates a successful booking.
- Critical transactional success is never returned before durable state is established.
- Workers must tolerate duplicate job delivery.
- Permanently failing jobs must not retry indefinitely.
- Important failure paths must be observable.

---

# 14. Related Documents

- `08-search-architecture.md`
- `09-booking-orchestration.md`
- `10-caching-strategy.md`
- `12-database-design.md`
- `13-security.md`
- `14-observability.md`