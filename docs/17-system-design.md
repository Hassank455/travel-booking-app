# 16. System Design

## 1. Purpose

This document presents the high-level system design of the Travel Booking Platform.

The design is derived from the platform's functional requirements, non-functional requirements, estimated traffic, data characteristics, and previously documented architectural decisions.

The goal is to explain how the main components work together and why specific architectural choices were made.

---

## 2. Scope

The system currently supports:

- Flight search and booking.
- Hotel search and booking.
- Multi-provider integration.
- Offer revalidation.
- Transaction processing.
- Customer booking management.
- Notifications.

Flight and hotel inventory is owned by external providers rather than the platform.

Current primary providers include:

- Duffel for flights.
- LiteAPI for hotels.

---

## 3. Design Approach

The system design is developed in the following order:

1. Define requirements and constraints.
2. Estimate expected traffic and storage.
3. Identify system bottlenecks.
4. Define major components.
5. Design critical request flows.
6. Evaluate scaling and reliability.
7. Document architectural trade-offs.

---

## 4. Back-of-the-Envelope Estimation

Back-of-the-envelope estimation provides an approximate understanding of the expected system scale.

The goal is not to predict exact production traffic.

Instead, the estimates help determine:

- Expected request volume.
- Read/write characteristics.
- External provider traffic.
- Cache requirements.
- Database storage requirements.
- Network bandwidth.
- Initial scaling requirements.

All numbers in this section are design assumptions and should be validated against production metrics when the platform becomes operational.

---

### 4.1 Initial Assumptions

For the initial system design, assume:

| Assumption | Value |
|---|---:|
| Registered Users | 1,000,000 |
| Daily Active Users | 100,000 |
| Average Searches per Active User | 5 / day |
| Search-to-Booking Conversion | 2% |
| Peak Traffic Factor | 3× average |
| Search Cache Hit Ratio | 60% |
| Average Search Response Size | ~200 KB |
| Average Persisted Booking Aggregate | ~8–12 KB |

These numbers represent a medium-scale travel platform rather than the scale of a global company such as Booking.com.

---

# 4.2 Search Traffic

Given:

```text
Daily Active Users = 100,000

Average Searches per User = 5
```

Daily searches:

```text
100,000 × 5

= 500,000 searches / day
```

Therefore:

```text
Searches per day
≈ 500K
```

---

### Average Search QPS

There are:

```text
86,400 seconds / day
```

Therefore:

```text
500,000 / 86,400

≈ 5.8 requests / second
```

Average Search QPS:

```text
≈ 6 QPS
```

---

### Peak Search QPS

Traffic is not distributed evenly throughout the day.

Using a peak factor of:

```text
3×
```

we get:

```text
5.8 × 3

≈ 17.4 QPS
```

Peak search traffic is approximately:

```text
≈ 18 QPS
```

---

### Architectural Implication

This traffic level does not require an extremely distributed architecture.

However, search requests may trigger expensive external provider calls.

The main bottleneck is therefore not necessarily:

```text
Backend CPU
```

but:

```text
External Provider Latency
+
Provider Rate Limits
+
External API Cost
```

This reinforces the importance of:

- Redis caching.
- Parallel provider calls.
- Provider timeout isolation.
- Horizontal backend scalability.

---

# 4.3 Booking Traffic

Assume:

```text
2% of searches eventually result in a booking
```

Therefore:

```text
500,000 × 0.02

= 10,000 bookings / day
```

Daily booking volume:

```text
≈ 10K bookings
```

---

### Average Booking QPS

```text
10,000 / 86,400

≈ 0.116 QPS
```

Approximately:

```text
1 booking every 8–9 seconds
```

on average.

---

### Peak Booking QPS

Applying the same peak factor:

```text
0.116 × 3

≈ 0.35 QPS
```

This means approximately:

```text
1 booking every 3 seconds
```

during the assumed peak period.

---

### Architectural Implication

There is a significant difference between:

```text
Search Traffic
```

and:

```text
Booking Traffic
```

Approximately:

```text
500,000 Searches / Day

vs

10,000 Bookings / Day
```

This confirms that the system is heavily **read/search oriented**.

Search infrastructure therefore requires more attention to:

- Caching.
- Provider latency.
- Network traffic.
- Rate limits.

Booking infrastructure requires more attention to:

- Correctness.
- Transactions.
- Idempotency.
- Reconciliation.
- Persistence.

