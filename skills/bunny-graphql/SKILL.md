---
name: bunny-graphql
description: Direct GraphQL API integration with Bunny — the subscription billing and management platform. Covers the endpoint URL (https://<subdomain>.bunny.com/graphql), Bearer-token authentication, OAuth2 client-credentials flow with automatic token refresh, pagination via Relay-style connections, error response shape, and the most common queries and mutations for accounts, subscriptions, quotes, invoices, and payments. Use when calling Bunny's GraphQL API from a language without an official SDK (Go, Python, Rust, Elixir, PHP, Java, .NET, etc.), when the Node or Ruby SDKs don't cover an operation you need, or when you need to understand the raw request / response shape. For Node prefer the `bunny-node-sdk` skill; for Ruby prefer the `bunny-ruby-sdk` skill.
---

# Bunny GraphQL

Bunny exposes its full API as a single GraphQL endpoint per tenant. This skill
covers calling it directly over HTTPS — auth, pagination, errors, and the
operations integrators reach for most.

For official docs, see
[docs.bunny.com/developer/using-the-graphql-api](https://docs.bunny.com/developer/using-the-graphql-api/api).

## Credential safety

Access tokens grant write access to live billing data. Treat them like database
passwords:

- **Read tokens from environment variables only** (`process.env.BUNNY_ACCESS_TOKEN`,
  `ENV['BUNNY_ACCESS_TOKEN']`, `os.getenv("BUNNY_ACCESS_TOKEN")`). Never inline.
- **Never log request bodies or auth headers** — token values will leak into
  log aggregators otherwise.
- **Never ship tokens to client-side JavaScript bundles** — they belong on your
  server.
- **Don't commit `.env` files.** Use your platform's secret manager in
  production (Doppler, Vault, AWS Secrets Manager, Rails credentials, etc.).
- **Rotate tokens regularly** and scope each token to the minimum set of
  permissions it needs.

## Endpoint

```text
https://<subdomain>.bunny.com/graphql
```

Replace `<subdomain>` with your Bunny tenant's subdomain (the part before
`.bunny.com` in your admin URL).

All requests:

- Use `POST`
- Set `Content-Type: application/json`
- Set `Authorization: Bearer <ACCESS_TOKEN>`

## Authentication

Bunny supports two auth modes. Pick based on how long your process runs.

### Access token (simplest)

Generate a long-lived access token in the admin portal:
**Settings → API Clients → Default API client → Generate Access Token**.

Use it directly in the `Authorization` header. No token exchange needed.

```bash
curl -sS "https://example.bunny.com/graphql" \
  -H "Authorization: Bearer $BUNNY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ currentUser { scopes user { id name email } } }"}'
```

Downside: if the token expires or is rotated, your requests fail until you
update the env var.

### OAuth2 client credentials (recommended for long-running services)

Create an API Client in the admin portal and get a `client_id`, `client_secret`,
and `scope`. Exchange them for a short-lived access token; refresh
automatically when it expires.

```typescript
const {
  BUNNY_BASE_URL,       // e.g. https://example.bunny.com
  BUNNY_CLIENT_ID,
  BUNNY_CLIENT_SECRET,
  BUNNY_SCOPE = "standard:read standard:write",
} = process.env;

let cached: { token: string; expiresAt: number } | null = null;

async function accessToken(): Promise<string> {
  if (cached && Date.now() < cached.expiresAt - 60_000) return cached.token;
  const res = await fetch(`${BUNNY_BASE_URL}/oauth/token`, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "client_credentials",
      client_id: BUNNY_CLIENT_ID!,
      client_secret: BUNNY_CLIENT_SECRET!,
      scope: BUNNY_SCOPE,
    }),
  });
  if (!res.ok) throw new Error(`Token exchange failed: ${res.status}`);
  const body = await res.json();
  cached = {
    token: body.access_token,
    expiresAt: Date.now() + body.expires_in * 1000,
  };
  return cached.token;
}

export async function graphql<T>(query: string, variables?: object): Promise<T> {
  const res = await fetch(`${BUNNY_BASE_URL}/graphql`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${await accessToken()}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ query, variables: variables ?? {} }),
  });
  return res.json();
}
```

Note on units: `body.expires_in` is **seconds** (the TypeScript sample
multiplies by `1000` to reach JS epoch milliseconds). Language implementations
that already work in seconds (Python `time.time()`, Go `time.Duration`) skip
the multiplication.

If the token endpoint returns `{"error":"unauthorized_client","error_description":"The client is not authorized to perform this request using this method."}` with otherwise valid credentials, the issue is server-side: the API client's `grant_flows` array doesn't include `client_credentials`. Enable it under **Settings → API Clients → \<your client\>** in the Bunny admin. The error name is misleading — it sounds like wrong credentials, but it's a client-config mismatch.

The official Node (`@bunnyapp/api-client`) and Ruby (`bunny_app`) SDKs handle
this refresh loop for you — prefer them when available. See also
`bunny-node-sdk` and `bunny-ruby-sdk`.

## Pagination

List queries return Relay-style connections with `edges { node }` and
`pageInfo { endCursor hasNextPage }`. Paginate forward by passing `after`.

```graphql
query ListAccounts($cursor: String) {
  accounts(first: 50, after: $cursor) {
    edges {
      node { id name state createdAt }
    }
    pageInfo { endCursor hasNextPage }
  }
}
```

Loop until `pageInfo.hasNextPage` is false, passing the previous
`endCursor` as the next `cursor`. Keep page sizes ≤ 100 to stay well under
request timeouts.

## Finding the right operation

Bunny's schema has hundreds of operations and the names don't always match
what you'd guess — e.g. quotes are applied via `quoteApplyChanges`, not
`quoteApply`; subscription renewals are driven through a renewal quote
(`quoteSubscriptionRenew`) rather than a direct mutation. **Do not guess
operation names from the user's prompt.**

To find the correct operation:

1. **Use the Bunny dev MCP** — `search_docs("<intent>")` surfaces the right
   operation with a link to its docs page; `validate_graphql("<document>")`
   confirms the shape before you ship it.
2. **Reach for the sibling skill for the domain area** — `bunny-subscriptions`
   for lifecycle operations, `bunny-quoting` for quote flows, `bunny-billing`
   for invoices / payments / credit notes, `bunny-catalog` for products and
   pricing, `bunny-analytics` for reporting reads.
3. **Introspect the schema** directly if you need the exact shape and no doc
   search is available.

### Minimal end-to-end: create an account

Bunny's argument and payload shapes are not the generic `input:
XxxCreateInput!` you may expect from other GraphQL APIs. **Validate your
mutation against the live schema before shipping** — either via the
Bunny dev MCP (`validate_graphql("<document>")`) or a one-off
introspection query. The example below is correct as of the `Last
verified` footer; if it rejects with `Field doesn't accept argument
'attributes'` or similar, the schema has moved.

```typescript
const QUERY = /* GraphQL */ `
  mutation CreateAccount($attributes: AccountAttributes!) {
    accountCreate(attributes: $attributes) {
      account { id name }
      errors
    }
  }
`;

const res = await fetch(`${process.env.BUNNY_BASE_URL}/graphql`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.BUNNY_ACCESS_TOKEN}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    query: QUERY,
    variables: {
      attributes: {
        name: "Acme Corp",
        billingContact: { email: "ap@acme.example" },
      },
    },
  }),
});

