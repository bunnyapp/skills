# Bunny Components — Prop Reference

## BunnyProvider

Wraps all Bunny components. Must appear exactly once per billing context.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `token` | string | ✓ | Portal session token or API token |
| `apiHost` | string | ✓ | Base URL, e.g. `https://acme.bunny.com` |

```tsx
<BunnyProvider token={token} apiHost="https://acme.bunny.com">
  {/* components here */}
</BunnyProvider>
```

---

## `<Subscriptions />`

View, upgrade, downgrade, and cancel subscriptions for the authenticated account.

No required props. Reads subscription data from the BunnyProvider context.

```tsx
<Subscriptions />
```

---

## `<BillingDetails />`

Allows users to update their billing address and payment method (card). Two-panel layout: billing address on the left, payment method on the right.

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `isCardEnabled` | boolean | `true` | When `false`, removes card/panel styling |
| `shadow` | string | `'shadow-md'` | Tailwind shadow class applied to the component |
| `className` | string | `undefined` | Additional Tailwind CSS classes |
| `hidePaymentMethodForm` | boolean | `false` | When `true`, hides the right-side payment method panel |
| `hideBillingDetailsForm` | boolean | `false` | When `true`, hides the left-side billing address panel |
| `countryListFilter` | `(country: { value: string }) => boolean` | `undefined` | Filter function to exclude specific countries from the dropdown |

```tsx
// Show only billing address (hide payment method panel)
<BillingDetails hideBillingDetailsForm={false} hidePaymentMethodForm={true} />

// Exclude specific countries
const blockedCountries = ["US", "CA"];
<BillingDetails
  countryListFilter={(country) => !blockedCountries.includes(country.value)}
/>

// Full example
<BillingDetails
  isCardEnabled={true}
  shadow="shadow-md"
  className="rounded-lg"
  hidePaymentMethodForm={false}
  hideBillingDetailsForm={false}
/>
```

---

## `<Invoice />`

Displays a single invoice. Supports viewing, downloading, and paying unpaid invoices.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `id` | string | ✓ | Invoice ID to display |
| `shadow` | string | | Tailwind shadow class |
| `className` | string | | Additional Tailwind CSS classes |
| `hideDownloadButton` | boolean | | When `true`, hides the download button |
| `backButtonName` | string | | Custom label for the back button |
| `onBackButtonClick` | `() => void` | | Called when the back button is clicked |
| `onInvoiceDownloadError` | `(error: Error) => void` | | Called if invoice download fails |
| `onPaymentSuccess` | `(usedSavedMethod: boolean) => void` | | Called after successful payment |
| `onInvoiceLoaded` | `(invoice: FormattedInvoice) => void` | | Called when invoice data is loaded |
| `onQuoteLoaded` | `(quote: FormattedQuote) => void` | | Called when quote data is loaded |
| `invoiceQuoteViewComponent` | React component | | Replaces the default invoice/quote view with custom component |

```tsx
<Invoice
  id="inv_12345"
  backButtonName="Back to invoices"
  onBackButtonClick={() => router.push("/billing")}
  onPaymentSuccess={(usedSavedMethod) => {
    toast.success("Payment complete");
    if (!usedSavedMethod) refetchPaymentMethods();
  }}
  onInvoiceDownloadError={(err) => toast.error("Download failed")}
  hideDownloadButton={false}
/>
```

---

## `<Transactions />`

Displays a paginated list of invoices, payments, and refunds for the account.

No required props. Reads data from BunnyProvider context.

```tsx
<Transactions />
```

---

## `<Quotes />`

Displays a list of pending quotes awaiting acceptance.

No required props.

```tsx
<Quotes />
```

---

## `<Quote />`

Displays a single quote for review and acceptance.

```tsx
<Quote />
```

---

## `<Signup />`

Public-facing subscription signup form. Creates a new account and subscription.

> **Important**: Signup requires its own dedicated `BunnyProvider` with a token that has **only** `signup:read` and `signup:write` scopes. Never share this provider with other components.

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `companyName` | string | ✓ | Your company/app name identifier |
| `entityId` | string | ✓ | Entity reference ID |
| `priceListCode` | string | ✓ | The plan/price list code (e.g., `"business-monthly"`) |
| `returnUrl` | string | | URL to redirect to after successful signup |
| `couponCode` | string | | Coupon/promo code applied automatically |
| `defaultFirstName` | string | | Pre-populates the first name field |
| `defaultLastName` | string | | Pre-populates the last name field |
| `defaultEmail` | string | | Pre-populates the email field |
| `defaultCompanyName` | string | | Pre-populates the company name field |
| `defaultBillingCountry` | string | | Pre-populates the country (ISO 2-letter code, e.g., `"US"`) |

```tsx
// Minimal
<BunnyProvider token={signupScopedToken} apiHost="https://acme.bunny.com">
  <Signup
    companyName="Acme"
    entityId="1"
    priceListCode="business-monthly"
    returnUrl="https://app.acme.com/dashboard"
  />
</BunnyProvider>

// Pre-filled from OAuth/SSO data
<BunnyProvider token={signupScopedToken} apiHost="https://acme.bunny.com">
  <Signup
    companyName="Acme"
    entityId="1"
    priceListCode="business-monthly"
    returnUrl="https://app.acme.com/dashboard"
    defaultFirstName={user.firstName}
    defaultLastName={user.lastName}
    defaultEmail={user.email}
    defaultCompanyName={user.orgName}
    defaultBillingCountry="US"
    couponCode={promoCode}
  />
</BunnyProvider>
```
