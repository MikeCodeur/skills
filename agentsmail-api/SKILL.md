---
name: agentsmail-api
description: >
  Drive the AgentsMail email marketing API v1 from an API key — lists, contacts,
  tags, templates, campaigns, automated sequences, sender addresses, sending
  domains and stats.
  Use when the user says "AgentsMail API", "send a campaign", "import contacts",
  "create an email template", "build a drip sequence", "check campaign stats",
  "verify sending domain", "why was my send refused", or when automating email
  marketing programmatically.
license: MIT
compatibility: Requires curl or any HTTP client. An AgentsMail API key is required.
metadata:
  author: mikecodeur
  version: '2.0'
---

# AgentsMail API — Email Marketing Skill

Automate email marketing through the AgentsMail REST API: build audiences, write templates,
render them safely, send campaigns, run automated sequences, and read results — without touching
the web UI.

## Safety rule, first

**Never trigger a real send without explicit human approval for that specific send.**
Drafting, importing, rendering: safe. `POST /send` reaches real inboxes and is paid for in
sender reputation. Approval for one send never covers the next one.

**A `test-send` is a real send too.** It goes through the same guards, consumes quota, and lands
in a real inbox. `POST /render` is the only side-effect-free endpoint.

---

## Authentication

**Get your API key:** log in to your AgentsMail instance → Account → API keys → create a key.

A key belongs to exactly one organization. That organization is carried inside the key, so
**no request body ever takes an `organizationId`** — it is always derived from the key.

```bash
# Hosted AgentsMail — note the www, and no trailing slash
export AGENTMAIL_BASE_URL="https://www.agentsmail.io"
export AGENTMAIL_API_KEY="your-api-key-here"

curl -s -H "x-api-key: $AGENTMAIL_API_KEY" "$AGENTMAIL_BASE_URL/api/v1/me"
```

