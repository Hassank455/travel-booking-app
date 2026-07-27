## Cache Keys

Cache keys uniquely identify cached business data.

A well-designed cache key must represent the business meaning of the cached entry rather than the raw technical request that produced it.

Cache keys should be:

- Deterministic
- Stable
- Unique
- Predictable
- Versioned
- Based on business-relevant parameters
- Independent of request-specific metadata

Poor cache key design may cause either:

- Multiple cache entries for the same business request.
- Different business requests incorrectly sharing the same cache entry.

Both cases can negatively affect performance and correctness.

---

### Cache Key Structure

A cache key should clearly identify:

1. The business domain.
2. The cached resource type.
3. The cache key version.
4. The provider, when applicable.
5. A normalized request identifier.

Recommended examples:

```text
flight-search:v1:{requestHash}
hotel-search:v1:{requestHash}
flight-provider:{providerId}:v1:{requestHash}
hotel-provider:{providerId}:v1:{requestHash}
provider:{providerId}:metadata:v1
reference:airports:v1
session:{sessionId}
```

The version component allows the platform to safely change normalization rules or cached response structures without conflicting with previously generated cache entries.

For example:

```text
flight-search:v1:{hash}
```

may later become:

```text
flight-search:v2:{hash}
```

Old entries can remain until their TTL expires without affecting the new cache format.

---

## Search Cache Key Normalization

Search cache keys must be generated from normalized business parameters rather than raw request payloads.

Two requests may look technically different while representing the same business search.

For example, the following flight search requests are semantically identical:

```json
{
  "origin": "amm",
  "destination": "DXB",
  "departureDate": "2026-08-10",
  "adults": 1
}
```

```json
{
  "destination": "DXB",
  "origin": " AMM ",
  "departureDate": "2026-08-10T00:00:00.000Z",
  "adults": "1"
}
```

If the cache key is generated directly from the raw request, the platform may generate different cache entries for the same search.

This results in:

```text
Same Business Search
        ↓
Different Raw Requests
        ↓
Different Cache Keys
        ↓
Repeated Provider Calls
        ↓
Lower Cache Hit Rate
```

Before creating the cache key, the platform should normalize the search request into a canonical business representation.

---

### Normalization Rules

Typical normalization rules include:

#### Trim Whitespace

Values such as:

```text
" AMM "
```

should become:

```text
"AMM"
```

---

#### Normalize Letter Case

Business codes should use a consistent case.

Examples:

```text
amm → AMM
usd → USD
economy → ECONOMY
```

This commonly applies to:

- Airport codes
- Airline codes
- Currency codes
- Country codes
- Cabin classes
- Provider identifiers

---

#### Normalize Data Types

Equivalent values must use the same data type.

Examples:

```text
"2" → 2
"true" → true
null children → 0
```

The platform should perform explicit validation and conversion before generating the key.

---

#### Normalize Dates

Dates should use a consistent business format.

For date-based flight and hotel searches, the platform commonly normalizes values to:

```text
YYYY-MM-DD
```

For example:

```text
2026-08-10T00:00:00.000Z
```

becomes:

```text
2026-08-10
```

Date normalization must respect the business timezone and should avoid unintentionally converting the date to the previous or next calendar day.

---

#### Apply Default Values

Omitted optional values and explicitly provided defaults should produce the same normalized request.

These requests may be equivalent:

```json
{
  "adults": 1
}
```

```json
{
  "adults": 1,
  "children": 0,
  "infants": 0,
  "cabinClass": "ECONOMY"
}
```

The normalized request should explicitly include the defaults:

```json
{
  "adults": 1,
  "children": 0,
  "infants": 0,
  "cabinClass": "ECONOMY"
}
```

---

#### Sort Unordered Collections

When the order of values does not affect the business request, collections should be sorted.

For example:

```json
{
  "airlines": ["QR", "EK"]
}
```

and:

```json
{
  "airlines": ["EK", "QR"]
}
```

should normalize to:

```json
{
  "airlines": ["EK", "QR"]
}
```

This may apply to:

