---
name: bunny-webhooks
description: Build webhook handlers that receive events from Bunny. Covers the two webhook flavours (Platform webhooks for tenant provisioning and checkout validation; Workflow webhooks for arbitrary object-change events across Accounts, Subscriptions, Invoices, Quotes, and 20+ other types), HMAC signature validation via the `x-bunny-signature` header against the raw request body, the optional Bearer-token `Authorization` header on platform webhooks, default payload envelope with `type` and `payload` fields, custom templated payloads with `{{placeholder}}` syntax, idempotency patterns, Bunny's per-event-class retry and timeout policy, and the Node / Ruby SDK verification helpers (`bunny.webhooks.validate`, `BunnyApp::Webhook.verify`). Use when receiving Bunny webhooks on a server you control.
---

# Bunny Webhooks

Bunny sends HTTP POST webhooks to URLs you configure. Two flavours:

| Flavour | What it's for | Configured at |
| --- | --- | --- |
| **Platform webhook** | Tenant provisioning — spin customer tenants up/down when subscriptions change state; checkout validation | Admin portal → Settings → Platforms → Provisioning |
| **Workflow webhook** | Arbitrary object-change events across ~24 object types (Account, Subscription, Invoice, Quote, Contact, etc.) with custom templated payloads | Admin portal → Workflows → your workflow → actions |

