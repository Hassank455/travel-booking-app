# 08. Search Architecture

## 1. Purpose

This document describes the search architecture of the Travel Booking Platform.

The platform supports searching for:

- Flights.
- Hotels.

Search results are retrieved from multiple external providers, normalized into a unified platform model, aggregated, filtered, ranked, cached, and returned to the customer.

This document focuses on the search flow and its architectural responsibilities.

It does not define:

- Provider-specific API details.
- Database schema.
- Booking confirmation workflow.
- Payment processing.
- User interface implementation.

---

## 2. Search Goals

The search architecture should:

- Return relevant offers from multiple providers.
- Hide provider-specific differences from the customer.
- Reduce response time where possible.
- Continue returning useful results when one provider fails.
- Support filtering and sorting.
- Avoid unnecessary provider requests.
- Keep search results temporary and revalidatable.
- Scale independently from other backend workloads when necessary.

---

## 3. Core Principles

### 3.1 Search Results Are Temporary

A search result represents availability and pricing at the time it was retrieved.

It is not a confirmed reservation.

Every selected offer must be revalidated before booking.

### 3.2 Providers Are Independent

A failure in one provider should not prevent the platform from returning valid results from other providers.

### 3.3 Provider Data Must Be Normalized

Each provider may use different:

- Field names.
- Response structures.
- Price formats.
- Currency representations.
- Hotel identifiers.
- Airport identifiers.
- Fare rules.
- Room structures.

Provider responses must be converted into a unified internal offer model before they are returned to the customer.

### 3.4 Search Must Support Partial Results

When one or more providers fail or time out, the platform may return available results from successful providers.

### 3.5 Search and Booking Are Separate

The Search Module finds and presents offers.

It does not confirm reservations.

The Booking Module owns:

- Offer revalidation.
- Availability validation.
- Final price validation.
- Provider confirmation.

---

## 4. Main Search Components

### API Layer

Receives search requests from:

- Customer Web Application.
- Customer Mobile Application.

Responsibilities:

- Authentication when applicable.
- Request validation.
- Request routing.
- Response serialization.
- Calling the Search Module.

The API Layer contains no search business logic.

### Search Module

Owns the search use cases.

Responsibilities:

- Validate search criteria.
- Build a normalized search query.
- Check cached results.
- Invoke the Search Orchestrator.
- Apply platform-level filters.
- Sort and rank results.
- Return a unified search response.

### Search Orchestrator

Coordinates the complete provider search process.

Responsibilities:

- Determine eligible providers.
- Send provider requests in parallel.
- Apply provider-specific timeouts.
- Collect successful responses.
- Handle partial failures.
- Send responses for normalization.
- Aggregate normalized offers.
- Return the final provider result set.

The Search Orchestrator coordinates work but should not contain provider-specific mapping logic.

### Provider Integration Module

Provides a stable interface between the Search Module and external travel providers.

It contains provider adapters such as:

```text
FlightProviderAdapter
HotelProviderAdapter
```

Each adapter is responsible for:

- Translating the platform search request into the provider request format.
- Calling the external provider.
- Translating provider errors into platform errors.
- Returning provider data for normalization.
- Preserving provider references required for revalidation.

### Offer Normalizer

Transforms provider-specific offers into the platform's unified offer model.

Responsibilities:

- Normalize identifiers.
- Normalize price structures.
- Normalize currencies.
- Normalize dates and times.
- Normalize airports and locations.
- Normalize flight segments.
- Normalize room and rate data.
- Normalize policies and restrictions.

### Offer Aggregator

Combines normalized offers from multiple providers.

Responsibilities:

- Merge result collections.
- Detect equivalent offers.
- Deduplicate when applicable.
- Preserve provider source information.
- Group comparable offers.
- Produce a unified result set.

### Ranking and Filtering Component

Applies platform-level presentation rules.

Possible filters include:

- Price range.
- Airline.
- Number of stops.
- Departure time.
- Flight duration.
- Hotel rating.
- Property type.
- Amenities.
- Cancellation policy.

Possible sorting options include:

- Recommended.
- Lowest price.
- Shortest duration.
- Earliest departure.
- Highest hotel rating.

### Search Cache

Stores temporary search results to reduce repeated external provider calls.

The Search Module may use cached results when:

- The normalized search criteria match.
- The cache entry has not expired.
- The cache policy allows reuse.

Cached offers must still be revalidated before booking.

Detailed cache design is documented in:

[Caching Strategy](10-caching-strategy.md)

---

## 5. High-Level Search Flow

See the diagram:

**Diagram:** [Search Flow](./diagrams/search/search-flow.mmd)

---

## 6. Flight Search Flow

A flight search request may include:

- Origin.
- Destination.
- Departure date.
- Return date when applicable.
- Traveler count.
- Traveler types.
- Cabin class.
- Direct-flight preference.
- Currency.
- Market or locale.

See the sequence diagram:

**Diagram:** [Flight Search Sequence](./diagrams/search/sequence/flight-search-sequence.mmd)

---

## 7. Hotel Search Flow

A hotel search request may include:

