# API v1 contract

Verified against the shipped route handlers on 2026-09-11. These are the supported
public endpoints, not every feature in the application. Paths below include `/api/v1`.
Every request uses `x-api-key`; JSON writes use `Content-Type: application/json`.

## Endpoint inventory

| Method | Path | Organization role |
|---|---|---|
| POST | `/api/v1/bookings/{bookingId}/cancel` | member (CASL applies) |
| POST | `/api/v1/bookings/{bookingId}/complete` | member (CASL applies) |
| POST | `/api/v1/bookings/{bookingId}/confirm` | member (CASL applies) |
| POST | `/api/v1/bookings/{bookingId}/reschedule` | member (CASL applies) |
| GET | `/api/v1/bookings/{bookingId}` | member (CASL applies) |
| GET | `/api/v1/bookings` | member (CASL applies) |
| POST | `/api/v1/bookings` | member (CASL applies) |
| GET | `/api/v1/customers/{customerId}` | member (CASL applies) |
| PATCH | `/api/v1/customers/{customerId}` | member (CASL applies) |
| POST | `/api/v1/customers/bulk` | member (CASL applies) |
| GET | `/api/v1/customers` | member (CASL applies) |
| POST | `/api/v1/customers` | member (CASL applies) |
| GET | `/api/v1/me` | member (CASL applies) |
| POST | `/api/v1/properties/{propertyId}/archive` | admin/owner |
| GET | `/api/v1/properties/{propertyId}/availability` | member (CASL applies) |
| GET | `/api/v1/properties/{propertyId}/opening-hours` | member (CASL applies) |
| PUT | `/api/v1/properties/{propertyId}/opening-hours` | admin/owner |
| POST | `/api/v1/properties/{propertyId}/photo` | admin/owner |
| GET | `/api/v1/properties/{propertyId}/rates` | member (CASL applies) |
| PUT | `/api/v1/properties/{propertyId}/rates` | admin/owner |
| GET | `/api/v1/properties/{propertyId}` | member (CASL applies) |
| PATCH | `/api/v1/properties/{propertyId}` | admin/owner |
| GET | `/api/v1/properties` | member (CASL applies) |
| POST | `/api/v1/properties` | admin/owner |
| GET | `/api/v1/settings/booking` | member (CASL applies) |
| PATCH | `/api/v1/settings/booking` | admin/owner |

`owner` includes admin capabilities. The key acts as its creator, with current
membership and service CASL checks. A customer role is not a staff integration role.
An admin endpoint can return 403 at the role gate; a service-level authorization
failure is deliberately mapped to 404, so not every denied operation is a 403.

## Request contracts

### Context and pagination

GET /me returns `data.organization {id,name,slug}`, `data.key {userId,organizationRole}`,
`data.settings {timezone,currency,bufferMinutes,holdExpirationHours,configured}` and
`data.plan {name,hasSubscription,limits}`. Each properties/bookings/customers limit
contains `{limit,usage,remaining}`. For all booking settings use GET /settings/booking.

GET /properties and GET /customers accept `page` (default 1), `limit` (default 20,
maximum 100), `search`. Invalid/negative pagination falls back to defaults;
limits over 100 clamp to 100. Properties also accept `status=active|archived|all`,
default active. Success: `{success:true,data:[...],pagination:{...}}`.

### Villas

POST /properties: `{"name":"Villa Example"}`. PATCH /properties/{propertyId}:
`{"name":"New name"}`. Neither accepts nightlyRate or arbitrary model fields.
POST /properties/{propertyId}/archive needs no body; it archives, not deletes.

PUT opening-hours: `{"openingHours":[{"dayOfWeek":1,"openMinute":480,"closeMinute":1080}]}`.
Sunday=0 through Saturday=6; minutes since local midnight. One range per open day;
omitted days are closed. GET returns the array in `data`.

PUT rates: `{"rates":[{"durationMinutes":120,"amount":1500000}]}`.
The whole grid is replaced. Amounts are positive integers in minor units, durations
60,120,180,240,300,360,420,480,540,600. GET returns the array in `data`.

POST photo: multipart `file` (nonempty File), optional `level=light|balanced|strong`
(default balanced). Do not set multipart content-type manually; let the client set
its boundary. Upload is compressed and appended to the villa gallery. The first
media becomes default; later uploads do not promise to become default. Returns a
property projection in `data`, including `defaultMediaUrl`, not the full media list.
Compression settings: light max dimension 1600/quality .82; balanced 1200/.72;
strong 900/.6. These are compression targets, not permission to send arbitrary files.

