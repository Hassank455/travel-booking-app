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