- Destination.
- Check-in date.
- Check-out date.
- Number of rooms.
- Adult count.
- Child count and ages.
- Currency.
- Market or locale.
- Optional filters.

**Diagram:** [Hotel Search Sequence](./diagrams/search/sequence/hotel-search-sequence.mmd) 

---

## 8. Search Request Normalization

Before using a search request, the platform should convert it into a canonical form.

Examples:

- Airport codes converted to uppercase.
- Dates converted into a standard date format.
- Currency converted to a standard currency code.
- Traveler types ordered consistently.
- Hotel destination resolved to a stable location identifier.
- Optional values replaced with explicit defaults.
- Filters sorted consistently.

This allows logically equivalent searches to produce the same cache key.

Example:

```text
Flight Search

Origin: AMM
Destination: DXB
Departure Date: 2026-08-10
Adults: 2
Children: 0
Cabin: ECONOMY
Currency: USD
```

A normalized search request should be independent from:

- Request field order.
- Client platform.
- UI representation.
- Provider request formats.

---

## 9. Unified Offer Model

The platform should not expose raw provider responses directly.

Every provider offer is converted into a normalized offer.

### Common Offer Fields

```text
Offer
├── Offer ID
├── Offer Type
├── Provider ID
├── Provider Offer Reference
├── Total Price
├── Currency
├── Expiration Time
├── Policies
├── Availability Information
└── Provider Metadata
```

### Flight Offer Model

```text
Flight Offer
├── Offer Information
├── Itinerary
│   └── Flight Segments
├── Travelers Pricing
├── Fare Class
├── Cabin Class
├── Baggage Allowance
├── Fare Rules
└── Ticketing Information
```

### Hotel Offer Model

```text
Hotel Offer
├── Offer Information
├── Hotel
├── Stay Period
├── Room
├── Rate Plan
├── Occupancy
├── Meal Plan
├── Cancellation Policy
└── Property Charges
```

---

## 10. Offer Identification

Every offer returned to the client should have a platform-generated offer identifier.

The identifier may reference temporary server-side search data.

It should allow the platform to retrieve:

- Provider identifier.
- Provider offer reference.
- Search context.
- Normalized offer snapshot.
- Expiration information.
- Data required for revalidation.

The client should not be responsible for understanding provider-specific identifiers.

---

## 11. Parallel Provider Execution

Provider searches should normally execute in parallel.

Sequential execution:

```text
Provider A → Provider B → Provider C
```

increases total response time.

Parallel execution:

```text
Provider A
Provider B
Provider C
```

allows the overall search time to be closer to the slowest accepted provider response rather than the sum of all provider response times.

Each provider call should have:

- A timeout.
- Error isolation.
- Provider-specific monitoring.
- A clear success or failure result.

---

## 12. Provider Selection

The Search Orchestrator should determine which providers are eligible for a search.

Provider selection may depend on:

- Search type.
- Origin and destination.
- Customer market.
- Supported currency.
- Provider availability.
- Provider contract.
- Provider health.
- Rate limits.
- Administrative configuration.

Not every search must be sent to every provider.

---

## 13. Timeouts and Partial Results

Each external provider call should have a bounded response time.

Example behavior:

```text
Provider A: Success
Provider B: Timeout
Provider C: Success
```

The search may return results from Provider A and Provider C.

The response may include metadata indicating that the result is partial.

A provider timeout should not automatically fail the complete search request.

The complete request may fail only when:

- No provider returns usable results.
- The search request is invalid.
- A critical internal error prevents result processing.

---

## 14. Error Handling

### Validation Error

The customer supplied invalid search criteria.

Examples:

- Invalid dates.
- Missing origin.
- Invalid guest count.

### Provider Error

A provider rejected or failed the request.

Examples:

- Provider authentication failure.
- Rate limit exceeded.
- Invalid provider response.
- Provider service unavailable.

### Timeout Error

The provider did not respond within the allowed time.

### Normalization Error

The provider response could not be converted into the platform model.

The affected offer or provider result may be excluded without failing successful results from other providers.

### No Results

The search completed successfully but no matching offers were found.

This is a valid business result, not necessarily a system failure.

---

## 15. Deduplication

Multiple providers may return equivalent travel products.

Examples:

- The same flight itinerary.
- The same hotel and room rate.
- The same rate plan sold through different suppliers.

The platform may:

- Keep all provider offers.
- Group equivalent offers.
- Display the lowest-priced option.
- Rank a preferred provider first.

Deduplication must preserve provider traceability.

Two offers should not be treated as equivalent unless their important commercial conditions are comparable.

These conditions may include:

- Total price.
- Currency.
- Fare or rate rules.
- Baggage.
- Cancellation policy.
- Payment conditions.
- Included taxes and fees.

---

## 16. Filtering and Sorting

Filtering and sorting should operate on normalized fields.

This prevents provider differences from affecting platform behavior.

### Flight Filters

- Price range.
- Stops.
- Airlines.
- Airports.
- Departure time.
- Arrival time.
- Duration.
- Baggage.
- Refundability.

### Hotel Filters

