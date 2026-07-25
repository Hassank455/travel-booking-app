# 09. Booking Orchestration

## 1. Purpose

This document describes how the Travel Booking Platform coordinates the booking process after a customer selects a flight or hotel offer.

The orchestration begins when the customer submits a booking request and ends when the booking is confirmed, rejected, or compensated after a failure.

The document focuses on architectural responsibilities and business flow. It does not describe controllers, repositories, framework classes, or provider-specific API payloads.

---

## 2. Scope

This document covers:

- Offer selection and retrieval.
- Offer revalidation.
- Creation of a pending booking.
- Payment processing.
- Provider reservation confirmation.
- Booking persistence.
- Customer notification.
- Failure handling and compensation.
- Idempotency and duplicate-request protection.

It does not define:

- Search aggregation and ranking.
- Exact payment gateway integration.
- Database table structures.
- API endpoint contracts.
- Cache TTL values.
- Provider-specific reservation protocols.

---

## 3. Booking Principles

### 3.1 A Booking Starts from an Offer

Every booking request must reference a valid offer produced by the Search Module.

The client should send a platform offer reference rather than raw provider data.

### 3.2 Revalidation Is Mandatory

Search results are temporary. Before continuing, the platform must confirm that:

- The offer still exists.
- The requested inventory remains available.
- The current price matches the accepted price policy.
- Important commercial rules have not changed.

### 3.3 Payment and Booking Have Separate Lifecycles

A successful payment does not automatically mean that the travel provider confirmed the reservation.

Likewise, a booking may be pending while payment is still being processed.

### 3.4 Provider Confirmation Is Required

The booking is considered confirmed only after the travel provider returns a successful reservation confirmation.

### 3.5 Failures Must Not Leave Partial Business State

When a later step fails after an earlier step succeeds, the platform must run an appropriate compensation action.

Example:

```text
Payment succeeds
        ↓
Provider confirmation fails
        ↓
Refund or void payment
        ↓
Mark booking as failed
```

### 3.6 Notification Is Non-Blocking

A notification failure must not change a successfully confirmed booking into a failed booking.

Notifications should be retried asynchronously.

---

## 4. High-Level Booking Flow

The complete business-level flow is maintained separately:

**Diagram:** [booking-orchestration.mmd](./diagrams/booking/booking-orchestration.mmd)

The normal flow is:

```text
Customer selects offer
        ↓
Offer is revalidated
        ↓
Pending booking is created
        ↓
Payment is processed
        ↓
Provider confirms reservation
        ↓
Booking is marked as confirmed
        ↓
Customer is notified
```

---

## 5. Booking Participants

### Customer

Provides:

- Selected offer reference.
- Traveler or guest information.
- Contact information.
- Payment choice.
- Required booking preferences.

### API Layer

Responsibilities:

- Authenticate the request when required.
- Validate the request structure.
- Forward the request to the Booking Module.
- Return the orchestration result to the client.

The API Layer does not own booking decisions.

### Booking Module

Acts as the orchestration owner.

Responsibilities:

- Coordinate the booking steps.
- Maintain booking state.
- Enforce booking business rules.
- Record provider and payment references.
- Trigger compensation when required.
- Produce booking-related domain events.

### Search Module

Provides the temporary offer context needed by the Booking Module, including:

- Provider identity.
- Provider offer reference.
- Normalized offer snapshot.
- Expiration information.
- Accepted price.

### Provider Integration Module

Responsibilities:

- Revalidate the selected offer.
- Translate provider-specific reservation requests.
- Confirm the reservation with the external travel provider.
- Normalize provider errors and confirmation references.

### Payment Module

Responsibilities:

- Authorize or capture the customer payment.
- Track payment status independently.
- Prevent duplicate successful charges.
- Void or refund a payment when compensation is required.

### Persistence Layer

Stores durable booking records, payment references, provider references, status changes, and immutable confirmation snapshots.

### Notification Module

Sends customer communications such as:

- Booking confirmation.
- Payment failure.
- Booking failure.
- Cancellation confirmation.
- Refund completion.

