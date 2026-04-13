# Bunny Node.js SDK Reference

Package: `@bunnyapp/api-client`
Repo: https://github.com/bunnyapp/bunny-node

## Client Initialization

```typescript
import Bunny from "@bunnyapp/api-client";

// Client Credentials (recommended — auto token refresh)
const bunny = new Bunny({
  baseUrl: "https://<subdomain>.bunny.com",
  clientId: "<bunny-client-id>",
  clientSecret: "<bunny-client-secret>",
  scope: "standard:read standard:write",
});

// Static Access Token
const bunny = new Bunny({
  baseUrl: "https://<subdomain>.bunny.com",
  accessToken: "<bunny-access-token>",
});

// With Webhook Signing Token
const bunny = new Bunny({
  baseUrl: "https://<subdomain>.bunny.com",
  accessToken: "<bunny-access-token>",
  webhookSigningToken: "<secret-signing-token>",
});
```

## Subscriptions

### `bunny.subscriptionCreate(priceListCode, options)`

Creates a subscription. Provide either `accountId` (existing) or `accountName` (new).

```typescript
// New account
const subscription = await bunny.subscriptionCreate("price-list-code", {
  trial: false,
  evergreen: true,
  accountName: "Acme Corp",
  ownerUserId: "user-123",
  phone: "555-0123",
  fax: "555-0124",
  website: "www.acme.com",
  billingStreet: "123 Main St",
  billingCity: "San Francisco",
  billingState: "CA",
  billingZip: "94105",
  billingCountry: "US",
  firstName: "John",
  lastName: "Doe",
  email: "john@acme.com",
  mobile: "555-0125",
  salutation: "Mr",
  title: "CEO",
  mailingStreet: "456 Market St",
  mailingCity: "San Francisco",
  mailingState: "CA",
  mailingZip: "94105",
  mailingCountry: "US",
  tenantCode: "acme-team",
  tenantName: "Acme Team",
});

// Existing account (minimal)
const subscription = await bunny.subscriptionCreate("price-list-code", {
  accountId: "account-123",
  firstName: "Jane",
  lastName: "Doe",
  email: "jane@acme.com",
});
```

### `bunny.subscriptionCancel(subscriptionId)`

```typescript
const cancelled = await bunny.subscriptionCancel("subscription-123");
```

## Tenants

### `bunny.tenantByCode(code)`

```typescript
const tenant = await bunny.tenantByCode("acme-team");
```

### `bunny.tenantCreate(name, code, platformCode, accountId, subscriptionId)`

```typescript
const tenant = await bunny.tenantCreate(
  "Acme Team",         // name
  "acme-team",         // code
  "platform-code",     // platform code
  "account-123",       // account ID
  "subscription-123"   // subscription ID
);
```

### `bunny.tenantUpdate(tenantId, code, name)`

```typescript
const tenant = await bunny.tenantUpdate("tenant-123", "new-code", "New Name");
```

### `bunny.tenantMetricsUpdate(tenantCode, measuredAt, value, additionalMetrics?)`

```typescript
await bunny.tenantMetricsUpdate(
  "acme-team",
  new Date(),
  42,
  { activeProjects: 10 }
);
```

## Accounts

### `bunny.accountUpdateByTenantCode(tenantCode, fields)`

```typescript
const account = await bunny.accountUpdateByTenantCode("acme-team", {
  name: "Acme Corp",
  billingStreet: "123 Main Street",
  billingCity: "Pleasantville",
  billingState: "CA",
  billingZip: "90210",
  billingCountry: "US",
  phone: "555-0123",
  fax: "555-0124",
  website: "www.acme.com",
  description: "A leading provider of widgets",
  employees: 250,
  annualRevenue: 5000000,
  netPaymentDays: 30,
});
```

## Portal Sessions

### `bunny.portalSessionCreate(tenantCode, returnUrl?, expiryHours?)`

```typescript
// Minimal
const token = await bunny.portalSessionCreate("acme-team");

// With return URL
const token = await bunny.portalSessionCreate(
  "acme-team",
  "https://app.example.com/billing"
);

// With expiry
const token = await bunny.portalSessionCreate(
  "acme-team",
  "https://app.example.com/billing",
  4   // hours
);

res.redirect(`https://<subdomain>.bunny.com/portal?token=${token}`);
```

## Feature Usage (Metered Billing)

### `bunny.featureUsageCreate(featureCode, quantity, subscriptionId, usageAt?)`

```typescript
// Without timestamp (defaults to now)
const usage = await bunny.featureUsageCreate(
  "api-calls",
  100,
  "subscription-123"
);

// With explicit timestamp
const usage = await bunny.featureUsageCreate(
  "api-calls",
  100,
  "subscription-123",
  "2024-03-15T10:00:00Z"
);
```

## Webhooks

### `bunny.webhooks.validate(signature, rawBody, signingToken?)`

```typescript
// Uses signingToken from client config
const valid = bunny.webhooks.validate(signature, rawBody);

// Override signing token
const valid = bunny.webhooks.validate(signature, rawBody, "<signing-token>");
```

Full Express handler:
```typescript
// MUST use raw body parser — JSON parser will break signature validation
app.use("/webhook/bunny", express.raw({ type: "application/json" }));

app.post("/webhook/bunny", (req, res) => {
  const signature = req.headers["x-bunny-signature"] as string;

  if (!bunny.webhooks.validate(signature, req.body)) {
    return res.status(401).send("Invalid signature");
  }

  const event = JSON.parse(req.body.toString());
  console.log("Event type:", event.type);

  res.status(200).send("OK");
});
```

## Raw GraphQL Queries

For operations not covered by helper methods:

```typescript
const query = `
  query tenants($filter: String, $limit: Int) {
    tenants(filter: $filter, limit: $limit) {
      id
      name
      code
      platform {
        id
        name
        code
      }
    }
  }
`;

const response = await bunny.query<{ tenants: Tenant[] }>(query, {
  filter: "",
  limit: 10,
});
```

All requests go to `POST https://<subdomain>.bunny.com/graphql` with `Authorization: Bearer <token>` and `Content-Type: application/json`.
