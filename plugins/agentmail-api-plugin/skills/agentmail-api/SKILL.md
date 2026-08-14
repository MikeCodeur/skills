---
name: agentmail-api
description: >
  Drive the AgentMail email marketing API v1 from an API key — lists, contacts,
  tags, templates, campaigns, sending domains and stats.
  Use when the user says "AgentMail API", "send a campaign", "import contacts",
  "create an email template", "check campaign stats", "verify sending domain",
  "why was my send refused", or when automating email marketing programmatically.
license: MIT
compatibility: Requires curl or any HTTP client. An AgentMail API key is required.
metadata:
  author: mikecodeur
  version: "1.0"
---

# AgentMail API — Email Marketing Skill

Automate email marketing through the AgentMail REST API: build audiences, write templates,
render them safely, send campaigns, and read results — without touching the web UI.

## Safety rule, first

**Never trigger a real send without explicit human approval for that specific send.**
Drafting, importing, rendering: safe. `POST /send` reaches real inboxes and is paid for in
sender reputation. Approval for one send never covers the next one.

**A `test-send` is a real send too.** It goes through the same guards, consumes quota, and lands
in a real inbox. `POST /render` is the only side-effect-free endpoint.

---

## Authentication

**Get your API key:** log in to your AgentMail instance → Account → API keys → create a key.

A key belongs to exactly one organization. That organization is carried inside the key, so
**no request body ever takes an `organizationId`** — it is always derived from the key.

```bash
export AGENTMAIL_BASE_URL="https://your-agentmail-instance.example"
export AGENTMAIL_API_KEY="your-api-key-here"

curl -s -H "x-api-key: $AGENTMAIL_API_KEY" "$AGENTMAIL_BASE_URL/api/v1/me"
```

All routes live under `/api/v1` and require the header:

```
x-api-key: $AGENTMAIL_API_KEY
```

The acting identity is the key **owner**, with their permissions. If the owner leaves the
organization, the key stops working. Rate limit: **120 requests per minute per key** (429 above).

---

## Start with the diagnostic, not with the campaign

`GET /api/v1/me` answers "can I send, and if not why".

```json
{
  "success": true,
  "data": {
    "organization": {"id": "...", "name": "...", "slug": "..."},
    "organizationRole": "OWNER",
    "keyId": "...",
    "sending": {"ready": false, "blockers": ["domain_not_verified"]}
  }
}
```

`blockers` is a **stable, untranslated machine vocabulary** — branch your remediation logic on it:

| Blocker | Meaning |
|---------|---------|
| `sending_disabled` | Application-wide kill switch is on. Nothing an API caller can fix. |
| `organization_not_allowed` | The organization is not cleared to send. |
| `domain_not_verified` | No sending domain with `dkimStatus: SUCCESS`. |
| `no_sender_identity` | Never blocking on its own — without a configured identity, mail falls back to the application address. It only appears alongside a real blocker. |

If `ready` is `false`, **fix it before building a campaign** instead of discovering the refusal
at the last step.

---

## API Reference

**Base URL:** `$AGENTMAIL_BASE_URL/api/v1`

### Endpoints Overview

| Method | Endpoint | Description |
|--------|----------|-------------|
| **Discovery & settings** | | |
| GET | `/me` | Organization behind the key + sending readiness |
| GET | `/settings` | Sending identity, opt-in settings and readiness in one read |
| PATCH | `/settings/sending` | Sender email, sender name, postal address |
| PATCH | `/settings/optin` | Double opt-in toggle and confirmation email |
| POST | `/settings/optin/test-send` | Test the confirmation email (real send) |
| **Sending domains** | | |
| GET | `/domains` | Domains, statuses and DNS records to publish |
| POST | `/domains` | Declare a domain and create its sending identity |
| GET | `/domains/:domainId` | One domain with its DNS records |
| POST | `/domains/:domainId/verify` | Force a fresh verification check |
| DELETE | `/domains/:domainId` | Remove the declaration (keeps the sending identity) |
| **Lists & contacts** | | |
| GET | `/lists` | Paginated lists with contact counts |
| POST | `/lists` | Create a list |
| GET | `/lists/:listId` | List detail |
| POST | `/lists/:listId/contacts` | Upsert one contact (emits side effects) |
| POST | `/lists/:listId/contacts/bulk` | Import up to 500 contacts (emits nothing) |
| GET | `/contacts?listId=…` | Paginated contacts — `listId` is **required** |
| GET | `/contacts/:id` | Contact detail |
| PATCH | `/contacts/:id` | Update `firstName` / `lastName` only |
| POST | `/contacts/:id/tags` | Attach a tag by name (get-or-create, idempotent) |
| DELETE | `/contacts/:id/tags/:tagId` | Detach a tag (idempotent) |
| POST | `/contacts/:id/unsubscribe` | Unsubscribe (idempotent) |
| POST | `/contacts/:id/resubscribe` | Resubscribe (idempotent) |
| **Tags** | | |
| GET | `/tags` | Tags with contact counts, `?search=` supported |
| POST | `/tags` | Create a tag (names are unique, case-insensitive) |
| **Templates** | | |
| GET | `/templates` | Paginated templates |
| POST | `/templates` | Create a template |
| GET | `/templates/:id` | Template detail |
| PATCH | `/templates/:id` | Update name, default subject, content |
| DELETE | `/templates/:id` | Delete a template |
| POST | `/templates/:id/test-send` | Send a test (real send) |
| **Campaigns** | | |
| GET | `/campaigns` | Paginated campaigns, `?status=` and `?search=` |
| POST | `/campaigns` | Create a draft campaign |
| GET | `/campaigns/:id` | Campaign + recipient count + send progress |
| PATCH | `/campaigns/:id` | Edit a **draft** campaign |
| POST | `/campaigns/:id/test-send` | Send a test (real send) |
| POST | `/campaigns/:id/send` | Trigger or schedule the send |
| GET | `/campaigns/:id/stats` | Delivery, opens, clicks, unsubscribes |
| **Rendering** | | |
| POST | `/render` | Render HTML with warnings — **no side effects** |

