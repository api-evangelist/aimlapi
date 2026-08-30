---
name: aimlapi-track-what-a-call-cost
description: >-
  Read the exact cost of an AI/ML API request from the response itself, and
  reconcile it later against the account ledger. Use whenever an agent's spend
  must be attributed to a task, an order, or a customer.
api: AIMLAPI
base_url: https://api.aimlapi.com
operations:
  - _v1_chat_completions
  - _v2_logs
  - GET /v2/usage
  - GET /v2/billing
  - GET /v2/billing/transactions
generated: '2026-08-30'
method: generated
source: >-
  Grounded in https://docs.aimlapi.com/capabilities/request-tracing-and-cost and
  https://docs.aimlapi.com/api-references/service-endpoints/usage-logs
---

# Track what an AI/ML API call cost

AI/ML API returns the price of a call **on the response to that call**. You do
not have to poll a reporting endpoint to learn what a step cost, which is what
makes per-task budgeting possible for an autonomous agent.

## 1. Tag the request with your own id

```
X-Client-Request-Id: order-8f21c4
```

1-128 characters from `A-Z a-z 0-9 . _ : -`. It is stored, echoed back, and
reported as `client_request_id` in the usage logs. A value outside that alphabet
is silently dropped and `client_request_id` stays `null` — with no error.

## 2. Read the cost off the response

| Header | When |
| --- | --- |
| `x-inference-id` | always |
| `x-client-request-id` | when you sent a valid one |
| `x-aimlapi-credits-used` | non-streaming JSON responses |
| `x-aimlapi-usd-spent` | non-streaming JSON responses |

All four are in `Access-Control-Expose-Headers`, so browser code can read them
cross-origin.

Two cases carry no cost header:

- **Streaming (SSE).** Headers flush before the first token. The total arrives in
  the final chunk under `meta.usage` as `{credits_used, usd_spent}`.
- **Binary audio (wav).** The cost only exists after the audio. Read it from the
  usage logs instead.

The correlation headers are present in both cases; only the cost moves.

## 3. Reconcile

```
GET https://api.aimlapi.com/v2/logs?period=24h&status=succeeded&status=failed
Authorization: Bearer <YOUR_AIMLAPI_KEY>
```

One row per request: `created`, `status`, `origin`, `model`, `cost.usd`,
`cost.credits`, `tokens.*`, `request_id`, `inference_id`, `client_request_id`.
Paginate with `limit` (1-100, default 50) and `offset`; the `pagination` object
returns `total` and `has_more`.

Filters do **not** split on commas. Repeat the parameter instead:
`?status=succeeded&status=failed`. `?model=a,b` is read as one model literally
named `a,b` and silently matches nothing.

Windows: either `period` (`24h`, `7d`) or `start`+`end`, never both, and never
more than 92 days.

Rate limit: **200 requests per 60 seconds**, returning 429. This is the only
published numeric rate limit in the whole API, and there is no `Retry-After`
header — back off on your own schedule.

## 4. Join to the ledger

`x-inference-id` is the join key across the platform:

- it is `inference_id` in `GET /v2/logs`
- it is `reference_id` on the matching `MODEL_USAGE` charge in `GET /v2/billing/transactions`
- for an asynchronous generation it is the same value `submit` returned as `generation_id`

`GET /v2/usage` gives totals for a window; `GET /v2/logs` gives the individual
rows including failures. A failed request is billed nothing — the hold is rolled
back — so it appears in the logs with zero cost and never in the usage totals.
That is why a log row count can legitimately exceed the `requests` figure for the
same window.

`GET /v2/billing` returns the current balance.