- Airline filters
- Provider identifiers
- Hotel amenities
- Meal types
- Star ratings
- Allowed cabin classes

---

#### Remove Duplicate Values

Duplicate values should be removed before generating the cache key.

For example:

```json
{
  "airlines": ["EK", "QR", "EK"]
}
```

should become:

```json
{
  "airlines": ["EK", "QR"]
}
```

---

#### Remove Non-Business Fields

Request-specific metadata that does not affect the search result should not be included in the cache key.

Typical fields to exclude include:

- Request ID
- Correlation ID
- Trace ID
- Device ID
- Request timestamp
- Logging metadata
- Client-generated page request ID

Including these fields would create a new cache entry for every request, even when the underlying search is identical.

---

### Business-Relevant Parameters

The cache key must include every parameter that can materially change the provider response or final search result.

For flight searches, this commonly includes:

- Origin
- Destination
- Departure date
- Return date
- Trip type
- Adult count
- Child count
- Infant count
- Cabin class
- Currency
- Passenger nationality, when pricing depends on it
- Residency, when required by the provider
- Direct-flight preference, when sent to providers
- Provider selection strategy, when it affects the result set

For hotel searches, this commonly includes:

- Destination identifier
- Check-in date
- Check-out date
- Room count
- Adults per room
- Child ages per room
- Currency
- Guest nationality
- Residency
- Market
- Language, when it changes the provider response
- Provider selection strategy

A key that omits a business-relevant parameter may cause incorrect cache reuse.

For example, this key is not sufficient:

```text
flight-search:AMM:DXB
```

It ignores:

- Travel dates
- Passenger count
- Cabin class
- Currency
- Trip type

Different searches could therefore incorrectly receive the same cached result.

---

### Flight Search Normalization Example

Raw request:

```json
{
  "origin": " amm ",
  "destination": "dxb",
  "departureDate": "2026-08-10T00:00:00.000Z",
  "adults": "2",
  "children": null,
  "cabinClass": "economy",
  "airlines": ["QR", "EK", "QR"],
  "currency": "usd",
  "requestId": "req-789"
}
```

Normalized request:

```json
{
  "origin": "AMM",
  "destination": "DXB",
  "departureDate": "2026-08-10",
  "returnDate": null,
  "tripType": "ONE_WAY",
  "adults": 2,
  "children": 0,
  "infants": 0,
  "cabinClass": "ECONOMY",
  "airlines": ["EK", "QR"],
  "currency": "USD"
}
```

The `requestId` is removed because it does not affect the search result.

The normalized request can then be serialized deterministically and hashed.

Resulting key:

```text
flight-search:v1:8a23f4b91d...
```

---

### Hotel Search Normalization Example

Raw request:

```json
{
  "city": " Dubai ",
  "checkIn": "2026-09-01T00:00:00.000Z",
  "checkOut": "2026-09-05T00:00:00.000Z",
  "rooms": [
    {
      "adults": "2",
      "childrenAges": [8, 4]
    }
  ],
  "currency": "usd"
}
```

Normalized request:

```json
{
  "destination": "DUBAI",
  "checkIn": "2026-09-01",
  "checkOut": "2026-09-05",
  "rooms": [
    {
      "adults": 2,
      "childrenAges": [4, 8]
    }
  ],
  "currency": "USD"
}
```

Child ages are sorted only when their order does not carry business meaning.

However, rooms themselves should not always be reordered blindly. If room-specific occupancy must remain associated with a particular room, each room must first be normalized carefully before deciding whether room ordering is semantically irrelevant.

---

## Deterministic Serialization

After normalization, the request must be serialized in a deterministic manner.

This means that equivalent normalized objects must always produce exactly the same serialized representation.

For example, these objects are equivalent:

```json
{
  "origin": "AMM",
  "destination": "DXB"
}
```

```json
{
  "destination": "DXB",
  "origin": "AMM"
}
```

But direct serialization may produce different strings because the property order is different.

The platform should therefore:

- Construct a canonical object with a predefined field order.
- Sort nested unordered collections.
- Remove undefined fields.
- Represent missing optional values consistently.
- Use stable serialization before hashing.