GET availability requires `date=YYYY-MM-DD` and `durationMinutes`. Response `data`:
`{propertyId,date,timezone,durationMinutes,slots:[{startAt,endAt}]}`. Slot timestamps
are ISO instants. Empty slots can mean closed or fully booked.

### Customers

POST /customers requires `name`; optional `email`, `phone`, `notes` become null when
omitted. PATCH /customers/{customerId} preserves omitted fields; explicit null clears
email/phone/notes. Name must remain valid. Email must be valid and unique within the
organization. GET returns the same customer projection, including internal notes.
POST /customers/bulk: see [imports.md](imports.md).

### Bookings

GET /bookings requires `from` and `to` ISO instants, optional `propertyId` UUID.
Use timezone-qualified values (e.g. `2026-09-11T00:00:00+08:00`) and encode `+` in
query parameters. Response data: `{from,to,timezone,currency,bookings:[...]}`.
This is a period query, not a page/limit list.

POST /bookings requires:
```json
{"propertyId":"<uuid>","customerId":"<uuid>","date":"2026-09-12","startMinute":540,"durationMinutes":120,"status":"HOLD"}
```
Optional `priceAmount` (positive integer; absent/null uses villa grid) and
`depositPercent` (integer 1–100; absent/null means full payment requested).
Start minutes: 0–1439, divisible by 30. Calendar date must exist.
The route creates hourly shootings, not overnight bookings. `kind`, `forceConflict`,
`preferredPaymentMethodCode`, `paymentStatus`, `createdAt` and notification-suppression
flags are not forwarded by this route. Do not send them expecting a result.

POST confirm/complete/cancel needs no body. Confirm only an eligible hold;
complete only a confirmed booking. Rescheduling applies to eligible bookings and
requires `propertyId`, `date`, `startMinute`, `durationMinutes`, optional `priceAmount`.
It is not a partial patch: supply the full target slot, preserving current fields
where appropriate. Omit the price to retain it. Business state/collision checks run
again on the server; do not infer success from a previous availability response.

GET /bookings/{bookingId} and list bookings include villa/customer names and
`blocksSlot`. Creation/transitions return the narrower booking projection. All
include UUIDs, ISO timestamps, kind, status, paymentStatus, priceAmount, currency,
depositPercent, bufferMinutes and holdExpiresAt. Public access tokens are excluded.
The app detail URL is `/{locale}/team/{organization.slug}/bookings/{booking.id}`;
it requires a web session, not an API-key query string.

### Settings

PATCH /settings/booking accepts currency (IDR/EUR/USD), timezone,
holdExpirationHours (1–168), holdBlocksSlot, bufferMinutes (0–480),
publicBookingEnabled, minimumNights (1–30). It preserves omitted fields; null also
falls back to the current value. Changing minimumNights does not expose overnight
creation through this API. Supported timezones: UTC, Asia/Makassar, Asia/Jakarta,
Asia/Singapore, Asia/Dubai, Europe/Paris, Europe/London, America/New_York.

GET returns organizationId, currency, timezone, timezoneConfigured, bufferMinutes,
holdExpirationHours, holdBlocksSlot, publicBookingEnabled, minimumNights, configured,
updatedAt. Changing settings does not rewrite prices/currency/buffers on old bookings.

## Response and failure handling

- 200 read/update, 201 create: `{success:true,data:...}`.
- Errors carry `{error: "..."}`, not necessarily `success:false`.
- 401 missing/invalid/revoked key or no usable organization membership.
- 403 explicit organization-role gate; 404 missing/cross-tenant/service authorization.
- 409 business conflict: `{error,detail}`. Creation refuses with `detail.reason`.
- 413 customer batch above 200; 422 invalid parameters with optional `details` array.
- 429 rate limit (keys created by the app: 120/minute); back off, no rapid retry loop.
- 500 unexpected failure; reconcile writes before retrying.

Creation refusal reasons include OVERLAP, CLOSED, OUTSIDE_OPENING_HOURS,
INVALID_INTERVAL, DURATION_NOT_PRICED, CURRENCY_NOT_CONFIGURED,
PROPERTY_NOT_BOOKABLE, CUSTOMER_NOT_FOUND. Plan conflicts instead provide limitType,
limit, usage (and other plan information) in `detail`; they are not slot conflicts.

A 404 does not distinguish another tenant from absence. Never probe other tenants.
After a timeout/5xx on POST, read back and reconcile before creating again: API v1
has no public idempotency-key contract. Read-only diagnostics never require a real
booking, payment, email, or production mutation.
