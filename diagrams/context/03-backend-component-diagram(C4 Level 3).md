# 03. Backend Component Diagram

## 1. Purpose

This document presents the C4 Level 3 Component Diagram for the Backend Application of the Travel Booking Platform.

It describes the primary business modules inside the Backend Application, their responsibilities, and the relationships between them.

Unlike the Container Diagram, which focuses on deployable units, this diagram explains the logical organization of the Backend Application implemented as a **Modular Monolith**.

The diagram intentionally omits implementation details such as classes, packages, controllers, repositories, and database schema.

---

# 2. Backend Overview

The Backend Application is implemented as a **Modular Monolith**.

Each business capability is implemented as an independent module with a clearly defined responsibility.

Modules communicate through well-defined interfaces while remaining part of the same deployable application.

This architecture provides:

- Clear separation of responsibilities.
- Low coupling between business capabilities.
- High maintainability.
- Simpler deployment compared to microservices.
- The ability to extract modules into independent services in the future if required.

---

# 3. Backend Components

## API Layer

The API Layer is the public entry point of the Backend Application.

Its responsibilities include:

- Routing requests.
- Authentication.
- Authorization.
- Request validation.
- Response serialization.
- Invoking the appropriate business module.

The API Layer does not contain business logic.

---

## Authentication Module

Responsible for:

- User authentication.
- Token management.
- Authorization.
- Password management.
- Identity verification.

---

## Customer Module

Responsible for:

- Customer accounts.
- Customer profiles.
- Preferences.
- Account settings.

---

## Traveler Module

Responsible for:

- Traveler profiles.
- Passenger information.
- Reusable traveler data.
- Traveler validation.

---

## Search Module

Responsible for:

- Flight search.
- Hotel search.
- Search orchestration.
- Provider aggregation.
- Offer normalization.
- Search filtering.
- Search caching.

Detailed behavior is documented in:

`08-search-architecture.md`

---

## Booking Module

Responsible for:

- Booking orchestration.
- Offer revalidation.
- Booking creation.
- Booking management.
- Booking cancellation.
- Booking status.

Detailed behavior is documented in:

`09-booking-orchestration.md`

---

## Payment Module

Responsible for:

- Payment creation.
- Payment tracking.
- Refund processing.
- Payment reconciliation.

---

## Notification Module

Responsible for:

- Email notifications.
- SMS notifications.
- Push notifications.
- Notification templates.
- Delivery tracking.

---

## Administration Module

Responsible for:

- Provider configuration.
- Administrative operations.
- Platform configuration.
- Monitoring support.

---

## Provider Integration Module

Responsible for communicating with external providers.

It exposes a unified interface to the rest of the application while hiding provider-specific implementation details.

Responsibilities include:

- Flight provider adapters.
- Hotel provider adapters.
- Payment gateway adapters.
- Notification provider adapters.

---

## Shared Infrastructure

Provides reusable technical capabilities used by all modules, including:

- Logging.
- Configuration.
- Validation.
- Exception handling.
- Domain events.
- Background job publishing.
- Caching abstraction.
- Common utilities.

---

## Persistence Layer

Responsible for:

- Data persistence.
- Repository implementations.
- Database access.
- Transaction management.

Business modules depend on repository interfaces rather than direct database access.

---

# 4. Component Diagram

````mermaid
flowchart TB

    API["API Layer"]

    Auth["Authentication Module"]
    Customer["Customer Module"]
    Traveler["Traveler Module"]
    Search["Search Module"]
    Booking["Booking Module"]
    Payment["Payment Module"]
    Notification["Notification Module"]
    Admin["Administration Module"]

    Providers["Provider Integration Module"]

    Shared["Shared Infrastructure"]

    Persistence["Persistence Layer"]

    %% API

    API --> Auth
    API --> Customer
    API --> Traveler
    API --> Search
    API --> Booking
    API --> Payment
    API --> Notification
    API --> Admin

    %% Business Relationships

    Booking --> Traveler
    Booking --> Payment
    Booking --> Notification

    Search --> Providers
    Booking --> Providers
    Payment --> Providers
    Notification --> Providers

    %% Persistence

    Auth --> Persistence
    Customer --> Persistence
    Traveler --> Persistence
    Search --> Persistence
    Booking --> Persistence
    Payment --> Persistence
    Notification --> Persistence
    Admin --> Persistence

    %% Shared Infrastructure

    Shared -.-> Auth
    Shared -.-> Customer
    Shared -.-> Traveler
    Shared -.-> Search
    Shared -.-> Booking
    Shared -.-> Payment
    Shared -.-> Notification
    Shared -.-> Admin
````

---

# 5. Component Relationships

## API Layer

The API Layer delegates requests to the appropriate business module.

It never contains business logic.

---

## Search Module

The Search Module communicates with the Provider Integration Module to retrieve travel offers.

It does not create bookings.

---

## Booking Module

The Booking Module coordinates the booking lifecycle.

It uses:

- Traveler Module
- Payment Module
- Notification Module
- Provider Integration Module

---

## Payment Module

The Payment Module communicates with external payment providers through the Provider Integration Module.

---

## Notification Module

The Notification Module sends customer communications through the Provider Integration Module.

---

## Provider Integration Module

Acts as the single integration boundary between the Backend Application and external systems.

Business modules never communicate directly with external providers.

---

## Shared Infrastructure

Provides cross-cutting technical capabilities used by all business modules.

---

## Persistence Layer

The Persistence Layer provides data access for all modules while isolating database implementation details.

---

# 6. Architectural Principles

The Backend Application follows these architectural principles:

- Single Responsibility Principle.
- Separation of Concerns.
- Low Coupling.
- High Cohesion.
- Dependency Inversion.
- Modular Design.
- Explicit Module Boundaries.

Business modules communicate through well-defined interfaces rather than direct implementation dependencies.

---

# 7. What This Diagram Does Not Define

This diagram does not describe:

- Internal classes.
- Controllers.
- Services.
- Repositories.
- Database schema.
- External provider APIs.
- Search algorithms.
- Booking workflow details.
- Payment workflow.
- Notification templates.

These topics are documented separately in their corresponding architecture documents.

---

# 8. Related Documents

```text
01-system-context.md
02-container-diagram.md
04-functional-requirements.md
05-non-functional-requirements.md
08-search-architecture.md
09-booking-orchestration.md
10-caching-strategy.md
11-database-design.md
12-api-design.md
```
