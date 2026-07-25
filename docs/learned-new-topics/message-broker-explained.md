# Message Broker

## 1. Introduction

A **Message Broker** is a middleware component that allows different services in a system to communicate by sending and receiving messages.

Instead of one service calling another service directly, the first service publishes a message to the Message Broker. The broker stores and delivers that message to the appropriate consumer.

A Message Broker is commonly used in distributed systems, microservices, booking platforms, payment systems, notification systems, and background processing.

---

## 2. The Basic Idea

Without a Message Broker, services communicate directly:

```text
Booking Service
      |
      | Direct API Call
      v
Notification Service
```

The Booking Service must know:

- Where the Notification Service is located.
- Whether it is currently available.
- Which API endpoint it exposes.
- How to handle its failure.
- How long to wait for its response.

With a Message Broker:

```text
Booking Service
      |
      | Publish: BookingCreated
      v
+--------------------+
|   Message Broker   |
+--------------------+
      |
      | Deliver Message
      v
Notification Service
```

The Booking Service only publishes an event. It does not need to know who will consume it.

---

## 3. Main Components

### 3.1 Producer

The **Producer** is the service that sends or publishes a message.

Example:

```text
Booking Service
```

After creating a booking, it publishes:

```text
BookingCreated
```

---

### 3.2 Message Broker

The broker receives, stores, routes, and delivers messages.

Examples:

- RabbitMQ
- Apache Kafka
- Amazon SQS
- Google Cloud Pub/Sub
- Azure Service Bus
- Redis Streams

---

### 3.3 Consumer

The **Consumer** is the service or worker that receives and processes a message.

Examples:

- Notification Service
- Analytics Service
- Background Worker
- Audit Service
- Email Worker

---

## 4. Example Message

A message is usually represented as JSON.

```json
{
  "eventId": "evt_734982",
  "eventType": "BookingCreated",
  "occurredAt": "2026-07-25T08:30:00Z",
  "bookingId": 2154,
  "customerId": 18,
  "bookingType": "FLIGHT"
}
```

The message contains the minimum information consumers need to process the event.

Avoid placing sensitive or very large data directly inside messages. Consumers can use identifiers, such as `bookingId`, to retrieve additional data when necessary.

---

## 5. Why Use a Message Broker?

### 5.1 Loose Coupling

The producer does not need to know which services consume the message.

```text
Booking Service
      |
      | BookingCreated
      v
Message Broker
   |       |       |
   v       v       v
Email   Analytics  Audit
```

Adding a new consumer does not require modifying the Booking Service.

---

### 5.2 Asynchronous Processing

The user does not need to wait for every secondary operation.

For example, after creating a booking:

1. Save the booking.
2. Return a successful response to the customer.
3. Send confirmation email in the background.
4. Update analytics in the background.
5. create an audit record in the background.

This improves response time.

---

### 5.3 Fault Tolerance

If the Notification Service is temporarily unavailable, the broker can keep the message until the service becomes available again.

```text
Booking Service
      |
      v
Message Broker
      |
      | Notification Service is unavailable
      |
      | Message remains pending
      v
Notification Service starts again
      |
      v
Message is processed
```

---

### 5.4 Scalability

Multiple consumers can process messages from the same queue.

```text
                    +--> Worker 1
Queue of Jobs ------+--> Worker 2
                    +--> Worker 3
```

Each worker processes a different message, allowing the system to handle more work.

---

### 5.5 Retry Support

If message processing fails, the broker or consumer can retry it.

Example:

```text
Send confirmation email
        |
        v
Email provider timeout
        |
        v
Retry after 10 seconds
```

Retry policies must be controlled to avoid endless retries.

---

## 6. Synchronous vs Asynchronous Communication

### Synchronous Communication

The caller waits for the result.

```text
Client
  |
  | Search Request
  v
Search Service
  |
  | External Provider API
  v
Flight Provider
```

This is appropriate when the user needs an immediate response.

Examples:

- Searching for flights.
- Searching for hotels.
- Checking the latest price.
- Revalidating an offer.
- Requesting payment authorization.

---

### Asynchronous Communication

The producer sends a message and continues without waiting for all consumers.

```text
Booking Service
      |
      | Publish BookingCreated
      v
Message Broker
      |
      v
Notification Service
```

This is appropriate when the result does not need to be returned immediately.

Examples:

- Sending confirmation emails.
- Recording audit logs.
- Updating analytics.
- Running reconciliation jobs.
- Retrying failed provider operations.
- Expiring unpaid bookings.

---

## 7. Queue vs Topic

## 7.1 Queue

A queue is commonly used when one message should be processed by one consumer instance.

```text
Producer
   |
   v
Queue
 | | |
 v v v
Worker 1
Worker 2
Worker 3
```

Although several workers listen to the queue, each message is normally handled by only one worker.

### Example

```text
GenerateTicketJob
```

If three workers are running, only one worker generates the ticket for that job.

Good use cases:

- Background jobs.
- Image processing.
- Email sending.
- Provider reconciliation.
- Booking expiration.
- Retry operations.

---

## 7.2 Topic or Publish/Subscribe

A topic is used when multiple independent consumers need to receive the same event.

```text
BookingCreated
      |
      v
Topic
  |       |       |
  v       v       v
Email  Analytics  Audit
```

Each consumer receives its own copy of the event.

Good use cases:

- Domain events.
- Notifications.
- Analytics.
- Auditing.
- Loyalty systems.
- Reporting.

---

## 8. Travel Booking Platform Example

Assume a customer completes a flight booking.

### Main Request Flow

```text
Customer
   |
   v
Backend API
   |
   v
Booking Service
   |
   +--> Revalidate the offer
   |
   +--> Create booking with provider
   |
   +--> Save booking in PostgreSQL
   |
   +--> Publish BookingCreated
   |
   v
Return success response
```

### Event Processing Flow

```text
Booking Service
      |
      | BookingCreated
      v
+--------------------+
|   Message Broker   |
+--------------------+
   |          |          |
   v          v          v
Notification Analytics  Audit
Service      Service     Worker
```

The customer does not need to wait for all secondary operations.

---

## 9. Booking Event Example

```json
{
  "eventId": "evt_booking_2154",
  "eventType": "BookingCreated",
  "eventVersion": 1,
  "occurredAt": "2026-07-25T08:30:00Z",
  "data": {
    "bookingId": 2154,
    "customerId": 18,
    "bookingType": "FLIGHT",
    "status": "PENDING_CONFIRMATION"
  }
}
```

Possible consumers:

| Consumer | Responsibility |
|---|---|
| Notification Service | Send a booking confirmation |
| Analytics Consumer | Record a new booking |
| Audit Consumer | Save an audit entry |
| Loyalty Service | Add reward points |
| Operations Dashboard | Update operational metrics |

---

## 10. Payment Example

The Payment Service successfully processes a payment.

```text
Payment Service
      |
      | Publish PaymentSucceeded
      v
Message Broker
   |            |
   v            v
Booking      Notification
Consumer     Consumer
```

Example message:

```json
{
  "eventId": "evt_payment_985",
  "eventType": "PaymentSucceeded",
  "eventVersion": 1,
  "occurredAt": "2026-07-25T08:35:00Z",
  "data": {
    "paymentId": 985,
    "bookingId": 2154,
    "amount": 820,
    "currency": "USD"
  }
}
```

Consumers may perform the following operations:

- Booking consumer changes the booking state.
- Notification consumer sends a payment confirmation.
- Accounting consumer records the financial transaction.
- Analytics consumer updates revenue metrics.

---

## 11. Cancellation and Refund Example

A customer cancels a booking.

```text
Booking Service
      |
      | BookingCancelled
      v
Message Broker
      |
      v
Refund Worker
      |
      v
Payment Gateway
```

After completing the refund:

```text
Payment Service
      |
      | RefundCompleted
      v
Message Broker
      |
      v
Notification Service
```

