---
name: bunny-node-sdk
description: Official Node.js / TypeScript SDK for Bunny — `@bunnyapp/api-client`. Covers installation, OAuth2 client-credentials setup (recommended, with automatic token refresh) and access-token setup, helper methods for creating and cancelling subscriptions, managing tenants, updating accounts, generating customer-portal sessions, recording metered feature usage, validating incoming webhooks with the `x-bunny-signature` header, and dropping down to raw GraphQL via `bunny.query()`. Use when integrating Bunny into a Node or TypeScript app (Express, Next.js, NestJS, Koa, serverless handlers, CLIs). For Ruby see `bunny-ruby-sdk`; for raw GraphQL see `bunny-graphql`; for React UI embedding see `bunny-components`.
---

# Bunny Node SDK — `@bunnyapp/api-client`

Official Node / TypeScript client for Bunny. Wraps the GraphQL API, handles
OAuth2 token refresh, and exposes typed helpers for the highest-traffic
operations.

Repo: [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node).
Public docs: [docs.bunny.com/developer/bunny-sdk](https://docs.bunny.com/developer/bunny-sdk/install-the-sdk).

## Credential safety

Load every credential from environment variables — `BUNNY_CLIENT_ID`,
`BUNNY_CLIENT_SECRET`, `BUNNY_ACCESS_TOKEN`, `BUNNY_WEBHOOK_SIGNING_TOKEN`.
Never inline, never log, never commit `.env`. Rotate tokens regularly and
scope each token to the minimum set of permissions it needs.

## Install

```sh
npm install @bunnyapp/api-client --save
```

## Setup

Two auth modes. **Prefer client credentials** for long-running services — the
client refreshes access tokens automatically when they expire.

```typescript
import Bunny from "@bunnyapp/api-client";

const bunny = new Bunny({
  baseUrl: process.env.BUNNY_BASE_URL!,            // https://<subdomain>.bunny.com
  clientId: process.env.BUNNY_CLIENT_ID!,
  clientSecret: process.env.BUNNY_CLIENT_SECRET!,
  scope: "standard:read standard:write security:write",
  webhookSigningToken: process.env.BUNNY_WEBHOOK_SIGNING_TOKEN, // optional
});
```

`security:write` is required for `portalSessionCreate` (gated by Bunny's
`SecurityPolicy`). Drop it if you're not issuing portal-session tokens;
add narrower scopes (`billing:*`, `product:*`, `legendary:*`) as your
integration grows.

Scopes layer a **reach** (`account` < `warren` < `bunny`) with a
**resource:ability** pair. `standard:*` covers the common account-scoped
models (accounts, subscriptions, quotes, invoices, …); other domains
(`workflow:*`, `product:*`, `billing:*`, `legendary:*`) are separate
opt-ins and some mutations require a specific `mutation:<name>` scope.
Expect `Rejected by reach` errors when a helper touches a model your
token's reach doesn't cover. See the `bunny-graphql` skill for the full
scope reference.

For short-lived scripts or sandboxes, an access token is faster to set up:

```typescript
const bunny = new Bunny({
  baseUrl: process.env.BUNNY_BASE_URL!,
  accessToken: process.env.BUNNY_ACCESS_TOKEN!,
});
```

If the access token expires, requests will fail — use client credentials for
anything that outlives a single script run.

## Common operations

### Create a subscription

`subscriptionCreate(priceListCode, options)` — supply either `accountId`
(attach to an existing account) or `accountName` (create the account inline).

```typescript
import { Subscription } from "@bunnyapp/api-client";

const subscription: Subscription = await bunny.subscriptionCreate("starter", {
  accountName: "Acme Corp",
  firstName: "Jane",
  lastName: "Smith",
  email: "jane@acme.example",
  evergreen: true,                 // auto-renew (default: true)
  trial: false,                    // optional trial period
  tenantCode: "acme-team",         // provision a tenant alongside
  tenantName: "Acme Team",
});

console.log(subscription.id, subscription.state, subscription.tenant.id);
```

Passing `tenantCode` requires tenant provisioning to be enabled on the
product's platform, and (if you want webhooks to fire on provisioning
changes) a configured webhook URL + signing key on the platform.
Without those, the call fails with `Tenant provisioning is not enabled`.

Attach to an existing account instead:

```typescript
await bunny.subscriptionCreate("starter", {
  accountId: "account-123",
  firstName: "Jane",
  lastName: "Smith",
  email: "jane@acme.example",
});
```

### Helper catalogue