> **Use the `www.` host.** `https://agentsmail.io/api/v1/…` answers **308** and redirects to
> `https://www.agentsmail.io/api/v1/…`. A 308 does preserve the method and the body, so `curl -L`
> survives it — but a client that does not follow redirects receives an empty 308 and looks like it
> failed for no reason. Point straight at `www.` and the problem disappears.
>
> Self-hosting? Override `AGENTMAIL_BASE_URL` with your own origin. Nothing else in this skill is
> host-specific.

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
    "organizationRole": "admin",
    "keyId": "...",
    "sending": {
      "ready": false,
      "blockers": ["domain_not_verified"],
      "setup": {"status": "dns_pending", "provisioning": "provisioned"}
    }
  }
}
```

`blockers` is a **stable, untranslated machine vocabulary** — branch your remediation logic on it,
never on a message:

| Blocker | Meaning | How to clear it |
|---------|---------|-----------------|
| `sending_disabled` | Sending is switched off application-wide | Nothing an API caller can fix |
| `organization_not_allowed` | The organization is not cleared to send | Ask the operator |
| `sending_paused` | The email provider **suspended** this organization, usually over a bounce or complaint rate | **Not retryable.** It clears on its own when the provider re-enables sending. Campaign test sends stay available meanwhile — they go out under the application address |
| `domain_not_verified` | A domain is declared but its DKIM is not `SUCCESS` | Publish the DNS records, then `verify` |
| `domain_ownership_unproven` | DKIM is `SUCCESS`, but this organization has not proved it owns the domain | Publish the `_agentmail-challenge` `TXT` record, then `verify` |
| `sender_not_on_verified_domain` | A domain is verified, but the **default** sender address is not on it (nor on one of its subdomains) | Set or promote an address on the verified domain |
| `no_sender_identity` | Never blocking on its own — without a sender address, mail falls back to the application address. It only appears alongside a real blocker, to say the account is bare | Add an address (`POST /settings/identities`) |

`sending.setup` reports the same state the account owner reads on screen, as one code instead of a
list: `status` (e.g. `dns_pending`) and `provisioning`.

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
| PATCH | `/settings/sending` | Postal address and non-address sending settings |
| PATCH | `/settings/optin` | Double opt-in toggle and confirmation email |
| POST | `/settings/optin/test-send` | Test the confirmation email (real send) |
| **Sender address book** | | |
| GET | `/settings/identities` | Every sender address, default first |
| POST | `/settings/identities` | Add an address — `{senderEmail, senderName?, isDefault?}` |
| PATCH | `/settings/identities/:identityId` | Edit an address, or promote it with `{"isDefault": true}` |
| DELETE | `/settings/identities/:identityId` | Remove an address from the book |
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
| **Sequences** | | *20 routes — see the Sequences section* |
| GET / POST | `/sequences` | List sequences / create one **and** its version 1 draft |
| GET / PATCH / DELETE | `/sequences/:id` | Detail / rename, activate, pause / delete |
| GET | `/sequences/:id/enrollments` | Who is walking the journey, every status |
| GET / POST | `/sequences/:id/versions` | Version history / open a draft |
| PATCH / DELETE | `/sequences/:id/versions/:vId` | Change a draft's trigger / discard a draft |
| POST | `/sequences/:id/versions/:vId/publish` | Publish — **the graph is validated here** |
| POST | `/sequences/:id/versions/:vId/rollback` | Republish an old version as a **new** one |
| GET / POST | `/sequences/:id/versions/:vId/nodes` | Read nodes / add one to a draft |
| PATCH / DELETE | `/sequences/:id/versions/:vId/nodes/:nId` | Edit a node / remove it, stitching the journey |
| POST | `/sequences/:id/versions/:vId/nodes/:nId/move` | Move a node one step up or down |
| GET / POST | `/sequences/:id/versions/:vId/edges` | Read wires / wire two nodes |
| DELETE | `/sequences/:id/versions/:vId/edges/:eId` | Unwire two nodes |
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
| 404 | resource missing, **owned by another organization**, or **belonging to another version** (sequences) — never 403 |
| 409 | state conflict (already sent, incomplete configuration, test refused) |
| 413 | bulk batch over 500 contacts; the body names the `limit` |
| 429 | per-key rate limit exceeded |

**The 404 is a trap.** It does not mean "does not exist", it means "not yours". The API
deliberately refuses to confirm that an id exists to a caller who has no right to it. On
sequences it goes further: a `versionId` that is not a version of the `sequenceId` in the URL
answers 404, and so does a `nodeId` that is not in that version. **An id is only ever read inside
the path that leads to it.**

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

`POST /campaigns` and `PATCH /campaigns/:id` also take **`sendingIdentityId`** — which sender
address signs this campaign. Absent or `null` means "the organization's default". Read the ids from
`GET /settings/identities`.

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
2. `GET /domains` — `dnsRecords` holds the records to publish: the DKIM records **and the
   `_agentmail-challenge` `TXT` record**. They are never stored; they are read back from the
   provider on every call, because they change if the identity is recreated.
3. Publish the DNS records, then `POST /domains/:domainId/verify` to force a fresh check, and
   repeat until `verified: true`.

**Two conditions, not one.** DKIM proves the domain can sign; the challenge `TXT` proves **this
organization** owns it. A domain can therefore sit at `dkimStatus: SUCCESS` and still block with
`domain_ownership_unproven`. And once any domain is verified, the default sender address must be on
it — otherwise `sender_not_on_verified_domain` blocks the send. Without that check the provider
accepts the batch and then fails every message with *"Email address is not verified"*, with nothing
in the product explaining why.

`sesUnavailable: true` on a read means you are seeing the last known state, dated by
`lastCheckedAt` — not the current truth. On `verify`, a provider outage surfaces as an error
instead, because verification is exactly what was asked for.

`DELETE /domains/:id` removes the declaration but keeps the underlying sending identity: another
organization may have declared the same domain, and a verified identity is expensive to rebuild.

---

## The sender address book

An organization registers **several** verified sender addresses and marks one as the default.
There is no single sender address hanging off the organization any more.

| Route | Role | Effect |
|-------|------|--------|
| `GET /settings/identities` | member | The whole book, default first |
| `POST /settings/identities` | admin | Add an address — `{senderEmail, senderName?, isDefault?}` |
| `PATCH /settings/identities/:identityId` | admin | Change the address or display name, or promote it with `{"isDefault": true}` |
| `DELETE /settings/identities/:identityId` | admin | Remove an address from the book |

Rules that govern the book:

- An address only lives on a domain that is **verified and proven** (both conditions — see above).
- **Exactly one default** per organization. To change it, promote another address; deleting the
  default is refused while others remain. Deleting the last one is allowed.
- Deleting an address referenced by a campaign is **refused, and the campaigns are named** —
  whatever their status.
- A campaign picks its address with `sendingIdentityId`. Absent or `null` means "the default".

`GET /settings` still returns everything in one read: `sending` (including `effectiveSender` — what
will actually be used, fallback included), `optin`, and `readiness`.

`PATCH /settings/sending` remains available and backward-compatible for settings that are not
addresses (notably `postalAddress`), with **strict PATCH semantics**: an absent field is left
untouched, a field set to `null` is cleared and falls back. Never resend the whole object just to
change one field.

## Opt-in

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

**A contact belongs to exactly one list.** The same person in two lists is two contacts, with two
distinct unsubscribe links — so unsubscribing is per list, not per person. Plan accordingly when
you model an audience: there is no cross-list identity.

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

## Sequences — an automated journey

A contact enters on a trigger, then walks a graph of nodes: send an email, wait, add a tag, remove
a tag, call a webhook, stop.

**Four objects:**

| Object | What it is |
|--------|-----------|
| **Sequence** | The journey's identity: a name, a status (`active` / `paused`), and a pointer to the version currently running |
| **Version** | A **snapshot** of the journey, with its trigger. A `draft` is editable; a `published` one never changes again |
| **Node** | One step: `send_email`, `wait`, `add_tag`, `remove_tag`, `end_sequence`, `webhook` |
| **Edge** | A wire from one node to the next. Adding a node wires it for you; edges exist so you can rewire |

Two consequences run through everything:

- **You never edit a running journey.** Editing means opening a draft version, changing it, and
  publishing it. Contacts already walking the old version finish on the old version.
- **The trigger lives on the version**, not on the sequence. You set it when you create the
  sequence — that call creates version 1 with it — and you change it like anything else: `PATCH` the
  draft, then publish.

**There is no endpoint to enrol a contact directly.** A contact enters through the trigger and only
through it: joining a list (`list_joined`) or receiving a tag (`tag_added`). So you add the contact
to the list, or tag it, and enrolment follows.

> **The trap that costs hours:** `POST /lists/:listId/contacts/bulk` **emits nothing** — so a bulk
> import enrols nobody, whatever the sequence is configured to do. Use the single-contact upsert, or
> apply the tag afterwards, when you want enrolment.

```bash
# Create a sequence and its version 1 draft, in one transaction → 201
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"name":"Onboarding","triggerType":"list_joined","triggerTargetId":"<listId>"}' \
  "$BASE/sequences"

