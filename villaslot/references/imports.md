# Importing customers and bookings

Use the environment and organization explicitly authorized by the user. Start with
GET /me and inspect villas, hours, rates, timezone, currency and plan ceilings.

## Reconcile before writing

Build a source-ID → VillaSlot UUID ledger for customers, villas and bookings.
Match customers by email in the organization; names alone can be ambiguous.
Validate local dates, start-minute alignment and supported/priced durations. Past
and future hourly bookings use the same creation endpoint; neither bypasses rate,
opening-hour, collision or plan checks. There is no `createdAt`/external source-ID
field accepted by POST /bookings, so import time differs from original creation time.

## Notifications

API v1 has no request flag that suppresses email or internal notifications. If the
user requires a silent import, have them disable the relevant application delivery
settings and verify that setting before writes. Do not assume an API key disables
side effects. Never test silence by sending a message to a real customer.

## Customers

POST /customers/bulk takes `{"customers":[{"name":"Tomas","email":"tomas@example.com"}]}`.
Maximum 200 rows; >200 returns 413 before any creation. Response status is 200 with
`data.summary {received,created,failed}`, `data.created[]`, and
`data.failed[] {index,error}`. Indices are zero-based in the submitted batch.
A row failure does not undo earlier successful rows. Empty batches return zero counts.
Persist results, reconcile ambiguous network failures, and retry only failed rows.
No upsert behaviour is promised; an existing email is a validation failure.

## Bookings and payments

Create bookings individually; there is no booking batch endpoint. Save each response
before the next write. Read the period and compare customer, villa and exact timestamps
before retrying after a lost response. Those attributes are reconciliation evidence,
not a guaranteed external-ID uniqueness constraint.

Unsupported source durations need a user-approved mapping; do not silently round or
split bookings. Do not manufacture PAID/REFUNDED by passing those fields in a request:
the API ignores them, and creating CONFIRMED does not record a receipt. Source payment
records cannot be imported via API v1. Report them as not imported, and use the app's
payment workflow separately when authorized. Never invent cash receipts to balance a
financial dashboard.
