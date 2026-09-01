# Fan-Out

## 1. What Is Fan-Out?

**Fan-Out** is a pattern where one request, event, or piece of data
causes work to be distributed to multiple downstream destinations.

In simple terms:

``` text
One input
   |
   +--> Output A
   +--> Output B
   +--> Output C
   +--> Output D
```

One operation becomes many operations.

------------------------------------------------------------------------

## 2. Simple Example

Suppose a user publishes a post and has 100,000 followers.

``` text
New Post
   |
   +--> Follower 1
   +--> Follower 2
   +--> Follower 3
   |
   ...
   |
   +--> Follower 100,000
```

One event may trigger thousands of downstream operations.

That is fan-out.

------------------------------------------------------------------------

## 3. Fan-Out in Messaging Systems

Imagine an `OrderCreated` event.

Several services are interested in it:

``` text
                 +--> Payment Service
                 |
OrderCreated ----+--> Notification Service
                 |
                 +--> Analytics Service
                 |
                 +--> Inventory Service
```

One event is distributed to multiple consumers.

This is common in event-driven architectures.

------------------------------------------------------------------------

## 4. Fan-Out in Search Systems

Suppose a travel platform searches several flight providers.

The client sends one request:

``` http
GET /flights/search
```

The Search Service distributes it:

``` text
                 +--> Provider A
                 |
Search Request --+--> Provider B
                 |
                 +--> Provider C
                 |
                 +--> Provider D
```

The results are then collected:

``` text
Provider A ----+
Provider B ----+
Provider C ----+--> Aggregator --> Client
Provider D ----+
```

This is a form of fan-out combined with a **Scatter-Gather** pattern.

------------------------------------------------------------------------

## 5. Why Use Fan-Out?

Fan-out is useful when one event or request needs to trigger multiple
independent operations.

Common use cases include:

-   Social-media feeds
-   Notifications
-   Search across multiple providers
-   Event-driven microservices
-   Email delivery
-   Analytics pipelines
-   Cache invalidation
-   Webhooks
-   Parallel processing

------------------------------------------------------------------------

# Fan-Out on Write vs Fan-Out on Read

A classic System Design topic is generating social-media feeds.

Suppose Alice publishes a post and has:

``` text
100,000 followers
```

There are two major strategies.

------------------------------------------------------------------------

## 6. Fan-Out on Write

With **Fan-Out on Write**, when Alice creates a post, the system
immediately distributes a reference or copy of that post into followers'
feeds.

``` text
Alice publishes post
        |
        v
    Fan-Out Worker
        |
        +--> Feed User 1
        +--> Feed User 2
        +--> Feed User 3
        |
        ...
        |
        +--> Feed User 100,000
```

The work happens mainly during the **write**.

### Advantages

Reading the feed is fast.

``` text
User opens feed
      |
      v
Read precomputed feed
      |
      v
Return result
```

The feed has already been prepared.

### Disadvantages

Publishing can cause huge amounts of work.

If a celebrity has:

``` text
50,000,000 followers
```

one post could theoretically trigger millions of feed updates.

This can create:

-   High write amplification
-   Large queue backlogs
-   Storage overhead
-   Backpressure
-   Hot partitions
-   Delayed propagation

------------------------------------------------------------------------

## 7. Fan-Out on Read

With **Fan-Out on Read**, the system does not push a new post into every
follower's feed.

Instead, posts are stored normally.

When a user opens their feed:

``` text
User opens feed
      |
      v
Find followed accounts
      |
      v
Fetch their recent posts
      |
      v
Merge
      |
      v
Rank / Sort
      |
      v
Return feed
```

The expensive work happens mainly during the **read**.

### Advantages

Writes are relatively cheap.

``` text
Create Post
   |
   v
Store Post
```

No need to immediately update millions of feeds.

### Disadvantages

Reads become more expensive because the system may need to fetch and
merge posts from many accounts.

------------------------------------------------------------------------

