# Bunny Components — Setup & Integration Guide

## CORS Whitelisting

Before any component will work in a browser, the app's domain must be whitelisted in the Bunny admin.

1. Log in to the Bunny admin
2. Navigate to Settings > API Clients (or equivalent CORS/domains settings)
3. Add each domain that will host the components:
   - `localhost` (for local development)
   - `https://app.yourcompany.com` (production)
   - Any staging/preview domains

Failing to whitelist causes browser errors like:
```
Access to fetch at 'https://acme.bunny.com/graphql' from origin 'https://app.acme.com' has been blocked by CORS policy
```

## Token Types and When to Use Each

### Portal Session Token (Recommended for customer-facing pages)

- Scoped to a single account
- Generated server-side with the Bunny SDK
- Short-lived (configurable expiry, typically 1–4 hours)
- Safe to send to the browser — carries no admin privileges

```typescript
// Server-side (Node.js)
import Bunny from "@bunnyapp/api-client";

const bunny = new Bunny({
  baseUrl: process.env.BUNNY_BASE_URL,
  clientId: process.env.BUNNY_CLIENT_ID,
  clientSecret: process.env.BUNNY_CLIENT_SECRET,
  scope: "standard:read standard:write",
});

// In your billing page API route
export async function GET(req: Request) {
  const tenantCode = getCurrentUserTenantCode(req); // your auth logic
  const token = await bunny.portalSessionCreate(tenantCode, returnUrl, 4);
  return Response.json({ token });
}
```

### API Token (Admin / multi-account views)

- Access across all accounts
- Generated in Bunny admin: Settings > API Clients > Generate Access Token
- Use only in trusted server environments — **never expose in browser**
- Appropriate for internal admin dashboards where the server renders and injects the token

### Signup Token

- Narrowly scoped to `signup:read` and `signup:write` only
- Can be a static token (not per-user) since signup is pre-authentication
- Store as an environment variable and inject at build time or via a public API route

## Next.js Integration Pattern

### App Router — Server Component fetching token

```tsx
// app/billing/page.tsx
import { BillingPortal } from "@/components/BillingPortal";
import { getBunnyToken } from "@/lib/bunny";

export default async function BillingPage() {
  const token = await getBunnyToken(); // server-side token fetch
  return <BillingPortal token={token} />;
}
```

```tsx
// components/BillingPortal.tsx  ("use client")
"use client";
import { BunnyProvider, Subscriptions, BillingDetails, Transactions } from "@bunnyapp/components";

export function BillingPortal({ token }: { token: string }) {
  return (
    <BunnyProvider token={token} apiHost={process.env.NEXT_PUBLIC_BUNNY_HOST!}>
      <Subscriptions />
      <BillingDetails />
      <Transactions />
    </BunnyProvider>
  );
}
```

### App Router — Client-side token fetch

```tsx
"use client";
import { useEffect, useState } from "react";
import { BunnyProvider, Subscriptions, BillingDetails } from "@bunnyapp/components";

export function BillingPortal() {
  const [token, setToken] = useState<string | null>(null);

  useEffect(() => {
    fetch("/api/bunny/session")
      .then((r) => r.json())
      .then((data) => setToken(data.token));
  }, []);

  if (!token) return <div>Loading...</div>;

  return (
    <BunnyProvider token={token} apiHost={process.env.NEXT_PUBLIC_BUNNY_HOST!}>
      <Subscriptions />
      <BillingDetails />
    </BunnyProvider>
  );
}
```

### API Route for token generation

```typescript
// app/api/bunny/session/route.ts
import { NextResponse } from "next/server";
import { getBunnyClient } from "@/lib/bunny";
import { getCurrentUser } from "@/lib/auth";

export async function GET(req: Request) {
  const user = await getCurrentUser(req);
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const bunny = getBunnyClient();
  const token = await bunny.portalSessionCreate(
    user.tenantCode,
    `${process.env.NEXT_PUBLIC_APP_URL}/billing`,
    4
  );

  return NextResponse.json({ token });
}
```

## Vite / Create React App Pattern

```tsx
// src/pages/BillingPage.tsx
import { useState, useEffect } from "react";
import { BunnyProvider, Subscriptions, BillingDetails, Transactions } from "@bunnyapp/components";

export function BillingPage() {
  const [token, setToken] = useState<string>("");

  useEffect(() => {
    // Fetch token from your backend
    fetch("/api/billing/token")
      .then((r) => r.json())
      .then(({ token }) => setToken(token));
  }, []);

  if (!token) return <div>Loading billing...</div>;

  return (
    <BunnyProvider token={token} apiHost={import.meta.env.VITE_BUNNY_HOST}>
      <Subscriptions />
      <BillingDetails />
      <Transactions />
    </BunnyProvider>
  );
}
```

## Environment Variables

```bash
# Server-side (never expose to browser)
BUNNY_CLIENT_ID=your-client-id
BUNNY_CLIENT_SECRET=your-client-secret
BUNNY_BASE_URL=https://acme.bunny.com

# Client-safe (public)
NEXT_PUBLIC_BUNNY_HOST=https://acme.bunny.com
# or for Vite:
VITE_BUNNY_HOST=https://acme.bunny.com

# Signup token — can be a static scoped token
NEXT_PUBLIC_BUNNY_SIGNUP_TOKEN=your-signup-scoped-token
```

## Common Mistakes

### Multiple BunnyProvider instances
```tsx
// ❌ Wrong — state won't be shared
<BunnyProvider token={token} apiHost={host}>
  <Subscriptions />
</BunnyProvider>
<BunnyProvider token={token} apiHost={host}>
  <BillingDetails />
</BunnyProvider>

// ✓ Correct — single provider wraps all components
<BunnyProvider token={token} apiHost={host}>
  <Subscriptions />
  <BillingDetails />
</BunnyProvider>
```

### Signup sharing a provider with billing components
```tsx
// ❌ Wrong — signup needs its own scoped token and provider
<BunnyProvider token={billingToken} apiHost={host}>
  <Subscriptions />
  <Signup priceListCode="starter" companyName="Acme" entityId="1" />
</BunnyProvider>

// ✓ Correct — separate providers
<BunnyProvider token={billingToken} apiHost={host}>
  <Subscriptions />
</BunnyProvider>

// On a different page/route:
<BunnyProvider token={signupToken} apiHost={host}>
  <Signup priceListCode="starter" companyName="Acme" entityId="1" />
</BunnyProvider>
```

### Exposing secrets to the browser
```typescript
// ❌ Wrong — never put clientSecret or API tokens in client code
const bunny = new Bunny({ clientSecret: process.env.NEXT_PUBLIC_BUNNY_SECRET });

// ✓ Correct — generate token server-side and pass only the token to the client
const token = await bunny.portalSessionCreate(tenantCode);
```
