---
name: villaslot
description: >
  Manage VillaSlot through its organization-scoped API v1: villas, opening hours,
  hourly rates, availability, bookings, customers and imports. Use for VillaSlot
  booking operations and integrations, including diagnosing refused bookings.
  Does not expose Stripe charging, refunds, payment recording or SaaS administration.
license: MIT
metadata:
  author: mikecodeur
  version: '1.1.0'
---

# VillaSlot

Requires an HTTP client and a VillaSlot organization API key.

## Connect and discover

Use `https://www.villaslot.io` for production, or the explicit environment supplied by
the user (e.g. `http://localhost:3000`). Never switch between them silently.
Create a key in Account → API Keys, selecting the intended organization.
Keep its value in `VILLASLOT_API_KEY`; do not write it into scripts, reports or this skill.

```bash
export VILLASLOT_BASE_URL="https://www.villaslot.io"
curl --silent --show-error --fail-with-body \
  -H "x-api-key: $VILLASLOT_API_KEY" "$VILLASLOT_BASE_URL/api/v1/me"
```

Read `/me` before operating: confirm the organization, bearer role, timezone,
currency, `settings.configured` and plan limits. A key targets one organization;
request parameters cannot override it. Responses may contain `organizationId`,
but clients must not use that to assume they can select another tenant.

Read [the API contract](references/api-v1.md) for the exact routes, bodies, bounds,
response envelopes and errors. Read [import operations](references/imports.md)
before a batch import or historical migration.

## Booking workflow

1. Find the customer with `/customers?search=...`; reuse their UUID or create the
   record authorized by the user. Email uniqueness is scoped to the organization.
2. List villas and read the selected villa's hours and rates. Default villa list
   shows active villas only. A duration without a rate is not bookable.
3. Ask `/properties/{id}/availability?date=YYYY-MM-DD&durationMinutes=N`.
4. Create with `propertyId`, `customerId`, organization-local `date`, `startMinute`,
   `durationMinutes`, and explicit `status: HOLD | CONFIRMED`. Use the status requested
   by the user; a hold is appropriate when the booking is still provisional.
5. Save the returned UUID. Read `/bookings/{id}` to reconcile state. Confirm,
   reschedule, complete or cancel only within the user's requested scope.

Availability is a snapshot: a creation or confirmation can still lose a race.
On `409`, inspect `detail.reason` and recheck availability. Do not repeat the same
request indefinitely or silently override a conflict.

## Rules that prevent expensive mistakes

- Booking `date` is a local calendar day, but calendar-list `from`/`to` are ISO
  instants. `startMinute` is minutes since local midnight, aligned to 30 minutes.
- Supported hourly durations are **60 through 600 minutes in steps of 60**;
  only the durations priced for the selected villa can be booked.
- Money is an integer in the currency's minor unit: **IDR has no decimals**;
  1,500,000 IDR is `1500000`. EUR/USD 25.00 is `2500`.
- The booking price is frozen at creation. Omit `priceAmount` to use the rate.
  On reschedule, omit it to preserve the old price; sending it renegotiates it.
- Booking status and payment status are separate. Confirming or completing is
  not payment. Cancelling does not refund. No API route writes payment status.
- In the app, manual cash/bank payments above the current net balance are rejected.
  That app feature is **not** an API endpoint; do not invent `/payments`.
- `PUT` hours/rates replaces the complete collection. Read it first and retain
  entries that the user did not ask to remove. An empty list clears it.
- Cancellation and archiving have operational consequences; use existing explicit
  task authorization and ask only when intent or affected scope is ambiguous.
- Creating a hold is a write: it may consume quota and block availability.
  Do not use live bookings as harmless connectivity probes.
- Writes can trigger product side effects. There is no API `dryRun`,
  `suppressEmail`, force-conflict switch, booking bulk endpoint, or idempotency key.
  After an ambiguous timeout, reconcile reads before retrying a creation.
- Keep a local source-ID → returned-UUID ledger for imports. Never replay a successful
  batch merely because other rows failed.

## Capabilities deliberately not exposed

The application includes features beyond API v1. This API does not provide:
manual payments, Stripe checkout/refunds, reports/CSV, customer access-link emails,
booking history or public-link tokens, organization membership, payment-method
settings/Stripe Connect, creating overnight bookings, media reorder/default/delete,
or editing a villa's nightly rate. Photo upload appends a media item; it does not
replace the gallery. Use the app for these capabilities and report unsupported
requests honestly. Do not call internal Server Actions as a substitute public API.

## Completion report

Report the environment and organization, successful operations with returned IDs,
refusals with their reasons, and anything skipped or unsupported. Distinguish
source-code checks, mocked tests and live execution; never describe an unexecuted
production operation as verified.
