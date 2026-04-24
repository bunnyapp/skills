---
name: bunny-components
description: Official React component library for Bunny — `@bunnyapp/components`. Embeddable UI for signup, subscription management, invoices, quotes, billing details, and transaction history. Covers installation, wrapping an app tree in `BunnyProvider` (required `apiHost` and — for any functional rendering — a `token` prop), generating portal session tokens server-side with the `portalSessionCreate` GraphQL mutation, the single-provider pattern across multiple components, CORS subdomain whitelisting, and the component catalogue (BillingDetails, Invoice, Signup, Subscriptions, Quote, Quotes, Transactions). Use when embedding Bunny billing UI into a React, Next.js, Remix, or Vite app. For the hosted (non-React) alternatives see `bunny-customer-portal`; for server-side SDKs see `bunny-node-sdk` and `bunny-ruby-sdk`.
---

# Bunny Components — `@bunnyapp/components`

Official React component library that embeds Bunny's signup, subscription,
invoice, and portal UIs directly into your app. Each component is a drop-in
view wrapped around Bunny's GraphQL API, requiring only a session token and
your tenant's `apiHost`.

Public docs: [docs.bunny.com/developer/bunny-components](https://docs.bunny.com/developer/bunny-components/bunny-components).
Repo: published as `@bunnyapp/components` on npm.

> **Status**: the library is currently in a pre-1.0 beta (`1.8.0-beta.N` at
> time of writing). Expect minor breaking changes between betas. Pin an exact
> version in `package.json` and review changelog before bumping.

## Credential safety

- **Never expose API tokens to the browser.** The `token` prop passed to
  `BunnyProvider` must be a **portal session token**, not an API access
  token, when scoping to a single customer. Generate session tokens on your
  server via `portalSessionCreate` and pass only that short-lived token to
  the React tree.
- **`<Signup />` is the exception** — it needs a separate, narrowly-scoped
  token (`signup:read signup:write`) in its own `BunnyProvider`. A
  portal-session token can't call the signup mutations. See "Checkout →
  subscription" below.
- API tokens are only appropriate when the integrator is the signed-in
  admin viewing cross-account data in their own internal tool — and even
  then they stay in server components or a signed cookie, not a public
  JavaScript bundle.
- Load server-side secrets from environment variables or your platform's
  secret manager.

## Install

```sh
npm install @bunnyapp/components --save
```

Peer dependencies (ensure your app already uses compatible versions): React
18, `@tanstack/react-query`, Ant Design, `@ant-design/icons`, Stripe JS
(`@stripe/stripe-js` + `@stripe/react-stripe-js`), Font Awesome. The exact
versions are in the package's `peerDependencies`.

## Wrap your tree in a single `BunnyProvider`

Mount **one** `BunnyProvider` above every Bunny component you render. It
owns the API client, the query cache, and the session — mounting multiple
providers creates state mismatches (stale caches, duplicate requests, lost
mutations).

```tsx
import { BunnyProvider, Subscriptions, BillingDetails } from "@bunnyapp/components";

export function BillingPage({ sessionToken }: { sessionToken: string }) {
  return (
    <BunnyProvider
      token={sessionToken}
      apiHost={process.env.NEXT_PUBLIC_BUNNY_API_HOST!} // https://<subdomain>.bunny.com
    >
      <Subscriptions />
      <BillingDetails />
    </BunnyProvider>
  );
}
```

Anti-pattern (don't do this):

```tsx
// Wrong — each Provider builds its own state, caches drift apart.
<BunnyProvider token={t} apiHost={h}><Subscriptions /></BunnyProvider>
<BunnyProvider token={t} apiHost={h}><BillingDetails /></BunnyProvider>
```

In server-rendered apps (Next.js App Router, Remix, Nuxt), mint the token
in the server layer (server component, loader, or API route) and hand it
to a `"use client"` boundary as a prop — `BunnyProvider` has to run
client-side. Don't render the token-minting page at build time; tokens
are short-lived (default 24 h).

## Generate the session token server-side

Mint the `token` prop server-side via `portalSessionCreate` — see the
`bunny-node-sdk`, `bunny-ruby-sdk`, or `bunny-graphql` skill for the exact
call in your language. One Bunny-specific scope rule applies regardless:

> **The token-minting API client must hold `security:write`.**
> `portalSessionCreate` is gated by Bunny's `SecurityPolicy`; a client
> without `security:write` fails with an authorization error. This is
> separate from — and in addition to — the standard `standard:read` /
> `standard:write` scopes the rest of your integration uses.

Portal session tokens default to 24 h expiry; pass an override if you need
shorter. The `returnUrl` only controls the portal's "Back" link, not a
post-action redirect.

## CORS: whitelist your subdomains

`BunnyProvider` makes XHR/fetch calls from the browser to `apiHost`. You
must whitelist the subdomain(s) serving your app in Bunny's admin portal,
otherwise every request fails CORS preflight. Include at minimum:

- `http://localhost:3000` (or whatever port you run in development)
- Your production origin, e.g. `https://app.example.com`

Add staging and preview deployments (Vercel, Netlify preview URLs) as
needed. Subdomain whitelisting lives in tenant settings; ask your Bunny
admin to add them if you don't have access yourself.

## Component catalogue

| Component | Purpose |
| --- | --- |
| `Signup` | Sign up for a new subscription (handles plan selection, billing details, payment) |
| `Subscriptions` | View, upgrade, downgrade, and cancel subscriptions |
| `Quote` | Show a single quote and let the customer accept it |
| `Quotes` | List view of quotes |
| `Invoice` | Show a single invoice and collect payment |
| `Transactions` | List invoices, payments, and refunds |
| `BillingDetails` | Update account details, billing address, and payment methods |

Internal-use-only: `PaymentForm` — don't mount it directly, it's rendered by
the user-facing components above.

Each component reads the `token` and `apiHost` from its nearest
`BunnyProvider`. Styling inherits from Ant Design's theme, so if your app
already customises `ConfigProvider`, the Bunny components will pick that up.

## Signup has a separate scope model

`<Signup />` is the one component that does **not** run under a portal
session token. It calls unauthenticated signup mutations
(`quoteAccountSignup`, `accountSignup`, `quoteChangeAddCoupon`,
`quoteChangeRemoveCoupon`, `quoteRecalculateTaxes`) which are only covered
by the `signup:read` / `signup:write` scope aliases. A portal-session
token — scoped to an existing account — cannot call them and fails with
`Rejected by reach`.

The signup `token` is an **OAuth2 access token** from a `signup:read
signup:write`-scoped API client — *not* a portal session token. Mint it
server-side via the OAuth2 client-credentials flow (`POST /oauth/token`
with `grant_type=client_credentials`) documented in `bunny-graphql`, then
pass the returned `access_token` string as the provider's `token` prop.
Cache it for its `expires_in` window; rotate credentials infrequently.
`@bunnyapp/api-client` does not publicly expose its internal access
token — mint your own from the OAuth endpoint when you need a raw token
string for this case.

Wrap the signup flow in its own `BunnyProvider`:

```tsx
import { BunnyProvider, Signup } from "@bunnyapp/components";

// signupToken = access_token from a signup:read signup:write client
// via /oauth/token. NOT the portal-session token used elsewhere.
<BunnyProvider token={signupToken} apiHost={apiHost}>
  <Signup
    priceListCode="starter-annual"       // preselect a plan
    onSuccess={(subscription) => {
      // Post-signup redirect target — the component won't navigate for you.
      // <Signup /> uses `onSuccess`, NOT `returnUrl` (which is portal-only).
    }}
  />
</BunnyProvider>
```

Alternatively, use the hosted signup page (`bunny-customer-portal`) instead
of the embedded `Signup` component — the hosted flow is easier when you
don't need to wrap Bunny's UI inside your own layout.

Every other component (Subscriptions, BillingDetails, Transactions, Quote,
Quotes, Invoice) follows the same shape: render one or more under the
portal-session `BunnyProvider` and the component handles its own queries,
mutations, and state.

## References

- [docs.bunny.com/developer/bunny-components](https://docs.bunny.com/developer/bunny-components/bunny-components) — official docs
- Sibling skills: `bunny-customer-portal` (hosted UI, non-React), `bunny-node-sdk`, `bunny-ruby-sdk`, `bunny-graphql` (server-side token generation), `bunny-quoting` (quote lifecycle behind `Quote`)

---

Last verified against @bunnyapp/components v1.8.0-beta.19 and bunnyapp/api release 2026-04-21-1