This is one reason Search and Booking are designed as separate architectural responsibilities.

---

# 4.4 External Provider Traffic

Without caching, every search would require at least one external provider request.

With:

```text
500,000 searches / day
```

the platform could generate approximately:

```text
500,000 provider requests / day
```

when using one provider for that search category.

---

### With Search Cache

Assume:

```text
Cache Hit Ratio = 60%
```

Therefore:

```text
Cache Miss Ratio = 40%
```

Only:

```text
500,000 × 0.40

= 200,000
```

searches require provider access.

Provider-facing search traffic becomes approximately:

```text
200K provider search operations / day
```

instead of:

```text
500K / day
```

---

### Provider QPS

Average:

```text
200,000 / 86,400

≈ 2.3 provider operations / second
```

Peak:

```text
2.3 × 3

≈ 6.9 provider operations / second
```

Approximately:

```text
7 provider operations / second
```

at peak under the current assumptions.

---

### Multi-Provider Expansion

If one search is sent to multiple providers, provider traffic increases accordingly.

For example:

```text
1 Search
   ↓
3 Providers
```

then:

```text
Provider Requests
=
Cache Misses × Active Providers
```

For three active providers:

```text
200,000 × 3

= 600,000 provider requests / day
```

This demonstrates why provider aggregation and caching become increasingly important as the number of integrations grows.

---

# 4.5 Cache Impact

Without cache:

```text
500K searches
       ↓
External Providers
```

With a 60% cache hit ratio:

```text
500K searches
       ↓
┌───────────────┐
│               │
300K Cache Hits 200K Cache Misses
                    ↓
                 Providers
```

The cache therefore prevents approximately:

```text
300,000 provider search operations / day
```

under the current single-provider-per-search-category assumption.

That represents approximately:

```text
60%
```

less provider traffic.

---

# 4.6 Approximate Redis Capacity

Assume:

```text
Average Cached Search Result
≈ 200 KB
```

At peak:

```text
Cache misses
≈ 7 / second
```

If search results use a TTL between approximately:

```text
3–5 minutes
```

then the number of simultaneously active cache entries may roughly fall around:

```text
7 × 180
≈ 1,260 entries
```

to:

```text
7 × 300
≈ 2,100 entries
```

Approximate payload memory:

For 3 minutes:

```text
1,260 × 200 KB
≈ 250 MB
```

For 5 minutes:

```text
2,100 × 200 KB
≈ 420 MB
```

Additional memory must be reserved for:

- Redis key overhead.
- Serialization overhead.
- Metadata.
- Locks.
- Idempotency records.
- Fragmentation.
- Operational headroom.

Therefore Redis should not be sized using payload size alone.

---

### Architectural Implication

The current traffic assumptions do not require a very large Redis cluster.

However, cache size grows significantly with:

- More traffic.
- Larger search responses.
- Longer TTLs.
- More providers.
- Provider-level caching.

Redis memory should therefore be monitored rather than permanently sized from design assumptions.

---

# 4.7 Search Bandwidth

Assume:

```text
Average Search Response
≈ 200 KB
```

With:

```text
500,000 searches / day
```

daily response payload becomes approximately:

```text
500,000 × 200 KB

≈ 100 GB / day
```

This represents response payload only and excludes:

- HTTP overhead.
- TLS overhead.
- Provider traffic.
- Internal traffic.
- Retries.

---

### Peak Response Bandwidth

At approximately:

```text
18 Search QPS
```

and:

```text
200 KB / response
```

peak outbound payload is approximately:

```text
18 × 200 KB

≈ 3.6 MB / second
```

This is manageable for the assumed system size.

However, search responses are significantly heavier than typical booking or profile responses.

---

# 4.8 Booking Storage

Assume:

```text
10,000 bookings / day
```

Annual bookings:

```text
10,000 × 365

= 3,650,000 bookings / year
```

Approximately:

```text
3.65 million bookings / year
```

---

### Booking Aggregate Size

A booking is not represented by only one database row.

A flight booking may include:

```text
Flight Booking
Passengers
Segments
Transaction
Status History
Audit Records
```

A hotel booking may include:

```text
Hotel Booking
Guests
Rooms
Transaction
Status History
Audit Records
```

Assume the average complete booking aggregate consumes approximately:

```text
8–12 KB
```

before significant index and database overhead.

---

### Raw Annual Storage

At approximately:

```text
3.65M bookings
```

