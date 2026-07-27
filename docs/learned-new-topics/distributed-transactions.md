# Distributed Transactions

> A comprehensive guide to Distributed Transactions, their challenges, common implementation strategies, and real-world usage in distributed systems and microservices.

---

# What is a Distributed Transaction?

A **Distributed Transaction** is a transaction that spans **multiple services, databases, or systems**, where all participating components must reach a consistent final state.

Unlike a traditional database transaction, a distributed transaction coordinates work across multiple independent resources.

Example:

```
Booking Service
    ↓
Inventory Service
    ↓
Payment Service
    ↓
Notification Service
```

Each service owns its own database.

```
Booking DB
Inventory DB
Payment DB
```

There is **no single database transaction** that can wrap all of them together.

---

# Traditional Transaction vs Distributed Transaction

## Traditional Transaction

Everything happens inside one database.

```
BEGIN;

Update Account A
Update Account B

COMMIT;
```

If anything fails:

```
ROLLBACK;
```

The database guarantees:

- Atomicity
- Consistency
- Isolation
- Durability (ACID)

---

## Distributed Transaction

Operations span multiple databases.

Example:

```
Booking DB

↓

Inventory DB

↓

Payment DB

↓

Notification Service
```

No database can rollback changes made by another database.

---

# The Problem

Suppose a customer books a flight.

The booking workflow:

```
1. Create Booking

2. Reserve Seat

3. Charge Credit Card

4. Send Confirmation Email
```

Now imagine:

```
Booking Created ✅

Seat Reserved ✅

Payment Failed ❌
```

Current system state:

```
Booking Exists

Seat Reserved

No Payment
```

The system is inconsistent.

---

# Why Distributed Transactions are Difficult

Every service owns its own data.

```
Booking Service
      │
Booking Database
```

```
Payment Service
      │
Payment Database
```

Neither service knows anything about the other's transaction.

There is no global transaction manager built into databases.

---

# Common Solutions

There are two major approaches.

1. Two-Phase Commit (2PC)
2. Saga Pattern

---

# Two-Phase Commit (2PC)

## Idea

A central coordinator asks every participant if they are ready.

```
             Coordinator

          /      |       \
Booking   Payment   Inventory
```

It executes two phases.

---

## Phase 1 — Prepare

Coordinator asks:

```
Can you commit?
```

Booking

```
YES
```

Inventory

```
YES
```

Payment

```
YES
```

No one commits yet.

Everything is prepared.

---

## Phase 2 — Commit

Coordinator sends:

```
COMMIT
```

All participants permanently save their changes.

---

## Failure Example

Suppose Payment replies:

```
NO
```

Coordinator sends:

```
ROLLBACK
```

Everyone discards their prepared work.

---

## Advantages

- Strong consistency
- ACID-like behavior
- Simple conceptually

---

## Disadvantages

### Blocking

If the coordinator crashes...

```
Coordinator ❌
```

Participants remain waiting.

---

### Slow

Everyone waits for everyone else.

---

### Poor Scalability

Large distributed systems become slower.

---

### Network Sensitive

Temporary network failures may block the entire transaction.

---

# Why Large Companies Rarely Use 2PC

Modern cloud systems prioritize:

- Availability
- Scalability
- Fault tolerance

2PC hurts all three.

Therefore companies like:

- Amazon
- Netflix
- Uber
- Booking.com

rarely rely on 2PC for business workflows.

---

# Saga Pattern

Saga is the modern alternative.

Instead of one huge transaction...

```
Booking

↓

Inventory

↓

Payment

↓

Notification
```

Each step commits independently.

---

# Compensating Transactions

If something fails later...

Instead of rollback...

Execute another transaction that undoes the previous work.

Example:

```
Booking Created
```

Compensation:

```
Cancel Booking
```

Example:

```
Seat Reserved
```

Compensation:

```
Release Seat
```

---

# Complete Example

Customer books a flight.

```
Booking Created

↓

Seat Reserved

↓

Payment Failed
```

Compensation starts.

```
Release Seat

↓

Cancel Booking
```

Final state:

```
No Booking

Seat Available
```

The system becomes consistent again.

---

# Saga Workflow Example

```
Customer

↓

Booking Service

↓

Booking Created

↓

Inventory Service

↓

Seat Reserved

↓

Payment Service

↓

Charge Card
```

If payment succeeds:

```
Issue Ticket

↓

Send Email
```

If payment fails:

```
Release Seat

↓

Cancel Booking
```

---

# Types of Saga

There are two implementations.

---

# 1. Choreography

Services communicate only using events.

