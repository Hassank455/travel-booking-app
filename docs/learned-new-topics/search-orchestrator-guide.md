# Orchestrator and Search Orchestrator

## 1. What Is an Orchestrator?

An **Orchestrator** is a component responsible for coordinating multiple services, operations, or external systems to complete one business workflow.

It does not necessarily perform all the work itself. Instead, it decides:

- Which services should be called
- In what order they should be called
- Whether calls should run sequentially or in parallel
- How failures and timeouts should be handled
- How results should be combined
- What final response should be returned

A useful analogy is an orchestra conductor. Each musician knows how to play an instrument, but the conductor coordinates everyone so that they produce one complete performance.

```text
Service A knows how to perform Task A
Service B knows how to perform Task B
Service C knows how to perform Task C

The Orchestrator coordinates all three services.
```

## 2. Why Use an Orchestrator?

Without an Orchestrator, a controller or service may become responsible for too many details:

```text
Controller
   ├── Calls Provider A
   ├── Calls Provider B
   ├── Handles Provider C timeout
   ├── Converts different responses
   ├── Removes duplicates
   ├── Sorts results
   ├── Writes to cache
   └── Builds the final response
```

With an Orchestrator:

```text
Controller
   │
   ▼
Orchestrator
   ├── Coordinates providers
   ├── Handles failures
   ├── Combines results
   └── Returns one final response
```

The controller remains focused on HTTP responsibilities:

- Reading request parameters
- Validating the request
- Calling the application layer
- Returning the HTTP response

# Search Orchestrator

## 3. What Is a Search Orchestrator?

A **Search Orchestrator** is an application component that coordinates the complete search workflow.

It is especially useful when search data comes from multiple sources.

For example, a flight and hotel booking platform may use:

```text
Flight Providers:
- Amadeus
- Travelport
- Kiwi
- Other flight suppliers

Hotel Providers:
- Hotel Supplier A
- Hotel Supplier B
- Hotel Supplier C
```

Each provider may have:

- A different API
- A different authentication method
- A different request structure
- A different response structure
- Different timeout limits
- Different pricing rules
- Different error formats

The Search Orchestrator hides these differences from the rest of the application.

## 4. High-Level Search Flow

```text
Client
  │
  ▼
Search Controller
  │
  ▼
Search Orchestrator
  │
  ├── Provider A
  ├── Provider B
  └── Provider C
  │
  ▼
Normalize Results
  │
  ▼
Merge Results
  │
  ▼
Remove Duplicates
  │
  ▼
Rank and Sort
  │
  ▼
Cache Results
  │
  ▼
Return Unified Response
```

# Main Responsibilities

## 5. Prepare the Search Request

The Orchestrator receives a normalized request from the controller.

```ts
interface FlightSearchQuery {
  origin: string;
  destination: string;
  departureDate: string;
  returnDate?: string;
  adults: number;
  children?: number;
  cabinClass?: 'ECONOMY' | 'BUSINESS' | 'FIRST';
  currency: string;
}
```

It may:

- Apply defaults
- Normalize airport or city codes
- Build a cache key
- Generate a search ID
- Decide which providers support the request

## 6. Check Redis Cache

Before calling external APIs, the Orchestrator can check Redis.

```text
Search Request
   │
   ▼
Generate Cache Key
   │
   ▼
Redis
   ├── Cache Hit  → Return cached results
   └── Cache Miss → Call providers
```

Example:

```text
flight-search:AMM:DXB:2026-08-10:1:ECONOMY:USD
```

Caching reduces:

- External API calls
- Provider cost
- Response time
- Repeated work

## 7. Select Providers

Not every provider must be called for every search.

Selection may depend on:

- Supported routes
- Country or market
- Search type
- Currency
- Provider health
- Provider reliability
- Provider cost
- Feature flags
- Rate limits

## 8. Call Providers in Parallel

Independent provider calls should usually run in parallel.

```ts
const results = await Promise.allSettled(
  providers.map((provider) => provider.search(query)),
);
```