Public docs: [docs.bunny.com/developer/webhooks](https://docs.bunny.com/developer/webhooks/webhooks).

## Credential safety

- Store the **signing key** / **auth token** for each webhook endpoint in
  environment variables: `BUNNY_WEBHOOK_SIGNING_TOKEN`,
  `BUNNY_PLATFORM_WEBHOOK_AUTH_TOKEN`. Never log them, never commit them.
- Reject any request whose signature you can't verify. "Fail closed" —
  better a missed event (which will retry) than a forged event.
- Rotate signing keys on a cadence; rotate immediately if a leak is
  suspected. Bunny's admin portal generates a fresh key on demand.

## Handler contract

Bunny delivers events with these properties:

- **Method**: HTTP POST
- **Content-Type**: `application/json`
- **Body**: JSON (see payload shapes below)
- **Headers**:
  - `x-bunny-signature` — hex-encoded HMAC-SHA1 of the raw body using
    your configured signing key (no `sha1=` prefix — just the hex digest)
  - `Authorization: Bearer <webhook_auth_token>` — **platform webhooks
    only**, if you set an auth token on the Platform. Verify this too.
- **Response**: return `2xx` within the timeout (see below). Non-2xx
  (or timeout) triggers a retry if the event class allows it.
- **Timeout + retries depend on the event class**:

  | Event class | Timeout | Max attempts |
  | --- | --- | --- |
  | Workflow webhooks (default) | 20 s | 2 |
  | Tenant provisioning (`TenantProvisioningChange`) | 5 s | 2 |
  | Checkout validation (synchronous — blocks the signup flow) | 5 s | 1 |
  | "Send test webhook" from the admin portal | 5 s | 1 |

  After attempts are exhausted the delivery is dropped — Bunny does not
  keep a dead-letter queue on its side, so your endpoint is the source
  of truth for what succeeded.

Because retries are automatic, handlers **must be idempotent**. Dedupe on
a stable ID you extract from the payload (usually the object ID plus the
event type, or a `webhook_event_id` if present).

## Signature verification (the only rule that actually matters)

Verify against the **raw request body**, not a re-serialised JSON object.
Re-serialising can alter characters like `&`, `<`, `>`, whitespace, and
numeric precision — which will break the HMAC match even if the JSON is
semantically identical.

### Node / TypeScript

```typescript
import express from "express";
import Bunny from "@bunnyapp/api-client";

const bunny = new Bunny({
  baseUrl: process.env.BUNNY_BASE_URL!,
  accessToken: process.env.BUNNY_ACCESS_TOKEN!,
  webhookSigningToken: process.env.BUNNY_WEBHOOK_SIGNING_TOKEN!,
});

const app = express();

// Capture the raw body BEFORE any json() middleware on this route.
app.use("/bunny/webhook", express.raw({ type: "application/json" }));

app.post("/bunny/webhook", (req, res) => {
  const signature = req.headers["x-bunny-signature"] as string | undefined;
  const rawBody = req.body as Buffer;

  if (!signature || !bunny.webhooks.validate(signature, rawBody)) {
    return res.status(401).send("Invalid signature");
  }

  const event = JSON.parse(rawBody.toString());
  void handleEvent(event).catch(console.error);

  // Acknowledge fast; do the real work async.
  res.status(200).send("OK");
});
```

**Workflows can carry their own signing key** (configured per workflow
action in the admin portal). Pass it as the optional third argument to
`validate` so you don't have to drop to raw HMAC for one route:

```typescript
const perWorkflowKey = process.env.BUNNY_ORDER_WORKFLOW_SIGNING_KEY!;
if (!bunny.webhooks.validate(signature, rawBody, perWorkflowKey)) {
  return res.status(401).send("Invalid signature");
}
```

`validate()` throws if no signing token is available (neither on the
client nor supplied per-call). Wrap the call in `try/catch` and surface
a misconfig as `401` rather than a `500`.

### Ruby / Rails

```ruby
# config/routes.rb
post "/bunny/webhook", to: "bunny_webhooks#create"

# app/controllers/bunny_webhooks_controller.rb
class BunnyWebhooksController < ApplicationController
  skip_before_action :verify_authenticity_token

  def create
    payload     = request.raw_post
    signature   = request.headers["x-bunny-signature"]
    signing_key = ENV.fetch("BUNNY_WEBHOOK_SIGNING_TOKEN")

    unless BunnyApp::Webhook.verify(signature, payload, signing_key)
      return head :unauthorized
    end

    event = JSON.parse(payload)
    ProcessBunnyEventJob.perform_later(event)   # do the work async
    head :ok
  end
end
```

### Raw HMAC (any language)

If you can't use an SDK, compute `HMAC-SHA1(raw_body, signing_key)`,
hex-encode it, and compare constant-time against the header value.

```go
// Go
h := hmac.New(sha1.New, []byte(os.Getenv("BUNNY_WEBHOOK_SIGNING_TOKEN")))
h.Write(rawBody)
expected := hex.EncodeToString(h.Sum(nil))
if !hmac.Equal([]byte(expected), []byte(r.Header.Get("X-Bunny-Signature"))) {
    http.Error(w, "invalid signature", http.StatusUnauthorized)
    return
}
```

```python
# Python
import hmac, hashlib, os
expected = hmac.new(
    os.environ["BUNNY_WEBHOOK_SIGNING_TOKEN"].encode(),
    raw_body,
    hashlib.sha1,
).hexdigest()
if not hmac.compare_digest(expected, request.headers["x-bunny-signature"]):
    abort(401)
```

SHA-1 is legacy but HMAC-SHA1 is still cryptographically sound for
message authentication when the key stays secret. The security property
is HMAC's, not the underlying hash's collision resistance.

## Payload shape

Default envelope — both platform and workflow webhooks use it unless a
custom payload is configured on the workflow:

```json
{
  "type": "<ObjectName>",
  "payload": { "...": "fields of the object" }
}
```

### Platform webhook — `TenantProvisioningChange`

Fires when a subscription affects a tenant (created, cancelled, trialled,
quantity changed, feature flags changed):

```json
{
  "type": "TenantProvisioningChange",
  "payload": {
    "tenant": { "id": "...", "code": "acme-team", "subdomain": "acme" },
    "subscriptions": [
      { "id": "...", "state": "active", "plan": { "code": "starter" } }
    ],
    "features": [
      { "code": "seats", "enabled": true, "value": 25 }
    ],
    "account": { "id": "...", "name": "Acme Corp" }
  }
}
```

Your handler typically:

1. Looks up the tenant by `payload.tenant.code` in your own database
2. Updates feature flags / seat limits / deprovisions if no active subscriptions

### Platform webhook — `CheckoutValidation` (synchronous)

Fires during checkout and **blocks the signup flow** until your endpoint
responds. The envelope is the same default `{ type, payload }`. Unlike
every other webhook, the HTTP status and body you return are the business
answer:

- **Return `2xx`** (any body, typically empty) to allow checkout to
  proceed.
- **Return non-2xx** with a JSON body that is an **array of error
  strings** — each string becomes a user-visible error on the signup
  form. Bunny does `JSON.parse(body)` and iterates, so the top-level
  must be an array, not an object.

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

["Email domain not on your allowlist", "Reserved tenant code"]
```

```http
HTTP/1.1 200 OK
```

If the body isn't valid JSON on a non-2xx response, Bunny surfaces a
generic "Checkout validation failed due an invalid response from the
platform" error to the end user — integrators debugging a blocked
signup should check both status and body shape.

### Workflow webhook — default

When a workflow fires, the payload is the workflow's object serialised
via its `webhook_payload` method. For example, an Invoice workflow sends:

```json
{
  "type": "Invoice",
  "payload": {
    "id": "inv-123",
    "number": "INV-0001",
    "state": "issued",
    "amount": 120.00,
    "charges": [{ "...": "..." }],
    "lineItems": [{ "...": "..." }]
  }
}
```

Object types that can drive a workflow webhook include Account, Contact,
Deal, Quote, QuoteChange, QuoteCharge, Subscription, SubscriptionCharge,
Plan, Product, Feature, Coupon, Invoice, Payment, CreditNote, Tenant, and
more — roughly 24 types.

### Workflow webhook — custom payload

Workflows support a templated payload string. Placeholders use `{{dotted.path}}`
syntax relative to the workflow's object. The output must be valid JSON.

```yaml
# Example workflow action (admin-portal-authored, shown in YAML form)
- method: webhook
  url: https://app.example.com/bunny/subscription-state
  signing_key: <SECRET_FROM_BUNNY>
  payload: >
    {
      "subscription_id": "{{subscription.id}}",
      "tenant_code":     "{{subscription.tenant.code}}",
      "new_state":       "{{subscription.state}}",
      "changed_at":      "{{subscription.updatedAt}}"
    }
```

Custom payloads are recommended when your handler only cares about a few
fields — they reduce bandwidth and narrow your integration surface.

## Idempotency

Bunny retries on non-2xx or timeout, so the **same event can arrive
multiple times**. Make handlers idempotent:

- **Dedupe** using a combination of object ID + event type + a monotonic
  timestamp from the payload (e.g. `payload.updatedAt`). Store the tuple
  in a table with a unique constraint.
- **Short-circuit** if the incoming state matches your already-applied
  state (e.g. subscription already active).
- **Process async.** ACK `200` fast, then queue the real work. Fewer
  false timeouts mean fewer retries.
- **Ack-then-async requires a durable enqueue.** Bunny will not retry a
  `200` — so if your handler ACKs, crashes before the work is persisted,
  and restarts, that event is gone. Write the job to a durable queue
  (Sidekiq / SQS / a row in Postgres) **before** you `res.status(200)`,
  not after. With max-attempts of 2 (workflow) or 1 (checkout), there's
  no second chance.

```typescript
const seen = new Set<string>();                   // use durable storage in production

function dedupeKey(event: any): string {
  return `${event.type}:${event.payload?.id}:${event.payload?.updatedAt}`;
}

async function handleEvent(event: any) {
  const key = dedupeKey(event);
  if (seen.has(key)) return;                      // already processed
  seen.add(key);
  // ... real work
}
```

## Routing events to handlers

For workflow webhooks with the default envelope, route on `event.type`:

```typescript
switch (event.type) {
  case "TenantProvisioningChange":
    return provisionTenant(event.payload);
  case "Subscription":
    return updateSubscriptionMirror(event.payload);
  case "Invoice":
    return recordInvoice(event.payload);
  case "CreditNote":
    return recordCreditNote(event.payload);
  default:
    console.warn("Unhandled Bunny event type:", event.type);
}
```

For custom-payload workflows, you define the schema — route on whatever
field you put in the template.

## Testing webhooks locally

- Use [ngrok](https://ngrok.com), [RequestBin](https://requestbin.com),
  or similar to expose your local server to Bunny.
- In the admin portal, each workflow action and each platform has a
  **Send test webhook** button that fires a sample payload against your
  configured URL without needing a real event to occur.
- Keep a raw-body log (redacting signatures) on failed deliveries so you
  can re-verify signatures manually when debugging.

## References

- [docs.bunny.com/developer/webhooks](https://docs.bunny.com/developer/webhooks/webhooks) — official docs
- Sibling skills: `bunny-node-sdk` (`bunny.webhooks.validate`), `bunny-ruby-sdk` (`BunnyApp::Webhook.verify`), `bunny-subscriptions` (lifecycle state changes that drive TenantProvisioningChange), `bunny-billing` (invoice / payment / credit-note events)

---

Last verified against bunnyapp/api release 2026-04-21-1