### Pagination

Index endpoints take `?page=1&limit=20`. `limit` is capped at **100**. Nonsense values fall back
to the default instead of failing. Responses carry a `pagination` object next to `data`.

### Response envelope and status codes

Success: `{"success": true, "data": …, "pagination": …}`
Failure: `{"error": "…", "details": [ …zod issues… ]}`

| Code | When |
|------|------|
| 200 / 201 / 202 | read or upsert / creation / send accepted for background processing |
| 400 | validation failed — `details` carries the field-level issues |
| 401 | missing or invalid key, or owner no longer a member |
| 403 | valid key, insufficient organization role |
| 404 | resource missing **or owned by another organization** — never 403 |
| 409 | state conflict (already sent, incomplete configuration, test refused) |
| 413 | bulk batch over 500 contacts; the body names the `limit` |
| 429 | per-key rate limit exceeded |

**The 404 is a trap.** It does not mean "does not exist", it means "not yours". The API
deliberately refuses to confirm that an id exists to a caller who has no right to it.

### Permissions

| Operation | Required |
|-----------|----------|
| Reads | membership in the key's organization |
| Writes | organization role **ADMIN** or above |
| `POST /campaigns/:id/send` | organization ADMIN **plus** an application-level admin role |

Mass sending is gated behind an application-level allowlist. A `404` on `send` can therefore mean
"you are not permitted to send", not "the campaign is gone".

---

## Agent workflow

```bash
BASE="$AGENTMAIL_BASE_URL/api/v1"
AUTH="x-api-key: $AGENTMAIL_API_KEY"

# 1. Can I send at all?
curl -s -H "$AUTH" "$BASE/me"

# 2. Find the audience
curl -s -H "$AUTH" "$BASE/lists"

# 3. Create a template
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"name":"Newsletter","defaultSubject":"Hello","content":"<html>…{{unsubscribeUrl}}…</html>"}' \
  "$BASE/templates"

# 4. Render it — repeat freely, nothing is sent
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"templateId":"<uuid>"}' "$BASE/render"

# 5. Create the campaign (content is COPIED from the template)
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"name":"August","subject":"Hello","listId":"<uuid>","templateId":"<uuid>"}' \
  "$BASE/campaigns"

# 6. Test in a real inbox
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"email":"you@example.com"}' "$BASE/campaigns/<id>/test-send"

# 7. Send — only after explicit human approval
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{}' "$BASE/campaigns/<id>/send"

# 8. Follow it
curl -s -H "$AUTH" "$BASE/campaigns/<id>"
curl -s -H "$AUTH" "$BASE/campaigns/<id>/stats"
```

**Key design fact:** a campaign's `content` **is** the HTML that gets sent, never a live reference
to the template. Editing the template afterwards changes nothing for campaigns already created.
`templateId` is optional — without it the campaign starts from a default skeleton.

Scheduling: `POST /campaigns/:id/send` accepts `{"scheduledAt": "2026-01-01T09:00:00+01:00"}`,
ISO 8601 **with offset**, at least 60 seconds in the future → `202 {"status": "scheduled"}`.
Without a body → `202 {"status": "send_requested"}`.

Only `draft` and `scheduled` campaigns can be sent. A `sent` campaign returns 409 and is never
replayed — duplicate it instead.

---

## Setting up a sending domain

