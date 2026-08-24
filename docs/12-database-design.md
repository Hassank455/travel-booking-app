# 12. Database Design

## 1. Purpose

This document defines the persistence design of the Travel Booking Platform.

It focuses on the core business data that must be stored permanently in PostgreSQL and explains the main entities, relationships, and persistence boundaries.

The design intentionally separates temporary search data from transactional business data.

Temporary data such as search results and offers is not persisted as permanent business entities.

---

## 2. Persistence Principles

The database design follows these principles:

- PostgreSQL is the source of truth for platform-owned transactional data.
- External providers remain the source of truth for live flight and hotel inventory, pricing, and availability.
- Search results and offers remain temporary and are not stored as permanent database entities.
- Flight bookings and hotel bookings are modeled separately because they belong to different business domains.
- Historical booking data must remain stable after confirmation.
- Provider-specific identifiers are stored using generic platform fields rather than provider-specific column names.
- Transactional data is normalized unless a clear performance requirement justifies controlled denormalization.
- Booking, transaction, and provider references must preserve strong referential integrity where possible.

---

# 3. Flight Booking Persistence Design

## 3.1 Overview

`flight_bookings` represents the platform's persistent record of a flight booking.

It does not represent a search result or an offer.

The selected flight offer is temporary and may exist in Redis during the search and booking workflow.

After the booking is successfully created, the platform stores the important accepted business information as a permanent booking snapshot.

The flight booking is associated with:

- A customer.
- A provider.
- One transaction.
- One or more passengers.
- One or more flight segments.

The initial design remains normalized.

Flight-specific information such as departure and arrival timestamps is stored in `flight_segments` instead of being duplicated in `flight_bookings`.

---

## 3.2 `flight_bookings`

### Proposed Fields

```text
flight_bookings

id
customer_id
transaction_id

provider_id
provider_offer_reference
provider_booking_reference
provider_confirmation_reference

status
trip_type

total_amount
currency

booked_at
created_at
updated_at
```

---

## 3.3 Field Definitions

### `id`

**Purpose**

Represents the unique internal identifier of the flight booking inside the platform.

This identifier belongs to our system and is independent from any external provider identifier.

**Example**

```text
fb_01J8X92H8F4K...
```

or, depending on the selected identifier strategy:

```text
982341
```

**Important**

The internal booking ID should never depend on Duffel or any future provider.

---

### `customer_id`

**Purpose**

References the customer who created and owns the booking.

The customer does not necessarily have to be one of the passengers.

For example, a customer may book a flight for:

- Themselves.
- A family member.
- A friend.
- Multiple travelers.

**Example**

```text
customer_id = 1842
```

Relationship:

```text
Customer
   1
   │
   └──── *
      FlightBooking
```

---

### `transaction_id`

**Purpose**

References the financial transaction associated with the booking.

The current business rule assumes:

```text
One Flight Booking
        ↓
One Transaction
```

The foreign key is stored in `flight_bookings` following the reverse foreign key approach.

**Example**

```text
transaction_id = 74091
```

Recommended constraint:

```text
UNIQUE FOREIGN KEY
```

This prevents the same transaction from being associated with multiple flight bookings.

The exact transaction model will be documented separately.

---

### `provider_id`

**Purpose**

Identifies the external provider responsible for the booking.

The platform should not use provider-specific columns such as:

```text
duffel_id
duffel_order_id
```

Instead, the provider relationship remains generic.

**Example**

```text
provider_id = 1
```

Where:

```text
providers

1 → Duffel
2 → LiteAPI
```

For a flight booking:

```text
provider_id = 1
```

would currently represent Duffel.

This allows another flight provider to be introduced later without changing the database schema.

---

### `provider_offer_reference`

**Purpose**

Stores the external provider reference of the offer from which the booking was created.

The offer itself is temporary and is not stored permanently as a full database entity.

However, keeping the original provider offer reference is useful for:

- Debugging.
- Customer support.
- Reconciliation.
- Tracing the booking back to the original provider offer.

**Duffel Example**

```text
provider_offer_reference = off_0000A3B4C5...
```

This value corresponds to the external Duffel Offer ID.

The platform still uses the generic name:

```text
provider_offer_reference
```

rather than:

```text
duffel_offer_id
```

---

### `provider_booking_reference`

**Purpose**

Stores the provider's primary identifier for the created booking.

This identifies the booking inside the external provider's system.

**Duffel Example**

Duffel represents a confirmed flight booking using an Order.

An example value may look like:

```text
provider_booking_reference = ord_00009hthhsUZ8W4Lx
```

In this case:

```text
provider_booking_reference
```

maps to the Duffel Order ID.

The generic name allows future flight providers to use the same field.

---

### `provider_confirmation_reference`

**Purpose**

Stores the booking confirmation reference issued by the airline or underlying travel supplier when available.

This reference may be different from the provider's internal booking identifier.

For flight bookings, this is commonly similar to a:

```text
PNR
Booking Reference
Record Locator
```

**Example**

```text
provider_confirmation_reference = RZPNX8
```

Conceptually:

```text
provider_booking_reference
→ Duffel booking/order identifier

provider_confirmation_reference
→ Airline booking reference / PNR
```

Both values are useful because they identify the booking at different integration layers.

---

### `status`

**Purpose**

Represents the current lifecycle state of the flight booking inside our platform.

Possible values may include:

```text
PENDING
PROCESSING
CONFIRMED
FAILED
CANCELLED
COMPLETED
MANUAL_REVIEW
```

**Example**

```text
status = CONFIRMED
```

The platform booking status should not blindly mirror the provider's raw status.

Provider-specific statuses should first be mapped into the platform's normalized booking lifecycle.

---

### `trip_type`

**Purpose**

Represents the general structure of the booked journey.

Initial supported values may include:

```text
ONE_WAY
ROUND_TRIP
```

Future support may include:

```text
MULTI_CITY
```

**Example**

```text
trip_type = ROUND_TRIP
```

This field describes the business meaning of the overall itinerary.

Detailed route information remains inside `flight_segments`.

---

### `total_amount`

**Purpose**

Stores the final amount accepted by the customer for the confirmed flight booking.

This is part of the historical booking snapshot.

The value must represent the amount accepted during the final booking process rather than a newer provider price retrieved later.

**Example**

```text
total_amount = 420.50
```

A precise decimal type should be used when the physical schema is defined.

---

### `currency`

**Purpose**

Stores the currency associated with `total_amount`.

**Example**

```text
currency = USD
```

Other examples:

```text
EUR
GBP
AED
```

The currency must be preserved with the historical booking because:

```text
420 USD
```

and:

```text
420 EUR
```

represent completely different monetary values.

---

### `booked_at`

**Purpose**

Represents the time at which the booking was successfully created or confirmed according to the platform's business definition.

**Example**

```text
2026-08-15T11:35:42Z
```

This is different from `created_at`.

For example:

```text
created_at
→ Pending booking record created

booked_at
→ Provider successfully confirmed booking
```

A booking may therefore have:

```text
created_at = 11:35:20
booked_at  = 11:35:42
```

---

### `created_at`

**Purpose**

Represents when the flight booking record was first created inside the platform.

The record may initially be created while the booking is:

```text
PENDING
```

or:

```text
PROCESSING
```

**Example**

```text
created_at = 2026-08-15T11:35:20Z
```

---

### `updated_at`

**Purpose**

Represents the most recent time the flight booking record was updated.

Updates may occur because of:

- Booking confirmation.
- Provider status synchronization.
- Cancellation.
- Manual review.
- Other approved booking lifecycle changes.

**Example**

```text
updated_at = 2026-08-15T11:35:42Z
```

---

## 3.4 Flight Booking Relationships

The initial relationship model is:

```text
Customer
   1
   │
   └──── *
      FlightBooking
          │
          ├──── 1 Transaction
          │
          ├──── 1 Provider
          │
          ├──── * FlightBookingPassenger
          │
          └──── * FlightSegment
```

The flight booking acts as the main persistent record of the booking lifecycle.

Passenger and itinerary details are stored separately to preserve normalization and support multiple passengers and flight segments.

---

## 3.5 Flight Booking Snapshot

The database must preserve the information accepted at booking time.

The system must not depend on the original cached offer after confirmation.

The booking snapshot is distributed across the related booking tables.

### `flight_bookings`

Stores booking-level data such as:

```text
total_amount
currency
trip_type
provider references
booking status
booking timestamps
```

### `flight_booking_passengers`

Will store the passenger information used for this specific booking.

Examples include:

```text
given_name
family_name
date_of_birth
passenger_type
travel_document information when required
```

A passenger snapshot remains unchanged even if the customer's reusable traveler profile is modified later.

### `flight_segments`

Will store the booked itinerary.

Examples include:

```text
origin
destination
departure_at
arrival_at
flight_number
marketing_carrier
operating_carrier
cabin_class
```

---

## 3.6 No Initial Denormalization

The initial database design intentionally avoids duplicating flight segment information in `flight_bookings`.

For example, the following fields are not initially stored in the main booking table:

```text
departure_at
arrival_at
origin
destination
```