---

## 6. Booking Lifecycle

The booking lifecycle is maintained separately:

**Diagram:** [booking-state-machine.mmd](./diagrams/booking/booking-state-machine.mmd)

Suggested booking states:

| State | Meaning |
|---|---|
| `PENDING` | The booking request has been accepted but processing is incomplete. |
| `REVALIDATING` | The selected offer is being checked with the provider. |
| `AWAITING_PAYMENT` | The offer is valid and payment is required. |
| `CONFIRMING_PROVIDER` | Payment succeeded and provider confirmation is in progress. |
| `CONFIRMED` | The provider confirmed the reservation. |
| `PAYMENT_FAILED` | The payment could not be completed. |
| `COMPENSATION_REQUIRED` | A successful previous step must be reversed. |
| `FAILED` | The booking could not be completed. |
| `CANCELLED` | A confirmed booking was cancelled successfully. |
| `COMPLETED` | The trip or stay has completed. |
| `EXPIRED` | The booking expired according to its business policy. |

Flight bookings and hotel bookings may extend this lifecycle with their own domain-specific states, but their state transitions should remain explicit and independent.

---

## 7. Booking Orchestration Steps

### 7.1 Receive Booking Request

The Booking Module receives:

- An idempotency key.
- A platform offer identifier.
- Traveler or guest details.
- Customer contact details.
- Payment method information.
- The price accepted by the customer.

The request is rejected when required information is missing or invalid.

### 7.2 Load Offer Context

The Booking Module requests the temporary offer context from the Search Module or offer store.

The offer context must contain enough information to identify the original provider offer without trusting client-supplied commercial data.

The orchestration stops when:

- The offer cannot be found.
- The offer has expired.
- The offer does not belong to the expected search context.
- The offer reference has been altered.

### 7.3 Revalidate Offer

The Provider Integration Module revalidates the offer with the original travel provider.

The result may be:

#### Offer Still Valid

The current price, availability, and important policies remain acceptable.

#### Price Changed

The platform should not silently charge more than the amount accepted by the customer.

Depending on product policy, it may:

- Reject the booking and return the new price.
- Ask the customer to explicitly accept the updated offer.
- Continue only when the price change is within an approved tolerance.

#### Availability Changed

The platform returns an availability failure and may suggest a new search.

#### Commercial Rules Changed

Changes to baggage, cancellation rules, room type, occupancy, or included services may require explicit customer acceptance.

### 7.4 Create Pending Booking

After successful revalidation, the platform creates a durable pending booking record.

This record provides:

- A platform booking reference.
- A stable orchestration identifier.
- A place to record status transitions.
- A durable audit trail.

The pending record must not be presented as a confirmed reservation.

### 7.5 Process Payment

The Payment Module processes the required payment.

The exact operation depends on the payment strategy:

- Immediate capture.
- Authorization followed by capture.
- Provider-side payment.
- Pay-at-property hotel booking.
- Deferred payment when supported.

The Payment Module returns a clear result:

- Successful.
- Failed.
- Pending.
- Unknown and requiring reconciliation.

An unknown payment outcome must not be treated as a definite failure because retrying may create a duplicate charge.

### 7.6 Confirm Reservation with Provider

After the required payment condition is satisfied, the platform sends the booking request to the external travel provider.

The provider request may include:

- Provider offer reference.
- Traveler or guest information.
- Contact details.
- Payment confirmation or guarantee data.
- Flight or hotel-specific information.

The provider response should include:

- Provider booking reference.
- Confirmation status.
- Ticketing or voucher information when available.
- Confirmed itinerary or stay details.
- Confirmed price and currency.
- Provider policies.

The provider confirmation is the source of truth for reservation success.

### 7.7 Persist Confirmed Booking

After provider confirmation, the Booking Module stores a durable confirmation snapshot.

The snapshot should preserve the important details accepted and confirmed at booking time, even when external data changes later.

Examples:

- Confirmed itinerary or stay.
- Traveler or guest information.
- Final price.
- Currency.
- Provider reference.
- Fare or rate rules.
- Cancellation policy.
- Ticket, voucher, or confirmation details.

### 7.8 Publish Booking Event

The Booking Module publishes a `BookingConfirmed` event after durable confirmation is stored.

Consumers may include:

- Notification Module.
- Audit and analytics components.
- Customer history projections.
- Administrative reporting.

### 7.9 Notify Customer

The Notification Module sends the booking confirmation asynchronously.

A failed email, SMS, or push notification should be retried without rolling back the confirmed booking.

---

## 8. Interaction Sequence

The detailed interaction view is maintained separately:

**Diagram:** [booking-sequence.mmd](./diagrams/booking/booking-sequence.mmd)

The sequence diagram shows interactions between:

- Customer.
- API Layer.
- Booking Module.
- Search Module.
- Payment Module.
- Provider Integration Module.
- Persistence Layer.
- Notification Module.

---

## 9. Failure and Compensation

The compensation flow is maintained separately:

**Diagram:** [booking-failure-flow.mmd](./diagrams/booking/booking-failure-flow.mmd)

### 9.1 Offer Revalidation Failure

Possible actions:

- Stop the orchestration.
- Return the updated price or availability state.
- Do not create a confirmed booking.
- Do not process payment.

### 9.2 Payment Failure

Possible actions:

- Mark the booking as payment failed.
- Do not call provider confirmation.
- Allow a safe payment retry according to policy.
- Notify the customer when appropriate.

### 9.3 Payment Outcome Unknown

Examples include timeout after the payment gateway may have processed the charge.

The platform should:

- Avoid immediately retrying with a new payment operation.
- Query the payment gateway using the same payment reference.
- Reconcile the final status.
- Continue only after the payment state is known.

### 9.4 Provider Confirmation Failure After Payment

This is the most important compensation scenario.

The platform should:

1. Mark the booking as requiring compensation.
2. Void an authorization when possible.
3. Otherwise initiate a refund.
4. Record the compensation reference.
5. Mark the booking as failed after compensation is accepted.
6. Notify the customer.

The compensation may complete asynchronously when the payment provider does not return an immediate final result.

### 9.5 Persistence Failure After Provider Confirmation

The platform must not lose a confirmed external reservation.

It should preserve enough correlation data to recover the booking using:

- Provider booking reference.
- Idempotency key.
- Correlation identifier.
- Payment reference.

This scenario requires reconciliation rather than blindly repeating the provider booking request.

### 9.6 Notification Failure

The platform should:

- Keep the booking confirmed.
- Queue a retry.
- Record the notification failure.
- Allow support or administration to resend confirmation.

---

## 10. Idempotency

Every booking request must be idempotent.

A customer may submit the same request more than once because of:

- Double-clicks.
- Mobile network retries.
- Client timeouts.
- API gateway retries.
- Application restarts.

The client should send a unique idempotency key for one logical booking attempt.

The platform should associate the key with:

- Customer or guest context.
- Request payload hash.
- Booking reference.
- Current orchestration status.
- Final response when available.

### Required Behavior

#### Same Key and Same Request

Return the existing booking result or continue the existing orchestration.

#### Same Key and Different Request

Reject the request because the key was reused incorrectly.

#### Provider Calls

Provider confirmation requests should also use a stable provider-side reference or request identifier when supported.

#### Payment Calls

Payment operations must use stable idempotency references to prevent duplicate successful charges.

---

## 11. Consistency and Transaction Boundaries

The complete booking flow cannot normally be executed inside one database transaction because it involves external systems.

The platform should use:

- Small local transactions for each durable state change.
- Explicit booking statuses.
- Idempotent external operations.
- Domain events or an outbox mechanism.
- Compensation instead of distributed database rollback.
- Reconciliation jobs for uncertain external outcomes.

This orchestration behaves like a saga coordinated by the Booking Module.

---

## 12. Flight and Hotel Differences

The orchestration structure is shared conceptually, but flight and hotel bookings retain independent business rules.