and:

```text
8–12 KB / booking
```

raw booking data would be approximately:

```text
29–44 GB / year
```

---

### Including Indexes and Operational Overhead

Indexes, row metadata, audit information, and database overhead may significantly increase the real footprint.

A reasonable planning estimate may therefore be:

```text
~60–90 GB / year
```

for booking-related persistent data under these assumptions.

Over three years:

```text
~180–270 GB
```

This remains very manageable for PostgreSQL.

---

# 4.9 Read vs Write Characteristics

The platform is strongly read-heavy.

Approximate daily operations:

```text
Searches
≈ 500,000

Bookings
≈ 10,000
```

Ratio:

```text
~50 Searches : 1 Booking
```

Not every search creates a database write because live travel inventory is not persisted in PostgreSQL.

Therefore:

```text
Search Path

Redis
+
External Providers
```

while:

```text
Booking Path

PostgreSQL
+
External Providers
+
Transaction Processing
```

---

# 4.10 Initial Scaling Implications

The estimates suggest several architectural conclusions.

### Backend

Peak request traffic is moderate.

The backend should remain stateless and horizontally scalable, but large-scale compute infrastructure is not initially required.

---

### PostgreSQL

Booking write volume is relatively low.

A well-configured primary PostgreSQL database should comfortably support the initial workload.

Database sharding is not justified by the current estimates.

---

### Redis

Redis primarily reduces:

- Provider traffic.
- Search latency.
- Provider cost.

The initial cache footprint is relatively small, but should be monitored as traffic and provider count increase.

---

### External Providers

External provider latency and quotas are likely to become bottlenecks before backend compute or database throughput.

Therefore the architecture should prioritize:

- Parallel calls.
- Timeouts.
- Caching.
- Rate limiting.
- Circuit-breaking/failure isolation where appropriate.
- Provider observability.

---

### Booking

Booking throughput is relatively low but correctness requirements are high.

The design should optimize booking for:

```text
Correctness
Reliability
Idempotency
Reconciliation
```

rather than extreme write throughput.

---

## 4.11 Estimation Summary

| Metric | Estimate |
|---|---:|
| Registered Users | 1M |
| Daily Active Users | 100K |
| Searches / Day | 500K |
| Average Search QPS | ~6 |
| Peak Search QPS | ~18 |
| Bookings / Day | 10K |
| Average Booking QPS | ~0.12 |
| Peak Booking QPS | ~0.35 |
| Cache Hit Ratio | 60% |
| Provider Search Operations / Day | ~200K |
| Peak Provider Operations / Second | ~7 |
| Estimated Search Egress | ~100 GB/day |
| Estimated Raw Booking Storage | ~29–44 GB/year |
| Estimated DB Storage with Overhead | ~60–90 GB/year |

---

## 4.12 Key Conclusion

The estimates indicate that the primary scalability challenge is not currently database write throughput.

The dominant characteristics of the platform are:

```text
High Search Volume
        +
External Provider Dependency
        +
Relatively Low Transactional Write Volume
```

Therefore the initial system design should focus on:

- Efficient search caching.
- Provider isolation.
- Horizontal application scaling.
- Reliable booking orchestration.
- Strong transactional persistence.

More advanced techniques such as database sharding or large distributed compute clusters are not justified by the current estimated scale.

---

## 5. Capacity & Bottleneck Analysis

The back-of-the-envelope estimation shows that the platform is primarily constrained by search traffic and external provider dependencies rather than transactional database writes.

The goal of this section is to identify the components most likely to become bottlenecks as traffic grows and define the architectural response for each one.

---

### 5.1 Bottleneck Prioritization

Based on the current estimates, the expected bottlenecks are prioritized as follows:

```text
1. External Provider Latency and Rate Limits

2. Search Fan-Out

3. Redis Cache Efficiency

4. Network Bandwidth and Large Search Responses

5. Backend Concurrency

6. Background Worker / Queue Capacity

7. PostgreSQL

8. Database Storage
```

This ordering may change as production traffic and provider behavior become measurable.

---

# 5.2 External Provider Latency

External providers are expected to be one of the earliest performance bottlenecks.

A search request may require communication with one or more external APIs.

Example:

```text
Customer Search
      ↓
Backend
      ↓
┌───────────────┬───────────────┬───────────────┐
Provider A      Provider B      Provider C
   300 ms          900 ms           4 s
```

