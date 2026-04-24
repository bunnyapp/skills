---
name: bunny-customer-portal
description: Hosted Bunny Customer Portal — three embedding variants for non-React apps (or when you prefer hosted over embedded). Covers the popup portal (via the `https://cdn.bunny.com/v1/bunny.js` SDK with `new Bunny(subdomain, token).popup({page})`), the standalone redirect portal at `https://<subdomain>.bunny.com/portal?token=<token>` with deep links to `/portal/subscriptions`, `/portal/transactions`, `/portal/payment-method`, and the hosted signup page at `https://<subdomain>.bunny.com/portal/signup` with pre-selectable `priceListCode`, `returnUrl`, `couponCode`, and prefilled customer fields. Also covers generating portal session tokens server-side via `portalSessionCreate` and the separate `signup:read` / `signup:write` token scopes the signup page requires. Use when embedding billing UI without using the React components, or when the simplest integration is a redirect.
---

# Bunny Customer Portal (hosted)

Bunny ships three hosted portal surfaces. They are independent of the React
component library — reach for these when (a) you don't want to mount React
components from Bunny inside your app, or (b) the simplest integration is a
redirect or a popup.

Public docs: [docs.bunny.com/developer/customer-portal](https://docs.bunny.com/developer/customer-portal).

| Variant | Use when | Integration shape |
| --- | --- | --- |
| **Popup** | You want the portal to float over your existing UI (e.g. an "Upgrade" button in a settings page) | `<script>` include + `bunny.popup()` |
| **Standalone** | You want a full-page Bunny experience for billing; simplest redirect flow | URL redirect with `?token=` |
| **Signup** | You want a marketing-style signup link that preselects a plan and optional coupon | URL redirect with plan + prefill params |

All three variants authenticate via a **portal session token**; generate it
server-side. See the `bunny-node-sdk` / `bunny-ruby-sdk` / `bunny-graphql`
skills for how.

## Credential safety

- Portal session tokens are short-lived (default 24 h, configurable). Never
  mint a long-lived API token and give it to the browser.
- The signup page accepts tokens scoped specifically to `signup:read`
  `signup:write`. Do **not** reuse a general API token with broader scopes
  there — mint a narrow one for the signup flow.
- `returnUrl` is trusted; never echo user-controlled input into it without
  an allowlist. An attacker who controls the returnUrl controls post-signup
  redirects.

## Popup portal

Floats Bunny's portal over your page as a modal. Ideal for "Manage
subscription" buttons in settings pages.

Mint a portal session token server-side (see the `bunny-node-sdk` /
`bunny-ruby-sdk` / `bunny-graphql` skill) and hydrate it into the page,
then include the Bunny JS SDK and open the popup on a user interaction:

```html
<script type="text/javascript" src="https://cdn.bunny.com/v1/bunny.js"></script>
<script>
  const bunny = new Bunny("<subdomain>", window.__BUNNY_TOKEN);
  document.querySelector("#upgrade").addEventListener("click", () => {
    bunny.popup({ page: "subscriptions" });
  });
</script>
```

### Supported `page` values

| Value | Shows |
| --- | --- |
| `"subscriptions"` (default) | Manage, upgrade, cancel subscriptions |
| `"transactions"` | Invoices, payments, refunds |
| `"payment-method"` | Stored payment methods and address |

## Standalone portal (redirect)

Full-page portal served by Bunny. The integration is a single redirect from
your server — typically when the user clicks a "Billing" link.

### URL patterns

```text
https://<subdomain>.bunny.com/portal?token=<TOKEN>
https://<subdomain>.bunny.com/portal/subscriptions?token=<TOKEN>
https://<subdomain>.bunny.com/portal/transactions?token=<TOKEN>
https://<subdomain>.bunny.com/portal/payment-method?token=<TOKEN>
```

### Flow

1. User clicks "Billing" in your app
2. Your server calls `portalSessionCreate(tenantCode, returnUrl?, expiryInHours?)`
3. Server 302-redirects the browser to the portal URL above with the token
   appended
4. Bunny validates the token and shows the portal

```typescript
// Next.js route handler — same shape in any language; call your SDK's
// portalSessionCreate helper and 302 to the portal URL with the token.
export async function GET(req: Request) {
  const user = await currentUser(req);
  const token = await bunny.portalSessionCreate(
    user.tenantCode,
    `${origin(req)}/settings`, // optional return URL
  );
  return Response.redirect(
    `${process.env.BUNNY_BASE_URL}/portal/subscriptions?token=${encodeURIComponent(token)}`,
    302,
  );
}
```

The portal is a one-way redirect — Bunny does not bring the user back to
your app automatically after specific actions. On the standalone portal,
`returnUrl` only controls where the portal's top-level "Back" link points.

> **Name collision to watch for:** `returnUrl` means two different things
> across these surfaces. On the **standalone portal** it's just the "Back"
> link target. On the **signup page** (below) it's the post-signup
> redirect. Same query-param name; different behaviour.

## Signup page

Marketing-friendly hosted signup form with plan, coupon, and customer
fields preselectable via query string. Hosted on Bunny; not embeddable.

### URL pattern

```text
https://<subdomain>.bunny.com/portal/signup?priceListCode=<CODE>&token=<TOKEN>&returnUrl=<URL>
```

### Supported query parameters

| Parameter | Purpose |
| --- | --- |
| `token` | **Required.** API token scoped to `signup:read` `signup:write`. |
| `priceListCode` | Preselects a plan (e.g. `business-annual`). |
| `returnUrl` | Redirect target after successful signup. |
| `couponCode` | Applies a discount code on signup. |
| `firstName` / `lastName` | Prefill customer name. |
| `email` | Prefill email. |
| `companyName` | Prefill account name. |
| `billingCountry` | Prefill billing country (ISO alpha-2). |

### Example

```text
https://acme.bunny.com/portal/signup
  ?priceListCode=business-annual
  &token=<SIGNUP_SCOPED_TOKEN>
  &returnUrl=https://app.example.com/welcome
  &couponCode=LAUNCH20
  &email=jane@acme.example
  &companyName=Acme+Corp
```

### Minting the signup-scoped token

Create a dedicated API client in the Bunny admin portal with scopes
`signup:read signup:write` only. **Don't reuse your general-purpose token**
— a narrower token reduces blast radius if the URL leaks (in referrer
logs, browser history, email previews). Cache the token; regenerate it
infrequently.

## Choosing between popup, standalone, components

| Need | Pick |
| --- | --- |
| Modal over your existing page, minimal layout impact | **Popup** |
| Simplest: just redirect to Bunny for billing | **Standalone** |
| Marketing landing page with preselected plan | **Signup** |
| Deeply integrated UI, customer doesn't see a domain change, you control layout | **[bunny-components](..) (React)** |

## References

- [docs.bunny.com/developer/customer-portal](https://docs.bunny.com/developer/customer-portal) — official docs + popup / standalone / signup sub-pages
- Sibling skills: `bunny-components` (React embed), `bunny-node-sdk` / `bunny-ruby-sdk` (token generation), `bunny-catalog` (price lists + coupons referenced here)

---

Last verified against bunnyapp/api release 2026-04-21-1