Example events:

```json
{
  "eventType": "BookingCancelled",
  "data": {
    "bookingId": 2154,
    "cancellationReason": "CUSTOMER_REQUEST"
  }
}
```

```json
{
  "eventType": "RefundCompleted",
  "data": {
    "bookingId": 2154,
    "refundId": 77,
    "amount": 750,
    "currency": "USD"
  }
}
```

---

## 12. Background Worker Example

Some operations should be handled by workers rather than directly inside the API request.

Example:

```text
Booking Service
      |
      | Publish IssueTicketCommand
      v
Ticket Queue
      |
      v
Ticket Worker
      |
      v
Flight Provider
```

The worker may:

1. Request ticket issuance from the provider.
2. Retry temporary failures.
3. Save the issued ticket.
4. Publish `FlightTicketIssued`.
5. Publish `FlightTicketIssuanceFailed` after final failure.

---

## 13. Events vs Commands

It is useful to distinguish between an **Event** and a **Command**.

### Event

An event describes something that already happened.

Examples:

- `BookingCreated`
- `PaymentSucceeded`
- `BookingCancelled`
- `RefundCompleted`

An event is written in the past tense.

```text
Something happened.
```

The publisher does not instruct a specific consumer.

---

### Command

A command requests an operation.

Examples:

- `SendBookingConfirmation`
- `IssueFlightTicket`
- `ExpireBooking`
- `ProcessRefund`

A command is an instruction:

```text
Please perform this operation.
```

Usually one logical consumer handles a command.

---

## 14. Recommended Events for a Travel Platform

| Event | Publisher | Possible Consumers |
|---|---|---|
| `BookingCreated` | Booking Service | Notification, Analytics, Audit |
| `BookingConfirmed` | Booking Service | Notification, Reporting |
| `BookingFailed` | Booking Service | Notification, Operations |
| `BookingCancelled` | Booking Service | Refund Worker, Notification |
| `PaymentSucceeded` | Payment Service | Booking Service, Notification |
| `PaymentFailed` | Payment Service | Booking Service, Notification |
| `RefundRequested` | Booking Service | Payment Service |
| `RefundCompleted` | Payment Service | Booking Service, Notification |
| `BookingExpired` | Background Worker | Booking Service, Notification |
| `FlightTicketIssued` | Ticket Worker | Booking Service, Notification |
| `HotelReservationConfirmed` | Provider Worker | Booking Service, Notification |
| `ProviderOperationFailed` | Provider Worker | Operations, Retry Worker |

---

## 15. Message Processing Problems

Using a Message Broker introduces important challenges.

### 15.1 Duplicate Messages

A broker may deliver the same message more than once.

Therefore, consumers should be **idempotent**.

Example:

```text
PaymentSucceeded is delivered twice.
```

The Booking Service must not confirm the booking twice or create duplicate records.

A consumer can store processed event IDs:

```text
processed_events
-------------------------
event_id
consumer_name
processed_at
```

Before processing:

```text
If eventId already exists:
    Ignore the duplicate message.
Else:
    Process the event.
    Save eventId.
```

---

### 15.2 Message Ordering

Messages may not always arrive in the expected order.

Example:

```text
PaymentSucceeded
BookingCancelled
```

A consumer must validate the current entity state before applying an event.

Use partitioning, routing keys, or entity-specific ordering only when strict ordering is required.

---

### 15.3 Poison Messages

A poison message is a message that repeatedly fails.

Example:

```json
{
  "eventType": "BookingCreated",
  "data": null
}
```

After several failed attempts, move it to a **Dead Letter Queue**.

```text
Main Queue
    |
    | Processing fails several times
    v
Dead Letter Queue
```

The operations team can inspect or replay these messages later.

---

### 15.4 Event Schema Changes

Message formats may change over time.

Use message versions:

```json
{
  "eventType": "BookingCreated",
  "eventVersion": 2,
  "data": {}
}
```

Consumers should know which versions they support.

