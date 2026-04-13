---
name: bunny-components
description: This skill should be used when the user asks to "add a billing portal", "embed subscription management", "add Bunny components to React", "implement self-service billing in my app", "add a billing page", "let users manage their subscription", "show invoices in my app", "add a signup flow with Bunny", "embed Bunny billing UI", "add payment method management", or any task involving the @bunnyapp/components React library for embedding Bunny billing UI into a React application.
version: 0.1.0
---

# Bunny React Components Integration

This skill guides embedding Bunny's pre-built React billing UI components (`@bunnyapp/components`) into a React application. These components allow users to self-service their subscriptions, update billing details, view invoices, manage payment methods, and sign up — all without leaving your app.

## Installation

```bash
npm install @bunnyapp/components
```

The package requires React. Styling uses Tailwind CSS — components accept `className` and `shadow` props for customisation.

## Core Concept: BunnyProvider

Every component must be a descendant of a single `BunnyProvider`. The provider manages shared state so that updates in one component (e.g. payment method saved in `BillingDetails`) are immediately reflected in others (e.g. `Subscriptions`).

```tsx
import { BunnyProvider, BillingDetails, Subscriptions, Transactions } from "@bunnyapp/components";

function BillingPage() {
  return (
    <BunnyProvider token={portalSessionToken} apiHost="https://acme.bunny.com">
      <Subscriptions />
      <BillingDetails />
      <Transactions />
    </BunnyProvider>
  );
}
```

**Do not** mount multiple `BunnyProvider` instances for normal billing components — state will not be shared and components will conflict.

## Authentication: Choosing the Right Token

Two token types are supported:

| Token type | Scope | Use when |
|---|---|---|
| Portal session token | Single account | Authenticated user viewing their own billing |
| API token | All accounts | Admin views, multi-tenant dashboards |

**For customer-facing pages**, always use a portal session token. Generate one server-side using the Bunny SDK (see `bunny-billing` skill) and pass it to the client. Never expose API tokens to the browser.

```tsx
// Server: generate token (Node.js example)
const token = await bunny.portalSessionCreate("acme-team", returnUrl, 4);

// Client: pass to BunnyProvider
<BunnyProvider token={token} apiHost="https://acme.bunny.com">
  ...
</BunnyProvider>
```

## CORS Setup

Before components will work in the browser, whitelist the app's domain in the Bunny admin. Add `localhost` for development and the production domain for live environments. Failing to do this causes CORS errors.

## Component Overview

| Component | Purpose |
|---|---|
| `<Subscriptions />` | View, upgrade, downgrade, and cancel subscriptions |
| `<BillingDetails />` | Update billing address and payment methods |
| `<Invoice id="..." />` | View and pay a specific invoice |
| `<Transactions />` | List of invoices, payments, and refunds |
| `<Quotes />` | List of outstanding quotes |
| `<Quote />` | View and accept a specific quote |
| `<Signup />` | Public-facing subscription signup flow |

`<PaymentForm />` is for internal Bunny use — do not use it directly.

## Common Layout Patterns

### Full Billing Portal Page

```tsx
import { BunnyProvider, Subscriptions, BillingDetails, Transactions } from "@bunnyapp/components";

export function BillingPortal({ sessionToken }: { sessionToken: string }) {
  return (
    <BunnyProvider token={sessionToken} apiHost={process.env.NEXT_PUBLIC_BUNNY_HOST}>
      <div className="max-w-4xl mx-auto py-8 space-y-6">
        <Subscriptions />
        <BillingDetails />
        <Transactions />
      </div>
    </BunnyProvider>
  );
}
```

### Invoice Deep-Link Page

```tsx
import { BunnyProvider, Invoice } from "@bunnyapp/components";

export function InvoicePage({ invoiceId, token }: { invoiceId: string; token: string }) {
  return (
    <BunnyProvider token={token} apiHost={process.env.NEXT_PUBLIC_BUNNY_HOST}>
      <Invoice
        id={invoiceId}
        onPaymentSuccess={(usedSavedMethod) => {
          console.log("Payment succeeded, saved method:", usedSavedMethod);
        }}
        onBackButtonClick={() => router.back()}
        backButtonName="Back to billing"
      />
    </BunnyProvider>
  );
}
```

### Signup Page (Public — Separate Provider)

The `<Signup />` component **must** use its own `BunnyProvider` with a token scoped to `signup:read signup:write` only. Never share a signup provider with billing components.

```tsx
import { BunnyProvider, Signup } from "@bunnyapp/components";

export function SignupPage() {
  // Token generated server-side with signup:read signup:write scopes only
  const signupToken = process.env.NEXT_PUBLIC_BUNNY_SIGNUP_TOKEN;

  return (
    <BunnyProvider token={signupToken} apiHost={process.env.NEXT_PUBLIC_BUNNY_HOST}>
      <Signup
        companyName="Acme"
        entityId="1"
        priceListCode="business-monthly"
        returnUrl="https://acme.com/dashboard"
      />
    </BunnyProvider>
  );
}
```

## Key Rules

1. **One BunnyProvider per billing context** — all normal components share a single provider.
2. **Signup always gets its own provider** — and a narrowly-scoped token.
3. **Generate portal session tokens server-side** — never expose Bunny client secrets to the browser.
4. **Whitelist domains in Bunny admin** — required to avoid CORS errors.
5. **`className` and `shadow` props** accept Tailwind CSS classes for styling.

## Additional Resources

- **`references/components.md`** — All component props with types and defaults
- **`references/setup.md`** — Token generation, CORS whitelisting, Next.js / Vite integration patterns

For server-side token generation, see the `bunny-billing` skill.
