---
name: bunny-ruby-sdk
description: Official Ruby gem for Bunny — `bunny_app`. Covers installation, Rails initializer generator (`bin/rails g bunny_app:install`), OAuth2 client-credentials configuration (recommended, with automatic token refresh) and access-token configuration, module methods for creating and cancelling subscriptions, updating subscription quantities, converting trials, managing tenants and platforms, pushing tenant metrics, recording feature usage, generating customer-portal sessions, validating incoming webhooks with the `x-bunny-signature` header, and dropping down to raw GraphQL via `BunnyApp.query` / `BunnyApp.query_async`. Use when integrating Bunny into a Ruby or Rails app. For Node / TypeScript see `bunny-node-sdk`; for raw GraphQL see `bunny-graphql`.
---

# Bunny Ruby SDK — `bunny_app` gem

Official Ruby client for Bunny. Wraps the GraphQL API, handles OAuth2 token
refresh, and exposes module-level helpers for the highest-traffic operations.
Rails-friendly with a generator for the config initializer.

Repo: [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby).
Public docs: [docs.bunny.com/developer/bunny-sdk](https://docs.bunny.com/developer/bunny-sdk/install-the-sdk).

## Credential safety

Load every credential from environment variables or a secret store
(Doppler, Vault, Rails credentials). Never inline, never log, never commit
`.env`. Rotate tokens regularly and scope each token to the minimum set of
permissions it needs.

## Install

```ruby
# Gemfile
gem 'bunny_app'
```

```sh
bundle install
# or
gem install bunny_app
```

Requires Ruby 3.1+.

## Configure (Rails)

Generate an initializer:

```sh
bin/rails g bunny_app:install
```

This creates `config/initializers/bunny_app.rb` reading credentials from
env vars with the `BUNNY_APP_` prefix (matching the generator template).
Prefer **client credentials** so access tokens refresh automatically when
they expire:

```ruby
BunnyApp.config do |c|
  c.client_id     = ENV['BUNNY_APP_CLIENT_ID']
  c.client_secret = ENV['BUNNY_APP_CLIENT_SECRET']
  c.scope         = ENV.fetch('BUNNY_APP_SCOPE', 'standard:read standard:write security:write')
  c.base_uri      = ENV['BUNNY_APP_BASE_URL']     # https://<subdomain>.bunny.com
end
```

`security:write` is required for `portalSessionCreate` (gated by Bunny's
`SecurityPolicy`). Drop it if you're not issuing portal-session tokens;
add narrower scopes (`billing:*`, `product:*`, `legendary:*`) as your
integration grows.

### Access-token fallback

If you manage the token lifecycle yourself, pass it directly.
`BunnyApp::AuthorizationError` is raised when the token expires.

```ruby
BunnyApp.config do |c|
  c.access_token = ENV['BUNNY_APP_ACCESS_TOKEN']
  c.base_uri     = ENV['BUNNY_APP_BASE_URL']
end
```

## Common operations

### Create a subscription

```ruby
subscription = BunnyApp::Subscription.create(
  price_list_code: 'starter',
  options: {
    account_name: 'Acme Corp',
    first_name:   'Jane',
    last_name:    'Smith',
    email:        'jane@acme.example',
    trial:        true,
    tenant_code:  'acme-team',
    tenant_name:  'Acme Team',
  }
)
# subscription['id'], subscription['state'], subscription['tenant'], ...
```

Attach to an existing account by `account_id` instead of supplying `account_name`.

A few preconditions catch first-time integrators off-guard:

- **Booleans in `options:` must be real booleans.** The SDK forwards
  values to GraphQL untouched, and GraphQL rejects string `"true"` /
  `"false"` with `Could not coerce value "false" to Boolean`. If you're
  wiring a signup controller that forwards request params, coerce
  first (e.g. `ActiveModel::Type::Boolean.new.cast(value)`).
- **`trial: true` is not universally valid.** Price lists can disable
  trials; expect `BunnyApp::ResponseError` with `Trial is not allowed on
  this price list` and fall back to non-trial creation if the caller
  isn't sure of the price list's policy.
- **Price list codes are typically known out-of-band.** Listing queries
  (`priceLists`) may return empty under `standard:read` due to view
  gating; direct lookup by `price_list_code` in `Subscription.create`
  works regardless. Wire the codes into config rather than relying on
  a list fetch to discover them.

### Update subscription quantities

```ruby
quote = BunnyApp::Subscription.quantity_update(
  subscription_id: '456123',
  quantities:      [{ code: 'users', quantity: 25 }],
  options: {
    invoice_immediately:           true,
    start_date:                    '2026-06-01',
    name:                          'Add users — June',
    allow_quantity_limits_override: false,
  }
)
# Returns a quote hash; the actual charge change happens when the quote is applied.
```

### Convert a trial to paid

```ruby
# By price list code
BunnyApp::Subscription.trial_convert(
  subscription_id: '456123',
  price_list_code: 'starter',
)

# Or by price list ID + a stored payment method
BunnyApp::Subscription.trial_convert(
  subscription_id: '456123',
  price_list_id:   '789',
  payment_id:      '101112',
)
# → { 'invoice' => { 'id' =>…, 'number' =>… }, 'subscription' => { 'id' =>… } }
```

### Helper catalogue

The gem also exposes rote CRUD helpers whose method names are their
documentation. Each takes the obvious keyword-argument shape and returns
a hash (or `true` on success). See the
[gem README](https://github.com/bunnyapp/bunny-ruby#readme) for return
shapes.

- `BunnyApp::Subscription.cancel(subscription_id:)` — returns `true`
- `BunnyApp::Platform.create(name:, code:)` / `BunnyApp::Tenant.create(name:, code:, account_id:, platform_code:)` / `BunnyApp::Tenant.find_by(code:)`
- `BunnyApp::TenantMetrics.update(code:, last_login:, user_count:, utilization_metrics:)` — health-scoring / churn-signal pushes. `last_login` is an ISO-8601 string; `utilization_metrics` is a free-form hash, e.g. `{ projects_created: 10, exports_run: 3 }`.
- `BunnyApp::PortalSession.create(tenant_code:, return_url:, expiry_hours:)` — tokens for the hosted portal UI (see the `bunny-customer-portal` skill for the end-to-end flow)
- `BunnyApp::FeatureUsage.create(quantity:, feature_code:, subscription_id:, usage_at:)` — metered-charge reporting (see `bunny-subscriptions` for lifecycle context)

### Validate an incoming webhook

Bunny signs every outbound webhook with the `x-bunny-signature` header.
**Verify against the raw request body**, not a re-serialised JSON string.

```ruby
# app/controllers/bunny_webhooks_controller.rb
class BunnyWebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def create
    payload     = request.raw_post
    signature   = request.headers['x-bunny-signature']
    signing_key = ENV.fetch('BUNNY_APP_WEBHOOK_SIGNING_TOKEN')

    # Guard the nil case: Webhook.verify(nil, …) raises TypeError because
    # OpenSSL::HMAC rejects a nil signature. Surface a missing header as 401.
    unless signature.present? && BunnyApp::Webhook.verify(signature, payload, signing_key)
      return head :unauthorized
    end

    event = JSON.parse(payload)
    case event['type']
    when 'SubscriptionProvisioningChange'
      # provision / deprovision tenant ...
    end
    head :ok
  end
end
```

Signatures are computed as `OpenSSL::HMAC.hexdigest('sha1', signing_key,
raw_body)`. Useful for writing tests or simulating webhooks locally with
a signed curl — `BunnyApp::Webhook.verify` does the same comparison.

See the `bunny-webhooks` skill for event-type routing, idempotency patterns,
and retry behaviour.

## Escape hatch: raw GraphQL

When a helper doesn't cover your operation, call the API directly.

```ruby
QUERY = <<~GRAPHQL
  query GetTenant($code: String!) {
    tenant(code: $code) {
      id name
      account { id name }
    }
  }
GRAPHQL

response = BunnyApp.query(QUERY, { code: 'acme-team' })
tenant   = response['data']['tenant']
```

`BunnyApp.query` takes two positional arguments; pass `{}` as the second
when the query has no variables (`BunnyApp.query('{ viewer { id } }')`
alone raises `ArgumentError`).

Mutations follow the same pattern — pass variables, check the payload's
`errors` field. Bunny mutations generally take an `id:` (or attribute
key) plus an `attributes:` input object and return `{ <resource>, errors }`:

```ruby
MUTATION = <<~GRAPHQL
  mutation UpdateAccount($id: ID!, $attributes: AccountAttributes!) {
    accountUpdate(id: $id, attributes: $attributes) {
      account { id name }
      errors
    }
  }
GRAPHQL

response = BunnyApp.query(MUTATION, { id: account_id, attributes: { billingCity: 'SF' } })
payload  = response.dig('data', 'accountUpdate')
raise BunnyApp::ResponseError, payload['errors'].join(', ') if payload&.dig('errors')&.any?
```

Model-validation errors may surface at the top level (`response['errors']`
with `path: [<mutationName>]` and `data.<op> = nil`) rather than inside
the payload — see `bunny-graphql` for the full error envelope.

### Fire-and-forget in a background thread

For non-critical tracking calls where you don't want to block the request
cycle (e.g. pushing tenant metrics from a web action):

```ruby
BunnyApp.query_async(QUERY, variables)
```

Prefer a proper job queue (Sidekiq, GoodJob) for anything that must succeed.

For guidance on finding the right operation, error shape, and pagination
patterns, see the `bunny-graphql` skill.

## Error handling

| Exception | When raised |
| --- | --- |
| `BunnyApp::AuthorizationError` | Invalid or expired credentials |
| `BunnyApp::ResponseError` | API returned errors in the response body |

```ruby
begin
  BunnyApp::Subscription.cancel(subscription_id: '456123')
rescue BunnyApp::AuthorizationError => e
  Rails.logger.error("Bunny auth failed: #{e.message}")
  # re-authenticate or alert
rescue BunnyApp::ResponseError => e
  Rails.logger.error("Bunny error: #{e.message}")
  # surface to caller; don't retry validation-level errors
end
```

## References

- [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) — gem source and README
- [docs.bunny.com/developer/bunny-sdk](https://docs.bunny.com/developer/bunny-sdk/install-the-sdk) — official install docs
- Sibling skills: `bunny-graphql` (raw API), `bunny-webhooks` (event handling), `bunny-customer-portal` (hosted UI)

---

Last verified against bunny_app v2.4.0 and bunnyapp/api release 2026-04-21-1