---

## 16. Retry and Dead Letter Queue Example

```text
Booking Event
      |
      v
Notification Queue
      |
      v
Notification Consumer
      |
      | Provider timeout
      v
Retry Queue
      |
      | Delay
      v
Notification Queue
      |
      | Final failure
      v
Dead Letter Queue
```

Example retry policy:

```text
Attempt 1: immediately
Attempt 2: after 10 seconds
Attempt 3: after 1 minute
Attempt 4: after 5 minutes
Final failure: move to Dead Letter Queue
```

Use exponential backoff where appropriate.

---

## 17. Database and Message Consistency

A common problem is:

1. Save a booking in PostgreSQL.
2. Publish `BookingCreated`.
3. The application crashes between these operations.

Possible results:

- Booking saved, but no event published.
- Event published, but database transaction rolled back.

A common solution is the **Transactional Outbox Pattern**.

---

## 18. Transactional Outbox Pattern

Save the business record and event in the same database transaction.

```text
Database Transaction
    |
    +--> Insert Booking
    |
    +--> Insert Outbox Event
    |
    +--> Commit
```

Example tables:

```text
bookings
outbox_events
```

Example outbox record:

```json
{
  "id": "evt_booking_2154",
  "aggregateType": "BOOKING",
  "aggregateId": "2154",
  "eventType": "BookingCreated",
  "payload": {
    "bookingId": 2154,
    "customerId": 18
  },
  "publishedAt": null
}
```

A background publisher reads unpublished events:

```text
Outbox Publisher
      |
      | Read unpublished events
      v
Message Broker
      |
      | Publish succeeds
      v
Mark event as published
```

This prevents losing events after saving business data.

---

## 19. Example Pseudocode

### Creating a Booking

```typescript
await database.transaction(async (tx) => {
  const booking = await tx.booking.create({
    data: {
      customerId,
      type: "FLIGHT",
      status: "PENDING_CONFIRMATION"
    }
  });

  await tx.outboxEvent.create({
    data: {
      eventId: crypto.randomUUID(),
      eventType: "BookingCreated",
      aggregateType: "BOOKING",
      aggregateId: String(booking.id),
      payload: {
        bookingId: booking.id,
        customerId: booking.customerId
      }
    }
  });
});
```

### Publishing Outbox Events

```typescript
const events = await outboxRepository.findUnpublishedEvents();

for (const event of events) {
  await messageBroker.publish(event.eventType, event.payload);
  await outboxRepository.markAsPublished(event.eventId);
}
```

### Idempotent Consumer

```typescript
async function handleBookingCreated(message: BookingCreatedEvent) {
  const alreadyProcessed =
    await processedEventRepository.exists(
      message.eventId,
      "notification-service"
    );

  if (alreadyProcessed) {
    return;
  }

  await database.transaction(async (tx) => {
    await notificationService.createBookingNotification(
      message.data.bookingId,
      tx
    );

    await processedEventRepository.save(
      message.eventId,
      "notification-service",
      tx
    );
  });
}
```

---

## 20. RabbitMQ Example

RabbitMQ commonly uses:

- Producer
- Exchange
- Queue
- Binding
- Consumer

```text
Booking Service
      |
      | Routing Key: booking.created
      v
Booking Exchange
      |
      +--> Notification Queue
      |
      +--> Analytics Queue
      |
      +--> Audit Queue
```

Each queue receives a copy of the event through exchange bindings.

Example routing keys:

```text
booking.created
booking.confirmed
booking.cancelled
payment.succeeded
payment.failed
refund.completed
```

---

## 21. Kafka Example

Kafka stores messages in **topics**.

```text
Producer
   |
   v
booking-events Topic
   |
   +--> Notification Consumer Group
   |
   +--> Analytics Consumer Group
   |
   +--> Audit Consumer Group
```

Kafka keeps events for a configured retention period. Consumers track their own position using offsets.

Kafka is useful when:

- Event volume is very high.
- Events need to be retained and replayed.
- Several systems consume the same event stream.
- Real-time analytics is important.
- Ordering within partitions is needed.

---

## 22. RabbitMQ vs Kafka

| Area | RabbitMQ | Kafka |
|---|---|---|
| Main model | Queues and message routing | Distributed event log |
| Best for | Tasks, commands, workflows | Event streaming, analytics, replay |
| Message removal | Often removed after acknowledgment | Retained for a period |
| Routing | Flexible exchanges and routing keys | Topics and partitions |
| Replay | Not its primary model | Built-in through offsets |
| Complexity | Usually easier to start with | Higher operational complexity |
| Typical use | Email, jobs, retries, workflow messages | Large event streams and analytics |

For a normal travel booking MVP, RabbitMQ is often simpler and sufficient.

Kafka becomes more attractive when the platform reaches very high event volumes or requires event replay and stream processing.

---

## 23. Redis Streams

Redis Streams can also support message processing.

Advantages:

- Redis may already exist in the architecture.
- Supports consumer groups.
- Simple for moderate workloads.
- Useful for internal asynchronous jobs.

However, using Redis for caching, distributed locks, temporary sessions, and critical message delivery at the same time can increase operational risk.

For important booking and payment workflows, a dedicated broker is often easier to operate and reason about.

---

## 24. When Not to Use a Message Broker

Do not use asynchronous messages for every interaction.

A direct synchronous request is usually better when:

- The customer needs an immediate result.
- A decision is needed before continuing.
- A service must return data directly.
- The operation is simple and local.
- Adding messaging creates unnecessary complexity.

Examples:

```text
Search available flights
Get booking details
Validate a coupon
Revalidate the latest price
Request payment authorization
```

A Message Broker is more suitable for secondary work and asynchronous workflows.

---

## 25. Suggested Architecture for the Travel Platform

```text
Customer Applications
        |
        v
Backend API
        |
        +------------------+
        |                  |
        v                  v
Search Service       Booking Service
        |                  |
        v                  +--> PostgreSQL
External Providers         |
                           +--> Redis Locks
                           |
                           +--> Message Broker
                                      |
                       +--------------+--------------+
                       |              |              |
                       v              v              v
                Notification     Background      Analytics
                   Service         Workers        Consumer
```

Suggested responsibilities:

### PostgreSQL

- Bookings
- Payments
- Refunds
- Customers
- Provider references
- Audit and operational records
- Outbox events

### Redis

- Search cache
- Temporary offer data
- Distributed locks
- Idempotency keys
- Session data
- Rate limiting

### Message Broker

- Booking events
- Payment events
- Refund events
- Notification jobs
- Provider reconciliation jobs
- Retry workflows
- Background commands

---

## 26. Practical Recommendation

For the first version of the travel booking platform:

1. Keep search requests synchronous.
2. Use PostgreSQL for persistent business data.
3. Use Redis for cache, locks, temporary state, and idempotency.
4. Use RabbitMQ for background jobs and domain events.
5. Add a Dead Letter Queue for failed messages.
6. Make every consumer idempotent.
7. Include an `eventId`, `eventType`, `eventVersion`, and timestamp in every message.
8. Use the Transactional Outbox Pattern for critical booking and payment events.
9. Monitor queue size, consumer failures, retry counts, and processing time.
10. Avoid placing large or sensitive data directly inside messages.

---

## 27. Summary

A Message Broker acts as an intermediary between services.

It provides:

- Loose coupling.
- Asynchronous processing.
- Better scalability.
- Retry mechanisms.
- Temporary message storage.
- Fault isolation.
- Publish/subscribe communication.
- Background job processing.

In a travel booking platform, it is especially useful for:

- Booking events.
- Payment events.
- Refund processing.
- Notifications.
- Ticket issuance.
- Provider reconciliation.
- Retry operations.
- Analytics and auditing.

The Message Broker does not replace synchronous APIs or the database. It complements them by handling communication and work that does not need to complete inside the customer's immediate request.
