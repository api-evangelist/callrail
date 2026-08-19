---
name: Place an outbound call through CallRail
description: Use a CallRail tracking number as the caller ID to connect a business phone to a customer phone, with the safety checks this irreversible, billed operation requires.
api: openapi/callrail-calls-api-openapi.yml
operations:
  - listAccounts
  - createCall
  - getCall
generated: '2026-08-14'
method: generated
source: openapi/callrail-calls-api-openapi.yml + https://apidocs.callrail.com/
---

# Place an outbound call through CallRail

`createCall` rings two real telephones. Read the guardrails before the steps.

## Guardrails — read first

- **This is a physical-world action and it is not reversible.** There is no dry run, no sandbox and
  no test mode anywhere in the CallRail API. Every request hits production.
- **There is no idempotency key.** If a request times out you cannot tell whether the call was
  placed. Do **not** blind-retry: wait, then check with `listCalls` filtered by
  `direction=outbound` and the customer number before trying again.
- **A human must confirm the number and intent before the request is sent.** Never place a call from
  an inferred phone number.
- **It is billed and rate-limited separately** — 100/hour and 2,000/day, tighter than the general
  1,000/hour budget.
- Recording is subject to consent law in the caller's and recipient's jurisdictions. Do not set
  `recording_enabled: true` without explicit instruction.

## Authentication

```
Authorization: Token token="YOUR_API_KEY"
```

Base URL `https://api.callrail.com/v3`.

## Steps

### 1. Resolve the account — `listAccounts`

```
GET /v3/a.json
```

Note `outbound_recording_enabled` on the account — it tells you whether outbound recording is
available at all.

### 2. Place the call — `createCall`

```
POST /v3/a/{account_id}/calls.json
Content-Type: application/json

{
  "caller_id": "+14045558514",
  "customer_phone_number": "+12148654559",
  "business_phone_number": "+14045551234",
  "recording_enabled": false
}
```

All three of `caller_id`, `customer_phone_number` and `business_phone_number` are required.
`caller_id` must be a CallRail tracking number on the account — it is what the customer sees.
CallRail rings `business_phone_number` first, then connects to `customer_phone_number`.

### 3. Confirm — `getCall`

```
GET /v3/a/{account_id}/calls/{call_id}.json
```

Read `id` from the create response and confirm the record exists before reporting success. The call
outcome (`answered`, `duration`, `recording`) is not final at creation time — the Outbound Post Call
webhook fires once the call ends, and Outbound Call Modified fires on any later change.

## Failure modes

| Status | Meaning | What to do |
|---|---|---|
| 400 / 422 | Malformed or invalid parameters | Fix the payload; do not retry unchanged |
| 401 | Missing or wrong key format | Check for `Token token="…"`, not `Bearer` |
| 403 | Key valid, user lacks access to this account | Use a key for a user with access |
| 406 | Parameters not sent as JSON | POST bodies must be JSON, not query string |
| 429 | Outbound ceiling hit (100/hour, 2,000/day) | Stop. No `Retry-After` is sent — back off on your own clock |

## Related

- `asyncapi/callrail-webhooks.yml` — Outbound Post Call and Outbound Call Modified events
- `rate-limits/callrail-rate-limits.yml` — the separate outbound-call ceiling
- `agentic-access/callrail-agentic-access.yml` — recommended execution contract for this operation