# Add a step to the draft, then publish — the graph is validated at publish time
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"type":"send_email","templateId":"<uuid>"}' \
  "$BASE/sequences/<id>/versions/<vId>/nodes"

curl -s -X POST -H "$AUTH" "$BASE/sequences/<id>/versions/<vId>/publish"

# Nothing runs until the sequence itself is active
curl -s -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"status":"active"}' "$BASE/sequences/<id>"
```

Reading a sequence needs **membership**. Everything that writes — creating, editing a draft,
publishing, activating, deleting — needs **admin**.

**A 400 at publish time is not a bug**: it is the graph being refused. Validation happens when you
publish, not on every edit — so build freely, then read the refusal.

`GET /sequences/:id/enrollments` lists who is in the journey, paginated, **across every status** —
the only way to see a contact that finished or dropped out.

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

Resource ceilings — verifiable domains, contacts, monthly sends — depend on the organization's
plan. Exceeding one answers 400 and names the ceiling.

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
| Bulk-importing contacts to trigger a sequence | Bulk emits nothing, so it enrols nobody. Use the single upsert, or tag afterwards. |
| Retrying on `sending_paused` | The provider suspended the organization. Retrying never clears it. |
| Verifying DKIM and stopping there | Ownership is a second, separate condition (`_agentmail-challenge` `TXT`). |
| Expecting one unsubscribe to cover a person | A contact belongs to one list. The same person in two lists unsubscribes twice. |
| Editing a published sequence version | Published versions are frozen. Open a draft, change it, publish it. |
