# Product Overview

## Table of Contents

- [1. Introduction](#1-introduction)
- [2. Product Vision](#2-product-vision)
- [3. Problem Statement](#3-problem-statement)
- [4. Product Type](#4-product-type)
- [5. Target Users](#5-target-users)
- [6. Value Proposition](#6-value-proposition)
- [7. Core Product Capabilities](#7-core-product-capabilities)
- [8. High-Level System Context](#8-high-level-system-context)
- [9. Data Ownership and Source of Truth](#9-data-ownership-and-source-of-truth)
- [10. Initial Architecture Direction](#10-initial-architecture-direction)
- [11. Product Goals](#11-product-goals)
- [12. Initial Assumptions](#12-initial-assumptions)
- [13. Non-Goals of This Document](#13-non-goals-of-this-document)
- [14. Related Documents](#14-related-documents)
- [15. Summary](#15-summary)

---

## 1. Introduction

This project documents the analysis, design, and planned implementation of a travel booking platform for flights and hotels.

The platform allows users to:

- Search for flights and hotels.
- Compare offers from multiple external providers.
- Review prices and booking conditions.
- Select an offer.
- Complete a booking.
- Pay through the platform.
- View and manage confirmed bookings.

The platform does not own flight inventory, hotel rooms, or live travel pricing.

Instead, it integrates with external flight and hotel providers through their APIs and presents their offers through a unified user experience.

---

## 2. Product Vision

The product vision is to provide users with one reliable platform for discovering, comparing, and booking flights and hotels from multiple travel providers.

Users should not need to understand:

- Which provider returned an offer.
- How each provider structures its data.
- Which provider authentication method is used.
- How provider-specific booking workflows operate.
- How provider errors are handled.

The platform hides these differences and exposes one consistent experience to web and mobile clients.

From the user's perspective, the product behaves as one travel booking platform.

From the backend perspective, it is a multi-provider travel aggregation and booking orchestration system.

---

## 3. Problem Statement

Travel inventory is distributed across many external providers.

These may include:

- Airlines.
- Global Distribution Systems.
- Flight aggregators.
- Hotel wholesalers.
- Hotel aggregators.
- Direct hotel suppliers.

Each provider may use different:

- Authentication mechanisms.
- Request and response formats.
- Pricing structures.
- Availability rules.
- Booking workflows.
- Cancellation policies.
- Error codes.
- Timeout rules.
- Rate limits.
- Supported operations.

Allowing frontend applications to communicate directly with every provider would create strong coupling and expose provider-specific complexity throughout the system.

The platform solves this problem by introducing a unified backend layer that:

- Integrates with multiple providers.
- Normalizes provider responses.
- Aggregates and compares travel offers.
- Coordinates booking and payment workflows.
- Stores permanent business records.
- Handles provider failures safely.
- Exposes one consistent API to clients.

---

## 4. Product Type

The system is a:

> Multi-provider travel aggregation and booking platform.

It combines the responsibilities of:

- A flight and hotel search engine.
- An offer-comparison platform.
- A provider-integration layer.
- A booking platform.
- A booking-orchestration system.

External providers remain responsible for live availability, current pricing, provider-side reservations, and final provider confirmation.

---

## 5. Target Users

The platform is intended to support the following high-level user groups:

### Guests

Users who can search, filter, compare, and inspect flight and hotel offers without signing in.

### Registered Customers

Users who can create bookings, complete payments, manage traveler information, and review their booking history.

### Administrators

Internal users who manage providers, monitor bookings, review payment issues, and operate the platform.

### Support Agents

Internal users who investigate failed, delayed, or uncertain bookings and assist customers.

Detailed roles and permissions will be documented in [User Roles](03-user-roles.md).

---

## 6. Value Proposition

The platform provides value by:

- Giving users access to multiple providers through one interface.
- Increasing the number and variety of available offers.
- Simplifying offer comparison.
- Hiding provider-specific technical complexity.
- Providing a consistent booking experience.
- Reducing dependency on one external supplier.
- Supporting future provider expansion.
- Improving operational visibility into booking and payment workflows.

For the business, the platform creates a reusable integration layer that makes it easier to add, replace, enable, or disable travel providers.

---

## 7. Core Product Capabilities

The following capabilities describe the product at a high level.

Detailed behavior will be documented in [Functional Requirements](04-functional-requirements.md).

### 7.1 Flight Search

Users can search for flights based on criteria such as:

- Origin.
- Destination.
- Departure date.
- Return date.
- Passenger count.
- Passenger type.
- Cabin class.
- Trip type.

The platform collects flight offers from one or more external providers and returns a unified result.

---

### 7.2 Hotel Search

Users can search for hotels based on criteria such as:

- Destination.
- Check-in date.
- Check-out date.
- Room count.
- Guest count.
- Guest ages.
- Hotel rating.
- Price range.

The platform collects hotel and room offers from one or more external providers and returns normalized results.

---

### 7.3 Offer Aggregation

The platform combines offers from multiple providers.

This may include:

- Normalizing provider-specific data.
- Handling provider failures independently.
- Identifying duplicate or overlapping offers.
- Sorting and ranking results.
- Returning one consistent response to clients.

The detailed search workflow will be documented in [Search Architecture](08-search-architecture.md).

---

### 7.4 Offer Comparison

Users can compare offers using attributes such as:

#### Flights

- Price.
- Airline.
- Departure time.
- Arrival time.
- Duration.
- Number of stops.
- Cabin class.
- Baggage allowance.
- Fare conditions.

#### Hotels

- Price.
- Hotel rating.
- Location.
- Room type.
- Meal plan.
- Included services.
- Cancellation policy.

---

### 7.5 Offer Revalidation

Search results are temporary and may become outdated.

Before a booking is finalized, the platform must confirm that the selected offer is still available and that its price and conditions are still valid.

The detailed revalidation and booking workflow will be documented in [Booking Orchestration](09-booking-orchestration.md).

---

### 7.6 Flight Booking

The platform allows users to:

- Select a flight offer.
- Provide passenger information.
- Complete payment.
- Receive provider confirmation.
- View booking details.
- Access ticket information when available.

---

### 7.7 Hotel Booking

The platform allows users to:

- Select a hotel and room offer.
- Provide guest information.
- Complete payment.
- Receive provider confirmation.
- View booking details.
- Access a hotel voucher when available.

---

### 7.8 Booking Management

Users may be able to:

- View booking history.
- View booking details.
- Check booking status.
- Download available travel documents.
- Request cancellation.
- Track refund progress.

Available operations may depend on provider capabilities and booking conditions.

---

### 7.9 Payment Processing

The platform integrates with an external payment gateway to process booking payments.

The platform is responsible for:

- Creating payment attempts.
- Tracking payment status.
- Preventing duplicate charges.
- Linking payments to bookings.
- Initiating refunds when required.

The platform must not store raw payment-card information.

---

### 7.10 Notifications

The platform may notify users about important events such as:

- Booking confirmation.
- Payment success or failure.
- Ticket issuance.
- Voucher availability.
- Cancellation.
- Refund updates.
- Provider schedule changes.

Notification channels may include email, push notifications, or SMS.

---

## 8. High-Level System Context

```text
Guest or Customer
        |
        v
Travel Booking Platform
        |
        +----> Flight Providers
        |
        +----> Hotel Providers
        |
        +----> Payment Gateway
        |
        +----> Email Provider
        |
        +----> SMS Provider
        |
        +----> Push Notification Provider
```

Clients communicate only with the travel booking platform.

The platform communicates with external providers and hides provider-specific behavior from clients.

A detailed architecture diagram will be added in [Search Architecture](08-search-architecture.md) and [Booking Orchestration](09-booking-orchestration.md).

---

## 9. Data Ownership and Source of Truth

Different systems act as the source of truth for different data.

### External providers are the source of truth for:

- Live flight availability.
- Live hotel room availability.
- Current provider price.
- Fare and rate conditions.
- Provider reservation status.
- Ticket issuance.
- Hotel confirmation.
- Provider-side cancellation results.

### The platform database is the source of truth for:

- Internal user records.
- Internal booking records.
- Customer ownership of bookings.
- Payment and refund records.
- Provider references.
- Booking history.
- Audit history.
- Internal workflow progress.

### Redis is used for temporary data such as:

- Search results.
- Temporary offers.
- Search sessions.
- Provider access tokens.
- Locks.
- Rate-limit counters.
- Short-lived workflow data.

The detailed caching approach will be documented in [Caching Strategy](10-caching-strategy.md).

---

## 10. Initial Architecture Direction

The initial implementation should use a Modular Monolith with clear internal module boundaries.

A possible high-level structure is:

```text
Backend Application
├── Identity Module
├── Search Module
├── Offer Module
├── Provider Integration Module
├── Booking Module
├── Payment Module
├── Notification Module
└── Administration Module
```

Supporting infrastructure may include:

```text
PostgreSQL
Redis
Background Workers
Message Queue
External Flight APIs
External Hotel APIs
Payment Gateway
Notification Services
```

This approach keeps the first version easier to:

- Understand.
- Develop.
- Test.
- Deploy.
- Operate.

Modules may be extracted into separate services later if real scalability or operational requirements justify that decision.

Detailed technical decisions will be recorded in Architecture Decision Records.

---

## 11. Product Goals

The main product goals are:

- Provide one interface for searching flights and hotels.
- Aggregate offers from multiple providers.
- Allow users to compare prices and conditions.
- Hide provider-specific complexity.
- Provide a consistent booking experience.
- Prevent duplicate bookings and payments.
- Handle external provider failures safely.
- Store durable booking and payment records.
- Support new provider integrations with minimal impact.
- Provide operational visibility into booking workflows.
- Preserve clear module boundaries for future growth.

---

## 12. Initial Assumptions

This overview is based on the following assumptions:

1. Flights and hotels are supplied by external APIs.
2. The platform will eventually support multiple providers.
3. The first implementation may begin with one flight provider and one hotel provider.
4. Search results are temporary.
5. Prices and availability may change before booking.
6. Redis will be used for caching and temporary coordination.
7. PostgreSQL will store permanent business data.
8. Payments will be processed through an external gateway.
9. Raw payment-card information will not be stored.
10. Provider operations may fail or timeout.
11. Some uncertain operations may require manual review.
12. The first version will use a Modular Monolith.

The complete project boundaries are defined in [Project Scope](02-scope.md).

---

## 13. Non-Goals of This Document

This document does not define:

- The complete MVP scope.
- Detailed user roles and permissions.
- Complete functional requirements.
- Complete non-functional requirements.
- Detailed business rules.
- Final database tables.
- Exact API contracts.
- Exact Redis keys or TTL values.
- Detailed booking states.
- Detailed failure-handling workflows.
- Final deployment architecture.

These topics are documented separately to avoid duplication.

---

## 14. Related Documents

- [Project Scope](02-scope.md)
- [User Roles](03-user-roles.md)
- [Functional Requirements](04-functional-requirements.md)
- [Non-Functional Requirements](05-non-functional-requirements.md)
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

## 15. Summary

The product is a multi-provider travel aggregation and booking platform for flights and hotels.

It allows users to search and compare travel offers through one interface while hiding the differences between external providers.

The platform is responsible for aggregation, normalization, booking coordination, payment tracking, and permanent business records.

External providers remain responsible for live inventory, current pricing, and provider-side booking confirmation.