They remain inside `flight_segments`.

This keeps the persistence model normalized and avoids multiple representations of the same business data.

Controlled denormalization may be introduced later if production measurements demonstrate a clear read-performance requirement.

---

## 3.7 Provider Independence

The database must remain independent from Duffel's internal naming conventions.

For example:

```text
Duffel                  Platform

offer.id            →   provider_offer_reference
order.id            →   provider_booking_reference
booking_reference   →   provider_confirmation_reference
```

This creates an anti-corruption boundary between the platform domain and the provider API.

If Duffel is replaced or another flight provider is introduced, the core flight booking schema should remain stable.

---

## 3.8 Architectural Decisions

For flight booking persistence, the platform currently adopts the following decisions:

- Flight bookings use their own table.
- Flight offers are temporary and are not persisted as permanent entities.
- The confirmed booking stores a historical snapshot of accepted business information.
- Flight itinerary details are stored in separate flight segment records.
- Passenger details are stored as booking-specific snapshots.
- One flight booking currently has one transaction.
- Provider-specific identifiers are stored using generic provider reference fields.
- The initial model remains normalized.
- Denormalization will only be introduced when justified by measured performance requirements.

---

# 4. Hotel Booking Persistence Design

## 4.1 Overview

`hotel_bookings` represents the platform's permanent record of a hotel reservation.

It does not represent:

- A hotel search result.
- A room rate.
- A temporary offer.
- A prebook session.

LiteAPI follows a booking flow similar to:

```text
Search Rate
    ↓
Prebook
    ↓
Book
```

The prebook step validates availability and final pricing before the booking is confirmed. The final booking operation returns the provider booking identifier and confirmation information. citeturn420676search6turn420676search0

Temporary values such as the `prebookId` belong primarily to the booking orchestration workflow and do not need to become permanent hotel booking attributes unless they are retained for operational tracing or reconciliation.

The hotel booking is associated with:

- A customer.
- A hotel provider.
- One transaction.
- One or more guests.
- One or more booked rooms.

The initial database model remains normalized.

---

## 4.2 `hotel_bookings`

### Proposed Fields

```text
hotel_bookings

id
customer_id
transaction_id

provider_id
provider_offer_reference
provider_booking_reference
provider_confirmation_reference

status

hotel_reference
check_in_date
check_out_date

total_amount
currency

booked_at
created_at
updated_at
```

---

# 4.3 Field Definitions

## `id`

**Purpose**

Represents the unique internal identifier of the hotel booking inside our platform.

This ID belongs entirely to our system and does not depend on LiteAPI or any future hotel provider.

**Example**

```text
hb_01J8XB21T65...
```

or:

```text
384921
```

depending on the platform's identifier strategy.

Conceptually:

```text
Platform Hotel Booking ID
≠
Provider Booking ID
```

The distinction is important because external providers may change while our internal booking identity must remain stable.

---

## `customer_id`

**Purpose**

References the customer account that created and owns the booking.

The customer does not necessarily need to be one of the hotel guests.

For example:

```text
Customer: Hassan

Hotel Booking:
├── Guest: Ahmed
└── Guest: Sara
```

**Example**

```text
customer_id = 1842
```

Relationship:

```text
Customer
   1
   │
   └──── *
      HotelBooking
```

---

## `transaction_id`

**Purpose**

References the transaction associated with this hotel booking.

Based on the current business decision:

```text
One Hotel Booking
       ↓
One Transaction
```

The transaction reference is stored inside the booking table using the reverse foreign key approach.

**Example**

```text
transaction_id = 74102
```

Recommended relationship:

```text
hotel_bookings.transaction_id
    ↓
UNIQUE FK
    ↓
transactions.id
```

The `UNIQUE` constraint ensures that the same transaction cannot be associated with multiple hotel bookings.

The final transaction structure will be defined in the Transaction Persistence section.

---

## `provider_id`

**Purpose**

References the external provider through which the hotel booking was created.

Currently:

```text
LiteAPI
```

is the hotel provider.

However, the schema must remain provider-independent.

**Example**

```text
provider_id = 2
```

Where:

```text
providers

1 → Duffel
2 → LiteAPI
```

We intentionally avoid fields such as:

```text
liteapi_id
liteapi_booking_id
```

because these names would tightly couple the core database schema to one provider.

---

## `provider_offer_reference`

**Purpose**

Stores the external reference of the rate or offer selected during the hotel search workflow.

LiteAPI search APIs return bookable hotel rates/offers, which are then validated through the prebook flow. citeturn420676search8turn420676search12

The full offer itself remains temporary.

The provider offer reference is preserved mainly for:

- Debugging.
- Booking traceability.
- Customer support.
- Reconciliation.
- Provider issue investigation.

**Example**

```text
provider_offer_reference = rate_8f32...
```

The exact format depends on the provider.

The platform keeps the generic field name regardless of LiteAPI's internal terminology.

---

## `provider_booking_reference`

**Purpose**

Stores the primary booking identifier assigned by the external hotel provider after successful booking confirmation.

LiteAPI returns a Booking ID after the booking is completed. citeturn420676search0turn420676search7

**Example**

```text
provider_booking_reference = bk_123456789
```

Conceptually:

```text
provider_booking_reference
→ LiteAPI Booking ID
```

This reference is important for later operations such as:

- Retrieve booking details.
- Booking reconciliation.
- Cancellation.
- Support investigation.

LiteAPI exposes booking retrieval using the booking ID. citeturn420676search7

---

## `provider_confirmation_reference`

**Purpose**

Stores the reservation confirmation code returned by the hotel provider or underlying hotel supplier.

This value is different from the provider's internal booking ID.

Conceptually:

```text
provider_booking_reference
→ Provider system identifier

provider_confirmation_reference
→ Customer-facing / hotel confirmation code
```

**Example**

```text
provider_confirmation_reference = HTL8QX52
```

LiteAPI's booking flow returns both a booking identifier and confirmation information after successful booking. citeturn420676search0

Keeping both references makes support and reconciliation easier.

---

## `status`

**Purpose**

Represents the current lifecycle state of the hotel booking inside our platform.

Possible platform values may include:

```text
PENDING
PROCESSING
CONFIRMED
FAILED
CANCELLED
COMPLETED
MANUAL_REVIEW
```

**Example**

```text
status = CONFIRMED
```

Provider-specific statuses should not be copied directly into the core domain.

Instead:

```text
LiteAPI Status
      ↓
Provider Adapter
      ↓
Platform Booking Status
```

This keeps our booking lifecycle independent from provider-specific terminology.

LiteAPI also exposes booking retrieval with current booking status, making later synchronization possible when needed. citeturn420676search7

---

## `hotel_reference`

**Purpose**

Stores the identifier of the booked hotel/property as known by the provider or normalized hotel catalog.

This allows the booking to retain a stable reference to the property that was actually booked.

**Example**

```text
hotel_reference = htl_982731
```

This field does not replace the booking snapshot.

Hotel names, addresses, room information, and rate details accepted by the customer should still be preserved where required.

The identifier is mainly useful for:

- Provider reconciliation.
- Linking to hotel metadata.
- Support operations.
- Reporting.

---

## `check_in_date`

**Purpose**

Stores the confirmed check-in date of the hotel reservation.

Unlike flight segment timestamps, check-in and check-out dates belong directly to the hotel booking as a whole.

Therefore, keeping them inside `hotel_bookings` is not considered unnecessary denormalization.

**Example**

```text
check_in_date = 2026-09-10
```

The value represents a calendar date rather than an arbitrary timestamp.

---

## `check_out_date`

**Purpose**

Stores the confirmed check-out date.

**Example**

```text
check_out_date = 2026-09-14
```

Business invariant:

```text
check_out_date > check_in_date
```

These dates form part of the historical booking snapshot and should not change merely because later provider search data changes.

---

## `total_amount`

**Purpose**

Stores the final amount accepted by the customer for the confirmed hotel reservation.

This value must represent the confirmed booking amount, not an older search result price.

The prebook step exists specifically to validate availability and final pricing before confirmation. citeturn420676search6

**Example**

```text
total_amount = 685.00
```

The physical schema should later use an appropriate precise decimal representation.

---

## `currency`

**Purpose**

Stores the currency associated with `total_amount`.

**Example**

```text
currency = USD
```

The currency is part of the booking snapshot because:

```text
685 USD
```

and:

```text
685 EUR
```

represent different commercial agreements.

---

## `booked_at`

**Purpose**

Represents the time at which the hotel reservation was successfully confirmed.

This is different from when the local booking record was first created.

Example lifecycle:

```text
11:20:01
Create local PENDING booking

11:20:03
Prebook completed

11:20:08
Provider confirms booking

11:20:08
booked_at
```

**Example**

```text
booked_at = 2026-08-15T20:20:08Z
```

---

## `created_at`

**Purpose**

Represents when the hotel booking record was initially created in our system.

The record may initially exist with:

```text
PENDING
```

or:

```text
PROCESSING
```

before final provider confirmation.

**Example**

```text
created_at = 2026-08-15T20:20:01Z
```

---

## `updated_at`

**Purpose**