The same normalization and serialization logic must be used for both cache reads and cache writes.

Any difference between the two paths would cause continuous cache misses.

---

## Hashed Search Keys

Search requests may contain many parameters.

Embedding all parameters directly in the key can produce long and difficult-to-manage keys.

For example:

```text
flight-search:AMM:DXB:2026-08-10:2026-08-20:2:1:0:ECONOMY:USD:EK,QR
```

A hashed key is generally easier to manage:

```text
flight-search:v1:{normalizedRequestHash}
```

Advantages include:

- Fixed key length
- Safer handling of special characters
- Easier inclusion of many search parameters
- Stable key structure
- Reduced risk of oversized keys

The hash must be generated from the normalized and deterministically serialized request, never from the raw request body.

For operational visibility, the platform may store lightweight metadata alongside the cached response, such as:

```json
{
  "requestHash": "8a23f4b91d...",
  "searchType": "FLIGHT",
  "createdAt": "2026-08-01T10:00:00Z",
  "expiresAt": "2026-08-01T10:03:00Z"
}
```

Sensitive request data should not be exposed through cache key names or unnecessary metadata.

---

## Provider-Level Cache

A Provider-Level Cache stores the normalized response of each external provider separately.

Instead of caching only the final aggregated search result, the platform may cache each provider response using a provider-specific key.

Examples:

```text
flight-provider:amadeus:v1:{requestHash}
flight-provider:duffel:v1:{requestHash}
hotel-provider:hotelbeds:v1:{requestHash}
hotel-provider:expedia:v1:{requestHash}
```

The cached value should normally contain the provider response after it has been mapped into the platform's normalized internal model.

It should not necessarily contain the raw provider payload.

Recommended flow:

```text
Normalized Search Request
        ↓
Generate Provider-Specific Key
        ↓
Check Provider Cache
        ↓
Cache Hit
        │
        └── Return Normalized Provider Offers
        ↓
Cache Miss
        ↓هو
Call Provider
        ↓
Validate Provider Response
        ↓
Normalize Provider Response
        ↓
Store Provider Result
        ↓
Return Normalized Provider Offers
```

---

### Why Provider-Level Cache Is Useful

Provider-level caching provides several benefits.

#### Independent Provider Reuse

If one provider already has a cached response, the platform can reuse it while calling only the providers whose cache entries are missing.

For example:

```text
Amadeus  → Cache Hit
Duffel   → Cache Miss
Sabre    → Cache Hit
```

The platform only needs to call Duffel.

Without provider-level caching, it may need to call all providers again when the final aggregated cache is unavailable.

---

#### Different TTL Per Provider

Different providers may have different:

- Data freshness characteristics
- Rate limits
- Usage costs
- Contract restrictions
- Response times
- Reliability levels

Provider-level entries may therefore use different TTL policies.

For example:

```text
Provider A → Very short TTL
Provider B → Short TTL
Provider C → Cache disabled
```

The actual TTL values should remain configurable and should follow provider contracts and business freshness requirements.

---

#### Better Fault Isolation

If one provider becomes unavailable, the platform may continue building a partial result from:

- Cached data from that provider, when still acceptable.
- Fresh or cached results from other providers.

This helps avoid treating one provider failure as a complete search failure.

Any use of stale provider data must follow an explicit stale-data policy and must not bypass offer revalidation before booking.

---

#### Provider-Specific Observability

Provider-level caching allows the platform to measure:

- Cache hit rate per provider
- Cache miss rate per provider
- Provider response latency
- Provider error rate
- Number of avoided provider calls
- Cache freshness
- Partial search behavior

This makes it easier to understand the cost and performance characteristics of each integration.

---

#### Partial Search Reconstruction

Suppose the final aggregated search cache has expired, but some provider-level entries are still valid.

The Search Orchestrator can reuse those entries and call only the missing providers before reconstructing the aggregated result.

Example:

```text
Aggregated Cache Miss
        ↓
Check Provider-Level Cache
        ↓
Provider A Hit
Provider B Miss
Provider C Hit
        ↓
Call Provider B Only
        ↓
Merge A, B, and C Results
        ↓
Deduplicate and Rank
        ↓
Store New Aggregated Result
```