### Flight Booking Examples

- Passenger name validation.
- Segment and itinerary validation.
- Ticketing deadlines.
- Fare basis and cabin rules.
- Passport information for international travel.
- Airline reservation references.

### Hotel Booking Examples

- Room occupancy validation.
- Child age rules.
- Check-in requirements.
- Pay-at-property guarantees.
- Room and rate plan confirmation.
- Property cancellation rules.

The Booking Module may coordinate both flows without forcing them into one identical domain entity or lifecycle implementation.

---

## 13. Security Considerations

The orchestration must:

- Never trust price or provider identifiers supplied directly by the client without server-side offer context.
- Protect traveler and payment-related information.
- Avoid storing raw card details.
- Validate that the customer can access the referenced booking.
- Sanitize provider errors before exposing them.
- Record sensitive actions in an audit trail.
- Prevent replay attacks using idempotency and expiration controls.

---

## 14. Observability

Every booking attempt should have a correlation identifier used across:

- Booking logs.
- Provider requests.
- Payment operations.
- Domain events.
- Notifications.
- Support and reconciliation tools.

Useful metrics include:

- Revalidation success rate.
- Price-change rate.
- Payment success rate.
- Provider confirmation success rate.
- Compensation rate.
- Refund initiation rate.
- Booking completion time.
- Unknown payment outcomes.
- Notification retry count.

Logs must not expose card information, passwords, passport data, or unnecessary personal details.

---

## 15. Architectural Decisions

### AD-BOOKING-001: Booking Requires Revalidation

A cached or selected offer cannot be booked without provider revalidation.

### AD-BOOKING-002: Pending Booking Is Persisted Before External Completion

A durable pending record is created to track orchestration progress, failures, and recovery.

### AD-BOOKING-003: Payment Status Is Independent

Payment state is not inferred from booking state.

### AD-BOOKING-004: Provider Confirmation Is the Reservation Source of Truth

The platform marks a booking as confirmed only after receiving provider confirmation.

### AD-BOOKING-005: External Failures Use Compensation

The platform does not rely on distributed transactions across payment and travel providers.

### AD-BOOKING-006: Booking Requests Are Idempotent

Retries must not create duplicate bookings or duplicate successful charges.

### AD-BOOKING-007: Confirmed Booking Data Is Preserved as a Snapshot

Important confirmed commercial and travel details remain available even when provider data later changes.

### AD-BOOKING-008: Notifications Are Asynchronous

Notification delivery does not block or reverse booking confirmation.

### AD-BOOKING-009: Flight and Hotel Rules Remain Independent

Shared orchestration concepts do not require identical domain models or lifecycle implementations.

---

## 16. Related Diagrams

- [Booking Orchestration](./diagrams/booking/booking-orchestration.mmd)
- [Booking Sequence](./diagrams/booking/booking-sequence.mmd)
- [Booking State Machine](./diagrams/booking/booking-state-machine.mmd)
- [Booking Failure and Compensation](./diagrams/booking/booking-failure-flow.mmd)

---

## 17. Related Documents

- `04-functional-requirements.md`
- `05-non-functional-requirements.md`
- `07-domain-model.md`
- `08-search-architecture.md`
- `10-caching-strategy.md`
- `11-database-design.md`
- `12-api-design.md`
- `13-failure-scenarios.md`
- `14-observability.md`

---

## 18. Summary

The Booking Module coordinates the business process that converts a temporary travel offer into a confirmed flight or hotel reservation.

The core orchestration is:

```text
Load Offer
    ↓
Revalidate
    ↓
Create Pending Booking
    ↓
Process Payment
    ↓
Confirm with Provider
    ↓
Persist Confirmation
    ↓
Publish Event
    ↓
Notify Customer
```

The design prioritizes:

- Correctness over blind retries.
- Explicit lifecycle states.
- Provider confirmation as the reservation authority.
- Independent payment tracking.
- Idempotency.
- Compensation and reconciliation.
- Separation between flight and hotel business rules.
