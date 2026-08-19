---
name: Triage and annotate CallRail calls
description: Page through recent calls for an account, inspect a single call, then tag it, note it, set lead status, correct the customer name, or mark it as spam.
api: openapi/callrail-calls-api-openapi.yml
operations:
  - listAccounts
  - listCalls
  - getCall
  - updateCall
generated: '2026-08-14'
method: generated
source: openapi/callrail-calls-api-openapi.yml + https://apidocs.callrail.com/
---

# Triage and annotate CallRail calls

The review loop: find the calls, read one, write back what you learned.

## Authentication

```
Authorization: Token token="YOUR_API_KEY"
```

Base URL `https://api.callrail.com/v3`. The key inherits its user's visibility — calls belonging to
companies that user cannot see simply will not appear.

## Steps

### 1. Resolve the account — `listAccounts`

```
GET /v3/a.json
```

Take `accounts[].id` as `{account_id}`.

### 2. List the calls — `listCalls`

```
GET /v3/a/{account_id}/calls.json?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD&relative_pagination=true&per_page=250
```

Documented filters on this operation: `start_date`, `end_date`, `direction` (`inbound`|`outbound`),
`answered`, `company_id`, `tracker_id`, `tags`.

**Use relative pagination here.** This is the only endpoint that supports it and CallRail recommends
it for larger data sets. Follow `next_page` until `has_next_page` is false. Never mix relative and
offset pagination across requests for the same dataset. Page n+1 may not start exactly where page n
ended if a new call lands mid-walk — de-duplicate by call `id`.

Fields like `person_id`, `sentiment`, `transcription` and `zip_code` are **not** in the default
response. Request them explicitly:

```
&fields=person_id,sentiment,transcription
```

`transcription` and `sentiment` return null unless the account has a Premium Conversation
Intelligence subscription. A null there means "not entitled", not "no speech".

### 3. Read one call — `getCall`

```
GET /v3/a/{account_id}/calls/{call_id}.json?fields=person_id,conversational_transcript
```

`person_id` is the identity that ties this call to the same lead's form submissions and texts — use
it to correlate rather than matching on phone number.

### 4. Write back — `updateCall`

```
PUT /v3/a/{account_id}/calls/{call_id}.json
Content-Type: application/json

{ "tags": ["qualified"], "note": "…", "lead_status": "good_lead", "customer_name": "…", "spam": false }
```

Send only the fields you intend to change.

## Rules

- **No idempotency key.** CallRail publishes no idempotency or deduplication contract. A retried
  `PUT` is generally safe because it is a full-field assignment, but never retry blindly on a
  timeout — re-read the call with `getCall` and compare before writing again.
- **Updates fire webhooks.** A successful `updateCall` triggers the Call Modified webhook, whose
  `changes` array names exactly which fields changed (`tags`, `note`, `value`,
  `transcription_text`, `call_summary`, `recording_duration`, `auto_score`, `manual_score`).
  Downstream systems will see your edit. Marking a call as spam fires it too.
- **`spam: true` is a real business action.** It removes the call from lead reporting. Confirm intent
  with a human before setting it.
- **25-month retention wall** on any date range, including `all_time`.
- **Rate limits:** 1,000/hour, 10,000/day. HTTP 429 on exhaustion, no `Retry-After` header.
- Use `PUT`, not `PATCH` — CallRail supports GET/POST/PUT/DELETE only, and `PATCH` returns 405.

## Related

- `asyncapi/callrail-webhooks.yml` — the Call Modified event your write triggers
- `data-model/callrail-data-model.yml` — how `person_id` links calls, forms and texts
- `errors/callrail-problem-types.yml` — 400 vs 406 vs 422 on a bad body