This can significantly reduce external provider traffic.

---

## Aggregated Search Cache

An Aggregated Search Cache stores the final search result after provider responses have been:

- Normalized
- Combined
- Deduplicated
- Filtered
- Enriched
- Ranked

Example key:

```text
flight-search:v1:{requestHash}
```

or:

```text
hotel-search:v1:{requestHash}
```

Recommended flow:

```text
Search Request
        ↓
Normalize Request
        ↓
Check Aggregated Cache
        ↓
Cache Hit
        └── Return Final Results
        ↓
Cache Miss
        ↓
Resolve Provider Results
        ↓
Normalize and Aggregate
        ↓
Deduplicate and Rank
        ↓
Store Aggregated Result
        ↓
Return Final Results
```

The aggregated cache provides the fastest search response because the final platform-ready result is already prepared.

---

## Provider-Level Cache vs Aggregated Cache

| Aspect                         | Provider-Level Cache                | Aggregated Search Cache        |
| ------------------------------ | ----------------------------------- | ------------------------------ |
| Cached content                 | One provider's normalized response  | Final combined search response |
| Key includes provider ID       | Yes                                 | Usually no                     |
| Main benefit                   | Reuse individual provider responses | Fast final response            |
| Provider-specific TTL          | Supported                           | Not directly                   |
| Partial provider reuse         | Yes                                 | No                             |
| Aggregation required after hit | Yes                                 | No                             |
| Implementation complexity      | Higher                              | Lower                          |
| Observability per provider     | Strong                              | Limited                        |
| Best use case                  | Multi-provider optimization         | Fast repeated searches         |

---

## Using Both Cache Levels

The platform may use both provider-level and aggregated caching.

Recommended lookup flow:

```text
Search Request
        ↓
Normalize Request
        ↓
Check Aggregated Search Cache
        ↓
Hit ───────────────► Return Results
        ↓ Miss
Check Provider-Level Cache Entries
        ↓
Reuse Available Provider Results
        ↓
Call Missing Providers
        ↓
Normalize Provider Responses
        ↓
Store Provider-Level Entries
        ↓
Aggregate and Deduplicate
        ↓
Store Aggregated Search Result
        ↓
Return Results
```

This provides the best optimization opportunities but introduces additional complexity.

The platform must manage:

- Multiple cache entries for one search
- Different provider TTLs
- Partial cache hits
- Aggregated result expiration
- Provider enablement changes
- Provider-specific failures
- Cache observability

Because of this complexity, both cache levels should not be introduced automatically without a clear need.

---

## Recommended Adoption Strategy

For the initial implementation, the platform may begin with an Aggregated Search Cache.

This provides:

- Simpler implementation
- Easier invalidation
- Faster final search responses
- Lower operational complexity

Provider-Level Cache can be introduced when one or more of the following becomes important:

- Provider API costs are high.
- Rate limits are restrictive.
- Providers have significantly different TTL requirements.
- Provider calls have high latency.
- Partial provider reuse is valuable.
- Provider-specific observability is required.
- Provider failures frequently affect search completeness.

This allows the platform to avoid premature complexity while preserving a clear path for future optimization.

---

## Provider Selection and Cache Keys

If the set of active providers affects the aggregated search result, the provider selection context must be represented in the aggregated cache key.

For example, these searches may not be equivalent:

```text
Search using Providers A and B
```

```text
Search using Providers A, B, and C
```

Possible strategies include:

### Provider Set in the Key

```text
flight-search:v1:{providerSetHash}:{requestHash}
```

The provider set should be normalized and sorted before hashing.

---

### Provider Strategy Version

```text
flight-search:v1:strategy-v3:{requestHash}
```

When provider routing rules change, the strategy version changes.

This avoids reusing aggregated results generated under an older provider selection policy.

---

### Invalidate on Provider Configuration Change

The platform may invalidate affected aggregated entries when:

- A provider is enabled.
- A provider is disabled.
- Provider routing rules change.
- Supported markets change.
- Provider capabilities change.