Represents the latest update to the hotel booking record.

Possible changes include:

- Provider confirmation.
- Status synchronization.
- Cancellation.
- Manual review.
- Administrative correction.

**Example**

```text
updated_at = 2026-08-15T20:20:08Z
```

---

# 4.4 Hotel Booking Relationships

The initial relationship model is:

```text
Customer
   1
   │
   └──── *
      HotelBooking
          │
          ├──── 1 Transaction
          │
          ├──── 1 Provider
          │
          ├──── * HotelBookingGuest
          │
          └──── * HotelBookingRoom
```

This keeps hotel-specific data separate from flight-specific persistence.

---

# 4.5 Hotel Guest Snapshot

A hotel booking should not depend only on the customer's reusable traveler profile.

Instead, guest information used during the booking should be stored as a booking-specific snapshot.

A future table may look conceptually like:

```text
hotel_booking_guests

id
hotel_booking_id
traveler_profile_id (optional)

first_name
last_name
email
guest_type

is_primary_guest
```

LiteAPI booking confirmation requires guest information such as first name, last name, and email. citeturn420676search0

The snapshot remains stable even if the customer later modifies a saved traveler profile.

Example:

```text
Traveler Profile
Ahmed Mohammad
        ↓
Booking Created
        ↓
hotel_booking_guests
Ahmed Mohammad
        ↓
Traveler Profile later changed
Ahmed M. Mohammad
        ↓
Historical booking remains unchanged
Ahmed Mohammad
```

---

# 4.6 Hotel Room Snapshot

Room and rate information accepted during booking should also be preserved.

A future table may conceptually contain:

```text
hotel_booking_rooms

id
hotel_booking_id

room_reference
room_name

rate_plan_reference
rate_name

meal_plan
occupancy

cancellation_policy
room_amount
currency
```

This is necessary because the live provider rate may later:

- Disappear.
- Change price.
- Change cancellation policy.
- Change availability.

The confirmed booking must preserve what the customer actually purchased.

LiteAPI's search and rate APIs return detailed room/rate information before the prebook and booking steps. citeturn420676search15turn420676search12

---

# 4.7 Prebook Data

LiteAPI uses `prebookId` to represent the validated checkout/prebooking session between rate selection and final booking. citeturn420676search6turn420676search13

The initial design does **not** treat the prebook session as a permanent business entity.

Conceptually:

```text
Search Rate
    ↓
Offer stored temporarily
    ↓
Prebook
    ↓
prebookId
    ↓
Temporary orchestration state
    ↓
Book
    ↓
Permanent HotelBooking
```

The `prebookId` may be stored temporarily in:

```text
Redis
```

or another short-lived orchestration store.

It may also be included in logs or operational metadata when useful for reconciliation.

However:

```text
prebookId
```

is not considered the permanent identity of the hotel reservation.

After successful booking, the important provider identifiers are:

```text
provider_booking_reference
provider_confirmation_reference
```

---

# 4.8 Hotel Booking Snapshot

The permanent snapshot is distributed across the hotel booking aggregate.

### `hotel_bookings`

Stores booking-level information:

```text
provider references
hotel reference
status
check-in
check-out
total amount
currency
booking timestamps
```

### `hotel_booking_guests`

Stores booking-specific guest information.

### `hotel_booking_rooms`

Stores:

```text
room
rate plan
occupancy
pricing
meal plan
cancellation terms
```

The booking must remain understandable even after the temporary search cache and prebook session have expired.

---

# 4.9 Provider Independence

The persistence model must not expose LiteAPI-specific naming inside the core hotel domain.

Conceptual mapping:

```text
LiteAPI                       Platform

rate / offer reference    →   provider_offer_reference
bookingId                 →   provider_booking_reference
confirmationCode          →   provider_confirmation_reference
prebookId                 →   temporary orchestration state
```

This creates a clear boundary between:

```text
Hotel Booking Domain
```

and:

```text
LiteAPI Integration
```

If a second hotel provider is introduced later, the main schema should remain stable.

---

# 4.10 Architectural Decisions

For hotel booking persistence, the platform currently adopts the following decisions:

- Hotel bookings use a table independent from flight bookings.
- Hotel search rates and offers are temporary and are not stored as permanent entities.
- LiteAPI prebook sessions are treated as temporary orchestration state.
- Confirmed hotel bookings store their own historical snapshot.
- Guest data is stored as a booking-specific snapshot.
- Room and rate details are stored separately from the main hotel booking record.
- One hotel booking currently has one transaction.
- Provider identifiers use generic platform field names.
- Check-in and check-out dates belong directly to the hotel booking.
- The hotel persistence model remains independent from LiteAPI-specific schema terminology.

---

# 5. Transaction Persistence Design

## 5.1 Overview

`transactions` represents the financial transaction associated with a flight or hotel booking.

The transaction lifecycle is independent from the booking lifecycle.

A booking may be:

```text
PENDING
CONFIRMED
FAILED
CANCELLED
```

while the transaction may independently be:

```text
PENDING
AUTHORIZED
SUCCEEDED
FAILED
CANCELLED
```

This separation is important because:

```text
Payment Success
≠
Booking Confirmation
```

For example, a payment may succeed while provider booking confirmation is still pending.

The current business decision is:

```text
One Booking
    ↓
One Transaction
```

The transaction is therefore referenced by the booking using a reverse foreign key.

---

## 5.2 `transactions`

### Proposed Fields

```text
transactions

id
customer_id

provider_id
provider_transaction_reference

status
payment_method

amount
currency

authorized_at
completed_at

created_at
updated_at
```

---

# 5.3 Field Definitions

## `id`

**Purpose**

Represents the unique internal identifier of the transaction inside our platform.

This identifier is independent from the external payment provider.

**Example**

```text
txn_01J8Y5M8KQ...
```

or:

```text
74102
```

depending on the selected platform identifier strategy.

Conceptually:

```text
Platform Transaction ID
≠
Payment Provider Transaction ID
```

---

## `customer_id`

**Purpose**

References the customer responsible for the transaction.

This makes the transaction directly traceable to the customer who initiated the booking.

**Example**

```text
customer_id = 1842
```

Relationship:

```text
Customer
   1
   │
   └──── *
      Transaction
```

A customer may therefore have many transactions over time.

---

## `provider_id`

**Purpose**

Identifies the external payment provider used to process the transaction.

The field remains generic so the database does not depend on a specific payment gateway.

For example:

```text
providers

1 → Duffel
2 → LiteAPI
3 → Stripe
```

A transaction may use:

```text
provider_id = 3
```

to indicate that Stripe processed the payment.

Depending on the final provider model, travel providers and payment providers may later be separated by provider type or capability.

---

## `provider_transaction_reference`

**Purpose**

Stores the primary transaction identifier assigned by the external payment provider.

This identifier is used for:

- Reconciliation.
- Payment lookup.
- Customer support.
- Payment status synchronization.
- Failure investigation.

**Example**

```text
provider_transaction_reference = pi_3Qx8...
```

or another identifier returned by the selected payment provider.

The field is intentionally named:

```text
provider_transaction_reference
```

rather than:

```text
stripe_payment_intent_id
```

to preserve provider independence.

---

## `status`

**Purpose**

Represents the current financial state of the transaction inside the platform.

Possible values may include:

```text
PENDING
AUTHORIZED
SUCCEEDED
FAILED
CANCELLED
```

**Example**

```text
status = SUCCEEDED
```

The platform status should be normalized.

External gateway-specific states should be mapped through the payment integration layer.

Conceptually:

```text
Payment Provider Status
        ↓
Payment Adapter
        ↓
Platform Transaction Status
```

---

## `payment_method`

**Purpose**

Represents the payment method selected for the transaction.

Examples may include:

```text
CARD
WALLET
BANK_TRANSFER
PAY_AT_PROPERTY
```

**Example**

```text
payment_method = CARD
```

The exact supported methods depend on platform and provider capabilities.

This field describes the business payment method rather than gateway-specific implementation details.

---

## `amount`

**Purpose**

Stores the amount associated with the transaction.

This amount must match the amount approved by the customer.

**Example**

```text
amount = 420.50
```

The physical database schema should later use a precise decimal representation.

---

## `currency`

**Purpose**

Stores the currency associated with the transaction amount.

**Example**

```text
currency = USD
```

The combination:

```text
amount + currency
```

must always be interpreted together.

For example:

```text
420 USD
```

is not equivalent to:

```text
420 EUR
```

---

## `authorized_at`

**Purpose**

Represents the time at which the payment provider successfully authorized the transaction when authorization is part of the payment flow.

**Example**

```text
authorized_at = 2026-08-16T06:15:10Z
```

This value may be `NULL` when:

- Authorization is not used.
- Authorization failed.
- The payment provider uses immediate capture.

---

## `completed_at`

**Purpose**

Represents the time at which the transaction reached a successful final financial state.

For a standard card payment this may correspond to successful capture or payment completion.

**Example**

```text
completed_at = 2026-08-16T06:15:12Z
```

Conceptually:

```text
created_at
→ Transaction started

authorized_at
→ Funds authorized

completed_at
→ Financial operation completed
```