If the platform waits for every provider without timeout isolation, the slowest provider may dominate the customer response time.

---

### Why This Matters

Backend processing may take:

```text
10–50 ms
```

while external provider requests may take:

```text
hundreds of milliseconds
to
several seconds
```

Therefore optimizing application code may have little effect if provider latency dominates total search time.

---

### Architectural Response

The system should:

- Call independent providers concurrently.
- Use provider-specific timeouts.
- Isolate provider failures.
- Allow partial search results where business rules permit.
- Monitor provider latency independently.
- Avoid unlimited retries in the synchronous request path.

Conceptually:

```text
Search Request
      ↓
Parallel Provider Calls
      ↓
┌────────────┬────────────┬────────────┐
Fast         Slow         Failed
 ↓            ↓             ↓
Use          Timeout       Isolate
      ↓
Aggregate Available Results
```

---

### Capacity Signal

Monitor:

```text
provider_request_duration
provider_timeout_rate
provider_error_rate
```

per provider.

If provider latency consistently dominates request latency, backend scaling alone will not solve the problem.

---

# 5.3 Provider Rate Limits

Search traffic can create significantly more provider traffic than customer-facing QPS suggests.

For example:

```text
18 peak search QPS
```

with three providers may produce up to:

```text
54 external requests / second
```

before cache reduction.

Provider rate limits may therefore become a constraint earlier than backend CPU capacity.

---

### Architectural Response

The platform should use:

- Search caching.
- Request deduplication where appropriate.
- Provider-specific rate limiting.
- Backpressure.
- Configurable provider routing.
- Provider-Level Cache if justified later.

When provider quotas become constrained, the architecture should reduce unnecessary provider requests before adding more backend instances.

---

# 5.4 Search Fan-Out

Multi-provider search introduces a fan-out effect.

For example:

```text
1 Customer Request
      ↓
5 Provider Requests
```

At:

```text
100 Search QPS
```

this becomes:

```text
500 provider requests / second
```

before retries.

This multiplication effect is one of the most important characteristics of the search architecture.

---

### Architectural Response

Search fan-out should be bounded.

The platform should avoid:

- Calling every possible provider for every request.
- Unlimited retries.
- Waiting indefinitely for slow providers.

Possible future strategies include:

- Provider routing.
- Market-based provider selection.
- Provider scoring.
- Adaptive provider selection.
- Provider-Level Cache.

---

# 5.5 Redis Cache Efficiency

Redis capacity itself is unlikely to be an early bottleneck under the current assumptions.

However, poor cache behavior can indirectly create a major provider bottleneck.

Example:

```text
Cache Hit Ratio = 60%
```

results in:

```text
200K provider-facing searches / day
```

If the hit ratio falls to:

```text
20%
```

then:

```text
400K provider-facing searches / day
```

would require external calls.

Therefore cache effectiveness matters more than raw Redis throughput initially.

---

### Important Metrics

Monitor:

```text
cache_hit_ratio
cache_miss_ratio
cache_evictions
cache_memory_usage
cache_lookup_latency
```

---

### Architectural Response

If cache hit ratio is unexpectedly low, investigate:

- Poor cache-key normalization.
- TTL too short.
- Excessive key cardinality.
- Filters unnecessarily included in cache keys.
- Insufficient Redis memory.
- Provider-selection context generating fragmented keys.

The correct response is not automatically "increase Redis capacity."

---

# 5.6 Cache Stampede

A popular cache entry may expire while many users request the same search simultaneously.

Example:

```text
Popular Search Cache Expires
          ↓
100 Concurrent Requests
          ↓
100 Provider Calls
```

This is known as a cache stampede.

It can produce sudden provider traffic spikes even when the average search QPS is low.

---

### Architectural Response

Possible protections include:

- Request coalescing.
- Distributed locking.
- TTL jitter.
- Stale-while-revalidate where acceptable.

Conceptually:

```text
100 Requests
      ↓
Cache Miss
      ↓
One Request Refreshes
      ↓
Others Wait / Reuse Result
```

This mechanism should only be introduced where traffic patterns justify it.

---

# 5.7 Backend Concurrency

The backend is not expected to be CPU-bound initially.

However, external requests create many concurrent waiting operations.

For example:

```text
20 search QPS
×
3 providers
×
2 second average provider latency
```

may produce approximately:

```text
120 provider requests concurrently in flight
```

even though the customer-facing QPS is relatively small.

---