The SDK also exposes rote CRUD helpers whose method names are their
documentation. Each takes the obvious positional / attribute-bag shape
and returns the created / updated object. See the
[SDK README](https://github.com/bunnyapp/bunny-node#helper-methods) for
argument orders and exact return types.

- `bunny.tenantByCode(code)` / `bunny.tenantCreate(name, code, platformCode, accountId, subscriptionId)` / `bunny.tenantUpdate(id, code?, name?)`
- `bunny.tenantMetricsUpdate(code, lastLogin?, userCount?, utilizationMetrics?)` — health-scoring / churn-signal pushes. `lastLogin` accepts a `Date` or ISO-8601 string; `utilizationMetrics` is a free-form object, e.g. `{ activeProjects: 7, apiCalls: 1234 }`.
- `bunny.accountUpdateByTenantCode(tenantCode, attributes)` — address, tax number, payment terms
- `bunny.portalSessionCreate(tenantCode, returnUrl?, expiryInHours?)` — tokens for the hosted portal UI (see the `bunny-customer-portal` skill for the end-to-end flow)
- `bunny.featureUsageCreate(featureCode, quantity, subscriptionId, usageAt?)` — metered-charge reporting (see `bunny-subscriptions` for lifecycle context)

### Validate an incoming webhook

Bunny signs every outbound webhook with the `x-bunny-signature` header.
**Validate against the raw request body (Buffer), not the parsed JSON** —
re-serialising changes characters like `&`, `<`, `>` and breaks the signature.

```typescript
import express from "express";

// Raw body capture must come before any json() middleware on this route
app.use("/webhook", express.raw({ type: "application/json" }));

app.post("/webhook", (req, res) => {
  const signature = req.headers["x-bunny-signature"] as string;
  const rawBody = req.body as Buffer;

  let valid = false;
  try {
    valid = bunny.webhooks.validate(signature, rawBody);
  } catch {
    // webhooks.validate() throws if the signing token wasn't configured on
    // the client — treat as invalid so misconfiguration surfaces as 401,
    // not a 500.
  }
  if (!valid) return res.status(401).send("Invalid signature");

  const event = JSON.parse(rawBody.toString());
  // Route on event.type ...
  res.status(200).send("OK");
});
```

The signing token is read from the `webhookSigningToken` passed at
construction, or can be supplied per-call as a third argument. See the
`bunny-webhooks` skill for handler patterns (idempotency, retries, event
routing).

## Escape hatch: raw GraphQL

When a helper doesn't cover your operation, use `bunny.query<T>()` directly.
Full TypeScript types are generated from the schema.

```typescript
import { Tenant } from "@bunnyapp/api-client";

const QUERY = /* GraphQL */ `
  query ListTenants($filter: String, $limit: Int) {
    tenants(filter: $filter, limit: $limit) {
      id name code
      platform { id name code }
    }
  }
`;

const res = await bunny.query<{ tenants: Tenant[] }>(QUERY, {
  filter: "",
  limit: 10,
});
const tenants = res.data?.tenants ?? [];
```

For guidance on finding the right operation, error shape, and pagination
patterns, see the `bunny-graphql` skill.

## Error handling

Wrap calls in `try/catch`, but don't assume the rejected value is an
`Error`. Depending on the failure class the SDK may reject with:

- an `Error` (application / validation errors surfaced through the SDK)
- a bare string (OAuth failures — the SDK passes through
  `error_description` verbatim)
- a plain object `{ data, errors }` (GraphQL-level errors from the API)
- `undefined` (transport / network failures where no response body came
  back)

Log the raw rejected value — reading `error.message` alone silently drops
three of the four cases:

```typescript
try {
  await bunny.subscriptionCreate("starter", {
    accountName: "Acme Corp",
    firstName: "John",
    email: "john@acme.example",
  });
} catch (error) {
  console.error("bunny call failed:", error);

  const message =
    error instanceof Error       ? error.message
    : typeof error === "string"  ? error
    : (error as any)?.errors?.[0]?.message
    ?? "unknown error";
  // Don't retry validation-level errors (invalid email, bad state);
  // do retry transient transport failures with backoff.
}
```

Consider wrapping the client with a thin adapter that normalises every
rejection into an `Error` subclass so downstream callers have a single
shape to reason about.

## References

- [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) — SDK source and README
- [docs.bunny.com/developer/bunny-sdk](https://docs.bunny.com/developer/bunny-sdk/install-the-sdk) — official install docs
- Sibling skills: `bunny-graphql` (raw API), `bunny-webhooks` (event handling), `bunny-components` (React UI), `bunny-customer-portal` (hosted UI)

---

Last verified against @bunnyapp/api-client v3.2.1 and bunnyapp/api release 2026-04-21-1