`Promise.allSettled()` is usually safer than `Promise.all()` because one provider failure should not necessarily fail the entire search.

## 9. Apply Timeouts

A slow provider should not block the whole search.

```text
Provider A: 700 ms
Provider B: 1.2 seconds
Provider C: no response

Configured timeout: 3 seconds
```

After the timeout, Provider C can be ignored while the search continues with available results.

```ts
async function withTimeout<T>(
  promise: Promise<T>,
  timeoutMs: number,
): Promise<T> {
  return Promise.race([
    promise,
    new Promise<T>((_, reject) => {
      setTimeout(() => reject(new Error('Provider timeout')), timeoutMs);
    }),
  ]);
}
```

In production, cancel the underlying HTTP request when possible.

## 10. Handle Partial Failure

Example:

```text
Amadeus: Success
Travelport: Success
Provider C: Failure
```

The system can still return useful results.

This is called **graceful degradation**.

```json
{
  "searchId": "search_123",
  "partial": true,
  "providers": {
    "successful": 2,
    "failed": 1
  },
  "results": []
}
```

A complete failure is usually appropriate only when all providers fail.

## 11. Normalize Provider Responses

Different providers return different formats.

Provider A:

```json
{
  "fare": 120,
  "carrier": "QR",
  "from": "AMM",
  "to": "DXB"
}
```

Provider B:

```json
{
  "price": {
    "amount": 125,
    "currency": "USD"
  },
  "airline": "Qatar Airways",
  "origin": "AMM",
  "destination": "DXB"
}
```

Both should be converted into one internal model:

```ts
interface NormalizedFlightResult {
  provider: string;
  providerOfferId: string;
  airlineCode: string;
  airlineName: string;
  origin: string;
  destination: string;
  departureAt: string;
  arrivalAt: string;
  durationMinutes: number;
  stops: number;
  price: {
    amount: number;
    currency: string;
  };
}
```

Provider-specific transformation should normally live inside an Adapter or Mapper.

```text
Raw Provider Response
        │
        ▼
Provider Adapter
        │
        ▼
Normalized Internal Model
```

## 12. Merge Results

```text
Provider A: 50 results
Provider B: 30 results
Provider C: 20 results

Combined: 100 results
```

```ts
const mergedResults = successfulResponses.flatMap(
  (response) => response.results,
);
```

## 13. Remove Duplicates or Group Offers

The same flight or hotel may appear through multiple providers.

A flight deduplication key may use:

- Airline
- Flight number
- Origin
- Destination
- Departure time
- Arrival time

However, two offers for the same itinerary can still differ in:

- Price
- Baggage
- Refund rules
- Fare rules
- Provider fees

Therefore, grouping offers is often better than deleting duplicates completely.

```ts
interface FlightGroup {
  flightKey: string;
  itinerary: unknown;
  offers: Array<{
    provider: string;
    providerOfferId: string;
    price: number;
    baggage: unknown;
    fareRules: unknown;
  }>;
}
```

## 14. Rank and Sort Results

Common sorting options:

- Lowest price
- Shortest duration
- Earliest departure
- Fewest stops
- Highest hotel rating
- Closest hotel
- Best value

A best-value score may combine:

```text
Price Weight
+ Duration Weight
+ Stops Weight
+ Provider Reliability Weight
```

The exact formula is a business decision.

Ranking logic can be extracted into a dedicated service.

## 15. Cache the Final Results

```ts
await redis.set(
  cacheKey,
  JSON.stringify(searchResponse),
  {
    EX: 120,
  },
);
```

The cached payload may include:

- Search ID
- Results
- Provider offer IDs
- Creation time
- Expiration time
- Provider metadata

Provider offer IDs are important for later revalidation and booking.

## 16. Return One Unified Response