### Architectural Implication

The application should use efficient asynchronous I/O and remain stateless.

Horizontal scaling should be supported from the beginning:

```text
Load Balancer
      ↓
┌──────────┬──────────┬──────────┐
Backend 1  Backend 2  Backend 3
```

The number of instances should be driven by:

- Request concurrency.
- CPU utilization.
- Memory.
- Provider connection usage.
- Latency.

rather than QPS alone.

---

# 5.8 Search Response Size and Network Bandwidth

Search responses may be considerably larger than normal API responses.

Assumed average:

```text
~200 KB
```

At higher traffic, network bandwidth may become important before database throughput.

For example:

```text
1,000 Search QPS
×
200 KB
```

produces approximately:

```text
200 MB / second
```

of outbound payload before protocol overhead.

---

### Architectural Response

Possible optimizations include:

- Limit maximum returned offers.
- Pagination.
- Response compression.
- Avoid unnecessary provider fields.
- Normalize responses into compact platform models.
- Client-side incremental loading where appropriate.

The platform should not return raw provider responses.

---

# 5.9 PostgreSQL Write Capacity

Based on current estimates:

```text
~10K bookings / day
```

the database write rate is relatively low.

Even when each booking creates multiple records:

```text
Booking
Passengers / Guests
Segments / Rooms
Transaction
Status History
Audit Records
```

the write volume remains modest compared with the capabilities of a properly configured PostgreSQL instance.

---

### Architectural Conclusion

The current scale does not justify:

- Database sharding.
- Distributed SQL.
- Separate database per booking type.
- Complex multi-region write architecture.

Initial scaling should focus on:

- Good indexes.
- Connection pooling.
- Query monitoring.
- Backup and recovery.
- Read replicas only when justified.

---

# 5.10 Database Connection Capacity

Although query throughput may be low, horizontal backend scaling can create excessive database connections.

Example:

```text
10 Backend Instances
×
50 Connections
=
500 Database Connections
```

This can become a bottleneck even when database CPU remains low.

---

### Architectural Response

Use connection pooling and limit application connection counts.

Monitor:

```text
active_connections
idle_connections
connection_wait_time
query_latency
```

Database scaling decisions should be based on measured utilization rather than application instance count alone.

---

# 5.11 Background Worker Capacity

Some operations should not remain in the customer request path.

Examples:

- Notifications.
- Provider reconciliation.
- Retry workflows.
- Scheduled synchronization.
- Operational cleanup.

These operations may accumulate in queues during provider outages or traffic spikes.

Example:

```text
Notification Provider Down
        ↓
Jobs Continue Arriving
        ↓
Queue Depth Increases
```

---

### Architectural Response

Workers should scale independently from the Backend Application.

Conceptually:

```text
Message Broker
      ↓
┌──────────┬──────────┬──────────┐
Worker 1   Worker 2   Worker 3
```

Important metrics include:

```text
queue_depth
oldest_message_age
worker_processing_time
retry_rate
failed_job_count
```

The age of queued work is often more important than queue length alone.

---

# 5.12 Message Broker Capacity

The broker is not expected to carry the main search traffic.

It primarily handles asynchronous business operations.

Therefore broker throughput is likely to remain manageable initially.

However, a provider outage may create a retry storm.

Example:

```text
Provider Down
     ↓
10K Failed Jobs
     ↓
Immediate Retry
     ↓
Provider Still Down
     ↓
Another 10K Requests
```

---

### Architectural Response

Retry behavior should use:

- Exponential backoff.
- Bounded retry counts.
- Dead-letter handling.
- Jitter.
- Provider-aware backpressure.

Retries should not amplify an external outage.

---

# 5.13 Booking Correctness as a Capacity Constraint

Booking is unusual because its primary constraint is not throughput.

Its main constraint is correctness.

A booking system that handles:

```text
10,000 writes / second
```

but occasionally duplicates customer bookings is worse than a slower system that preserves correctness.

Therefore booking design prioritizes:

```text
Idempotency
Consistency
Reconciliation
Durability
Auditability
```

before extreme throughput optimization.

---

# 5.14 Likely First Bottlenecks

Under the current design assumptions, the expected progression is:

```text
1. Provider latency / provider limits

        ↓

2. Cache effectiveness

        ↓

3. Application concurrency

        ↓

4. Network bandwidth

        ↓

5. Worker backlog during failures

        ↓

6. PostgreSQL connections / query efficiency

        ↓

7. Raw database write throughput
```

