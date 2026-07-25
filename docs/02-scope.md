# Project Scope

## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Scope Definition](#2-scope-definition)
- [3. MVP Scope](#3-mvp-scope)
- [4. In Scope](#4-in-scope)
- [5. Out of Scope](#5-out-of-scope)
- [6. Future Scope](#6-future-scope)
- [7. System Boundaries](#7-system-boundaries)
- [8. External Dependencies](#8-external-dependencies)
- [9. Constraints](#9-constraints)
- [10. Scope Assumptions](#10-scope-assumptions)
- [11. MVP Success Criteria](#11-mvp-success-criteria)
- [12. Scope Change Rules](#12-scope-change-rules)
- [13. Related Documents](#13-related-documents)
- [14. Summary](#14-summary)

---

## 1. Purpose

This document defines the boundaries of the travel booking platform.

It clarifies:

- What will be included in the first implementation.
- What will not be included initially.
- Which features may be added later.
- Which responsibilities belong to the platform.
- Which responsibilities belong to external providers.

The purpose of this document is to prevent uncontrolled scope growth and keep architecture and implementation decisions aligned with the MVP.

For a high-level description of the product, see [Product Overview](01-product-overview.md).

---

## 2. Scope Definition

The project covers the analysis, design, and initial implementation of a travel booking platform for:

- Flights.
- Hotels.

The platform acts as an intermediary between:

- Users.
- Flight providers.
- Hotel providers.
- Payment gateways.
- Notification providers.

The platform does not own airline inventory or hotel inventory.

External providers remain responsible for:

- Live availability.
- Current provider pricing.
- Fare and room conditions.
- Provider-side booking confirmation.
- Provider-side cancellation results.

The platform is responsible for:

- User-facing search.
- Offer normalization.
- Offer comparison.
- Booking coordination.
- Payment tracking.
- Booking persistence.
- Cancellation and refund coordination.
- Operational visibility.

---

## 3. MVP Scope

The MVP is the smallest version that demonstrates a complete and usable flight and hotel booking flow.

### 3.1 Flight MVP

The initial flight scope includes:

- One-way search.
- Round-trip search.
- Origin and destination selection.
- Departure and return dates.
- Adult passenger count.
- Basic cabin-class selection.
- Flight offer listing.
- Basic filtering and sorting.
- Offer details.
- Offer revalidation.
- Passenger information.
- Payment.
- Provider booking confirmation.
- Booking details.
- Ticket reference storage when available.

The MVP may begin with one flight provider, while the architecture remains capable of supporting multiple providers.

---

### 3.2 Hotel MVP

The initial hotel scope includes:

- Destination search.
- Check-in and check-out dates.
- Room count.
- Adult guest count.
- Child guest count and ages when required.
- Hotel offer listing.
- Basic filtering and sorting.
- Hotel and room details.
- Room-offer revalidation.
- Guest information.
- Payment.
- Provider booking confirmation.
- Booking details.
- Hotel voucher or confirmation-reference storage.

The MVP may begin with one hotel provider, while the architecture remains capable of supporting multiple providers.

---

### 3.3 User MVP

The initial user scope includes:

- User registration.
- User login.
- User logout.
- Password recovery.
- Profile information.
- Basic traveler information.
- Viewing booking history.
- Viewing booking details.

Guests may search without signing in.

Authentication may be required before completing a booking.

---

### 3.4 Payment MVP

The initial payment scope includes:

- One external payment gateway.
- Payment-attempt creation.
- Payment-status tracking.
- Payment webhook processing.
- Duplicate-payment prevention.
- Linking payment records to bookings.
- Basic refund initiation and tracking.

The platform will not store raw payment-card data.

---

### 3.5 Notification MVP

The initial notification scope includes:

- Booking confirmation emails.
- Payment-failure emails.
- Cancellation-result emails.
- Refund-status emails.

Additional channels may be introduced later.

---

### 3.6 Administration MVP

The initial administration scope includes:

- Viewing users.
- Viewing bookings.
- Viewing booking status.
- Viewing payment status.
- Viewing provider references.
- Viewing failed or uncertain bookings.
- Enabling or disabling provider integrations.
- Basic audit visibility.

A full business intelligence or revenue dashboard is not part of the MVP.

---

## 4. In Scope

The following capabilities are included in the planned project.

### 4.1 Identity and User Management

- Registration.
- Authentication.
- Password recovery.
- Profile management.
- Traveler and guest information.
- Booking ownership.
- Basic role-based access.

Detailed roles and permissions will be documented in [User Roles](03-user-roles.md).

---

### 4.2 Guest Search

Guests can:

- Search for flights.
- Search for hotels.
- View results.
- Apply basic filters.
- Sort results.
- View offer details.

---

### 4.3 Flight Search

The platform includes:

- Search validation.
- Flight-provider integration.
- Unified flight-offer models.
- Basic result aggregation.
- Basic duplicate identification.
- Filtering.
- Sorting.
- Search-result caching.
- Partial-result handling when more than one provider is enabled.

Detailed search behavior will be documented in [Search Architecture](08-search-architecture.md).

---

### 4.4 Hotel Search

The platform includes:

- Search validation.
- Hotel-provider integration.
- Unified hotel and room-offer models.
- Basic result aggregation.
- Basic hotel matching where possible.
- Filtering.
- Sorting.
- Search-result caching.
- Partial-result handling when more than one provider is enabled.

---

### 4.5 Offer Revalidation

The platform includes revalidation of selected offers before booking.

Revalidation may confirm:

- Availability.
- Current price.
- Currency.
- Taxes and fees.
- Fare or rate conditions.
- Offer expiration.

Detailed revalidation rules will be documented in [Booking Orchestration](09-booking-orchestration.md).

---

### 4.6 Flight Booking

The platform includes:

- Passenger-data collection.
- Offer revalidation.
- Provider booking request.
- Payment coordination.
- Provider confirmation.
- Provider-reference storage.
- Internal booking persistence.
- Ticket-reference storage when available.
- Booking confirmation notification.

---

### 4.7 Hotel Booking

The platform includes:

- Guest-data collection.
- Room-offer revalidation.
- Provider booking request.
- Payment coordination.
- Provider confirmation.
- Provider-reference storage.
- Internal booking persistence.
- Voucher-reference storage when available.
- Booking confirmation notification.

---

### 4.8 Booking Management

Users can:

- View bookings.
- View booking details.
- View booking status.
- View payment status.
- Access tickets or vouchers when available.
- Request cancellation where supported.
- Track refund progress.

Detailed booking states will be documented in [Booking Orchestration](09-booking-orchestration.md).

---

### 4.9 Cancellation and Refund Tracking

The platform includes:

- Cancellation eligibility checks.
- Cancellation requests.
- Provider cancellation calls.
- Cancellation-status tracking.
- Refund-record creation.
- Refund-status tracking.
- Manual review for unresolved cases.


---

### 4.10 Provider Integration

The platform includes:

- Provider authentication.
- Search operations.
- Revalidation operations.
- Booking operations.
- Status checks.
- Cancellation operations where supported.
- Provider-specific adapters.
- Error normalization.
- Timeout handling.
- Provider enablement and disablement.

Provider-specific implementation details should remain isolated from the core domain.

---

### 4.11 Caching and Temporary Data

Redis is included for:

- Search-result caching.
- Temporary offer storage.
- Provider-token caching.
- Rate limiting.
- Short-lived locks.
- Idempotency support.
- Temporary workflow data.

Detailed Redis behavior will be documented in [Caching Strategy](10-caching-strategy.md).

---

### 4.12 Persistent Data

The platform includes permanent storage for:

- Users.
- Traveler profiles.
- Bookings.
- Flight-booking details.
- Hotel-booking details.
- Provider references.
- Booking-status history.
- Payment records.
- Refund records.
- Notification records.
- Audit records.
- Manual-review records.

The database structure will be documented in [Database Design](11-database-design.md).

---

### 4.13 Reliability and Failure Handling

The platform includes basic handling for:

- Provider timeouts.
- Provider errors.
- Expired offers.
- Price changes.
- Payment failures.
- Duplicate booking requests.
- Duplicate payment events.
- Unknown provider results.
- Notification failures.
- Refund failures.

Detailed failure scenarios will be documented in [Failure Scenarios](13-failure-scenarios.md).

---

### 4.14 Observability

The platform includes:

- Structured logs.
- Request identifiers.
- Booking identifiers.
- Provider identifiers.
- Payment identifiers.
- Health checks.
- Basic metrics.
- Error tracking.
- Audit history.

Detailed observability requirements will be documented in [Observability](15-observability.md).

---

## 5. Out of Scope

The following items are intentionally excluded from the MVP.

### 5.1 Additional Travel Products

- Car rental.
- Travel insurance.
- Airport transfers.
- Train booking.
- Bus booking.
- Cruises.
- Tours and activities.
- Visa services.
- Flight and hotel packages.

---

### 5.2 Advanced Flight Features

- Multi-city flight search.
- Group booking.
- Private and charter flights.
- Airline loyalty-point redemption.
- Advanced seat-map selection.
- Complex ticket exchange.
- Corporate travel policies.
- Interline and multi-provider ticket combinations.

---

### 5.3 Advanced Hotel Features

- Hotel-owner portal.
- Hotel inventory management.
- Direct room inventory entry.
- Property onboarding.
- Hotel extranet.
- Supplier contract management.
- Hotel payout management.
- Channel-manager functionality.

---

### 5.4 Advanced User Features

- Social login.
- Loyalty points.
- Referral programs.
- Wish lists.
- Shared trip planning.
- Family accounts.
- Group payments.
- User-generated reviews.

---

### 5.5 Advanced Commercial Features

- Dynamic markup engine.
- Complex commission engine.
- Coupon and promotion engine.
- Multi-currency settlement.
- Supplier invoicing.
- Corporate billing.
- B2B travel-agent portal.
- White-label booking portals.

---

### 5.6 Advanced Architecture

The MVP does not require:

- Full microservices.
- Multi-region active-active deployment.
- Service mesh.
- Full event sourcing.
- Complex event-streaming infrastructure.
- Machine-learning ranking.
- Dedicated data warehouse.
- Real-time recommendation engine.

These may be introduced only when justified by real requirements.

---

### 5.7 Inventory Ownership

The platform will not:

- Operate an airline.
- Own hotel rooms.
- Maintain primary flight inventory.
- Maintain primary hotel inventory.
- Control provider prices.
- Guarantee cached prices.
- Replace provider booking systems.

---

## 6. Future Scope

The following capabilities may be added in later phases.

### 6.1 Travel Expansion

- Car rental.
- Travel insurance.
- Airport transfers.
- Activities.
- Train booking.
- Bus booking.
- Travel packages.

---

### 6.2 Search Enhancements

- Flexible-date search.
- Nearby-airport search.
- Price calendar.
- Price alerts.
- Saved searches.
- Personalized ranking.
- Search history.
- Recommendation engine.

---

### 6.3 Booking Enhancements

- Multi-city flights.
- Seat selection.
- Baggage purchase.
- Meal selection.
- Booking modification.
- Ticket exchange.
- Partial cancellation.
- Split payments.
- Group booking.

---

### 6.4 User Enhancements

- Social login.
- Saved payment methods through payment-provider tokens.
- Loyalty program.
- Referral program.
- Family traveler profiles.
- Corporate accounts.
- Wish lists.

---

### 6.5 Business Enhancements

- Coupons.
- Promotions.
- Provider markup rules.
- Commission rules.
- Provider prioritization.
- B2B agent portal.
- White-label portals.
- Supplier settlement.
- Revenue reporting.

---

### 6.6 Technical Enhancements

- Independent search workers.
- Independent booking workers.
- Provider-specific queues.
- Advanced event-driven workflows.
- Multi-region deployment.
- Data warehouse.
- Advanced analytics.
- Automated anomaly detection.

---

## 7. System Boundaries

The platform boundary includes:

- Client-facing APIs.
- Authentication and authorization.
- Search orchestration.
- Offer normalization.
- Temporary offer management.
- Booking orchestration.
- Payment coordination.
- Booking persistence.
- Cancellation coordination.
- Refund tracking.
- Notifications.
- Provider adapters.
- Administration.
- Monitoring and auditing.

The platform boundary does not include:

- Airline reservation systems.
- Hotel property-management systems.
- Provider-owned inventory systems.
- Payment-gateway infrastructure.
- Email infrastructure.
- SMS infrastructure.
- Push-notification infrastructure.

---

## 8. External Dependencies

The project depends on external systems.

### Flight Providers

Responsible for:

- Live availability.
- Flight pricing.
- Fare rules.
- Reservations.
- Ticket issuance.
- Cancellation.
- Provider booking status.

### Hotel Providers

Responsible for:

- Hotel data.
- Room availability.
- Room pricing.
- Rate conditions.
- Reservations.
- Voucher generation.
- Cancellation.
- Provider booking status.

### Payment Gateway

Responsible for:

- Secure payment processing.
- Authorization.
- Capture.
- Refund execution.
- Payment webhooks.
- Transaction references.

### Notification Providers

Responsible for:

- Email delivery.
- SMS delivery.
- Push-notification delivery.

### Reference Data Providers

May supply:

- Airport data.
- City data.
- Country data.
- Currency data.
- Geographic coordinates.
- Destination mappings.

---

## 9. Constraints

The project may be affected by:

- Provider availability.
- Provider latency.
- Provider rate limits.
- Provider commercial agreements.
- Provider certification requirements.
- Limited provider test environments.
- Payment-gateway restrictions.
- Supported currencies.
- Supported countries.
- Personal-data regulations.
- Different provider cancellation rules.
- Different provider refund behavior.
- Search-offer expiration.
- Unknown external-operation results after timeouts.

Provider-specific constraints should be reviewed before each integration.

---

## 10. Scope Assumptions

The project scope assumes:

1. External providers expose search and booking APIs.
2. Provider contracts allow offer display and booking.
3. Search results may expire quickly.
4. Selected offers can be revalidated.
5. Providers may return overlapping offers.
6. Provider capabilities may differ.
7. Redis is available for temporary data.
8. PostgreSQL is used for persistent business data.
9. One payment gateway is sufficient for the MVP.
10. Email is sufficient as the first notification channel.
11. The MVP may support limited languages and currencies.
12. Some operations may require background processing.
13. Some failures may require manual review.
14. Provider responses must be normalized.
15. Raw payment-card details will not be stored.
16. The architecture will initially use a Modular Monolith.

---

## 11. MVP Success Criteria

The MVP will be considered successful when it can:

- Search at least one flight provider.
- Search at least one hotel provider.
- Return normalized results.
- Cache temporary search data.
- Revalidate selected offers.
- Complete one flight-booking flow.
- Complete one hotel-booking flow.
- Process payments safely.
- Prevent duplicate bookings.
- Prevent duplicate charges.
- Store confirmed bookings.
- Store provider references.
- Display user booking history.
- Handle common provider failures.
- Track booking and payment states independently.
- Send booking-confirmation emails.
- Provide logs sufficient to investigate failures.

---

## 12. Scope Change Rules

Any proposed scope change should be evaluated before implementation.

The evaluation should consider:

- Business value.
- User value.
- MVP priority.
- Provider support.
- Technical complexity.
- Security impact.
- Data-model impact.
- Payment impact.
- Operational impact.
- Implementation cost.

A feature should not be added to the MVP only because it is technically interesting.

Significant changes should be documented through:

- A scope update.
- An Architecture Decision Record.
- A new roadmap phase.

---

## 13. Related Documents

- [Product Overview](01-product-overview.md)
- [User Roles](03-user-roles.md)
- [Functional Requirements](04-functional-requirements.md)
- [Non-Functional Requirements](05-non-functional-requirements.md)
- [Domain Model](06-domain-model.md)
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

## 14. Summary

The project scope covers an MVP travel booking platform for flights and hotels.

The MVP includes:

- User authentication.
- Guest search.
- Flight and hotel search.
- Basic offer comparison.
- Offer revalidation.
- Flight booking.
- Hotel booking.
- Payment processing.
- Booking history.
- Basic cancellation and refund tracking.
- Email notifications.
- Basic administration.
- Redis caching.
- PostgreSQL persistence.

Advanced travel products, complex commercial models, full microservices, supplier portals, and machine-learning features are intentionally excluded from the MVP.