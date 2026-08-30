---
name: aimlapi-run-a-long-generation
description: >-
  Submit and poll an asynchronous AI/ML API generation — video, music or
  speech-to-text — and cancel a batch you no longer want. Use for any job that
  will not finish inside one HTTP request.
api: AIMLAPI
base_url: https://api.aimlapi.com
operations:
  - _v2_video_generations
  - _v1_stt_create
  - "_v1_stt_:generation_id"
  - _v2_generate_audio
  - _v2_generate_audio_preprocess
  - _v1_batches
  - _v1_batches_cancel_batch_id
generated: '2026-08-30'
method: generated
source: >-
  Grounded in openapi/aimlapi-inference-openapi.yml,
  https://docs.aimlapi.com/capabilities/batch-processing and
  https://docs.aimlapi.com/capabilities/request-tracing-and-cost
---

# Run a long AI/ML API generation

There are no webhooks. AI/ML API documents no callback URL and no event surface
of any kind — the only asynchronous pattern is **submit, then poll**.

## Submit / poll pairs

| Modality | Submit | Poll |
| --- | --- | --- |
| Video | `POST /v2/video/generations` | `GET /v2/video/generations` |
| Speech-to-text | `POST /v1/stt/create` | `GET /v1/stt/{generation_id}` |
| Music / audio | `POST /v2/generate/audio` | `GET /v2/generate/audio` |

Submit returns a `generation_id`. That value **is** the inference id — the same
string the `x-inference-id` response header carries — so the submit, every poll,
the usage-log row and the billing charge all share one identity. No mapping table
is needed.

If a poll comes back with **no** `x-inference-id` header, the id you polled with
was malformed. The request is not failed; the header is simply omitted.

Note that the published contract reuses one `operationId` for both the POST and
the GET on the video and audio paths, so a generated client may collide on the
names. See `overlays/aimlapi-inference-overlay.yaml`.

## Batches

For many independent requests, use the batch surface instead of polling
individually.

```
POST https://api.aimlapi.com/v1/batches
Authorization: Bearer <YOUR_AIMLAPI_KEY>

{"model": "...", "requests": [{"custom_id": "row-1", "params": {...}}]}
```

1 to 100,000 items. Each carries your own `custom_id` and an optional string
`metadata` map.

```
GET  https://api.aimlapi.com/v1/batches?batch_id={batch_id}
POST https://api.aimlapi.com/v1/batches/cancel/{batch_id}
```

The status response carries `processing_status`, `request_counts`
(`processing` / `succeeded` / `errored` / `canceled` / `expired`), `created_at`,
`expires_at`, `ended_at`, `cancel_initiated_at` and `results_url`.

## Cancelling — the only reversal in this API

`POST /v1/batches/cancel/{batch_id}` is the **one** operation in the AI/ML API
surface that undoes something. Everything else is irreversible: a completed
generation cannot be un-generated and its tokens cannot be refunded.

Read `expires_at` on the batch to know how long it has (24 hours after
`created_at` in the documented example). After cancelling, `processing_status`
becomes `canceling` and `cancel_initiated_at` is set. Work already completed is
reported in `request_counts.succeeded` and appears to be billed — no document
states that it is not, so assume it is.

## Timeouts

A synchronous generation that exceeds the time limit returns **504 Gateway
Timeout** ("Generation timeout"). Move long work to the asynchronous surface
rather than retrying — there is no idempotency key, so a retry bills again.