Without a verified domain, **no mass send goes out**: `POST /send` returns 409 with an incomplete
configuration error. The loop is designed to be automated:

1. `POST /domains` with `{"domain": "mail.example.com"}` — creates the sending identity.
2. `GET /domains` — `dnsRecords` holds the records to publish. They are never stored; they are
   read back from the provider on every call, because they change if the identity is recreated.
3. Publish the DNS records, then `POST /domains/:domainId/verify` to force a fresh check, and
   repeat until `verified: true` (i.e. `dkimStatus === "SUCCESS"`).

`sesUnavailable: true` on a read means you are seeing the last known state, dated by
`lastCheckedAt` — not the current truth. On `verify`, a provider outage surfaces as an error
instead, because verification is exactly what was asked for.

`DELETE /domains/:id` removes the declaration but keeps the underlying sending identity: another
organization may have declared the same domain, and a verified identity is expensive to rebuild.

---

## Sender identity and opt-in

`GET /settings` returns everything in one read: `sending` (`senderEmail`, `senderName`,
`postalAddress`, and `effectiveSender` — what will actually be used, fallback included), `optin`,
and `readiness`.

`PATCH /settings/sending` uses **strict PATCH semantics**: an absent field is left untouched, a
field set to `null` is cleared and falls back. Never resend the whole object just to change one
field.

`PATCH /settings/optin` takes `{required?, subject?, content?}`. Content without
`{{confirmationUrl}}` is rejected. Content identical to the default is stored as empty, so the
organization keeps inheriting future default improvements — that is what the returned
`customized` flag tells you.

---

## Writing templates

### Variables

A closed vocabulary of seven names:

`{{firstName}}` · `{{lastName}}` · `{{email}}` · `{{unsubscribeUrl}}` · `{{currentYear}}` ·
`{{postalAddress}}` · `{{confirmationUrl}}` (opt-in confirmation template only)

Unknown variables are left in place, never substituted, and reported in `unknownVariables`.
They never raise an error — so they are invisible unless you read the render output.

**`{{unsubscribeUrl}}` is mandatory.** Rendering verifies the final HTML actually contains the
link and **refuses** (400) otherwise. No footer is grafted into your design automatically: only
the template can carry it.

First and last names are frequently empty on imported audiences. **Never write a sentence that
collapses without them.**

### Pasting from Mailchimp

`*|FNAME|*`, `*|LNAME|*`, `*|EMAIL|*`, `*|UNSUB|*` and `*|CURRENT_YEAR|*` are converted
automatically on create and update. Anything else is left untouched and listed in
`unsupportedTags` in the response — **read it**, or a raw `*|MERGE7|*` ships to real inboxes.

### Dark mode — the rule that matters

**Never declare `<meta name="color-scheme">` without shipping the matching rules.** The
declaration tells the mail client "I handle dark mode myself", so it stops applying its own
coherent inversion — which flips background *and* text together. With no
`@media (prefers-color-scheme: dark)` block behind it, the background darkens and the text stays
dark. **The tag alone is worse than nothing.**

Three mechanisms, and all three are required — miss one and a whole family of clients breaks:

1. **Apple Mail** honours `prefers-color-scheme`. Ship an
   `@media (prefers-color-scheme: dark)` block keyed to the colors **actually present** in the
   document, with `!important`. Attribute selectors (`[style*="background-color:#ffffff"]`,
   `[bgcolor="#ffffff"]`) let you do this without restructuring the HTML.
2. **The Gmail app ignores media queries** and inverts its own way. The only defence is that
   **every element carrying a background also carries an explicit inline text color** — without
   it, Gmail flips the background and leaves inherited text as-is.
3. **Outlook.com** does neither: it prefixes selectors. Provide `[data-ogsc]` for text and
   `[data-ogsb]` for background, **outside** the media query.

**The governing rule:** in every dark declaration, set background **and** text together, never one
without the other. A remapped background without its text produces exactly the bug you are trying
to avoid.

**The `!important` trap:** an `!important` in a stylesheet does **not** beat an inline style that
also carries one — at equal importance the `style` attribute wins (CSS Cascade 4 §6.4.4). Email
templates are full of `style="margin:0 !important;…"`.

**How to verify:** flip your OS theme and reload the preview, and above all open a real
`test-send` **on Apple Mail iOS** — that is the client that breaks.

### Rendering before sending

`POST /render` takes `{templateId?, content?, subject?, variables?}` — `templateId` **or**
`content`, never neither. A supplied `content` wins, which lets you try an edit before saving it.

Returns `{html, text, subject, unknownVariables, warnings}`. Warning vocabulary:
`missing_postal_address`, `dark_mode_contrast_corrected`, `dark_mode_contrast_unverified`,
`dark_mode_declaration_removed`.

