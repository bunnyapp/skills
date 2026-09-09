# Connecting the Bunny MCP server

The skills in this repo describe how to operate Bunny through its MCP tools. This file covers getting
the tools connected.

## One endpoint, every account

```
https://auth.bunny.com/api/mcp
```

That URL is the same for everyone. It is not per-customer, so nothing in it identifies your Bunny
account — the account is carried by the token you authenticate with, and is fixed at the moment you
consent. This is why `mcp.json` in this repo ships a working URL rather than a placeholder.

The server publishes its metadata at:

```
https://auth.bunny.com/.well-known/oauth-protected-resource
https://auth.bunny.com/.well-known/oauth-authorization-server
```

An unauthenticated request to the endpoint answers `401` with a `WWW-Authenticate` header pointing at
the first of those, which is how a client discovers where to send you to log in.

## Authentication

OAuth 2.1 with the authorization code grant and PKCE.

**Most clients need no setup beyond the URL.** The server supports
[dynamic client registration](https://datatracker.ietf.org/doc/html/rfc7591), so a client that speaks
it enrols itself: it reads `registration_endpoint` from the authorization-server metadata, registers,
and takes you straight to the Bunny login and consent screen. There is no API client to create by
hand, no client ID or secret to copy, and no redirect URI to pre-register.

Registration is open but rate limited, and a newly registered client reaches no data at all until a
user logs in and consents. What you grant at that screen is what the client can do: the token is
scoped to your own permissions, and stamped with the account you consented for.

### If your client does not support dynamic registration

Some hosts still want a client ID and secret typed in. For those:

1. In Bunny, go to **Settings → API Clients** and create a client with the **Authorization Code**
   grant.
2. Set its **redirect URI** to the callback URL of the tool you are connecting from. Each tool has
   its own; it is shown in that tool's connector setup screen. This is the step that most often goes
   wrong — a URI that does not match exactly fails the OAuth flow with `redirect_uri` mismatch, after
   the login screen rather than before it.
3. Copy the **client ID** and **client secret** into the connector's OAuth fields.

Everything after that is discovered automatically.

## Transport

The server speaks **streamable HTTP**: `POST` for JSON-RPC, `GET` to open the server-initiated SSE
stream, `DELETE` to end a session. Clients connect to the URL directly — no local bridge process.

Hosts spell the same thing differently. This repo ships all four:

```jsonc
// mcp.json and .mcp.json — Agent Plugins spec, and Claude Code, which
// accepts "streamable-http" as an alias for its own "http"
{ "mcpServers": { "bunny": {
    "type": "streamable-http",
    "url": "https://auth.bunny.com/api/mcp" } } }

// gemini-extension.json — Gemini CLI: httpUrl is streamable HTTP,
// plain `url` would mean SSE
{ "mcpServers": { "bunny": { "httpUrl": "https://auth.bunny.com/api/mcp" } } }
```

```toml
# .codex/config.toml — Codex picks the HTTP transport from `url` alone
[mcp_servers.bunny]
url = "https://auth.bunny.com/api/mcp"
```

### Clients that only speak stdio

A host with no remote MCP support reaches the server through a bridge:

```json
{
  "mcpServers": {
    "bunny": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://auth.bunny.com/api/mcp"]
    }
  }
}
```

`mcp-remote` handles the OAuth flow, including dynamic registration, and opens a browser for the
consent step. Prefer the direct connection where the host supports it — the bridge is an extra
process to install, run and debug.

## Checking the connection by hand

Two calls confirm a working server. Both need a bearer token and an `Accept` header listing
`text/event-stream` — the transport rejects requests without it.

```bash
BUNNY_MCP=https://auth.bunny.com/api/mcp
TOKEN=…

curl -s -X POST "$BUNNY_MCP" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
        "protocolVersion":"2026-07-28","capabilities":{},
        "clientInfo":{"name":"curl","version":"1.0"}}}'

curl -s -X POST "$BUNNY_MCP" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
```

`initialize` should echo the protocol version and identify `bunny-api`. `tools/list` should return
the full tool set, each with a `title` and an `annotations` object describing whether it reads or
writes.

To confirm dynamic registration is available before wiring anything up:

```bash
curl -s https://auth.bunny.com/.well-known/oauth-authorization-server | grep registration_endpoint
```

No `registration_endpoint` in that response means the client will need a manually created API client
as described above.

## When it does not work

| Response | Cause |
|---|---|
| `401 Unauthorized` | Token missing or expired. The `WWW-Authenticate` header names the metadata document to start the OAuth flow from. |
| `404` on `/oauth/register` | The deployment does not offer dynamic client registration; create an API client by hand. |
| `-32602 Invalid params — Missing or invalid clientInfo` | `clientInfo` needs both `name` and `version`. |
| `406` | The `Accept` header must list both `application/json` and `text/event-stream`. |
| Connects, but reports no tools | The client completed the handshake and then hit an error opening the event stream. Check that nothing between you and Bunny (a proxy, a gateway) rejects `GET` on the endpoint — the stream is opened with `GET`, not `POST`. |
| `308 Permanent Redirect` | HTTP was used; the endpoint is HTTPS only. |
