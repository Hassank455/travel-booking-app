# ADR-012: Separate User Identity from Customer Profile

## Status

Accepted

---

## Context

The platform serves multiple types of users, including:

- Customers
- Administrators
- Support Agents
- Finance Users

Authentication and authorization represent infrastructure concerns, while customer information belongs to the business domain.

If all customer-related information is stored directly inside the `users` table, authentication becomes tightly coupled to booking and customer management.

As the platform grows, different user types may require authentication without being customers.

---

## Decision

The platform separates authentication identity from customer business data.

The `users` table owns identity-related information such as:

- Email
- Password Hash
- Account Status
- Email Verification
- Authentication Metadata

The `customers` table owns business-related information such as:

- Full Name
- Phone Number
- Traveler Profiles
- Addresses
- Bookings
- Preferences

Conceptually:

```text
User
 │
 │ 1 : 0..1
 ▼
Customer
```

Every Customer is associated with one User.

However, not every User must have a Customer profile.

For example:

```text
Admin User
    │
    └── No Customer Profile

Support User
    │
    └── No Customer Profile

Customer User
    │
    └── Customer Profile
```

This separation keeps authentication concerns independent from business data.

---

## Alternatives Considered

### Single User Table

Store authentication and customer information together.

**Pros**

- Simpler schema.
- Fewer joins.
- Faster initial implementation.

**Cons**

- Authentication becomes coupled to business logic.
- Admin and support users contain unnecessary customer fields.
- Harder future evolution.

---

### Separate Authentication Service

Move authentication into an independent service.

**Pros**

- Strong separation of concerns.
- Independent scalability.

**Cons**

- Additional infrastructure.
- Distributed authentication.
- Unnecessary complexity for the current Modular Monolith architecture.

---

## Consequences

### Positive

- Clear separation between identity and business domains.
- Supports multiple user types naturally.
- Easier future integration with external identity providers.
- Cleaner authorization model.
- Better long-term maintainability.

### Negative

- Additional relationship between tables.
- Slightly more complex queries.
- User and Customer creation must be coordinated.

---

## Related Documents

- `07-domain-model.md`
- `12-database-design.md`
- `13-security.md`