```json
{
  "searchId": "search_123",
  "partial": false,
  "currency": "USD",
  "total": 100,
  "results": [
    {
      "type": "FLIGHT",
      "airline": "Qatar Airways",
      "origin": "AMM",
      "destination": "DXB",
      "departureAt": "2026-08-10T10:00:00Z",
      "arrivalAt": "2026-08-10T13:00:00Z",
      "price": {
        "amount": 118,
        "currency": "USD"
      },
      "offers": [
        {
          "provider": "PROVIDER_A",
          "providerOfferId": "offer_123"
        }
      ]
    }
  ]
}
```

# Provider Adapter Pattern

## 17. Common Provider Interface

The Orchestrator should not contain provider-specific details.

```ts
interface FlightSearchProvider {
  readonly name: string;

  supports(query: FlightSearchQuery): boolean;

  search(
    query: FlightSearchQuery,
  ): Promise<NormalizedFlightResult[]>;
}
```

Each provider implements the same interface.

```ts
class AmadeusFlightProvider implements FlightSearchProvider {
  readonly name = 'AMADEUS';

  supports(query: FlightSearchQuery): boolean {
    return true;
  }

  async search(
    query: FlightSearchQuery,
  ): Promise<NormalizedFlightResult[]> {
    const rawResponse = await this.callAmadeus(query);
    return this.normalize(rawResponse);
  }

  private async callAmadeus(
    query: FlightSearchQuery,
  ): Promise<unknown> {
    return {};
  }

  private normalize(
    rawResponse: unknown,
  ): NormalizedFlightResult[] {
    return [];
  }
}
```

This allows new providers to be added without rewriting the Orchestrator.

# Simplified TypeScript Example

## 18. Flight Search Orchestrator

```ts
interface ProviderSearchResult {
  provider: string;
  results: NormalizedFlightResult[];
}

interface FlightSearchResponse {
  searchId: string;
  partial: boolean;
  results: NormalizedFlightResult[];
  failedProviders: string[];
}

class FlightSearchOrchestrator {
  constructor(
    private readonly providers: FlightSearchProvider[],
    private readonly cacheService: SearchCacheService,
    private readonly rankingService: FlightRankingService,
  ) {}

  async search(
    query: FlightSearchQuery,
  ): Promise<FlightSearchResponse> {
    const cacheKey = this.cacheService.createKey(query);

    const cached =
      await this.cacheService.get<FlightSearchResponse>(cacheKey);

    if (cached) {
      return cached;
    }

    const supportedProviders = this.providers.filter(
      (provider) => provider.supports(query),
    );

    const settledResults = await Promise.allSettled(
      supportedProviders.map(async (provider) => {
        const results = await withTimeout(
          provider.search(query),
          3000,
        );

        return {
          provider: provider.name,
          results,
        };
      }),
    );

    const successfulResults: ProviderSearchResult[] = [];
    const failedProviders: string[] = [];

    settledResults.forEach((result, index) => {
      const provider = supportedProviders[index];

      if (result.status === 'fulfilled') {
        successfulResults.push(result.value);
      } else {
        failedProviders.push(provider.name);
      }
    });

    if (successfulResults.length === 0) {
      throw new Error('All search providers failed');
    }

    const mergedResults = successfulResults.flatMap(
      (result) => result.results,
    );

    const deduplicatedResults =
      this.removeDuplicates(mergedResults);

    const rankedResults =
      this.rankingService.rank(deduplicatedResults);

    const response: FlightSearchResponse = {
      searchId: crypto.randomUUID(),
      partial: failedProviders.length > 0,
      results: rankedResults,
      failedProviders,
    };

    await this.cacheService.set(cacheKey, response, 120);

    return response;
  }

  private removeDuplicates(
    results: NormalizedFlightResult[],
  ): NormalizedFlightResult[] {
    const uniqueResults = new Map<
      string,
      NormalizedFlightResult
    >();

    for (const result of results) {
      const key = [
        result.airlineCode,
        result.origin,
        result.destination,
        result.departureAt,
        result.arrivalAt,
      ].join(':');

      const existing = uniqueResults.get(key);

      if (!existing || result.price.amount < existing.price.amount) {
        uniqueResults.set(key, result);
      }
    }

    return Array.from(uniqueResults.values());
  }
}
```

