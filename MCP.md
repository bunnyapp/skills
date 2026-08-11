# Connecting the Bunny MCP server

The skills in this repo describe how to operate Bunny through its MCP tools. This file covers getting
the tools connected.

## The endpoint is per account

```
https://<your-subdomain>.bunny.com/api/mcp
```

There is no single shared host — each Bunny account answers on its own subdomain, so the URL is the
one you already use to sign in. A request to the wrong subdomain will not authenticate even with a
valid token.

**This is why `mcp.json` in this repo cannot ship a working URL.** The value there is a placeholder;
replace `YOUR-SUBDOMAIN` with your own, or connect through ChatGPT where the URL is entered per
install.

Authentication is OAuth 2.1. The server publishes its metadata at:

```
https://<your-subdomain>.bunny.com/api/mcp/.well-known/oauth-protected-resource
```

## Register the client first

Bunny does not support dynamic client registration, so a client cannot enrol itself. Create the
credentials before connecting:

1. In Bunny, go to **Settings → API Clients** and create a client with the **Authorization Code**
   grant.
2. Set its **redirect URI** to the callback URL of the tool you are connecting from. Each tool has
   its own; it is shown in that tool's connector setup screen. This is the step that most often goes
   wrong — a URI that does not match exactly fails the OAuth flow with `redirect_uri` mismatch, after
   the login screen rather than before it.
3. Copy the **client ID** and **client secret** into the connector's OAuth fields.

Everything after that is discovered automatically: the endpoint returns a `WWW-Authenticate` header
pointing at its metadata, and the client follows it to the authorize and token endpoints.

## Connecting from ChatGPT

Add a connector pointing at your subdomain's endpoint and complete the OAuth flow. Since the URL
differs per customer, this is the route for normal use.

## Connecting a local MCP client

Remote MCP servers are reached through a stdio bridge:

```json
{
  "mcpServers": {
    "bunny": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://your-subdomain.bunny.com/api/mcp"]
    }
  }
}
```

## Checking the connection by hand

Two calls confirm a working server. Both need a bearer token and an `Accept` header listing
`text/event-stream` — the transport rejects requests without it.

```bash
BUNNY_MCP=https://your-subdomain.bunny.com/api/mcp
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

## When it does not work

| Response | Cause |
|---|---|
| `401 Unauthorized` | Token missing or expired, or issued for a different Bunny account than the subdomain in the URL. |
| `403 Forbidden: Invalid Host header` | The URL host does not match the account's own subdomain. |
| `-32602 Invalid params — Missing or invalid clientInfo` | `clientInfo` needs both `name` and `version`. |
| `406` | The `Accept` header must list both `application/json` and `text/event-stream`. |
| Connects, but reports no tools | The client completed the handshake and then hit an error opening the event stream. Check that nothing between you and Bunny (a proxy, a gateway) rejects `GET` on the endpoint — the stream is opened with `GET`, not `POST`. |
| `308 Permanent Redirect` | HTTP was used; the endpoint is HTTPS only. |
