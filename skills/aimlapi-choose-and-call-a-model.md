---
name: aimlapi-choose-and-call-a-model
description: >-
  Pick a model from the live AI/ML API catalogue and call it, without hardcoding
  a model id that may already have been withdrawn. Use when a task needs an LLM,
  an embedding, an image, a transcription or a generation and the specific model
  is not fixed in advance.
api: AIMLAPI
base_url: https://api.aimlapi.com
operations:
  - GET /v1/models
  - GET /v1/models/deprecations
  - _v1_chat_completions
  - _v1_embeddings
  - _v1_images_generations
generated: '2026-08-30'
method: generated
source: >-
  Grounded in openapi/aimlapi-inference-openapi.yml and
  https://docs.aimlapi.com/api-references/service-endpoints/complete-model-list
---

# Choose and call an AI/ML API model

AI/ML API resells 1000+ third-party models. Models are added and withdrawn
constantly — 132 entries sat in the deprecation feed on 2026-08-30 — so a model
id baked into your code is a future 404. Resolve the model at run time.

## 1. Read the live catalogue

```
GET https://api.aimlapi.com/v1/models
```

No API key is required. The response is `{"object":"list","data":[...]}` with one
entry per model: `id` (`source/alias`, e.g. `openai/gpt-5`), `info.name`,
`info.developer`, `info.description`, `info.releasedAt` and context length.
Pricing, modalities and capabilities are opt-in sections, so ask for them when
you need to compare cost.

`/v1/models` is the canonical path. `/models` and `/api/v1/models` serve the
same data.

## 2. Check the model has not been retired

```
GET https://api.aimlapi.com/v1/models/deprecations
```

Also keyless. Match your candidate against BOTH `id` and every entry in
`aliases[]` — a model may have been reachable under several public ids. Then read
`status`:

- `deprecated` — still serving, retirement announced. Plan a move; check `shutdown_at`.
- `superseded` — folded into another model; the id still resolves. Move to `replaced_by`.
- `withdrawn` — gone. Requests return 404. Move to `replaced_by`, or pick again from step 1.

The feed accepts `?status=` and `?since=YYYY-MM-DD` filters and returns a stable
weak ETag (`generated_at` is deliberately excluded from it), so poll it
conditionally rather than re-downloading.

Do NOT infer retirement from a model's absence in `/v1/models`. The catalogue
returns the same "not here" for a withdrawn model, a superseded model, and a
model that never existed. This feed is the only thing that tells them apart.

## 3. Call it

```
POST https://api.aimlapi.com/v1/chat/completions
Authorization: Bearer <YOUR_AIMLAPI_KEY>
Content-Type: application/json
X-Client-Request-Id: <your-correlation-id>

{"model": "openai/gpt-5", "messages": [{"role": "user", "content": "..."}]}
```

The surface is OpenAI-compatible — an OpenAI SDK configured with
`base_url = https://api.aimlapi.com/v1` reaches this endpoint unchanged. Video
and speech models are NOT reachable through the OpenAI SDK; call those with
plain REST.

Other modalities, same auth: `POST /v1/embeddings`, `POST /v1/images/generations`,
`POST /v1/images/edits`, `POST /v1/ocr`, `POST /v1/tts`.

## Conventions that will bite you

- `X-Client-Request-Id` must be 1-128 characters from `A-Z a-z 0-9 . _ : -`.
  Anything else is **silently dropped** — the call still runs and is still
  billed, and nothing in the response tells you. Sanitise ids built from user
  input.
- There is **no idempotency key**. Retrying a timed-out completion bills twice.
  See `conventions/aimlapi-conventions.yml`.
- There is **no reversal**. A completed generation cannot be refunded. Cap the
  key in dollars instead — see `aimlapi-run-an-agent-safely`.
- Set `provider` to pin one upstream source with no fallback (`openai`, `google`,
  `xai`, `alibaba`, `minimax`, `moonshot`, `baidu`, `togetherai`, `openrouter`).
  The default `auto` uses the full fallback chain, so the model may not run where
  you expect.