This is important because it prevents premature optimization of the wrong component.

---

## 5.15 Scaling Trigger Matrix

| Component | Warning Signal | First Response |
|---|---|---|
| External Provider | Latency / rate-limit errors increase | Cache, timeout, routing |
| Redis | Hit ratio decreases | Review keys and TTL |
| Backend | High concurrency / CPU / latency | Horizontal scaling |
| Network | High outbound bandwidth | Compression / result limits |
| Workers | Job age increases | Add workers / fix provider bottleneck |
| Broker | Queue backlog | Backpressure / consumer scaling |
| PostgreSQL | Slow queries | Index/query optimization |
| PostgreSQL | Connection saturation | Connection pooling |
| PostgreSQL | Read pressure | Consider read replicas |
| Storage | Rapid DB growth | Retention / archival strategy |

---

## 5.16 Architectural Conclusion

The system should scale **asymmetrically**.

Different parts of the platform have different scaling needs:

```text
Search
→ scale for concurrency and external calls

Booking
→ scale for correctness and reliability

Workers
→ scale for queue backlog

Redis
→ scale for cache memory and throughput

PostgreSQL
→ scale for durable transactional data
```

The architecture should therefore avoid assuming that all components require the same scaling strategy.

At the estimated initial scale, the platform does not require aggressive distributed-database techniques.

The first architecture should remain simple while preserving clear horizontal scaling paths.

---

# 6. High-Level System Architecture

## Purpose

This section presents the high-level architecture of the Travel Booking Platform.

The architecture is derived from:

- Functional requirements.
- Non-functional requirements.
- Back-of-the-envelope estimation.
- Capacity and bottleneck analysis.
- Previously documented architectural decisions (ADRs).

The objective is to show how the major components interact while keeping the design simple, scalable, and maintainable.

The platform follows a **Modular Monolith** architecture with clearly separated business modules that can evolve independently while remaining part of a single deployable application.

---

## 6.1 Architecture Overview

The architecture is designed around several key observations identified during the previous design stages.

### Search is Read-Heavy

Search traffic is expected to be significantly higher than booking traffic.

Search operations prioritize:

- Low latency.
- High throughput.
- Cache efficiency.
- External provider aggregation.

---

### Booking Prioritizes Correctness

Booking traffic is relatively low compared to search traffic.

Booking operations prioritize:

- Strong consistency.
- Idempotency.
- Reliable persistence.
- Provider reconciliation.
- Auditability.

---

### External Providers Own Travel Inventory

The platform does **not** own:

- Flight inventory.
- Hotel inventory.
- Live pricing.
- Availability.

These remain the responsibility of external providers.

The platform owns only its business data, including:

- Customers.
- Bookings.
- Transactions.

---

### Stateless Backend

The backend remains stateless.

Business state is stored in:

- PostgreSQL.
- Redis.
- External Providers.

This allows backend instances to scale horizontally without sharing local state.

---

## 6.2 Major Components

The high-level architecture consists of the following components.

### Client Applications

Responsible for interacting with the platform.

Components:

- Web Application
- Mobile Application

---

### Load Balancer

Distributes incoming requests across backend instances.

Responsibilities:

- Traffic distribution.
- High availability.
- Horizontal scaling support.

---

### Backend Application

Implemented as a **Modular Monolith**.

Business modules include:

- Search Module
- Booking Module
- Customer Module
- Payment Module
- Notification Module

Each module owns its own business logic while sharing the same deployment unit.

---

### Redis

Redis provides temporary, high-speed storage.

Used for:

- Search Cache
- Idempotency Keys
- Distributed Locks
- Temporary Workflow State

Redis is **not** the source of truth.

---

### PostgreSQL

The primary transactional database.

Stores:

- Users
- Customers
- Traveler Profiles
- Flight Bookings
- Hotel Bookings
- Transactions

PostgreSQL is the authoritative source for platform-owned business data.

---

### Message Broker

Supports asynchronous processing.

Examples of published events:

- Booking Confirmed
- Payment Completed
- Notification Requested
- Booking Reconciliation Required

---

### Background Workers

Consume asynchronous jobs.

Responsibilities include:

- Notifications
- Retry Operations
- Provider Reconciliation
- Scheduled Jobs
- Background Processing

Workers scale independently from backend instances.

---

### External Providers

The platform integrates with multiple external systems.