Not every payment method requires all three stages.

---

## `created_at`

**Purpose**

Represents when the transaction record was created in the platform.

The transaction may initially have:

```text
status = PENDING
```

**Example**

```text
created_at = 2026-08-16T06:15:08Z
```

---

## `updated_at`

**Purpose**

Represents the most recent modification of the transaction record.

Updates may occur because of:

- Authorization.
- Payment completion.
- Payment failure.
- Provider callback.
- Reconciliation.
- Manual operational correction.

**Example**

```text
updated_at = 2026-08-16T06:15:12Z
```

---

# 5.4 Booking Relationship

The transaction does not contain:

```text
flight_booking_id
hotel_booking_id
```

Instead, each booking references its transaction.

Conceptually:

```text
transactions
     ▲
     │
     │ UNIQUE FK
     │
flight_bookings.transaction_id
```

or:

```text
transactions
     ▲
     │
     │ UNIQUE FK
     │
hotel_bookings.transaction_id
```

This is the **Reverse Foreign Key** approach.

It avoids a polymorphic relationship such as:

```text
transactions

booking_type
booking_id
```

and also avoids nullable foreign keys such as:

```text
flight_booking_id
hotel_booking_id
```

inside the transaction table.

---

## Why Reverse Foreign Key?

The relationship is currently:

```text
One Booking
↔
One Transaction
```

Therefore the booking owns the reference to its transaction.

For example:

```text
flight_bookings

id = 1001
transaction_id = 74102
```

and:

```text
transactions

id = 74102
status = SUCCEEDED
amount = 420.50
currency = USD
```

The same transaction ID should not be used by another booking.

Recommended constraint:

```text
UNIQUE(transaction_id)
```

inside each booking table.

---

# 5.5 Flight Booking Example

```text
flight_bookings

id                          = 1001
customer_id                 = 1842
transaction_id              = 74102
provider_id                 = Duffel
provider_booking_reference  = ord_123...
status                      = CONFIRMED
```

Transaction:

```text
transactions

id                              = 74102
customer_id                     = 1842
provider_id                     = Payment Provider
provider_transaction_reference  = pi_123...
status                          = SUCCEEDED
payment_method                  = CARD
amount                          = 420.50
currency                        = USD
```

Conceptually:

```text
Flight Booking
       │
       │ transaction_id
       ▼
Transaction
```

---

# 5.6 Hotel Booking Example

```text
hotel_bookings

id                          = 2001
customer_id                 = 1842
transaction_id              = 75119
provider_id                 = LiteAPI
provider_booking_reference  = booking_...
status                      = CONFIRMED
```

Transaction:

```text
transactions

id                              = 75119
customer_id                     = 1842
provider_id                     = Payment Provider
provider_transaction_reference  = pi_456...
status                          = SUCCEEDED
payment_method                  = CARD
amount                          = 685.00
currency                        = USD
```

---

# 5.7 Transaction vs Booking Status

Booking and transaction statuses must remain independent.

For example:

```text
Transaction = SUCCEEDED
Booking     = PROCESSING
```

is a valid temporary state.

Another example:

```text
Transaction = FAILED
Booking     = PENDING
```

may indicate that the customer can retry payment before the booking expires.

The platform must not assume:

```text
Transaction SUCCEEDED
        =
Booking CONFIRMED
```

Provider confirmation remains part of the booking lifecycle.

---

# 5.8 Transaction Snapshot

The transaction stores the financial facts accepted during the operation.

At minimum, the snapshot includes:

```text
amount
currency
payment_method
provider_transaction_reference
transaction status
transaction timestamps
```

These values must remain traceable even if:

- Payment provider configuration changes.
- Customer payment preferences change.
- Booking details later change through an approved workflow.

---

# 5.9 No Refund Model in Initial Scope

The current platform scope does not include a dedicated refund persistence model.

Therefore the initial design does not introduce:

```text
refunds
```

or refund-specific transaction relationships.

If refund functionality becomes part of the product scope later, it should be introduced as a separate financial lifecycle rather than overloading the existing transaction status.

---

# 5.10 Provider Independence

The transaction model must remain independent from the selected payment provider.

Conceptual mapping:

```text
External Payment Provider          Platform

payment/payment-intent ID      →   provider_transaction_reference
provider payment state         →   status
provider method                →   payment_method
payment amount                 →   amount
payment currency               →   currency
```

Provider-specific fields should remain inside the payment integration layer unless there is a strong business reason to persist them.

---

# 5.11 Architectural Decisions

For transaction persistence, the platform currently adopts the following decisions:

- The entity remains named `Transaction`.
- One booking currently has exactly one transaction.
- Flight and hotel bookings reference their transaction using reverse foreign keys.
- `transaction_id` is unique within a booking relationship.
- Transaction status is independent from booking status.
- Transaction identifiers remain independent from payment-provider-specific naming.
- Amount and currency are preserved as historical financial data.
- Provider transaction references are stored for reconciliation.
- Refund persistence is outside the initial scope.
- Polymorphic booking references are avoided for financial relationships.

---

# 6. Flight Booking Passengers

## 6.1 Overview

`flight_booking_passengers` stores the passenger information associated with a confirmed flight booking.

This table represents a **booking-specific passenger snapshot** rather than the customer's reusable traveler profile.

The purpose of this table is to preserve the exact passenger information used during the booking process, even if the customer later updates their saved traveler profiles.

Each flight booking may contain one or more passengers.

Examples include:

- Single traveler
- Couple
- Family
- Group booking

---

## 6.2 Why a Separate Table?

Passenger information should not be stored directly inside `flight_bookings` because:

- A booking may contain multiple passengers.
- Each passenger has independent personal information.
- Passenger information may include travel documents.
- Passenger information should remain normalized.

More importantly, passenger information must remain historically accurate.

For example:

```text
Traveler Profile

Passport Number = A123456
```

After one year:

```text
Traveler Profile

Passport Number = B987654
```

The historical booking must still contain:

```text
Passport Number = A123456
```

Therefore, the booking stores its own immutable passenger snapshot.

---

## 6.3 Proposed Fields

```text
flight_booking_passengers

id

flight_booking_id

traveler_profile_id (optional)

first_name
last_name

date_of_birth

passenger_type

gender

nationality

document_type
document_number
document_expiry

created_at
```

---

## 6.4 Field Definitions

### `id`

**Purpose**

Unique internal identifier of the booking passenger record.

Each passenger inside a booking has its own identifier.

**Example**

```text
fbp_01J9....
```

---

### `flight_booking_id`

**Purpose**

References the flight booking to which the passenger belongs.

Relationship:

```text
FlightBooking
    1
    │
    └──── *
         FlightBookingPassenger
```

A booking may therefore contain multiple passenger records.

---

### `traveler_profile_id`

**Purpose**

Optionally references the reusable traveler profile used when creating the booking.

This field exists only for convenience.

The booking must never depend on the traveler profile remaining unchanged.

If the traveler profile is deleted or updated, the booking snapshot remains valid.

Example:

```text
traveler_profile_id = 52
```

This field may be NULL.

---

### `first_name`

**Purpose**

Stores the passenger's given name exactly as submitted during booking.

Example:

```text
Ahmed
```

---

### `last_name`

**Purpose**

Stores the passenger's family name.

Example:

```text
Mohammad
```

---

### `date_of_birth`

**Purpose**

Stores the passenger's birth date.

Many airlines require this information during booking.

Example:

```text
1998-05-14
```

---

### `passenger_type`

**Purpose**

Represents the passenger category.

Possible values:

```text
ADULT
CHILD
INFANT
```

Example:

```text
ADULT
```

---

### `gender`

**Purpose**

Stores the passenger gender only when required by the airline or provider.

Example:

```text
MALE
FEMALE
```

---

### `nationality`

**Purpose**

Stores the passenger nationality when required for ticket issuance or travel regulations.

Example:

```text
PS
```

---

### `document_type`

**Purpose**

Represents the travel document type used during booking.

Examples:

```text
PASSPORT

NATIONAL_ID
```

---

### `document_number`

**Purpose**

Stores the document number used during booking.

Example:

```text
P12345678
```

---

### `document_expiry`

**Purpose**

Stores the expiration date of the travel document.

Example:

```text
2032-08-01
```

---

### `created_at`

**Purpose**

Represents when the passenger snapshot was created.

Since the snapshot never changes, an `updated_at` field is not initially required.

---

## 6.5 Snapshot Principle

Passenger information stored in this table represents the booking at confirmation time.

Future changes to:

- Traveler profile
- Passport
- Nationality

must not modify historical booking records.

Conceptually:

```text
Traveler Profile
        │
        ▼
Booking Created
        │
        ▼
Passenger Snapshot
        │
        ▼
Traveler Updated
        │
        ▼
Snapshot remains unchanged
```

---

## 6.6 Architectural Decisions

The platform currently adopts the following decisions:

- Passenger information is stored separately from the booking.
- Passenger information is stored as an immutable snapshot.
- Traveler profiles are optional references.
- Historical booking records never depend on reusable traveler profiles.
- A flight booking may contain multiple passengers.

