# ADR-002: Use Provider Adapter Pattern

## Status

Accepted

## Context

The Travel Booking Platform integrates with multiple external providers for flights and hotels.

Different providers expose different:

- API contracts.
- Authentication mechanisms.
- Request formats.
- Response structures.
- Error formats.
- Status values.
- Offer identifiers.
- Booking identifiers.

If the core Search or Booking modules communicate directly with provider-specific APIs, provider implementation details would leak into the business domain.

This would make the platform tightly coupled to vendors such as Duffel or LiteAPI and would make future provider replacement or extension more difficult.

## Decision

The platform will use a Provider Adapter Pattern.

Each external provider will be accessed through an adapter responsible for translating between:

- Platform models.
- Provider-specific requests.
- Provider-specific responses.
- Provider-specific errors.

Conceptually:

```text
Search / Booking Module
        ↓
Provider Interface
        ↓
Provider Adapter
        ↓
External Provider API
```

Examples:

```text
FlightProvider
    └── DuffelAdapter

HotelProvider
    └── LiteAPIAdapter
```

The business domain will depend on platform abstractions rather than vendor-specific SDKs or response models.

## Alternatives Considered

### Direct Provider Integration

The Search and Booking modules could call provider APIs directly.

**Pros**

- Faster initial implementation.
- Fewer abstraction layers.

**Cons**

- Strong provider coupling.
- Provider-specific logic spreads across the application.
- Harder testing.
- Harder provider replacement.
- Difficult multi-provider aggregation.

### Shared Generic Provider Service Without Adapters

All provider-specific logic could be placed inside one large integration service.

**Pros**

- Centralized integration logic.
- Simple initial structure.

**Cons**

- Becomes difficult to maintain as providers increase.
- Provider-specific conditions accumulate in one component.
- Violates clear responsibility boundaries.

## Consequences

### Positive

- Core business modules remain provider-independent.
- New providers can be added with limited impact.
- Provider-specific errors can be normalized.
- Testing becomes easier using provider interface mocks.
- Search aggregation across multiple providers becomes simpler.

### Negative

- Additional abstraction and mapping code is required.
- Provider capabilities may not always map perfectly to one shared interface.
- Provider-specific features may require controlled extensions.

## Related Documents

- `08-search-architecture.md`
- `09-booking-orchestration.md`
- `12-database-design.md`
- `13-security.md`