The appropriate approach depends on the scale of the cache and the provider configuration model.

---

## Filters and Cache Keys

Not every client filter must be included in the search cache key.

The platform should distinguish between two filter categories.

### Provider-Level Filters

These parameters affect the provider request or the provider response and must usually be included in the cache key.

Examples include:

- Origin and destination
- Dates
- Passenger count
- Cabin class
- Room occupancy
- Guest nationality
- Currency
- Direct-flight-only preference, when sent to the provider
- Refundability, when sent to the provider

---

### Post-Search Filters

These parameters can be applied after retrieving the cached result and may not need to be included in the key.

Examples may include:

- Sort by price
- Sort by duration
- Local UI price range
- Hide specific airlines locally
- Client-side amenity filters
- Presentation preferences

Excluding post-search filters from the key can improve cache reuse.

However, if a filter changes the data requested from the provider, affects pagination, or changes the result population, it must be included.

---

## Pagination Considerations

Pagination requires careful cache key design.

If the platform caches the complete normalized result set and performs pagination internally, the page number may not need to be part of the provider search cache key.

Example:

```text
flight-search:v1:{requestHash}
```

The application can then return:

```text
Page 1
Page 2
Page 3
```

from the same cached result.

However, if the provider itself returns paginated results and each page requires a separate provider call, the pagination cursor or page identifier must be represented in the provider-level key.

Example:

```text
hotel-provider:{providerId}:v1:{requestHash}:cursor:{cursorHash}
```

Provider cursors may contain sensitive or oversized values and should therefore be hashed before being used in cache keys.

---

## Localization Considerations

Language should only be included in the cache key when it changes the cached response.

For example, language may affect:

- Hotel descriptions
- Amenity labels
- Cancellation policy text
- Destination names

If the cached internal model contains language-independent codes and localization is applied later, language should not be part of the key.

This improves cache reuse across languages.

If the provider returns localized content that is stored directly, language must be included:

```text
hotel-search:v1:lang:en:{requestHash}
hotel-search:v1:lang:ar:{requestHash}
```

---

## Currency Considerations

Currency must be included in the cache key when it affects:

- Provider pricing
- Currency conversion
- Taxes
- Fees
- Final displayed amounts

Example:

```text
flight-search:v1:currency:USD:{requestHash}
```

Alternatively, currency may already be part of the normalized request before hashing.

Different currency searches must not reuse the same cached price response unless the platform caches a base currency and performs a trusted conversion afterward.

---

## Cache Key Safety

Cache keys should not expose sensitive data.

Avoid placing the following values directly in cache key names:

- Customer names
- Email addresses
- Phone numbers
- Passport numbers
- Payment details
- Authentication tokens
- Full session payloads
- Raw provider credentials

When user-specific context must be represented, use a safe internal identifier or hash where appropriate.

For example:

```text
session:{sessionId}
```

is preferable to:

```text
session:{customerEmail}
```

---

## Cache Key Ownership

Each module should own the construction of cache keys for its business data.

Examples:

- Search Module owns aggregated search keys.
- Provider Integration owns provider-level keys.
- Authentication Module owns session keys.
- Reference Data Module owns reference-data keys.

Cache key generation should not be scattered across controllers or unrelated services.

A shared key builder may provide technical utilities such as:

- Versioning
- Hash generation
- Stable serialization
- Namespace formatting

However, each business module remains responsible for deciding which parameters are meaningful.

---

## Architectural Decisions

The platform follows these cache key decisions:

- Cache keys are generated from normalized business data.
- Raw HTTP requests are never used directly as cache keys.
- Search request serialization must be deterministic.
- Search keys include all parameters that materially affect results.
- Request-specific metadata is excluded.
- Unordered collections are sorted and deduplicated.
- Search cache keys are versioned.
- Long search parameter sets are represented using hashes.
- Provider-level keys include the provider identity.
- Aggregated keys represent the provider selection strategy when it affects results.
- Cache keys must not expose sensitive customer or payment data.
- The same normalization process is used for cache reads and writes.
- Provider-level caching may be added incrementally based on measurable operational needs.