---

# 7. Flight Segments

## 7.1 Overview

`flight_segments` stores the itinerary associated with a confirmed flight booking.

A flight itinerary may contain one or more flight segments.

Examples:

- Non-stop flight
- One connection
- Multiple connections

Each segment represents one physical flight operated by an airline.

---

## 7.2 Why a Separate Table?

Flight information should not be duplicated inside `flight_bookings`.

A booking may contain:

- One segment
- Two segments
- Four or more segments

Separating segments keeps the persistence model normalized and allows unlimited itinerary complexity.

---

## 7.3 Proposed Fields

```text
flight_segments

id

flight_booking_id

segment_order

origin_iata_code
destination_iata_code

departure_at
arrival_at

marketing_carrier
operating_carrier

flight_number

cabin_class

booking_class

created_at
```

---

## 7.4 Field Definitions

### `id`

**Purpose**

Unique internal identifier of the flight segment.

---

### `flight_booking_id`

**Purpose**

References the booking that owns this segment.

Relationship:

```text
FlightBooking
    1
    │
    └──── *
        FlightSegment
```

---

### `segment_order`

**Purpose**

Represents the order of the segment within the itinerary.

Example:

```text
1
```

First flight.

```text
2
```

Second connection.

This guarantees the itinerary can always be reconstructed.

---

### `origin_iata_code`

**Purpose**

Stores the departure airport.

Example:

```text
AMM
```

---

### `destination_iata_code`

**Purpose**

Stores the arrival airport.

Example:

```text
DXB
```

---

### `departure_at`

**Purpose**

Stores the scheduled departure timestamp.

Example:

```text
2026-08-20T10:30:00Z
```

---

### `arrival_at`

**Purpose**

Stores the scheduled arrival timestamp.

Example:

```text
2026-08-20T14:15:00Z
```

---

### `marketing_carrier`

**Purpose**

Represents the airline that marketed the flight.

Example:

```text
EK
```

---

### `operating_carrier`

**Purpose**

Represents the airline that actually operates the flight.

These values may differ during codeshare flights.

Example:

```text
Marketing Carrier = EK

Operating Carrier = QF
```

---

### `flight_number`

**Purpose**

Stores the airline flight number.

Example:

```text
EK903
```

---

### `cabin_class`

**Purpose**

Represents the cabin booked by the passenger.

Examples:

```text
ECONOMY
PREMIUM_ECONOMY
BUSINESS
FIRST
```

---

### `booking_class`

**Purpose**

Represents the airline fare booking class.

Example:

```text
Y

J

C
```

This value is different from Cabin Class.

For example:

```text
Cabin = ECONOMY

Booking Class = Y
```

---

### `created_at`

Represents when the segment snapshot was created.

---

## 7.5 Segment Snapshot

Each segment represents the itinerary confirmed during booking.

Future schedule changes from the provider must not overwrite the historical booking itinerary.

The booking always preserves what the customer originally accepted.

---

## 7.6 Architectural Decisions

The platform currently adopts the following decisions:

- Flight segments are stored separately from the booking.
- A booking may contain any number of flight segments.
- The itinerary remains normalized.
- Segment ordering is explicitly stored.
- Codeshare flights are supported.
- Flight segment data represents a historical booking snapshot.

---

# 8. Hotel Booking Guests

## 8.1 Overview

`hotel_booking_guests` stores the guest information associated with a confirmed hotel booking.

This table represents a **booking-specific guest snapshot**.

It is separate from reusable traveler profiles because the historical booking must preserve the exact guest information used at booking time.

A hotel booking may contain:

- One guest.
- Multiple guests.
- Multiple rooms with different guest allocations.

---

## 8.2 Why a Separate Table?

Guest information should not be stored directly inside `hotel_bookings` because:

- A booking may contain multiple guests.
- Guests may be assigned to different rooms.
- Guest information must remain historically accurate.
- A guest may not have a reusable traveler profile.
- A customer may create a hotel booking for someone else.

For example:

```text
Customer
Hassan
   │
   └── Hotel Booking
         ├── Ahmed
         └── Sara
```

The customer who owns the booking does not have to be one of the guests.

---

## 8.3 Proposed Fields

```text
hotel_booking_guests

id

hotel_booking_id
traveler_profile_id (optional)

first_name
last_name

email
phone_number

date_of_birth (optional)

guest_type

is_primary_guest

created_at
```

---

## 8.4 Field Definitions

### `id`

**Purpose**

Represents the unique internal identifier of the guest snapshot.

**Example**

```text
hbg_01J9...
```

---

### `hotel_booking_id`

**Purpose**

References the hotel booking that owns this guest.

Relationship:

```text
HotelBooking
    1
    │
    └──── *
       HotelBookingGuest
```

A hotel booking may therefore contain multiple guest records.

---

### `traveler_profile_id`

**Purpose**

Optionally references the reusable traveler profile from which the guest information was copied.

This relationship is only a convenience reference.

The historical booking must never depend on future changes to the traveler profile.

**Example**

```text
traveler_profile_id = 72
```

This field may be `NULL`.

---

### `first_name`

**Purpose**

Stores the guest's first name exactly as submitted during booking.

**Example**

```text
Ahmed
```

---

### `last_name`

**Purpose**

Stores the guest's family name.

**Example**

```text
Mohammad
```

---

### `email`

**Purpose**

Stores the guest email address when required by the hotel provider.

A provider may require contact information for the primary guest.

**Example**

```text
ahmed@example.com
```

This field may be optional depending on provider requirements.

---

### `phone_number`

**Purpose**

Stores the guest contact phone number when required.

**Example**

```text
+970599123456
```

This field may be optional.

---

### `date_of_birth`

**Purpose**

Stores the guest date of birth when required for age validation or hotel policy.

**Example**

```text
1995-04-12
```

This field may be `NULL` when the provider does not require it.

---

### `guest_type`

**Purpose**

Represents the guest category.

Possible values may include:

```text
ADULT
CHILD
```

**Example**

```text
guest_type = ADULT
```

The exact age boundaries may depend on the hotel provider or property rules.

---

### `is_primary_guest`

**Purpose**

Identifies the guest responsible for the reservation and check-in.

Every hotel booking should normally have one primary guest.

**Example**

```text
is_primary_guest = true
```

Business rule:

```text
One Hotel Booking
      ↓
Exactly One Primary Guest
```

unless the provider supports a different model.

---

### `created_at`

**Purpose**

Represents when the guest snapshot was created.

Because this record represents historical booking data, it should not normally be modified after confirmation.

---

## 8.5 Guest Snapshot Principle

The guest record is a historical snapshot.

For example:

```text
Traveler Profile
Ahmed Mohammad
email: old@example.com
        ↓
Booking Created
        ↓
Hotel Booking Guest Snapshot
Ahmed Mohammad
email: old@example.com
        ↓
Traveler Profile Updated
email: new@example.com
        ↓
Historical Booking Remains
email: old@example.com
```

This ensures previous reservations remain accurate.

---

## 8.6 Relationship to Rooms

Guests may later be assigned to hotel rooms.

Conceptually:

```text
HotelBooking
    │
    ├── HotelBookingRoom 1
    │      ├── Guest A
    │      └── Guest B
    │
    └── HotelBookingRoom 2
           └── Guest C
```

The exact room-to-guest relationship will be finalized together with `hotel_booking_rooms`.

---

## 8.7 Architectural Decisions

The platform currently adopts the following decisions:

- Hotel guests are stored separately from the main booking.
- Guest information is stored as a booking-specific snapshot.
- Traveler profile references are optional.
- A customer does not have to be one of the hotel guests.
- Hotel bookings support multiple guests.
- A primary guest is explicitly identified.
- Historical guest data remains unchanged after booking confirmation.

---

# 9. Hotel Booking Rooms

## 9.1 Overview

`hotel_booking_rooms` stores the room and rate information associated with a confirmed hotel booking.

A hotel booking may contain:

- One room.
- Multiple rooms.
- Different room types.
- Different occupancy configurations.

The room record preserves the exact accommodation and commercial terms accepted by the customer.

---

## 9.2 Why a Separate Table?

Room information should not be stored directly in `hotel_bookings` because one booking may contain multiple rooms.

For example:

```text
Hotel Booking

Room 1
- Deluxe King
- 2 Adults

Room 2
- Twin Room
- 1 Adult + 1 Child
```

Storing rooms separately keeps the model normalized and supports multi-room bookings.

---

## 9.3 Proposed Fields

```text
hotel_booking_rooms

id

hotel_booking_id

provider_room_reference
provider_rate_reference

room_name
room_type

rate_name
meal_plan

adult_count
child_count

room_amount
currency

cancellation_policy

created_at
```

---

## 9.4 Field Definitions

### `id`

**Purpose**

Represents the unique internal identifier of the booked room record.

---

### `hotel_booking_id`

**Purpose**

References the hotel booking that owns the room.

Relationship:

```text
HotelBooking
    1
    │
    └──── *
       HotelBookingRoom
```

---

### `provider_room_reference`

**Purpose**