## 8. Fan-Out on Write vs Fan-Out on Read

  Property            Fan-Out on Write          Fan-Out on Read
  ------------------- ------------------------- -----------------------
  Write cost          High                      Low
  Read cost           Low                       High
  Feed latency        Usually low               Potentially higher
  Storage             Higher                    Lower
  Celebrity problem   Significant               Less severe
  Complexity          Background distribution   Read-time aggregation

------------------------------------------------------------------------

## 9. Hybrid Fan-Out

Large systems may use a hybrid strategy.

For normal users:

``` text
Fan-Out on Write
```

For celebrities with millions of followers:

``` text
Fan-Out on Read
```

Example:

``` text
Normal User Post
      |
      v
Push into follower feeds


Celebrity Post
      |
      v
Store normally
      |
      v
Merge during feed reads
```

This avoids performing millions of writes every time a high-follower
account posts.

------------------------------------------------------------------------

## 10. Fan-Out and Message Queues

Fan-out is often implemented asynchronously.

Instead of:

``` text
API
 |
 +--> User 1
 +--> User 2
 +--> User 3
 +--> ...
```

the API publishes a job or event:

``` text
API
 |
 v
Queue / Broker
 |
 +--> Worker 1
 +--> Worker 2
 +--> Worker 3
```

Workers perform the expensive distribution.

Benefits include:

-   Better API latency
-   Retry support
-   Horizontal scaling
-   Failure isolation
-   Controlled concurrency

------------------------------------------------------------------------

## 11. The Fan-Out Problem

Fan-out can amplify traffic dramatically.

Suppose:

``` text
1 request
```

creates:

``` text
10,000 downstream operations
```

and the system receives:

``` text
100 requests/sec
```

Then downstream work can reach:

``` text
100 × 10,000
= 1,000,000 operations/sec
```

This is called **work amplification**.

Fan-out therefore needs careful capacity planning.

------------------------------------------------------------------------

## 12. Fan-Out and Backpressure

Fan-out can directly create backpressure.

Example:

``` text
Incoming Events
100 events/sec
```

Each event generates:

``` text
1,000 jobs
```

Therefore:

``` text
100 × 1,000
= 100,000 jobs/sec
```

But workers may process only:

``` text
30,000 jobs/sec
```

The backlog grows by:

``` text
70,000 jobs/sec
```

Architecture:

``` text
Event
  |
  v
Fan-Out
  |
  v
Queue
██████████████████
  |
  v
Workers
```

This is why fan-out systems commonly need:

-   Queues
-   Backpressure
-   Rate limiting
-   Batching
-   Horizontal scaling
-   Retry policies
-   Dead-letter queues
-   Monitoring

------------------------------------------------------------------------

## 13. Fan-Out vs Scatter-Gather

They are related but not identical.

### Fan-Out

One event/request is distributed to many destinations.

``` text
        +--> A
Input --+--> B
        +--> C
```

Returning a combined response is not necessarily required.

### Scatter-Gather

A request is sent to multiple destinations **and their responses are
collected and aggregated**.

``` text
        +--> A --+
Request +--> B --+--> Aggregate --> Response
        +--> C --+
```

For example, searching multiple travel providers and combining their
offers is a Scatter-Gather workflow built using fan-out.

------------------------------------------------------------------------

## 14. Important Design Questions

When designing fan-out, ask:

1.  How large can the fan-out factor become?
2.  Should processing be synchronous or asynchronous?
3.  What happens if one destination fails?
4.  Should failed operations be retried?
5.  Can operations be processed in batches?
6.  How do we prevent duplicate processing?
7.  What happens when queues become full?
8.  How do we handle celebrity or hot-key scenarios?
9.  How do we monitor queue depth and processing latency?
10. Is ordering required?

------------------------------------------------------------------------

## 15. Key Takeaway

Fan-out means:

``` text
One input
    |
    v
Many downstream operations
```

Two important feed strategies are:

``` text
Fan-Out on Write
    -> expensive writes
    -> fast reads

Fan-Out on Read
    -> cheap writes
    -> expensive reads
```

Fan-out can create massive **work amplification**, so large-scale
fan-out architectures commonly depend on queues, workers, batching, rate
limiting, and backpressure.