Current providers include:

Flights

- Duffel

Hotels

- LiteAPI

Additional providers can be added through the Provider Adapter Pattern.

---

### Payment Provider

Responsible for:

- Payment Authorization
- Payment Capture
- Payment Status

The payment provider remains external to the platform.

---

### Notification Provider

Responsible for:

- Email
- SMS
- Push Notifications

Notification delivery is asynchronous and does not affect booking correctness.

---

### Observability Platform

Collects operational telemetry from all major components.

Includes:

- Logs
- Metrics
- Traces
- Alerts

Observability is treated as a cross-cutting concern across the entire platform.

---

## 6.3 High-Level Request Paths

Although the platform contains many modules, two request paths dominate the architecture.

---

### Search Path

Search is optimized for:

- Low latency.
- High read throughput.
- Cache reuse.
- External provider aggregation.

High-level flow:

```text
Client
      ↓
Load Balancer
      ↓
Backend
      ↓
Redis
      ↓ Cache Miss
External Providers
      ↓
Normalize Results
      ↓
Redis
      ↓
Client
```

Search requests tolerate partial provider failures where appropriate.

---

### Booking Path

Booking is optimized for correctness rather than throughput.

High-level flow:

```text
Client
      ↓
Load Balancer
      ↓
Backend
      ↓
Offer Revalidation
      ↓
Payment Processing
      ↓
Provider Booking
      ↓
PostgreSQL
      ↓
Message Broker
      ↓
Background Workers
```

Booking never relies solely on cached search results.

Every selected offer is revalidated before confirmation.

---

## 6.4 Data Ownership

The architecture clearly defines ownership of data.

### PostgreSQL Owns

- Users
- Customers
- Traveler Profiles
- Flight Bookings
- Hotel Bookings
- Transactions

---

### Redis Owns

Nothing permanently.

Redis stores temporary data only.

Examples:

- Search Cache
- Idempotency Records
- Distributed Locks
- Temporary Workflow State

---

### External Providers Own

- Flight Availability
- Hotel Availability
- Live Prices
- Offers
- Inventory

The platform synchronizes with providers but does not own this data.

---

## 6.5 Scaling Strategy

Different parts of the platform scale differently.

### Backend

Scale horizontally.

```text
Load Balancer
      ↓
Backend 1

Backend 2

Backend 3
```

---

### Redis

Scale when justified by:

- Memory usage.
- Cache throughput.
- Cache hit ratio.

---

### PostgreSQL

Initially scale through:

- Query optimization.
- Proper indexing.
- Connection pooling.

Read replicas should only be introduced when read pressure justifies them.

Database sharding is not required at the current estimated scale.

---

### Background Workers

Workers scale independently.

Additional worker instances can be added without affecting customer-facing API capacity.

---

## 6.6 Design Principles

The architecture follows these principles.

- Keep the backend stateless.
- Separate Search from Booking.
- Cache temporary data only.
- Persist only business-owned data.
- Treat external providers as authoritative for travel inventory.
- Isolate provider failures.
- Prefer asynchronous processing for non-critical work.
- Scale components independently.
- Keep the initial architecture simple while preserving future scalability.

---

## 6.7 Architectural Decisions Applied

The high-level architecture incorporates the following ADRs.

- ADR-001 — Use Modular Monolith Architecture.
- ADR-002 — Use Provider Adapter Pattern.
- ADR-003 — Use Redis for Search Cache.
- ADR-004 — Revalidate Offers Before Booking.
- ADR-005 — Separate Flight and Hotel Booking Entities.
- ADR-006 — Store Booking Snapshots.
- ADR-007 — Use Reverse Foreign Keys for Transactions.
- ADR-008 — Separate Search from Booking.
- ADR-009 — Use Cache-Aside Strategy.
- ADR-011 — Redis Is Not the Source of Truth.

---

## 6.8 Related Diagrams

This section is supported by the following diagrams.

- `diagrams/system-design/high-level-system-design-v1.mmd`
- `diagrams/search/search-flow.mmd`
- `diagrams/sequence/flight-search-sequence.mmd`
- `diagrams/sequence/hotel-search-sequence.mmd`
- `diagrams/cache/cache-architecture.mmd`
- `diagrams/cache/cache-layers.mmd`
- `diagrams/database/core-erd.mmd`

Together, these diagrams describe the major architectural building blocks and the primary runtime interactions of the platform.

---