const body = await res.json();
if (res.status >= 400) throw new Error(`HTTP ${res.status}: ${JSON.stringify(body)}`);
if (body.errors?.length) throw new Error(body.errors[0].message);  // schema OR validation
const payload = body.data.accountCreate;
if (payload?.errors?.length) throw new Error(payload.errors.join("; "));  // auth
const account = payload.account;
```

Note the three-way error check — see "Error response shape" below for
why validation errors come through top-level `errors`, not the nested
`payload.errors` array.

## Error response shape

Bunny responses mix five observable failure modes. Inspect them in this order:

1. **HTTP status — transport / uncaught exception.** 4xx (commonly 401 on a
   stale token, 403 on OAuth `unauthorized_client`) or 5xx (network, auth
   service, or an uncaught Rails exception). A 500 body from the Rails
   backend is still JSON and includes an `exception` trace — it can look
   deceptively like a successful GraphQL response until you check the
   status. Retry 5xx with backoff; fix 4xx.
2. **Top-level `errors[]` — schema-level.** Unknown field, missing required
   argument, wrong type. Array of GraphQL error objects with `message` and
   `path`. Bug in your query; do not retry.
3. **Top-level `errors[]` with `path = ["<mutationName>"]` and `data.<op>: null` — model validation.**
   The most common mutation failure mode: blank required string, invalid
   email format, illegal state transition. Surface to the caller; do not
   retry.

   ```json
   {
     "errors": [
       { "message": "Name can't be blank",
         "path": ["accountCreate"] }
     ],
     "data": { "accountCreate": null }
   }
   ```

4. **`data.<op>.errors: [String!]` with `data.<op>.<resource>: null` — authorization.**
   Populated only for scope / permission failures, e.g.
   `"Missing required permission: write, can't access mutation PortalSessionCreate"`.
   The message drops the domain prefix (says `write`, not `portal:write`),
   so it can read as ambiguous. Introspect `currentUser { scopes }` to
   confirm what your token actually carries.

   ```json
   { "data": { "portalSessionCreate": {
       "token": null,
       "errors": ["Missing required permission: write, can't access mutation PortalSessionCreate"]
     } } }
   ```

5. **`data.<op>.errors` alongside a populated `data.<op>.<resource>` — partial / warnings.**
   Rare. Some mutations return soft-error arrays on otherwise-successful
   responses. Handle defensively.

Canonical check order in a consumer:

```python
if resp.status_code >= 400:              # 1: transport / exception
    raise TransportError(resp)