A production implementation should also include:

- Structured logging
- Metrics
- Distributed tracing
- Request cancellation
- Retry policies
- Circuit breakers
- Rate-limit handling
- Currency conversion
- Pagination
- Offer expiration
- Provider health monitoring

# Suggested Project Structure

```text
src/
└── modules/
    └── search/
        ├── controllers/
        │   └── search.controller.ts
        ├── orchestrators/
        │   ├── flight-search.orchestrator.ts
        │   └── hotel-search.orchestrator.ts
        ├── providers/
        │   ├── flight/
        │   │   ├── flight-search-provider.interface.ts
        │   │   ├── amadeus.provider.ts
        │   │   └── travelport.provider.ts
        │   └── hotel/
        │       ├── hotel-search-provider.interface.ts
        │       ├── provider-a.provider.ts
        │       └── provider-b.provider.ts
        ├── mappers/
        ├── ranking/
        ├── deduplication/
        ├── cache/
        ├── dto/
        └── models/
```

# Search Orchestrator and Elasticsearch

## 19. Does the Search Orchestrator Replace Elasticsearch?

No. They solve different problems.

### Elasticsearch

Elasticsearch is used for:

- Full-text search
- Filtering indexed data
- Ranking documents
- Autocomplete
- Geographic search

### Search Orchestrator

The Search Orchestrator:

- Checks Redis
- Calls multiple providers
- Queries Elasticsearch when needed
- Normalizes data
- Merges results
- Handles failures
- Removes duplicates
- Ranks results
- Returns one response

They can work together:

```text
Search Orchestrator
   ├── Redis Cache
   ├── Elasticsearch
   ├── Flight Provider A
   ├── Flight Provider B
   └── Hotel Provider C
```

# Search Versus Booking

## 20. Search Results Are Not Final Booking Data

Between search and booking:

- Price may change
- Availability may change
- Seats may sell out
- Rooms may become unavailable
- Offers may expire
- Taxes or fees may change

Correct workflow:

```text
1. Search
2. Display results
3. User selects an offer
4. Revalidate the offer
5. Retrieve current price and availability
6. Create the booking
7. Process payment
```

The Search Orchestrator handles discovery.

Booking should be handled by a separate Booking Service or Booking Orchestrator.

# Orchestration Versus Choreography

## 21. Orchestration

One central component controls the workflow.

```text
Search Orchestrator
   ├── Call Provider A
   ├── Call Provider B
   ├── Merge results
   └── Return response
```

Advantages:

- Clear workflow
- Centralized error handling
- Suitable for synchronous aggregation
- Easier to trace

Disadvantages:

- Can become too large
- Can become a central dependency
- Poor design may create tight coupling

## 22. Choreography

Services react to events without one central controller.

```text
Service A publishes an event
   │
   ▼
Service B reacts
   │
   ▼
Service C publishes another event
```

Choreography is useful for asynchronous workflows, but orchestration is usually clearer for a user waiting for an immediate multi-provider search response.

# Reliability Patterns

## 23. Timeout

Stop waiting after an acceptable duration.

## 24. Retry

Retry only temporary failures such as:

- Network interruptions
- HTTP 502
- HTTP 503
- HTTP 504

Use exponential backoff and jitter.

## 25. Circuit Breaker

If a provider fails repeatedly, temporarily stop calling it.

```text
Closed: calls are allowed
Open: calls are blocked
Half-Open: limited test calls are allowed
```

## 26. Bulkhead

Isolate provider resources so one slow provider cannot consume all application capacity.

## 27. Rate Limiting

Track provider limits such as:

- Requests per second
- Requests per minute
- Daily quota
- Concurrent requests

# Observability

## 28. Logging

Useful fields:

```text
searchId
provider
durationMs
resultCount
cacheHit
timeout
failureReason
partialResult
```

## 29. Metrics

Important metrics:

- Total search requests
- Average search duration
- Cache hit rate
- Provider response time
- Provider error rate
- Provider timeout rate
- Partial response count
- Empty result rate

