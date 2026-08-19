---
name: Connect to the Syncly Social MCP server
description: >-
  Authenticate against Syncly's OAuth 2.1 authorization server and open an MCP session to the
  Syncly Social MCP server, so an agent can query TikTok, Reels and Shorts social intelligence
  from a connected Syncly workspace.
api: openapi/syncly-social-mcp-openapi.json
mcp: https://mcp.syncly.app/mcp
operations:
  - oauth_protected_resource__well_known_oauth_protected_resource_mcp_get
  - oauth_protected_resource__well_known_oauth_protected_resource_get
  - health_check_health_get
generated: '2026-08-13'
method: generated
source: >-
  Grounded in openapi/syncly-social-mcp-openapi.json (operationIds verified verbatim),
  well-known/syncly-oauth-authorization-server.json, well-known/syncly-oauth-protected-resource.json,
  and the live 401 challenge from https://mcp.syncly.app/mcp.
---

# Connect to the Syncly Social MCP server

Syncly has no REST API. Everything an agent can do with Syncly data goes through one hosted
remote MCP server. This skill gets you a session; what you can then call is decided by the
workspace the token belongs to.

## What you are connecting to

| | |
|---|---|
| MCP endpoint | `https://mcp.syncly.app/mcp` |
| Transport | MCP Streamable HTTP, JSON-RPC 2.0 over POST |
| Authorization server | `https://social-server.syncly.app` |
| Scopes | `syncly:read`, `offline_access` |
| Server version | `0.1.0` |

## Step 1 — Confirm the server is up

`GET https://mcp.syncly.app/health` (`health_check_health_get`). A healthy server answers
`{"status":"ok"}`. This endpoint needs no token.

## Step 2 — Discover the authorization server

`GET https://mcp.syncly.app/.well-known/oauth-protected-resource/mcp`
(`oauth_protected_resource__well_known_oauth_protected_resource_mcp_get`).

Do not hardcode the issuer — read `authorization_servers[0]` from the response. The root path
`/.well-known/oauth-protected-resource`
(`oauth_protected_resource__well_known_oauth_protected_resource_get`) returns the same document.

Then `GET {issuer}/.well-known/oauth-authorization-server` for the RFC 8414 metadata:
`authorization_endpoint`, `token_endpoint`, `registration_endpoint`, `revocation_endpoint`.

## Step 3 — Register the client

Syncly supports dynamic client registration (RFC 7591). POST your client metadata to
`registration_endpoint`. `client_id_metadata_document_supported` is `true`, so a client ID
metadata document is also accepted. Do not send a client secret — this is a public client
(`token_endpoint_auth_methods_supported` is `["none"]`).

## Step 4 — Authorization code flow with PKCE

PKCE is required and only `S256` is supported. Request `scope=syncly:read offline_access`;
`offline_access` is what gets you a refresh token. Exchange the code at `token_endpoint`.

The end-user side of this is a provider-initiated connect flow: a Syncly user goes to
**Settings → My Account → Connected Apps**, picks their AI client, and approves. They revoke
the same way, or you can call `revocation_endpoint` directly.

## Step 5 — Open the MCP session

POST to `https://mcp.syncly.app/mcp` with:

- `Authorization: Bearer <access_token>`
- `Accept: application/json, text/event-stream`
- `Content-Type: application/json`

Send `initialize`, then `tools/list` to get the real tool names and their `inputSchema`.

**Do not assume tool names.** Syncly does not publish them. Its product page advertises six
capabilities in prose — monitor trends, compare competitors, discover creators, mine winning
content, search every conversation, report in seconds — but those are marketing labels, not
identifiers. Always read the wire-level names from `tools/list` at runtime.

## Handling failure

Every method returns HTTP 401 with a JSON-RPC error when the token is missing or invalid:

```json
{"jsonrpc":"2.0","error":{"code":-32001,"message":"Unauthorized",
 "data":{"_meta":{"mcp/www_authenticate":["Bearer resource_metadata=\"https://mcp.syncly.app/.well-known/oauth-protected-resource/mcp\", scope=\"syncly:read\", error=\"invalid_token\", error_description=\"Authentication is required\""]}}},"id":null}
```

The Bearer challenge is inside `error.data._meta`, **not** in a `WWW-Authenticate` response
header. Parse it there, restart discovery from the `resource_metadata` URL it names, and retry.

## What this API will not do for you

- **Nothing is idempotent-keyed.** Syncly documents no idempotency header. The sole resource
  scope is `syncly:read`, so calls should be safe to retry, but there is no written contract
  saying so — treat any future write tool as unsafe to blind-retry.
- **No rate limits are published.** No `RateLimit-*` or `Retry-After` header was observed.
  Back off on your own schedule; you will not be told what the ceiling is.
- **No status page, no changelog, no deprecation policy.** The server version is `0.1.0` and
  nothing announces breaking changes. Re-read `tools/list` at the start of every session rather
  than caching a tool set across days.