Stores the provider's identifier for the selected room when available.

The platform uses a generic field name instead of a LiteAPI-specific name.

**Example**

```text
provider_room_reference = room_87321
```

This reference is useful for:

- Support.
- Reconciliation.
- Provider tracing.

---

### `provider_rate_reference`

**Purpose**

Stores the external identifier of the booked hotel rate.

A hotel room may have multiple rates with different:

- Prices.
- Meal plans.
- Cancellation policies.
- Payment conditions.

Therefore the room identity and rate identity should remain separate.

**Example**

```text
provider_rate_reference = rate_98124
```

---

### `room_name`

**Purpose**

Stores the room name displayed and accepted by the customer.

**Example**

```text
Deluxe King Room
```

This is part of the historical booking snapshot.

---

### `room_type`

**Purpose**

Represents a normalized room category when available.

Examples:

```text
STANDARD
DELUXE
SUITE
TWIN
```

This field may be provider-dependent and should only be used if a reliable normalization exists.

---

### `rate_name`

**Purpose**

Stores the name or description of the selected rate plan.

Examples:

```text
Flexible Rate
Non-Refundable Rate
Breakfast Included
```

---

### `meal_plan`

**Purpose**

Represents the meal arrangement included in the booked rate.

Possible normalized values may include:

```text
ROOM_ONLY
BREAKFAST_INCLUDED
HALF_BOARD
FULL_BOARD
ALL_INCLUSIVE
```

**Example**

```text
meal_plan = BREAKFAST_INCLUDED
```

---

### `adult_count`

**Purpose**

Stores the number of adults assigned to this room.

**Example**

```text
adult_count = 2
```

---

### `child_count`

**Purpose**

Stores the number of children assigned to this room.

**Example**

```text
child_count = 1
```

Detailed child ages may later be stored through the room-to-guest relationship if required.

---

### `room_amount`

**Purpose**

Stores the amount associated with this booked room and rate.

**Example**

```text
room_amount = 342.50
```

For a multi-room booking:

```text
Room 1 = 342.50 USD
Room 2 = 342.50 USD

Booking Total = 685.00 USD
```

The room amounts should reconcile with the booking total according to pricing rules.

---

### `currency`

**Purpose**

Stores the currency of the room amount.

**Example**

```text
currency = USD
```

---

### `cancellation_policy`

**Purpose**

Preserves the cancellation terms accepted for this specific booked rate.

The cancellation policy may later change in live provider data, but the booking must preserve the original commercial terms.

The initial logical model may store the normalized policy as:

```text
FREE_CANCELLATION_UNTIL 2026-09-08
THEN 100% PENALTY
```

The exact physical representation will be decided later.

This may eventually become:

- Structured columns.
- JSON.
- A separate cancellation-policy snapshot entity.

We should not decide that yet without a clear provider requirement.

---

### `created_at`

**Purpose**

Represents when the booked room snapshot was created.

---

## 9.5 Room-to-Guest Relationship

A multi-room hotel booking requires us to know which guests belong to which room.

We should avoid embedding guest IDs directly inside a room as an array.

The normalized relationship is conceptually:

```text
HotelBookingRoom
      *
      │
      │
      *
HotelBookingGuest
```

This is a many-to-many relationship.

A small join table may therefore be required:

```text
hotel_booking_room_guests

hotel_booking_room_id
hotel_booking_guest_id
```

Example:

```text
Room 1
├── Ahmed
└── Sara

Room 2
├── Hassan
└── Ali
```

This keeps guest snapshots reusable within the same booking while preserving explicit room allocation.

---

## 9.6 Room Snapshot Principle

The booked room must remain understandable even when the live hotel rate disappears.

Therefore the platform preserves:

```text
room name
room/rate references
rate name
meal plan
occupancy
price
currency
cancellation terms
```

The platform must not depend on fetching the original LiteAPI rate later to understand a confirmed booking.

---

## 9.7 Architectural Decisions

The platform currently adopts the following decisions:

- Hotel rooms are stored separately from the main hotel booking.
- One booking may contain multiple rooms.
- Room and rate provider references are stored separately.
- Room pricing is preserved as historical data.
- Cancellation terms are preserved as a booking snapshot.
- Occupancy is stored per room.
- Room-to-guest allocation is represented explicitly.
- The initial model remains normalized.

---

# 10. Provider Persistence Design

## 10.1 Overview

`providers` represents external systems integrated with the Travel Booking Platform.

Current examples include:

```text
Duffel
LiteAPI
```

Future examples may include additional:

- Flight providers.
- Hotel providers.
- Payment gateways.
- Notification providers.

The purpose of this table is to keep platform business data independent from specific external provider names.

---

## 10.2 Why a Provider Table?

Without a provider abstraction, booking tables may end up containing provider-specific columns such as:

```text
duffel_order_id
liteapi_booking_id
amadeus_offer_id
```

This tightly couples the core schema to integration vendors.

Instead, the platform uses:

```text
provider_id
provider_offer_reference
provider_booking_reference
```

Conceptually:

```text
FlightBooking
      │
      └── provider_id → Duffel

HotelBooking
      │
      └── provider_id → LiteAPI
```

If providers change later, the booking schema remains stable.

---

## 10.3 Proposed Fields

```text
providers

id

name
code

type

status

created_at
updated_at
```

---

## 10.4 Field Definitions

### `id`

**Purpose**

Represents the unique internal identifier of the provider.

**Example**

```text
1
```

---

### `name`

**Purpose**

Stores the human-readable provider name.

Examples:

```text
Duffel
LiteAPI
```

---

### `code`

**Purpose**

Stores a stable internal provider code used by the application.

Examples:

```text
DUFFEL
LITE_API
```

The code should normally remain stable even if the display name changes.

---

### `type`

**Purpose**

Represents the capability or category of the provider.

Possible values:

```text
FLIGHT
HOTEL
PAYMENT
NOTIFICATION
```

**Example**

```text
Duffel
type = FLIGHT

LiteAPI
type = HOTEL
```

If one provider supports multiple capabilities in the future, we may replace this single value with a provider-capability relationship.

For the current scope, one primary type is sufficient.

---

### `status`

**Purpose**

Represents whether the provider is available for new platform operations.

Possible values:

```text
ACTIVE
INACTIVE
MAINTENANCE
```

**Example**

```text
status = ACTIVE
```

Disabling a provider must not affect historical bookings that reference it.

---

### `created_at`

**Purpose**

Represents when the provider configuration record was created.

---

### `updated_at`

**Purpose**

Represents the most recent update to the provider record.

---

## 10.5 What Should Not Be Stored Here?

The `providers` table should not become a generic configuration dump.

For example, avoid placing:

```text
api_key
secret_key
client_secret
```

directly in this table.

Secrets belong in a secure secret-management mechanism.

Similarly, detailed configuration such as:

```text
timeouts
rate limits
supported markets
provider-specific feature switches
```

may later belong in configuration or dedicated provider capability structures if required.

---

## 10.6 Provider References in Bookings

Bookings reference the provider through:

```text
provider_id
```

and preserve the external provider identifiers separately.

For flights:

```text
provider_id                    = Duffel
provider_offer_reference       = off_...
provider_booking_reference     = ord_...
provider_confirmation_reference = Airline PNR
```

For hotels:

```text
provider_id                    = LiteAPI
provider_offer_reference       = rate_...
provider_booking_reference     = bookingId
provider_confirmation_reference = confirmationCode
```

This pattern creates a provider-independent booking domain.

---

## 10.7 Relationship Model

Conceptually:

```text
Provider
   1
   │
   ├──── * FlightBooking
   │
   └──── * HotelBooking
```

A provider may therefore be associated with many historical bookings.

The provider record must not be deleted when historical references exist.

Deactivation is preferred over deletion.

---

## 10.8 Provider Independence

The database schema treats external providers as integration dependencies rather than core domain owners.

Conceptually:

```text
External Provider API
        ↓
Provider Integration Layer
        ↓
Normalized Platform Model
        ↓
Database
```

Provider-specific response structures should not leak into the core persistence model unless required for historical or reconciliation purposes.

---

## 10.9 Architectural Decisions

The platform currently adopts the following decisions:

- External providers have internal platform identities.
- Flight and hotel bookings reference providers using `provider_id`.
- Provider-specific column names are avoided.
- Provider references are stored separately from provider identity.
- Historical bookings remain valid even when a provider is disabled.
- Provider secrets are not stored in the core provider table.
- Provider-specific API structures remain inside the integration layer.

---

# 11. User Persistence Design

## 11.1 Overview

`users` represents the identity used to access the Travel Booking Platform.

The User entity is responsible for authentication-related information and account state.

It does not contain booking-specific customer information such as:

- Traveler profiles.
- Addresses.
- Booking history.
- Hotel guests.
- Flight passengers.

These responsibilities belong to other business entities.

---

## 11.2 Proposed Fields

```text
users

id

email
password_hash

status

email_verified_at

created_at
updated_at
```

---

## 11.3 Field Definitions

### `id`

**Purpose**

Represents the unique internal identifier of the user account.

**Example**

```text
user_01J...
```

---

