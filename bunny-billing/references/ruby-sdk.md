# Bunny Ruby SDK Reference

Gem: `bunny_app`
Repo: https://github.com/bunnyapp/bunny-ruby
Requirements: Ruby 2.5+

## Installation

```bash
# Gemfile
gem 'bunny_app'

# Or
gem install bunny_app
```

## Configuration

```ruby
# config/initializers/bunny_app.rb

# Client Credentials (recommended — auto token refresh)
BunnyApp.config do |c|
  c.client_id     = ENV['BUNNY_CLIENT_ID']
  c.client_secret = ENV['BUNNY_CLIENT_SECRET']
  c.scope         = 'standard:read standard:write'
  c.base_uri      = 'https://<subdomain>.bunny.com'
end

# Static Access Token
BunnyApp.config do |c|
  c.access_token = ENV['BUNNY_ACCESS_TOKEN']
  c.base_uri     = 'https://<subdomain>.bunny.com'
end
```

Rails users can generate a config template:
```bash
bin/rails g bunny_app:install
```

## Subscriptions

### `BunnyApp::Subscription.create(price_list_code:, options:)`

Create with a new account:
```ruby
subscription = BunnyApp::Subscription.create(
  price_list_code: 'starter',
  options: {
    account_name: 'Acme Corp',
    first_name: 'Jane',
    last_name: 'Smith',
    email: 'jane@acme.com',
    trial: true,
    tenant_code: 'acme-123',
    tenant_name: 'Acme Corp'
  }
)
```

Create against an existing account:
```ruby
subscription = BunnyApp::Subscription.create(
  price_list_code: 'starter',
  options: {
    account_id: '456',
    tenant_code: 'acme-123',
    tenant_name: 'Acme Corp'
  }
)
```

### `BunnyApp::Subscription.quantity_update(subscription_id:, quantities:, options:)`

Adjust seat/quantity charges, with optional immediate invoicing:
```ruby
quote = BunnyApp::Subscription.quantity_update(
  subscription_id: '456123',
  quantities: [{ code: 'users', quantity: 25 }],
  options: {
    invoice_immediately: true,
    start_date: '2024-06-01',
    name: 'Add users — June',
    allow_quantity_limits_override: false
  }
)
```

### `BunnyApp::Subscription.trial_convert(subscription_id:, price_list_code:)`

Convert a trial subscription to a paid plan:
```ruby
result = BunnyApp::Subscription.trial_convert(
  subscription_id: '456123',
  price_list_code: 'starter'
)
```

### `BunnyApp::Subscription.cancel(subscription_id:)`

```ruby
BunnyApp::Subscription.cancel(subscription_id: '456123')
```

## Tenants

### `BunnyApp::Tenant.create(name:, code:, account_id:, platform_code:)`

```ruby
tenant = BunnyApp::Tenant.create(
  name: 'Acme Corp',
  code: 'acme-123',
  account_id: '456',
  platform_code: 'main'
)
```

### `BunnyApp::Tenant.find_by(code:)`

```ruby
tenant = BunnyApp::Tenant.find_by(code: 'acme-123')
```

## Feature Usage (Metered Billing)

### `BunnyApp::FeatureUsage.create(quantity:, feature_code:, subscription_id:, usage_at:)`

```ruby
usage = BunnyApp::FeatureUsage.create(
  quantity: 5,
  feature_code: 'api_calls',
  subscription_id: '456123',
  usage_at: Time.current.iso8601   # ISO 8601 date string
)
```

## Portal Sessions

### `BunnyApp::PortalSession.create(tenant_code:, return_url:, expiry_hours:)`

```ruby
token = BunnyApp::PortalSession.create(
  tenant_code: 'acme-123',
  return_url: 'https://yourapp.com/billing',
  expiry_hours: 4
)

redirect_to "https://<subdomain>.bunny.com/portal?token=#{token}"
```

## Tenant Metrics

### `BunnyApp::TenantMetrics.update(...)`

Push usage/engagement data for analytics and health scoring:
```ruby
BunnyApp::TenantMetrics.update(
  tenant_code: 'acme-123',
  value: 42,
  measured_at: Time.current
)
```

## Platforms

### `BunnyApp::Platform.create(...)`

Organize tenants under logical platform groupings:
```ruby
platform = BunnyApp::Platform.create(
  name: 'Main Platform',
  code: 'main'
)
```

## Webhooks

### `BunnyApp::Webhook.verify(signature, payload, signing_key)`

```ruby
signature = request.headers['X-Bunny-Signature']
payload   = request.body.read

if BunnyApp::Webhook.verify(signature, payload, ENV['BUNNY_WEBHOOK_SECRET'])
  event = JSON.parse(payload)
  # handle event['type']
end
```

Full Rails controller:
```ruby
# app/controllers/webhooks_controller.rb
class WebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def bunny
    payload   = request.body.read
    signature = request.headers['X-Bunny-Signature']

    unless BunnyApp::Webhook.verify(signature, payload, ENV['BUNNY_WEBHOOK_SECRET'])
      return head :unauthorized
    end

    event = JSON.parse(payload)

    case event['type']
    when 'subscription.created'
      # provision resources
    when 'subscription.cancelled'
      # deprovision resources
    end

    head :ok
  end
end

# config/routes.rb
post '/webhooks/bunny', to: 'webhooks#bunny'
```

## Error Handling

Two exception classes:

| Exception | Meaning |
|-----------|---------|
| `BunnyApp::AuthorizationError` | Invalid credentials, expired token, insufficient scope |
| `BunnyApp::ResponseError` | API rejected the request (validation failure, not found, etc.) |

```ruby
begin
  BunnyApp::Subscription.create(
    price_list_code: 'starter',
    options: { account_name: 'Acme', email: 'jane@acme.com' }
  )
rescue BunnyApp::AuthorizationError => e
  Rails.logger.error "Bunny auth failed: #{e.message}"
  raise
rescue BunnyApp::ResponseError => e
  Rails.logger.error "Bunny API error: #{e.message}"
  raise
end
```