## 30. Distributed Tracing

Tracing helps follow one search across:

```text
API Gateway
   │
Search Service
   │
Search Orchestrator
   ├── Provider A
   ├── Provider B
   └── Redis
```

# Design Guidelines

## 31. Keep the Orchestrator Focused

The Orchestrator should coordinate the workflow, not contain every implementation detail.

```text
Orchestrator
   ├── Provider Registry
   ├── Cache Service
   ├── Deduplication Service
   ├── Ranking Service
   ├── Currency Service
   └── Metrics Service
```

## 32. Use a Common Provider Interface

All providers should implement the same contract.

This supports the Open/Closed Principle:

```text
Open for extension
Closed for modification
```

## 33. Make Provider Calls Independent

One provider failure should not break the others.

## 34. Return Partial Results When Useful

Two successful providers out of three may still provide a useful search response.

## 35. Make Cache Keys Deterministic

Normalize values before generating keys:

- Uppercase airport codes
- Sorted filters
- Normalized dates
- Default currency
- Removed whitespace

## 36. Revalidate Before Booking

Never trust search prices and availability as final booking data.

Search is optimized for speed and discovery.

Booking is optimized for consistency and correctness.

# Recommended Architecture

```text
                         ┌──────────────────┐
                         │ Mobile / Web App │
                         └─────────┬────────┘
                                   │
                                   ▼
                         Search Controller
                                   │
                                   ▼
                         Search Orchestrator
                                   │
                  ┌────────────────┼────────────────┐
                  │                │                │
                  ▼                ▼                ▼
             Redis Cache     Elasticsearch    Provider Registry
                                                     │
                                  ┌──────────────────┼──────────────────┐
                                  │                  │                  │
                                  ▼                  ▼                  ▼
                           Amadeus Adapter    Travelport Adapter   Hotel Adapter
                                  │                  │                  │
                                  └──────────────────┼──────────────────┘
                                                     │
                                                     ▼
                                              Normalize Results
                                                     │
                                                     ▼
                                                Merge Results
                                                     │
                                                     ▼
                                             Deduplicate or Group
                                                     │
                                                     ▼
                                                Rank and Sort
                                                     │
                                                     ▼
                                               Cache Response
                                                     │
                                                     ▼
                                             Unified API Response
```

# Implementation Roadmap

## Phase 1: Basic Integration

```text
1. Define the normalized search request.
2. Define the normalized search result.
3. Create a common provider interface.
4. Integrate one provider.
5. Implement the Search Orchestrator.
6. Return a unified response.
```

## Phase 2: Multiple Providers

```text
1. Add provider adapters.
2. Run calls in parallel.
3. Use Promise.allSettled().
4. Add timeout handling.
5. Support partial results.
6. Merge results.
```

## Phase 3: Search Quality

```text
1. Add deduplication or grouping.
2. Add sorting and ranking.
3. Add currency normalization.
4. Add pagination.
5. Add Redis caching.
```

## Phase 4: Production Reliability

```text
1. Add retries with exponential backoff.
2. Add circuit breakers.
3. Add provider rate-limit handling.
4. Add structured logging.
5. Add metrics and tracing.
6. Add provider health monitoring.
```

# Summary

An **Orchestrator** coordinates multiple components to complete one workflow.

A **Search Orchestrator** coordinates the complete search process across multiple providers and internal systems.

```text
Receive Search Request
        │
        ▼
Check Cache
        │
        ▼
Select Providers
        │
        ▼
Call Providers in Parallel
        │
        ▼
Handle Timeout and Failure
        │
        ▼
Normalize Results
        │
        ▼
Merge Results
        │
        ▼
Deduplicate or Group Offers
        │
        ▼
Rank and Sort
        │
        ▼
Cache and Return Unified Response
```

For a flight and hotel booking platform that depends on multiple external APIs, the Search Orchestrator is the central application component responsible for delivering one fast, reliable, and consistent search experience.