### `email`

**Purpose**

Represents the primary email used for authentication and account communication.

**Example**

```text
hassan@example.com
```

The email should normally be unique.

---

### `password_hash`

**Purpose**

Stores the password hash used for password-based authentication.

The platform must never store the original password.

Example conceptually:

```text
Password
    ↓
Hash
    ↓
password_hash
```

---

### `status`

**Purpose**

Represents the current account state.

Possible values may include:

```text
ACTIVE
SUSPENDED
DISABLED
PENDING_VERIFICATION
```

**Example**

```text
status = ACTIVE
```

---

### `email_verified_at`

**Purpose**

Represents when the user's email was successfully verified.

**Example**

```text
2026-08-22T12:00:00Z
```

A `NULL` value indicates that verification has not yet been completed.

---

### `created_at`

Represents when the user account was created.

---

### `updated_at`

Represents the most recent update to the user account.

---

## 11.4 Architectural Decisions

- Authentication identity is stored separately from the customer profile.
- Email is treated as a unique identity when email-based authentication is used.
- Passwords are never stored directly.
- Account status is represented directly on the user.
- Authentication provider-specific details may be added later if social login is introduced.

---

# 12. Customer Persistence Design

## 12.1 Overview

`customers` represents the business customer who interacts with the travel platform.

A customer is associated with a user account but belongs to the booking domain rather than the authentication domain.

Conceptually:

```text
User
  │
  └── Customer
```

The User answers:

> Who can access the system?

The Customer answers:

> Who owns searches, bookings, traveler profiles, and customer data?

---

## 12.2 Proposed Fields

```text
customers

id
user_id

first_name
last_name

phone_number

created_at
updated_at
```

---

## 12.3 Field Definitions

### `id`

Unique internal customer identifier.

---

### `user_id`

**Purpose**

References the user account associated with the customer.

Relationship:

```text
User
  1
  │
  0..1
Customer
```

Recommended constraint:

```text
UNIQUE(user_id)
```

A user account should not create multiple customer records.

---

### `first_name`

Stores the customer's first name.

**Example**

```text
Hassan
```

---

### `last_name`

Stores the customer's family name.

**Example**

```text
AlMosadder
```

---

### `phone_number`

Stores the customer's primary contact phone number.

**Example**

```text
+970599123456
```

---

### `created_at`

Represents when the customer profile was created.

---

### `updated_at`

Represents the latest customer profile update.

---

## 12.4 Customer Relationships

Conceptually:

```text
Customer
   │
   ├── * Addresses
   ├── * Traveler Profiles
   ├── * Flight Bookings
   ├── * Hotel Bookings
   └── * Transactions
```

The customer owns the booking relationship but does not necessarily represent the actual passenger or hotel guest.

---

## 12.5 Architectural Decisions

- User identity and customer business profile remain separate.
- One user currently maps to at most one customer.
- Customer data does not replace traveler or guest snapshots.
- Booking history is linked through booking entities rather than duplicated inside the customer table.

---

# 13. Address Persistence Design

## 13.1 Overview

`addresses` stores reusable addresses associated with a customer.

Addresses may later be used for:

- Billing.
- Customer profile.
- Payment information.
- Provider-required contact information.

A customer may have multiple addresses.

---

## 13.2 Proposed Fields

```text
addresses

id
customer_id

line1
line2

city
state
postal_code
country_code

type
is_default

created_at
updated_at
```

---

## 13.3 Field Definitions

### `id`

Unique internal address identifier.

---

### `customer_id`

References the customer who owns the address.

Relationship:

```text
Customer
   1
   │
   └──── *
       Address
```

---

### `line1`

Stores the primary address line.

**Example**

```text
15 King Street
```

---

### `line2`

Stores optional additional address information.

**Example**

```text
Apartment 4B
```

This field may be `NULL`.

---

### `city`

Stores the city.

**Example**

```text
Amman
```

---

### `state`

Stores the state, province, or region when applicable.

**Example**

```text
Amman Governorate
```

This field may be optional depending on country.

---

### `postal_code`

Stores the postal or ZIP code when applicable.

---

### `country_code`

Stores the normalized country code.

Recommended example:

```text
JO
PS
AE
US
```

Using country codes is preferable to storing inconsistent free-text country names.

---

### `type`

Represents the address purpose.

Possible values:

```text
HOME
BILLING
OTHER
```

This field may initially remain optional if the platform does not yet need address classification.

---

### `is_default`

Indicates whether this is the customer's preferred/default address.

**Example**

```text
is_default = true
```

---

### `created_at`

Represents when the address was created.

---

### `updated_at`

Represents the latest address update.

---

## 13.4 Architectural Decisions

- Addresses belong to customers rather than authentication users.
- One customer may have multiple addresses.
- Country values should use normalized codes.
- Address functionality remains reusable and independent from historical booking snapshots.
- Booking-specific address snapshots may be introduced later only if business requirements require preserving the exact billing/contact address used at booking time.

---

# 14. Traveler Profile Persistence Design

## 14.1 Overview

`traveler_profiles` stores reusable traveler information owned by a customer.

Its purpose is to avoid repeatedly entering the same traveler information during future bookings.

Examples:

```text
Customer Hassan
   │
   ├── Hassan
   ├── Ahmed
   ├── Sara
   └── Ali
```

A traveler profile is **not** the historical passenger or guest record.

During booking, the relevant information is copied into:

```text
flight_booking_passengers
```

or:

```text
hotel_booking_guests
```

as an immutable snapshot.

---

## 14.2 Proposed Fields

```text
traveler_profiles

id
customer_id

first_name
last_name

date_of_birth
gender
nationality

document_type
document_number
document_expiry

created_at
updated_at
```

---

## 14.3 Field Definitions

### `id`

Unique internal identifier of the traveler profile.

---

### `customer_id`

References the customer who owns the reusable traveler profile.

Relationship:

```text
Customer
   1
   │
   └──── *
     TravelerProfile
```

---

### `first_name`

Stores the traveler's first name.

---

### `last_name`

Stores the traveler's family name.

---

### `date_of_birth`

Stores the traveler's birth date.

This value is important for determining:

```text
ADULT
CHILD
INFANT
```

according to provider rules.

---

### `gender`

Stores gender when required for provider or travel-document purposes.

This field may be optional.

---

### `nationality`

Stores the traveler's nationality using a normalized country code where possible.

**Example**

```text
PS
```

---

### `document_type`

Represents the saved travel document type.

Examples:

```text
PASSPORT
NATIONAL_ID
```

---

### `document_number`

Stores the reusable travel-document number.

This information is sensitive and must be protected appropriately.

---

### `document_expiry`

Stores the document expiration date.

---

### `created_at`

Represents when the traveler profile was created.

---

### `updated_at`

Represents the latest update to the reusable traveler information.

---

## 14.4 Traveler Profile vs Booking Snapshot

This distinction is critical.

```text
Traveler Profile
        ↓
Reusable Data
        ↓
Booking Created
        ↓
Passenger / Guest Snapshot
```

Example:

```text
traveler_profiles

Passport = P111111
```

Booking created:

```text
flight_booking_passengers

Passport = P111111
```

Later the customer updates:

```text
traveler_profiles

Passport = P999999
```

The historical booking remains:

```text
flight_booking_passengers

Passport = P111111
```

---

## 14.5 Architectural Decisions

- Traveler profiles are reusable customer-owned data.
- Traveler profiles are not historical booking records.
- Flight passengers and hotel guests store independent snapshots.
- A customer may maintain multiple traveler profiles.
- Sensitive travel-document information must be protected.
- Updating a traveler profile never updates historical bookings.

---

# 15. Roles and User Roles Persistence Design

## 15.1 Overview

The platform uses role-based access control to assign permissions to users.

The initial design uses:

```text
roles
user_roles
```

instead of introducing both:

```text
user_type
```

and:

```text
role
```

without a clear business distinction.

This avoids unnecessary duplication.

---

## 15.2 `roles`

### Proposed Fields

```text
roles

id
code
name

created_at
```

---

### `id`

Unique internal identifier of the role.

---

### `code`

Stores a stable application role code.

Examples:

```text
CUSTOMER
ADMIN
SUPPORT
FINANCE
```

The code is intended for internal permission checks.

---

### `name`

Stores a human-readable role name.

Examples:

```text
Customer
Administrator
Support Agent
Finance Officer
```

---

### `created_at`

Represents when the role was created.

---

## 15.3 `user_roles`

`user_roles` represents the many-to-many relationship between users and roles.

### Proposed Fields

```text
user_roles

user_id
role_id

created_at
```

Relationship:

```text
User
  *
  │
  *
Role
```

A user may therefore hold more than one role.

Example:

```text
User #15
├── SUPPORT
└── ADMIN
```

---

## 15.4 Constraints

The pair:

```text
user_id + role_id
```

should be unique.

This prevents duplicate role assignments.

Conceptually:

```text
UNIQUE(user_id, role_id)
```

---

## 15.5 Why No `user_type` Table?

A separate `user_type` table is not currently required because roles already express the user's authorization context.

For example:

