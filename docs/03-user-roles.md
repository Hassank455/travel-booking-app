# User Roles

## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Role Model Overview](#2-role-model-overview)
- [3. External Users](#3-external-users)
  - [3.1 Guest](#31-guest)
  - [3.2 Registered Customer](#32-registered-customer)
- [4. Internal Users](#4-internal-users)
  - [4.1 Support Agent](#41-support-agent)
  - [4.2 Operations Agent](#42-operations-agent)
  - [4.3 Finance Agent](#43-finance-agent)
  - [4.4 Provider Manager](#44-provider-manager)
  - [4.5 Administrator](#45-administrator)
  - [4.6 Super Administrator](#46-super-administrator)
- [5. System Actors](#5-system-actors)
  - [5.1 Flight Provider](#51-flight-provider)
  - [5.2 Hotel Provider](#52-hotel-provider)
  - [5.3 Payment Gateway](#53-payment-gateway)
  - [5.4 Notification Provider](#54-notification-provider)
  - [5.5 Background Worker](#55-background-worker)
- [6. Role Permission Matrix](#6-role-permission-matrix)
- [7. Role Boundaries](#7-role-boundaries)
- [8. Ownership and Access Rules](#8-ownership-and-access-rules)
- [9. Sensitive Operations](#9-sensitive-operations)
- [10. Role Assignment Rules](#10-role-assignment-rules)
- [11. Audit Requirements](#11-audit-requirements)
- [12. MVP Role Scope](#12-mvp-role-scope)
- [13. Future Role Extensions](#13-future-role-extensions)
- [14. Related Documents](#14-related-documents)
- [15. Summary](#15-summary)

---

## 1. Purpose

This document defines the actors that interact with the travel booking platform and describes their responsibilities, permissions, and access boundaries.

The platform includes two main categories of users:

- External users who search for and book travel services.
- Internal users who operate, support, monitor, and administer the platform.

It also interacts with external systems and automated processes that participate in search, booking, payment, and notification workflows.

This document focuses on role responsibilities and authorization boundaries.

Detailed functional behavior will be documented in [Functional Requirements](04-functional-requirements.md), while authentication and authorization controls will be documented in [Security](14-security.md).

---

## 2. Role Model Overview

The initial role model includes the following actors:

```text
External Users
├── Guest
└── Registered Customer

Internal Users
├── Support Agent
├── Operations Agent
├── Finance Agent
├── Provider Manager
├── Administrator
└── Super Administrator

System Actors
├── Flight Provider
├── Hotel Provider
├── Payment Gateway
├── Notification Provider
└── Background Worker
```

The system should follow the principle of least privilege.

Each role should receive only the permissions required to perform its responsibilities.

A user may have one or more internal roles if the authorization model supports multiple role assignments.

---

## 3. External Users

## 3.1 Guest

A Guest is a user who interacts with the platform without being authenticated.

### Responsibilities

A Guest may:

- Search for flights.
- Search for hotels.
- View search results.
- Apply basic filters.
- Sort search results.
- View flight-offer details.
- View hotel and room details.
- Compare available offers.
- Start the booking process.
- Register a new account.
- Log in to an existing account.
- Request password recovery.

### Access Limitations

A Guest cannot:

- Complete a booking without providing the required identity and contact information.
- View another user's bookings.
- Access booking history.
- Save traveler profiles.
- Access internal administration features.
- View internal provider references.
- View internal payment details.
- Perform administrative operations.

The MVP may require the Guest to create an account or log in before completing payment or confirming a booking.

The final guest-booking policy will be defined in [Business Rules](06-business-rules.md).

---

## 3.2 Registered Customer

A Registered Customer is an authenticated user who can search for and book travel services.

### Responsibilities

A Registered Customer may:

- Perform all Guest actions.
- Manage personal profile information.
- Manage contact information.
- Save traveler or passenger information.
- Save hotel guest information.
- Select a flight offer.
- Select a hotel offer.
- Complete a booking.
- Complete payment.
- View personal booking history.
- View booking details.
- View payment status.
- Access available tickets.
- Access available hotel vouchers.
- Request cancellation where supported.
- Track cancellation status.
- Track refund status.
- Receive booking notifications.

### Ownership Rules

A Registered Customer may access only:

- Their own profile.
- Their own traveler profiles.
- Their own bookings.
- Their own payment summaries.
- Their own refund records.
- Their own tickets and vouchers.
- Bookings explicitly associated with their account.

A customer must not be able to retrieve another customer's data by changing a booking identifier, payment identifier, or provider reference.

Detailed ownership rules are described in [Security](14-security.md).

### Access Limitations

A Registered Customer cannot:

- Modify provider configurations.
- Enable or disable providers.
- View internal operational notes.
- View full payment-gateway payloads.
- Change internal booking states directly.
- Approve refunds manually.
- Access audit logs.
- Access bookings owned by other customers.
- Perform internal support or administration operations.

---

## 4. Internal Users

## 4.1 Support Agent

A Support Agent helps customers understand and resolve booking-related issues.

### Responsibilities

A Support Agent may:

- Search for customers.
- Search for bookings.
- View booking summaries.
- View booking status.
- View payment status.
- View cancellation status.
- View refund status.
- View provider confirmation references.
- View booking timeline information.
- View operational notes.
- Add support notes.
- Identify bookings requiring escalation.
- Resend eligible customer notifications.
- Guide customers through supported cancellation procedures.

### Access Limitations

A Support Agent should not:

- View raw payment-card information.
- Change payment records directly.
- Approve refunds without authorization.
- Change provider configuration.
- Enable or disable providers.
- Change user roles.
- Delete audit records.
- Change confirmed booking data without a controlled operation.
- Manually force a booking to a successful state.

A Support Agent may initiate controlled actions only when explicitly allowed by business rules.

Examples include:

- Resending a confirmation email.
- Requesting booking-status synchronization.
- Starting an approved cancellation workflow.
- Escalating an unresolved booking.

---

## 4.2 Operations Agent

An Operations Agent monitors booking workflows and resolves operational failures.

### Responsibilities

An Operations Agent may:

- View all operational bookings.
- View provider request status.
- View provider references.
- View booking workflow history.
- View failed booking operations.
- View uncertain provider operations.
- Retry approved operational steps.
- Trigger provider-status reconciliation.
- Move unresolved bookings to manual review.
- Add internal operational notes.
- Review bookings stuck in processing.
- Review provider timeout cases.
- Coordinate cancellation workflows.
- Coordinate booking recovery or compensation steps.

### Access Limitations

An Operations Agent should not:

- Change user roles.
- Modify payment-gateway credentials.
- Modify provider credentials unless separately authorized.
- Access raw payment-card data.
- Delete booking history.
- Delete provider-call history.
- Mark payments as successful manually.
- Issue refunds outside the approved refund workflow.

Detailed recovery and reconciliation behavior will be documented in [Failure Scenarios](13-failure-scenarios.md).

---

## 4.3 Finance Agent

A Finance Agent manages payment, refund, and financial reconciliation operations.

### Responsibilities

A Finance Agent may:

- View payment records.
- View payment transaction history.
- View refund records.
- View payment-gateway references.
- Review payment failures.
- Review refund failures.
- Initiate approved refunds.
- Retry supported refund operations.
- Reconcile platform records with gateway records.
- Review duplicate-payment risks.
- Review bookings with financial inconsistencies.
- Add finance-related internal notes.
- Export approved financial reports when available.

### Access Limitations

A Finance Agent should not:

- Access raw payment-card details.
- Modify flight or hotel offer data.
- Change provider configurations.
- Change customer identities.
- Change booking ownership.
- Change user roles.
- Delete financial audit records.
- Mark bookings as provider-confirmed without provider evidence.

Booking state and payment state must remain independently controlled.

For more details, see [Booking Orchestration](09-booking-orchestration.md).

---

## 4.4 Provider Manager

A Provider Manager manages external flight and hotel provider integrations.

### Responsibilities

A Provider Manager may:

- View configured providers.
- View provider capabilities.
- View provider health.
- View provider response times.
- View provider error rates.
- Enable or disable a provider.
- Configure provider priorities.
- Configure supported provider operations.
- Manage provider environment settings.
- Review provider rate-limit usage.
- Review provider integration errors.
- Coordinate provider certification.
- Manage provider-specific operational notes.

### Access Limitations

A Provider Manager should not:

- Access raw customer payment data.
- Approve customer refunds.
- Change customer roles.
- Delete booking records.
- Change confirmed booking ownership.
- View unnecessary personal customer information.
- Change core authorization policies.

Sensitive provider credentials should be stored securely and should not be displayed in plain text after configuration.

Detailed provider architecture will be documented in [Search Architecture](08-search-architecture.md) and [Booking Orchestration](09-booking-orchestration.md).

---

## 4.5 Administrator

An Administrator manages the day-to-day configuration and operation of the platform.

### Responsibilities

An Administrator may:

- View users.
- Activate or deactivate user accounts.
- View bookings.
- View payment and refund summaries.
- View provider configurations.
- View operational dashboards.
- View audit history.
- Manage selected internal configurations.
- Assign approved internal roles.
- Review manual-review cases.
- Manage platform content where applicable.
- Review system health.
- Review failed background operations.

### Access Limitations

An Administrator should not automatically receive unrestricted access to:

- Raw secrets.
- Raw payment-card data.
- Production database credentials.
- Infrastructure root access.
- Permanent audit-log deletion.
- Super Administrator operations.
- Security policy overrides.

Administrative privileges should be divided where practical to reduce the risk of misuse or accidental damage.

---

## 4.6 Super Administrator

A Super Administrator has the highest application-level administrative authority.

This role should be used only for a small number of trusted users.

### Responsibilities

A Super Administrator may:

- Perform all Administrator operations.
- Create and manage internal user accounts.
- Assign or revoke internal roles.
- Configure high-risk application settings.
- Approve sensitive operational changes.
- Manage authorization policies.
- Manage provider-manager access.
- Review complete audit history.
- Handle emergency access scenarios.
- Disable compromised internal accounts.
- Approve exceptional manual interventions.

### Access Limitations

Even a Super Administrator should not:

- Access raw payment-card data.
- Bypass external provider confirmation requirements.
- Delete immutable audit records.
- modify financial history without an auditable operation.
- Directly alter production data outside controlled procedures.
- Share secrets through the application interface.

Super Administrator actions must always be audited.

---

## 5. System Actors

System Actors are not human roles, but they interact with the platform and participate in important workflows.

## 5.1 Flight Provider

A Flight Provider supplies flight-related data and booking capabilities.

It may:

- Return flight availability.
- Return flight pricing.
- Return fare conditions.
- Revalidate flight offers.
- Create reservations.
- Confirm bookings.
- Issue ticket references.
- Return booking status.
- Process cancellation requests.

The Flight Provider remains the source of truth for provider-side availability and confirmation.

---

## 5.2 Hotel Provider

A Hotel Provider supplies hotel, room, and rate-plan data.

It may:

- Return hotel information.
- Return room availability.
- Return pricing and taxes.
- Return cancellation conditions.
- Revalidate room offers.
- Create hotel reservations.
- Confirm hotel bookings.
- Return voucher references.
- Return booking status.
- Process cancellation requests.

The Hotel Provider remains the source of truth for provider-side hotel availability and confirmation.

---

## 5.3 Payment Gateway

The Payment Gateway processes customer payments.

It may:

- Create payment sessions.
- Authorize payments.
- Capture payments.
- Return payment status.
- Send payment webhooks.
- Process refunds.
- Return transaction references.

The platform must verify gateway callbacks before updating internal payment records.

---

## 5.4 Notification Provider

A Notification Provider delivers customer notifications.

It may provide:

- Email delivery.
- SMS delivery.
- Push-notification delivery.

A notification-delivery failure must not invalidate an otherwise confirmed booking.

---

## 5.5 Background Worker

A Background Worker performs asynchronous system operations.

It may:

- Retry approved provider calls.
- Reconcile uncertain bookings.
- Process provider webhooks.
- Process payment webhooks.
- Send notifications.
- Retry failed notifications.
- Expire temporary data.
- Process cancellation requests.
- Track refunds.
- Detect bookings stuck in processing.

Background workers must use the same authorization and business rules as synchronous application flows.

---

## 6. Role Permission Matrix

The following matrix describes the expected permission level at a high level.

| Capability | Guest | Customer | Support | Operations | Finance | Provider Manager | Admin | Super Admin |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Search flights and hotels | Yes | Yes | Optional | Optional | No | Optional | Yes | Yes |
| View public offer details | Yes | Yes | Yes | Yes | No | Yes | Yes | Yes |
| Create personal booking | Limited | Yes | No | No | No | No | Optional | Optional |
| View own bookings | No | Yes | No | No | No | No | Yes | Yes |
| View customer bookings | No | No | Yes | Yes | Limited | Limited | Yes | Yes |
| View payment status | No | Own only | Summary | Yes | Yes | No | Yes | Yes |
| Initiate cancellation | No | Own only | Controlled | Controlled | No | No | Yes | Yes |
| Initiate refund | No | No | No | Controlled | Yes | No | Controlled | Yes |
| Retry booking operation | No | No | No | Controlled | No | No | Controlled | Yes |
| Reconcile payment | No | No | No | Limited | Yes | No | Yes | Yes |
| Manage providers | No | No | No | No | No | Yes | Yes | Yes |
| Assign internal roles | No | No | No | No | No | No | Limited | Yes |
| View audit logs | No | No | Limited | Limited | Limited | Limited | Yes | Yes |
| Manage authorization policies | No | No | No | No | No | No | No | Yes |

`Controlled` means the operation is available only through an approved workflow with validation, authorization, and audit logging.

The final permission matrix may change as the authorization model is refined.

---

## 7. Role Boundaries

The platform should preserve clear boundaries between responsibilities.

### Customer Support Boundary

Support Agents help customers and inspect booking information, but they should not control financial or provider configuration.

### Operations Boundary

Operations Agents resolve workflow issues, but they should not modify payment history or user authorization.

### Finance Boundary

Finance Agents manage payments and refunds, but they should not control provider booking confirmation.

### Provider Management Boundary

Provider Managers control integration configuration, but they should not access unnecessary customer or financial data.

### Administration Boundary

Administrators manage platform operations, but highly sensitive security and role-management actions remain restricted.

These boundaries reduce operational risk and improve accountability.

---

## 8. Ownership and Access Rules

The platform must enforce ownership and access rules at the backend.

Frontend visibility alone is not an authorization control.

### Customer Data Ownership

A customer may access only records associated with their account.

This includes:

- Profile information.
- Traveler profiles.
- Bookings.
- Payment summaries.
- Refund records.
- Tickets.
- Hotel vouchers.

### Internal Access

Internal access must be:

- Role-based.
- Limited to business need.
- Logged.
- Protected by authentication.
- Reviewed periodically.

### Provider References

Provider references may be visible to internal users where required.

Customers may receive selected provider references, such as:

- Flight reservation code.
- Ticket number.
- Hotel confirmation number.
- Hotel voucher reference.

Internal-only provider identifiers and raw payloads should not be exposed unnecessarily.

---

## 9. Sensitive Operations

The following operations are considered sensitive:

- Assigning or revoking internal roles.
- Enabling or disabling providers.
- Updating provider credentials.
- Initiating refunds.
- Retrying uncertain bookings.
- Forcing reconciliation.
- Deactivating internal accounts.
- Accessing detailed audit history.
- Modifying security configuration.
- Performing manual booking recovery.

Sensitive operations should require:

- Explicit authorization.
- Input validation.
- Reason or justification where appropriate.
- Audit logging.
- Idempotency where applicable.
- Additional approval for high-risk actions.

Some operations may require a dual-approval process in future versions.

---

## 10. Role Assignment Rules

The following rules should apply to internal role assignment:

1. External users must not assign roles to themselves.
2. Customers must not become internal users through public APIs.
3. Internal roles should be assigned only by authorized administrators.
4. Super Administrator assignment should be highly restricted.
5. Role changes must be audited.
6. Disabled internal users must immediately lose access.
7. Role removal must invalidate active privileged sessions where required.
8. Shared internal accounts should not be allowed.
9. Each internal user should have an individual account.
10. Temporary elevated access should expire automatically when supported.

The technical implementation will be documented in [Security](14-security.md).

---

## 11. Audit Requirements

The platform should audit important internal operations.

Audit records should capture:

- The acting user.
- The role used.
- The operation.
- The affected entity.
- The previous value when applicable.
- The new value when applicable.
- The date and time.
- The request identifier.
- The reason when required.
- The result of the operation.

Examples of auditable actions include:

- Role assignment.
- Account deactivation.
- Provider enablement.
- Provider disablement.
- Refund initiation.
- Booking recovery.
- Cancellation intervention.
- Manual-review resolution.
- Security configuration changes.

Audit logs should not be editable through normal application workflows.

Detailed audit requirements will be documented in [Observability](15-observability.md) and [Security](14-security.md).

---

## 12. MVP Role Scope

The MVP does not need to implement every internal role as a separate application role from the first release.

A practical MVP may begin with:

```text
GUEST
CUSTOMER
SUPPORT
ADMIN
SUPER_ADMIN
```

In the MVP:

### Support

The Support role may combine basic support and operational visibility.

### Admin

The Admin role may temporarily combine:

- Operations.
- Finance oversight.
- Provider management.
- Basic administration.

### Super Admin

The Super Admin role manages:

- Internal role assignment.
- High-risk settings.
- Emergency operations.

As the system and operations team grow, the broad Admin role should be separated into:

- Operations Agent.
- Finance Agent.
- Provider Manager.
- Administrator.

This avoids premature authorization complexity while preserving a clear future model.

---

## 13. Future Role Extensions

Future versions may introduce additional roles.

### Travel Agent

May create and manage bookings on behalf of customers.

### Corporate Travel Manager

May manage business travelers, travel policies, and company bookings.

### Finance Manager

May approve refunds above configured limits.

### Security Administrator

May manage security settings independently from business administration.

### Auditor

May access read-only operational and financial audit records.

### Content Manager

May manage public content, destination information, and platform help pages.

### Partner Administrator

May manage a white-label or business-to-business partner account.

These roles are outside the current MVP scope.

See [Project Scope](02-scope.md) for excluded and future capabilities.

---

## 14. Related Documents

- [Product Overview](01-product-overview.md)
- [Project Scope](02-scope.md)
- [Functional Requirements](04-functional-requirements.md)
- [Non-Functional Requirements](05-non-functional-requirements.md)
- [Business Rules](06-business-rules.md)
- [Domain Model](07-domain-model.md)
- [Search Architecture](08-search-architecture.md)
- [Booking Orchestration](09-booking-orchestration.md)
- [Database Design](11-database-design.md)
- [API Design](12-api-design.md)
- [Failure Scenarios](13-failure-scenarios.md)
- [Security](14-security.md)
- [Observability](15-observability.md)

---

## 15. Summary

The platform supports external users, internal users, and automated system actors.

External users include:

- Guests.
- Registered Customers.

Internal responsibilities are separated across:

- Customer support.
- Booking operations.
- Finance.
- Provider management.
- Administration.
- Security-sensitive administration.

The MVP may begin with a smaller role set, while preserving the ability to introduce more granular roles later.

All access must be enforced by the backend, follow the principle of least privilege, and generate audit records for sensitive operations.