```
Booking Created Event

↓

Inventory Service

↓

Seat Reserved Event

↓

Payment Service

↓

Payment Successful Event

↓

Notification Service
```

No central controller exists.

Each service listens for events.

---

## Advantages

- Highly decoupled
- Easy to scale
- No single point of failure

---

## Disadvantages

- Hard to understand
- Difficult debugging
- Event chains become complicated

---

# 2. Orchestration

A central Saga Orchestrator controls everything.

```
Saga Orchestrator

↓

Booking

↓

Inventory

↓

Payment

↓

Notification
```

If payment fails:

```
Orchestrator

↓

Release Seat

↓

Cancel Booking
```

---

## Advantages

- Easy to monitor
- Easier debugging
- Central business workflow
- Clear execution flow

---

## Disadvantages

- Orchestrator becomes an important component
- Slightly more coupling

---

# Real-World Example

Booking Platform

```
Customer

↓

Booking Service

↓

Flight Provider

↓

Payment Service

↓

Notification Service
```

Scenario:

```
Create Booking

↓

Reserve Seat

↓

Charge Customer

↓

Issue Ticket

↓

Send Email
```

Failure:

```
Payment Failed
```

Compensation:

```
Release Seat

↓

Cancel Booking

↓

Refund (if needed)
```

---

# Another Example

E-commerce Order

```
Create Order

↓

Reserve Inventory

↓

Process Payment

↓

Arrange Shipping
```

If shipping fails:

```
Cancel Shipment

↓

Refund Payment

↓

Return Inventory
```

---

# Banking Example

Money Transfer

```
Withdraw

↓

Deposit
```

Banks often require stronger consistency.

Some systems still use Two-Phase Commit.

Others use specialized distributed transaction protocols.

---

# Food Delivery Example

```
Create Order

↓

Reserve Food

↓

Process Payment

↓

Assign Driver
```

Driver unavailable?

```
Refund Customer

↓

Release Reserved Food

↓

Cancel Order
```

---

# Eventual Consistency

Distributed systems often sacrifice immediate consistency.

Instead they use:

```
Eventual Consistency
```

Meaning:

The system may be temporarily inconsistent.

Eventually...

All services converge to the correct state.

Example:

```
Booking Created

↓

Payment Processing...

↓

Inventory Updating...

↓

Notification Sending...
```

For a few seconds, services may show different states.

Eventually:

```
Everything becomes consistent.
```

---

# When Should You Use Distributed Transactions?

Use them whenever one business operation touches multiple services.

Examples:

- Flight booking
- Hotel booking
- Order processing
- Payment systems
- Ride booking
- Food delivery
- Warehouse management
- Insurance systems
- Banking systems

---

# When Should You Avoid 2PC?

Avoid it when:

- Building cloud-native applications
- Using microservices
- High scalability is required
- High availability is required
- Network latency is expected

Prefer Saga instead.

---

# Best Practices

✔ Keep each local transaction small.

✔ Design idempotent operations.

✔ Implement compensating transactions.

✔ Use reliable messaging.

✔ Store Saga state.

✔ Add retries.

✔ Handle duplicate events.

✔ Make compensation reversible whenever possible.

✔ Monitor every Saga execution.

✔ Log every business event.

---

# Comparison

| Feature | Traditional Transaction | Two-Phase Commit | Saga Pattern |
|----------|-------------------------|------------------|--------------|
| Single Database | ✅ | ❌ | ❌ |
| Multiple Databases | ❌ | ✅ | ✅ |
| Immediate Consistency | ✅ | ✅ | ❌ |
| Eventual Consistency | ❌ | ❌ | ✅ |
| Rollback | Database Rollback | Global Rollback | Compensation |
| Scalability | Medium | Low | High |
| Availability | High | Low | High |
| Complexity | Low | Medium | High |
| Best for Microservices | ❌ | Rarely | ✅ |

---

# Summary

Distributed Transactions solve the challenge of keeping multiple services consistent during a single business operation.

The two primary approaches are:

## Two-Phase Commit

- Strong consistency
- Coordinator based
- Blocking
- Rarely used in modern cloud systems

---

## Saga Pattern

- Local transactions
- Compensating transactions
- Eventual consistency
- Highly scalable
- Preferred for modern microservices

---

# Key Takeaways

- A database transaction cannot span multiple independent databases.
- Distributed Transactions coordinate work across multiple services.
- Two-Phase Commit guarantees consistency but sacrifices scalability.
- Saga Pattern embraces eventual consistency using compensating transactions.
- Nearly all modern booking, payment, and e-commerce platforms use Saga-based workflows rather than Two-Phase Commit.