```text
CUSTOMER
ADMIN
SUPPORT
```

can already be represented using `roles`.

Introducing both:

```text
USER_TYPE = ADMIN
ROLE = ADMIN
```

would duplicate the same business meaning.

A separate user type should only be introduced later if the domain discovers a genuinely different concept that cannot be represented using roles.

---

## 15.6 Permissions

The initial design does not require a dedicated `permissions` table.

For the current scope:

```text
roles
+
application authorization rules
```

may be sufficient.

If access control becomes more complex later, the model can evolve into:

```text
roles
permissions
role_permissions
user_roles
```

without changing the existing user/customer model.

---

## 15.7 Architectural Decisions

- The platform uses RBAC for authorization.
- `roles` and `user_roles` are sufficient for the initial model.
- `user_type` is not introduced without a distinct business meaning.
- One user may hold multiple roles.
- Role assignment is independent from customer profile creation.
- Fine-grained permissions may be introduced later if required.

---

# 16. Audit Log Persistence Design

## 16.1 Overview

`audit_logs` records important actions and changes performed within the platform.

Its purpose is to answer questions such as:

- Who performed an action?
- What entity was affected?
- What changed?
- When did the change happen?
- Why was the action performed?

Audit logs are especially important for sensitive operations involving:

- Bookings.
- Transactions.
- Customer accounts.
- Administrative actions.
- Provider configuration.

---

## 16.2 Proposed Fields

```text
audit_logs

id

actor_user_id

entity_type
entity_id

action

old_values
new_values

metadata

created_at
```

---

## 16.3 Field Definitions

### `id`

**Purpose**

Represents the unique internal identifier of the audit record.

---

### `actor_user_id`

**Purpose**

References the user who performed the action when the action was initiated by a platform user.

**Example**

```text
actor_user_id = 42
```

The field may be `NULL` for system-generated actions.

Example:

```text
Provider reconciliation worker
→ changes booking state
→ actor_user_id = NULL
```

---

### `entity_type`

**Purpose**

Identifies the type of entity affected by the action.

Examples:

```text
FLIGHT_BOOKING
HOTEL_BOOKING
TRANSACTION
CUSTOMER
PROVIDER
```

---

### `entity_id`

**Purpose**

Stores the identifier of the affected entity.

**Example**

```text
entity_type = FLIGHT_BOOKING
entity_id   = 1001
```

Together:

```text
FLIGHT_BOOKING #1001
```

This is intentionally a flexible audit reference rather than a database foreign key.

Audit logs should remain available even if the underlying entity lifecycle changes.

---

### `action`

**Purpose**

Describes the business or administrative action that occurred.

Examples:

```text
BOOKING_CONFIRMED
BOOKING_CANCELLED
TRANSACTION_STATUS_CHANGED
CUSTOMER_SUSPENDED
PROVIDER_DISABLED
```

---

### `old_values`

**Purpose**

Stores the relevant state before the change when useful.

Example:

```json
{
  "status": "PROCESSING"
}
```

---

### `new_values`

**Purpose**

Stores the relevant state after the change.

Example:

```json
{
  "status": "CONFIRMED"
}
```

---

### `metadata`

**Purpose**

Stores additional operational context that may help with investigation.

Examples may include:

```text
reason
correlation_id
provider_reference
admin_comment
source
```

This field should not become a replacement for core relational data.

---

### `created_at`

**Purpose**

Represents when the audited action occurred.

---

## 16.4 Example

```text
Actor
Admin #42

Action
BOOKING_CANCELLED

Entity
HOTEL_BOOKING #2001

Old State
CONFIRMED

New State
CANCELLED

Reason
Customer support request
```

---

## 16.5 Architectural Decisions

- Audit logs are append-only operational records.
- Audit logs do not control the current business state.
- Entity references are intentionally flexible.
- System-generated events may not have an actor user.
- Sensitive administrative and financial actions should be auditable.
- Audit metadata must not contain unnecessary secrets or sensitive payment credentials.

---

# 17. Booking Status History Persistence Design

## 17.1 Overview

The current booking status is stored directly inside:

```text
flight_bookings.status
hotel_bookings.status
```

However, the current status alone does not explain how the booking reached that state.

The platform therefore preserves important status transitions.

Because flight and hotel bookings are independent business entities, the initial design uses separate history tables:

```text
flight_booking_status_history
hotel_booking_status_history
```

This preserves strong foreign-key relationships and avoids polymorphic booking references.

---

# 17.2 Flight Booking Status History

## Proposed Fields

```text
flight_booking_status_history

id
flight_booking_id

previous_status
new_status

reason
source

changed_at
```

---

### `flight_booking_id`

References the flight booking whose status changed.

Relationship:

```text
FlightBooking
      1
      │
      └──── *
          StatusHistory
```

---

### `previous_status`

Stores the booking state before the transition.

Example:

```text
PROCESSING
```

For the initial status, this value may be `NULL`.

---

### `new_status`

Stores the state after the transition.

Example:

```text
CONFIRMED
```

---

### `reason`

Stores an optional explanation for the transition.

Examples:

```text
Provider confirmed order
Payment failed
Booking expired
Manual support action
```

---

### `source`

Identifies what initiated the status change.

Possible values:

```text
SYSTEM
CUSTOMER
ADMIN
PROVIDER
WORKER
```

---

### `changed_at`

Represents when the status transition occurred.

---

## 17.3 Hotel Booking Status History

The hotel history table follows the same principle:

```text
hotel_booking_status_history

id
hotel_booking_id

previous_status
new_status

reason
source

changed_at
```

It references `hotel_bookings` directly.

---

## 17.4 Example Lifecycle

```text
PENDING
   ↓
PROCESSING
   ↓
CONFIRMED
   ↓
COMPLETED
```

A failed flow may look like:

```text
PENDING
   ↓
PROCESSING
   ↓
FAILED
```

The main booking table stores only:

```text
status = CONFIRMED
```

while the history table preserves the full path.

---

## 17.5 Why Not a Single Polymorphic History Table?

An alternative would be:

```text
booking_status_history

booking_type
booking_id
```

This is flexible, but the database cannot enforce a real foreign key from `booking_id` to two different booking tables.

Because booking status is business-critical, the initial design prefers:

```text
flight_booking_status_history
hotel_booking_status_history
```

This maintains strong referential integrity.

---

## 17.6 Architectural Decisions

- Current status remains on the main booking table.
- Status history is append-only.
- Flight and hotel status histories remain separate.
- Status transitions preserve reason, source, and timestamp.
- Status history does not replace audit logging.
- Database foreign keys are preferred over polymorphic references for booking integrity.

---

# 18. Notification Persistence Design

## 18.1 Overview

`notifications` stores important customer communication attempts and delivery outcomes.

Typical notifications include:

- Booking confirmation.
- Booking failure.
- Transaction result.
- Cancellation confirmation.
- Provider-originated booking changes.

The initial design keeps notification persistence intentionally simple.

---

## 18.2 Proposed Fields

```text
notifications

id
customer_id

type
channel
status

related_entity_type
related_entity_id

sent_at

created_at
updated_at
```

---

## 18.3 Field Definitions

### `id`

Unique internal notification identifier.

---

### `customer_id`

References the customer receiving the notification.

---

### `type`

Represents the business purpose of the notification.

Examples:

```text
BOOKING_CONFIRMED
BOOKING_FAILED
TRANSACTION_SUCCEEDED
TRANSACTION_FAILED
BOOKING_CANCELLED
```

---

### `channel`

Represents the delivery channel.

Examples:

```text
EMAIL
SMS
PUSH
IN_APP
```

---

### `status`

Represents the notification delivery state.

Possible values:

```text
PENDING
SENT
FAILED
```

---

### `related_entity_type`

Identifies the business entity related to the notification.

Examples:

```text
FLIGHT_BOOKING
HOTEL_BOOKING
TRANSACTION
```

---

### `related_entity_id`

Stores the related entity identifier.

Example:

```text
related_entity_type = FLIGHT_BOOKING
related_entity_id   = 1001
```

This relationship is intentionally flexible.

Unlike booking and transaction integrity relationships, notification references do not control business correctness.

---

### `sent_at`

Represents when the notification was successfully sent.

The value remains `NULL` until successful delivery.

---

### `created_at`

Represents when the notification record was created.

---

### `updated_at`

Represents the latest delivery-status update.

---

## 18.4 Why No Notification Type or Status Tables?

The initial scope does not require:

```text
notification_types
notification_statuses
sms_provider_settings
email_provider_settings
```

Simple normalized values are sufficient.

More complex notification configuration may be introduced when the notification domain grows.

---

## 18.5 Notification Failure

Notification failure must never change the underlying booking or transaction result.

For example:

```text
Booking = CONFIRMED
Notification = FAILED
```

is a valid system state.

The notification may be retried asynchronously.

---

## 18.6 Architectural Decisions

- Notifications are persisted independently from booking state.
- Notification failure never rolls back a successful booking.
- Notification types and statuses remain simple values in v1.
- Flexible entity references are acceptable for notification traceability.
- Provider-specific email or SMS configuration remains outside the core notification table.