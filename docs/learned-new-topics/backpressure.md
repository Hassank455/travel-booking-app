# Backpressure

## 1. What Is Backpressure?

**Backpressure** is a mechanism used when a downstream component cannot
process data as fast as an upstream component produces it.

In simple terms:

> The producer is faster than the consumer, so the system needs a way to
> slow down, buffer, reject, or otherwise control incoming work.

------------------------------------------------------------------------

## 2. The Problem

Imagine a producer sends:

``` text
1,000 messages/second
```

but the consumer can process only:

``` text
200 messages/second
```

The queue grows by:

``` text
1,000 - 200 = 800 messages/second
```

After one minute:

``` text
800 × 60 = 48,000 pending messages
```

If this continues, the system may experience:

-   Increasing latency
-   Memory exhaustion
-   Queue overflow
-   Timeouts
-   Consumer crashes
-   Cascading failures

------------------------------------------------------------------------

## 3. Simple Example

``` text
Producer
1000 msg/s
    |
    v
+---------+
|  Queue  |  <--- backlog keeps growing
+---------+
    |
    v
Consumer
200 msg/s
```

The downstream consumer cannot keep up with the producer.

This is where backpressure is needed.

------------------------------------------------------------------------

## 4. Common Backpressure Strategies

### 4.1 Slow Down the Producer

The consumer or system signals that the producer should send data more
slowly.

``` text
Producer ---> Consumer
    ^           |
    |-----------|
      slow down
```

This is common in reactive streams and network protocols that support
flow control.

### 4.2 Buffering

Temporarily store work in a queue or buffer.

``` text
Producer ---> Queue ---> Consumer
```

This absorbs short traffic spikes.

However, a buffer is not an infinite solution. If production remains
faster than consumption, the queue will eventually become too large.

### 4.3 Reject Requests

When the system reaches its capacity, reject new work.

For an HTTP API, this may result in responses such as:

``` http
HTTP/1.1 429 Too Many Requests
```

or sometimes:

``` http
HTTP/1.1 503 Service Unavailable
```

### 4.4 Drop Low-Priority Work

Some systems can safely discard non-critical events when overloaded.

For example:

``` text
Critical payment event     -> Keep
Analytics event            -> May drop
Optional telemetry event   -> May drop
```

This technique is often called **load shedding**.

### 4.5 Scale Consumers

If consumers are the bottleneck, add more consumers.

``` text
             +--> Consumer 1
Producer --> Queue --> Consumer 2
             +--> Consumer 3
             +--> Consumer 4
```

This works only if the workload can be processed concurrently and the
underlying dependencies can also handle the increased load.

### 4.6 Batch Processing

Instead of processing one message at a time:

``` text
message -> process
message -> process
message -> process
```

process messages in batches:

``` text
100 messages -> process as batch
```

Batching can improve throughput by reducing per-message overhead.

------------------------------------------------------------------------

## 5. Backpressure With Kafka

Suppose producers write events to Kafka at:

``` text
50,000 events/sec
```

while consumers process:

``` text
30,000 events/sec
```

Consumer lag grows by approximately:

``` text
20,000 events/sec
```

Kafka can buffer events in its partitions, but continuously increasing
**consumer lag** indicates that consumers cannot keep up.

Possible responses include:

-   Add consumer instances
-   Increase partitions when appropriate
-   Optimize consumer processing
-   Batch database operations
-   Reduce unnecessary produced events
-   Apply rate limits upstream
-   Shed non-critical work

------------------------------------------------------------------------

## 6. Backpressure With APIs

Consider:

``` text
Client
  |
  v
API
  |
  v
Database
```

The API may handle 10,000 requests/sec, but the database may safely
handle only 2,000 requests/sec.

Allowing all requests through can overload the database.

A safer architecture might introduce controls:

``` text
Clients
   |
   v
Rate Limiter
   |
   v
API
   |
   v
Queue / Concurrency Limit
   |
   v
Database
```

The goal is to protect the slower downstream dependency.

------------------------------------------------------------------------

## 7. Backpressure vs Rate Limiting

They are related but not identical.

### Rate Limiting

Answers:

> How much traffic is a client allowed to send?

Example:

``` text
100 requests/minute/user
```

It is usually a policy or traffic-control mechanism.

### Backpressure

Answers:

> What should happen when downstream components cannot keep up?

It is primarily about managing a mismatch between production rate and
processing capacity.

Rate limiting can be **one technique used to implement backpressure**.

------------------------------------------------------------------------

## 8. Backpressure vs Buffering

Buffering does not automatically solve backpressure.

A queue can temporarily absorb spikes:

``` text
Producer ---> Queue ---> Consumer
```

But if:

``` text
Producer rate > Consumer rate
```

for a long period, the queue keeps growing.

A robust system therefore usually needs:

``` text
Buffering
+
Capacity limits
+
Scaling
+
Flow control / rate limiting
+
Load shedding when necessary
```

------------------------------------------------------------------------

## 9. Real-World Example

Imagine an image-processing service.

Users upload:

``` text
5,000 images/minute
```

Workers can process:

``` text
2,000 images/minute
```

Architecture:

``` text
Users
  |
  v
Upload API
  |
  v
Queue
  |
  +--> Worker
  +--> Worker
  +--> Worker
```

If uploads remain faster than processing, queue depth increases.

Possible backpressure strategy:

1.  Monitor queue depth and processing latency.
2.  Autoscale workers.
3.  Limit concurrent uploads or jobs.
4.  Reject or delay low-priority requests when capacity is exhausted.
5.  Retry rejected transient work with exponential backoff.

------------------------------------------------------------------------

## 10. Important Metrics

Useful metrics for detecting backpressure include:

-   Queue depth
-   Consumer lag
-   Processing throughput
-   Incoming request rate
-   Error rate
-   Retry rate
-   Processing latency
-   CPU utilization
-   Memory utilization
-   Database connection-pool usage

A common warning sign is:

``` text
Incoming Rate > Processing Rate
```

for a sustained period.

------------------------------------------------------------------------

## 11. Key Takeaway

Backpressure protects a system when:

``` text
Producer Speed > Consumer Capacity
```

The system must respond by doing one or more of:

``` text
Slow down
Buffer
Scale
Reject
Drop
Batch
```

The key idea is:

> Do not allow a fast upstream component to overwhelm a slower
> downstream component.