- Price range.
- Star rating.
- Guest rating.
- Property type.
- Amenities.
- Meal plan.
- Distance.
- Free cancellation.
- Pay-at-property availability.

Filtering should not alter the original normalized offer.

---

## 17. Ranking

The default ranking may consider several signals.

Possible signals include:

- Price.
- Travel duration.
- Number of stops.
- Provider reliability.
- Cancellation flexibility.
- Hotel rating.
- Offer completeness.
- Platform commercial preferences.

Commercial influence must not make sponsored results appear as neutral recommendations.

Promoted results should be identified clearly.

---

## 18. Search Cache Interaction

The Search Module checks the cache before calling providers.

### Cache Hit

```text
Search Request
    ↓
Normalized Cache Key
    ↓
Cached Search Result
    ↓
Filtering and Sorting
    ↓
Response
```

### Cache Miss

```text
Search Request
    ↓
Provider Search
    ↓
Normalization
    ↓
Aggregation
    ↓
Store in Cache
    ↓
Response
```

The cache stores temporary search data.

It does not guarantee:

- Current availability.
- Final price.
- Successful booking.

Offer revalidation remains mandatory.

---

## 19. Guest and Authenticated Searches

The search process should not require authentication unless business policy requires it.

Guest searches and authenticated searches may use the same normalized search cache.

Customer-specific information should not be included in a shared cache entry.

Examples of customer-specific information:

- Loyalty benefits.
- Private pricing.
- Personalized recommendations.
- Customer-specific discounts.

Such data should be applied separately or cached using a customer-safe strategy.

---

## 20. Search Result Pagination

The platform may paginate search results after aggregation.

Pagination must operate on a stable search result snapshot.

The platform should avoid rerunning all providers for every page request.

A search session or search result identifier may be used to retrieve later pages from temporary storage.

---

## 21. Search Session

A search session represents the temporary context of a customer search.

It may contain:

- Search identifier.
- Normalized search criteria.
- Aggregated offer references.
- Creation time.
- Expiration time.
- Result metadata.
- Provider execution summary.

A search session is temporary and is not a booking.

---

## 22. Observability

Search operations should be traceable using a correlation identifier.

Useful metrics include:

- Total search response time.
- Cache hit rate.
- Provider response time.
- Provider success rate.
- Provider timeout rate.
- Number of offers returned.
- Normalization failure count.
- Searches with partial results.
- Searches with no results.

Logs should include provider and request metadata without exposing sensitive customer information.

---

## 23. Security Considerations

The search architecture should:

- Protect provider credentials.
- Validate all customer inputs.
- Limit abusive search traffic.
- Avoid exposing raw provider errors.
- Avoid exposing internal provider credentials or contracts.
- Apply rate limiting when necessary.
- Sanitize provider responses.
- Protect temporary offer references from tampering.

---

## 24. Architectural Decisions

### Search Is Read-Oriented

The search flow primarily reads external inventory and returns temporary offers.

It does not create confirmed business reservations.

### Search Results Are Not Persisted as Confirmed Inventory

External flight and hotel inventory changes frequently.

The platform should not treat search results as permanently owned inventory.

Temporary search results may be cached.

Confirmed bookings are persisted separately.

### Provider Adapters Are the Integration Boundary

The Search Module must not communicate directly with external provider APIs.

All provider communication goes through the Provider Integration Module.

### Normalization Happens Before Aggregation

Raw provider responses cannot be reliably compared.

They must first be normalized into the platform model.

### Cache Supports Search but Does Not Replace Revalidation

Caching improves performance and reduces provider calls.

It does not guarantee the commercial validity of an offer.

---

## 25. What This Document Does Not Define

This document does not define:

- Exact cache TTL values.
- Redis key formats.
- Booking confirmation steps.
- Payment orchestration.
- Database tables.
- Provider credentials.
- API endpoint contracts.
- Ranking algorithm weights.
- Retry configuration.
- Circuit breaker implementation.
- Deployment topology.

These topics are covered in separate documents.

---

## 26. Related Documents

- `01-product-overview.md`
- `02-scope.md`
- `03-user-roles.md`
- `04-functional-requirements.md`
- `05-non-functional-requirements.md`
- `07-domain-model.md`
- `09-booking-orchestration.md`
- `10-caching-strategy.md`
- `11-database-design.md`
- `12-api-design.md`
- `13-failure-scenarios.md`
- `14-observability.md`

---

## 27. Summary

The Search Architecture coordinates searches across multiple flight and hotel providers.

The main flow is:

```text
Request Validation
        ↓
Search Request Normalization
        ↓
Cache Lookup
        ↓
Provider Selection
        ↓
Parallel Provider Calls
        ↓
Offer Normalization
        ↓
Aggregation and Deduplication
        ↓
Filtering and Ranking
        ↓
Temporary Caching
        ↓
Unified Search Response
```

The architecture is designed around the following principles:

- Search offers are temporary.
- Providers are isolated.
- Provider data is normalized.
- Partial results are supported.
- Search and booking are separate.
- Cached offers must be revalidated.
- Provider-specific complexity remains inside the integration boundary.
