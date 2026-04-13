---
name: bunny-billing
description: This skill should be used when the user asks to "integrate Bunny billing", "add subscription billing", "set up Bunny.com subscriptions", "implement Bunny billing in my app", "connect to Bunny for subscriptions", "add metered billing", "track feature usage with Bunny", "create a billing portal", "handle subscription webhooks from Bunny", or any task involving the Bunny.com subscription billing platform using the bunny-node or bunny-ruby SDKs.
version: 0.1.0
---

# Bunny Subscription Billing Integration

This skill guides implementation of subscription billing integrations with [Bunny.com](https://bunny.com) using the official Node.js (`@bunnyapp/api-client`) or Ruby (`bunny_app`) SDKs. Bunny is a subscription management platform that handles accounts, subscriptions, tenants, metered billing, and customer billing portals.

## Core Concepts

- **Account** — A customer (company or individual) that holds subscriptions.
- **Subscription** — A billing relationship between an account and a price list (plan).
- **Tenant** — A logical workspace/organization within your app, linked to an account and subscription. Used for multi-tenant SaaS apps.
- **Price List** — A plan/pricing tier (e.g., `starter`, `pro`, `enterprise`) configured in the Bunny admin.
- **Feature Usage** — Metered billing events (e.g., API calls, seats) recorded per subscription.
- **Portal Session** — A short-lived token granting access to the Bunny billing self-service portal.

## Authentication

Two approaches are available. Client credentials is preferred — it auto-refreshes tokens.

**Node.js:**
```typescript
import Bunny from "@bunnyapp/api-client";

// Preferred: Client Credentials (auto-refreshes tokens)
const bunny = new Bunny({
  baseUrl: "https://<subdomain>.bunny.com",
  clientId: process.env.BUNNY_CLIENT_ID,
  clientSecret: process.env.BUNNY_CLIENT_SECRET,
  scope: "standard:read standard:write",
});

// Alternative: Static access token
const bunny = new Bunny({
  baseUrl: "https://<subdomain>.bunny.com",
  accessToken: process.env.BUNNY_ACCESS_TOKEN,
});
```

**Ruby:**
```ruby
# config/initializers/bunny_app.rb (Rails)
BunnyApp.config do |c|
  c.client_id     = ENV['BUNNY_CLIENT_ID']
  c.client_secret = ENV['BUNNY_CLIENT_SECRET']
  c.scope         = 'standard:read standard:write'
  c.base_uri      = 'https://<subdomain>.bunny.com'
end
```

Use `bin/rails g bunny_app:install` to generate the Rails initializer template.

Always store credentials in environment variables — never hardcode them.

## Key Integration Patterns

### 1. Creating a Subscription (New Customer)

Called when a new user signs up. Creates the account, subscription, and tenant in one call.

**Node.js:**
```typescript
const subscription = await bunny.subscriptionCreate("starter", {
  accountName: "Acme Corp",
  firstName: "Jane",
  lastName: "Doe",
  email: "jane@acme.com",
  trial: true,
  tenantCode: "acme-team",
  tenantName: "Acme Corp",
});
```

**Ruby:**
```ruby
subscription = BunnyApp::Subscription.create(
  price_list_code: 'starter',
  options: {
    account_name: 'Acme Corp',
    first_name: 'Jane',
    last_name: 'Doe',
    email: 'jane@acme.com',
    trial: true,
    tenant_code: 'acme-team',
    tenant_name: 'Acme Corp'
  }
)
```

### 2. Creating a Subscription (Existing Account)

Used when adding a second subscription to an existing customer.

**Node.js:**
```typescript
const subscription = await bunny.subscriptionCreate("starter", {
  accountId: "account-123",
  tenantCode: "acme-team",
  tenantName: "Acme Corp",
});
```

**Ruby:**
```ruby
subscription = BunnyApp::Subscription.create(
  price_list_code: 'starter',
  options: { account_id: '456', tenant_code: 'acme-team', tenant_name: 'Acme Corp' }
)
```

### 3. Billing Portal (Self-Service)

Generate a portal session token and redirect the user to manage their billing.

**Node.js:**
```typescript
const token = await bunny.portalSessionCreate(
  "acme-team",                          // tenant code
  "https://yourapp.com/billing",        // return URL
  4                                     // expiry hours
);
res.redirect(`https://<subdomain>.bunny.com/portal?token=${token}`);
```

**Ruby:**
```ruby
token = BunnyApp::PortalSession.create(
  tenant_code: 'acme-team',
  return_url: 'https://yourapp.com/billing',
  expiry_hours: 4
)
redirect_to "https://<subdomain>.bunny.com/portal?token=#{token}"
```

### 4. Metered Feature Usage

Record usage events for metered billing (e.g., API calls, storage, seats).

**Node.js:**
```typescript
await bunny.featureUsageCreate(
  "api-calls",           // feature code (configured in Bunny admin)
  100,                   // quantity
  "subscription-123",    // subscription ID
  new Date().toISOString() // optional: defaults to now
);
```

**Ruby:**
```ruby
BunnyApp::FeatureUsage.create(
  quantity: 100,
  feature_code: 'api_calls',
  subscription_id: '456123',
  usage_at: Time.current.iso8601
)
```

### 5. Cancelling a Subscription

**Node.js:**
```typescript
await bunny.subscriptionCancel("subscription-123");
```

**Ruby:**
```ruby
BunnyApp::Subscription.cancel(subscription_id: '456123')
```

### 6. Tenant Metrics (Usage Tracking)

Push engagement/usage metrics for analytics and health scoring.

**Node.js:**
```typescript
await bunny.tenantMetricsUpdate(
  "acme-team",
  new Date(),
  42,                          // primary metric (e.g., active users)
  { activeProjects: 10 }       // additional metrics (custom key-value)
);
```

**Ruby:**
```ruby
BunnyApp::TenantMetrics.update(
  tenant_code: 'acme-team',
  value: 42,
  measured_at: Time.current
)
```

## Webhook Handling

Validate incoming webhooks using the signing token before processing.

**Node.js (Express):**
```typescript
// IMPORTANT: Must use raw body parser for webhooks
app.use("/webhook/bunny", express.raw({ type: "application/json" }));

app.post("/webhook/bunny", (req, res) => {
  const signature = req.headers["x-bunny-signature"] as string;
  const valid = bunny.webhooks.validate(signature, req.body);

  if (!valid) return res.status(401).send("Invalid signature");

  const event = JSON.parse(req.body.toString());
  // Handle event.type: subscription.created, subscription.cancelled, etc.
  res.status(200).send("OK");
});
```

**Ruby (Rails):**
```ruby
# app/controllers/webhooks_controller.rb
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def bunny
    payload = request.body.read
    signature = request.headers['X-Bunny-Signature']

    unless BunnyApp::Webhook.verify(signature, payload, ENV['BUNNY_WEBHOOK_SECRET'])
      return head :unauthorized
    end

    event = JSON.parse(payload)
    # Handle event['type']
    head :ok
  end
end
```

## Error Handling

**Node.js:** All methods throw on failure. Wrap in try/catch:
```typescript
try {
  const subscription = await bunny.subscriptionCreate("starter", options);
} catch (error) {
  // Log error.message; decide whether to retry or surface to user
  throw error;
}
```

**Ruby:** Two exception types:
- `BunnyApp::AuthorizationError` — credential or permission issues
- `BunnyApp::ResponseError` — API rejected the request (validation, not found, etc.)

```ruby
begin
  BunnyApp::Subscription.create(price_list_code: 'starter', options: options)
rescue BunnyApp::AuthorizationError => e
  # Credential/permission problem
rescue BunnyApp::ResponseError => e
  # API-level rejection
end
```

## Environment Setup

Required environment variables:
```
BUNNY_BASE_URL=https://<subdomain>.bunny.com
BUNNY_CLIENT_ID=<your-client-id>
BUNNY_CLIENT_SECRET=<your-client-secret>
BUNNY_WEBHOOK_SECRET=<webhook-signing-token>   # for webhook validation
```

Obtain credentials from the Bunny admin: **Settings > API Clients**.

## Installation

**Node.js:**
```bash
npm install @bunnyapp/api-client
```

**Ruby:**
```bash
bundle add bunny_app
# or: gem install bunny_app
```

## Implementation Checklist

When building a Bunny billing integration:

1. Install the SDK and configure credentials via environment variables
2. Implement `subscriptionCreate` on signup — decide whether to use `trial: true` or go straight to paid
3. Store the returned `subscription.id` and `tenant.code` in your database
4. Add a billing portal link using `portalSessionCreate` (redirect, don't embed)
5. Implement webhook endpoint with signature validation to react to billing events
6. Add `featureUsageCreate` calls at metered usage points
7. Handle `subscriptionCancel` for account deletion / churn flows
8. Test with Bunny's sandbox environment before going live

## Additional Resources

- **`references/node-sdk.md`** — Full Node.js SDK reference with all method signatures and parameters
- **`references/ruby-sdk.md`** — Full Ruby SDK reference with all method signatures and parameters

For raw GraphQL queries (advanced): use `bunny.query<T>(queryString, variables)` in Node.js. All requests go to `POST https://<subdomain>.bunny.com/graphql` with `Authorization: Bearer <token>`.