if body.get("errors"):                   # 2 or 3: schema or validation
    # path == [<mutationName>] + data.<op> is None => model validation
    raise ApiError(body["errors"])
payload = body["data"]["<op>"]
if payload and payload.get("errors"):    # 4 (or 5 if resource is also set)
    raise AuthError(payload["errors"])
```

Do not parse `message` strings for control flow — they are human-readable
and may change between releases.

## Scopes

Access tokens are scoped. Common scopes:

- `standard:read`, `standard:write` — most day-to-day operations
- `admin:read`, `admin:write` — tenant administration
- `billing:read`, `billing:write` — invoices, payments, credit notes
- `product:read`, `product:write` — catalog management
- `security:read`, `security:write` — API clients, tokens, audit
- `legendary:read`, `legendary:write` — advanced / internal operations

Request the narrowest set you need. Full scope list in the admin portal.

### Confirming a token's scopes at runtime

Query `currentUser { scopes }` to see what scopes the current token
actually carries. Useful when a mutation rejects with a scope error and
the message is ambiguous about which scope is missing (see error tier 4
above — the message strips the domain prefix, saying "Missing required
permission: **write**" rather than "**portal:**write").
Comparing `currentUser.scopes` against the mutation's
`REQUIRED_SCOPES` (visible in the schema's SDL comments) is usually the
fastest path from error to fix.

## References

- [docs.bunny.com/developer/using-the-graphql-api](https://docs.bunny.com/developer/using-the-graphql-api/api) — official docs
- [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) — Node SDK (handles auth + refresh for you)
- [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) — Ruby gem (handles auth + refresh for you)

---

Last verified against bunnyapp/api release 2026-04-21-1