**Gotcha:** `/render` does not inject the organization's postal address, so
`missing_postal_address` **always** appears on this endpoint. It is not a template defect — the
real address is injected at send time. Check `GET /settings` to know whether it is configured.

---

## Importing contacts

`POST /lists/:listId/contacts` — single upsert, **emits side effects**: enrolment event and a
double opt-in confirmation email when enabled. Returns 200 (not 201): it is an upsert.

`POST /lists/:listId/contacts/bulk` — **emits nothing**: no email, no event, whatever the
configuration. `doubleOptIn` is explicitly *rejected* here rather than ignored, so no caller ever
believes 500 confirmations went out.

```json
{
  "contacts": [
    {"email": "a@example.com", "firstName": "Ada", "lastName": "L", "tags": ["vip"],
     "status": "subscribed", "optinAt": "2025-03-01T10:00:00Z", "optinIp": "203.0.113.4"}
  ],
  "tags": ["import-march"]
}
```

- 500 contacts max per call — 413 above, never a silent truncation
- Batch `tags`: 5 max, unioned with each row's own `tags`
- `optinAt` / `optinIp` / `confirmedAt` / `confirmedIp` carry consent proof for a migrated audience
- Response: `{total, created, updated, rejected, errors: [{index, email, reason}]}`

`index` is the position in the batch you sent: **replay only the failed rows**, never all 500.

Contact statuses: `pending`, `subscribed`, `unsubscribed`, `bounced`, `complained`. A `bounced`
contact can never be resubscribed manually — a hard bounce is final.

---

## Following a send

`GET /campaigns/:id` returns the campaign plus three things nothing else gives you:

- `recipientCount` — how many subscribers the target hits **right now** (live, never stored)
- `progress` — `{total, sent, failed, pending, sending, lastSentAt}`
- `sendingEnabled` — the application-wide kill switch

Read them together:

- `sendingEnabled: false` → polling until `sent` **will never converge**. The campaign stays
  `draft` with zero progress.
- `sending: 0` with `pending` remaining on a `sending` campaign → it is stuck, not slow.
  `lastSentAt` is the only signal that separates the two.

Campaign statuses: `draft`, `scheduled`, `sending`, `sent`, `failed`.

`GET /campaigns/:id/stats` returns `sent`, `delivered`, `bounced`, `uniqueOpens`, `openRate`,
`uniqueClicks`, `clickRate`, `unsubscribed`, `unsubscribeRate`, `clicksByLink`, `clicksOverTime`.

Rates are computed on **delivered** (sent minus bounces) and read 0 while nothing is delivered
yet — judging a send by them before the progress completes is meaningless.

### Why a test was refused

A refused test returns 409 naming the cause: `blocked` (kill switch), `not_allowed`
(organization), `suppressed` (address on the suppression list after a bounce or complaint),
`not_sendable` (wrong state), `rate_limited` (test quota).

A test writes **no** statistics and changes **no** status: no open tracking, no click tracking.

---

## Deliverability

Ramp up in stages, starting with your most engaged contacts, and **wait 24 hours between
stages**. Bounces land within minutes, but **complaints** arrive hours later. Sending the next
stage before reading them is flying blind.

Provider thresholds (Amazon SES): bounces 5% (under review) / 10% (suspension), complaints
0.1% / 0.5%. At low volume a single ratio means nothing — one complaint out of 700 is already
0.14%.

Bounces and complaints feed the suppression list automatically. A suppressed address will not be
mailed again, and that is intentional.

---

## Limits

| What | Value |
|------|-------|
| API requests | 120 / minute / key |
| Contacts per `bulk` call | 500 (413 above) |
| Tags applied to a batch | 5 |
| Test sends | 10 / hour / user, campaigns and templates combined |
| Pagination `limit` | 100 |
| Minimum scheduling lead time | 60 seconds |

---

## Common mistakes

| Mistake | What actually happens |
|---------|----------------------|
| Building the campaign first, checking readiness last | The send is refused after all the work. Call `GET /me` first. |
| Reading a `404` as "deleted" | It also means "another organization's resource" or "not permitted". |
| Editing the template to fix a created campaign | The campaign holds a copy. Patch the campaign. |
| Ignoring `unsupportedTags` | Raw `*|MERGE|*` tags ship to real inboxes. |
| Trusting `missing_postal_address` from `/render` | It is always raised there. Check `GET /settings`. |
| Polling a campaign until `sent` | With `sendingEnabled: false` it never converges. |
| Sending the whole batch again after partial failures | Use `errors[].index` to replay only failed rows. |
| Treating `test-send` as a dry run | It is a real send, with real quota and a real inbox. |
