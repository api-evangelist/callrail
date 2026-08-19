---
name: Build a CallRail call attribution report
description: Resolve the CallRail account, then pull aggregate call counts by marketing source and a call-volume time series for a date range, without downloading individual call records.
api: openapi/callrail-calls-api-openapi.yml
operations:
  - listAccounts
  - getCallSummary
  - getCallTimeseries
generated: '2026-08-14'
method: generated
source: openapi/callrail-calls-api-openapi.yml + https://apidocs.callrail.com/
---

# Build a CallRail call attribution report

Use this when someone asks "how many calls did we get last month, and where did they come from?"
Aggregate endpoints answer this in two requests. Do **not** page through `listCalls` to count calls —
that burns the 1,000/hour request budget and returns data you then have to sum yourself.

## Authentication

Every request needs this exact header. It is not `Bearer`:

```
Authorization: Token token="YOUR_API_KEY"
```

The key is scoped to the CallRail user who created it and returns only what that user can see in the
CallRail interface. Base URL: `https://api.callrail.com/v3`.

## Steps

### 1. Resolve the account — `listAccounts`

```
GET /v3/a.json
```

Returns `accounts[]` with `id` (an `ACC…` identifier), `name`, `outbound_recording_enabled`, and
`hipaa_account`. Use the `id` as `{account_id}` everywhere below. If more than one account comes
back, ask which one before continuing — do not guess.

### 2. Aggregate by marketing dimension — `getCallSummary`

```
GET /v3/a/{account_id}/calls/summary.json?group_by=source&start_date=YYYY-MM-DD&end_date=YYYY-MM-DD
```

`group_by` accepts `source`, `keywords`, `campaign`, `medium`, `landing_page`, `referrer`. Run it
once per dimension the user asked about; each is a separate request.

Attribution semantics matter when you report the numbers: Source, Medium, Campaign, Referrer and
Landing Page reflect the lead's **first-touch** milestone. `gclid`/`fbclid` return the most valuable
associated value even when first touch was a different source. Say "first touch" in the output.

### 3. Plot volume over time — `getCallTimeseries`

```
GET /v3/a/{account_id}/calls/timeseries.json?time_range=day&start_date=YYYY-MM-DD&end_date=YYYY-MM-DD
```

Pass `time_zone` when the user's reporting timezone differs from the company default; CallRail
returns timestamps with an explicit offset.

## Rules

- **25-month retention wall.** Communication Records older than 25 months are deleted. A date range
  that reaches past it — including `all_time` — returns an error telling you to revise the range.
  Clamp the range and say so rather than retrying.
- **Rate limits.** 1,000 requests/hour and 10,000/day per account. Exhaustion returns HTTP 429 with
  no `Retry-After` and no `RateLimit-*` headers, so back off on your own schedule.
- **Read-only.** Nothing in this skill writes. If the user then asks to tag or annotate calls, switch
  to `callrail-triage-calls.md`.
- On 401, the header format is almost always the cause — check for `Token token="…"`.

## Related

- `conventions/callrail-conventions.yml` — pagination, field selection, filtering
- `errors/callrail-problem-types.yml` — status codes and remediation
- `rate-limits/callrail-rate-limits.yml` — published ceilings
