---
name: aimlapi-run-an-agent-safely
description: >-
  Provision an AI/ML API credential for an autonomous agent with the narrowest
  possible blast radius — scoped to the model categories it needs and capped in
  dollars — and connect it over the remote MCP server. Use before handing any
  agent an AIMLAPI key.
api: AIMLAPI
base_url: https://api.aimlapi.com
operations:
  - KeysController_createKey_v1
  - KeysController_listKeys_v1
  - PATCH /v1/keys/{prefix}
  - DELETE /v1/keys/{prefix}
  - GET /v2/usage
generated: '2026-08-30'
method: generated
source: >-
  Grounded in
  https://docs.aimlapi.com/api-references/service-endpoints/api-key-management,
  https://docs.aimlapi.com/faq/how-can-i-work-with-my-api-keys and
  https://docs.aimlapi.com/quickstart/mcp
---

# Run an agent against AI/ML API safely

Every AI/ML API call spends real money and **nothing can be refunded**. There is
no dry-run mode, no idempotency key, and one reversal operation in the entire
surface (cancelling a batch). The control that matters is therefore the ceiling,
not the undo.

## 1. Mint a scoped, capped key

```
POST https://api.aimlapi.com/v1/keys
Authorization: Bearer <YOUR_MANAGEMENT_KEY>

{
  "name": "nightly-summariser",
  "scopes": ["model:chat", "model:embeddings"],
  "limit": { "retention": "day", "threshold": 5 }
}
```

- **`scopes`** restricts which model categories the key may call:
  `model:chat`, `model:responses`, `model:image`, `model:audio`, `model:video`,
  `model:embeddings`, `model:speech`, `model:ocr`. The field is *nullable*, and a
  key created without it is unrestricted — so always pass it explicitly.
- **`limit`** caps spend in USD. `retention` is `no_reset`, `day`, `week` or
  `month`; periodic limits reset at **00:00 UTC**. Once the threshold is reached,
  charges on that key are blocked.

Copy the `key` from the response immediately — it is returned only at creation.
The first 8 characters are its `prefix`, which is what you use everywhere else.

Creating keys requires a **management key**, which can only be made in the
dashboard and never through the API. Never give an agent the management key: it
mints credentials.

## 2. Watch it

```
GET https://api.aimlapi.com/v2/usage?period=24h&key_prefix={prefix}
GET https://api.aimlapi.com/v2/logs?period=24h&key_prefix={prefix}
```

A regular key naming *another* key's prefix gets a 403; listing a different key's
activity needs the management key.

## 3. Revoke it

```
PATCH  https://api.aimlapi.com/v1/keys/{prefix}    # disable / update limits
DELETE https://api.aimlapi.com/v1/keys/{prefix}    # remove
```

Both take effect immediately and have no window.

## Connecting over MCP instead

AI/ML API runs a first-party remote MCP server:

```
https://mcp.aimlapi.com/mcp
```

Streamable HTTP with OAuth 2.1 + PKCE and dynamic client registration — the
client discovers the sign-in flow from
`https://mcp.aimlapi.com/.well-known/oauth-protected-resource`, so there is no
key to paste. Adding it to Claude Code is:

```bash
claude mcp add --transport http --scope user aimlapi https://mcp.aimlapi.com/mcp
```

then `/mcp` → `aimlapi` → Authenticate.

**Understand what you are granting.** The OAuth scopes are `openid`, `email`,
`offline_access` and `mcp:invoke` — and `mcp:invoke` is a single coarse scope
covering catalogue browsing, inference across every modality, job management and
account balance. There is no read-only MCP scope. An agent granted `mcp:invoke`
can spend money, and `offline_access` lets it keep doing so unattended.

If you need the model-category restriction and the dollar cap, use a scoped API
key against the REST API rather than OAuth over MCP — the API-key permission
model is strictly finer-grained than the OAuth one.

Removing a connector in the client does not always revoke the token. Revoke the
app under **Authorized apps / Connections** in the AI/ML API account as well.
