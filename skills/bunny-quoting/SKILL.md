---
name: bunny-quoting
description: Build quote flows in Bunny — the subscription billing platform. Covers the quote lifecycle state machine (DRAFT, IN_APPROVAL, APPROVED, SHARED, VIEWED, ACCEPTED, REJECTED, UNDONE), the `Quote → QuoteChange → QuoteCharge` three-level hierarchy, QuoteChangeKind (SUBSCRIBE, RENEW, UPDATE, UNSUBSCRIBE, ADJUSTMENT, DISCOUNT, QUANTITY_UPDATE, PRICE_UPDATE, …), the canonical create-change-apply flow, renewal/upgrade/add-on shortcuts (`quoteSubscriptionRenew`, `quoteSubscriptionUpgrade`, `quoteSubscriptionAddon`), coupon application (`quoteChangeAddCoupon`), the share/accept workflow (`quoteMarkAsShared` + `quoteAccept`), approval gating (`quoteApprovalStart` / `quoteApprove`), ramp deals (`quoteChangeCreateRampUp`) for stepped-quantity contracts, and the `quoteApplyChanges` output envelope (`{ quote, subscriptions, invoice, errors }`). Use when creating, modifying, or applying quotes; when building sales-assisted signup; or when orchestrating renewals and upgrades.
---

# Bunny Quoting

A **quote** is Bunny's unit of change against a billing relationship. Every
subscription-affecting operation — new sale, renewal, upgrade, cancellation,
discount, one-time adjustment — is modelled as a quote that you build,
optionally share with the customer, and then apply to realize the change.

This skill covers the lifecycle state machine, the three-level object
hierarchy, the canonical flows, and the non-obvious preconditions on
apply.

Public docs: [docs.bunny.com/developer](https://docs.bunny.com/developer).

## Credential safety

Quote mutations modify billing state — a stray `quoteApplyChanges` on a
draft quote with the wrong price list bills the customer immediately.

- **Load tokens from environment variables only** (`process.env.BUNNY_ACCESS_TOKEN`,
  `ENV['BUNNY_ACCESS_TOKEN']`). Never hardcode, never log, never commit.
- **Scope tokens to `standard:read standard:write`** for the common
  lifecycle. Read-only analytics dashboards can drop to `standard:read`
  or `quoting:read`.
- **`applyOnAccept: true` shortens the state machine** — the quote
  applies immediately when the customer accepts, bypassing the server
  `quoteApplyChanges` step. Make sure that's what you want before
  setting it (common for self-service signup, dangerous for
  sales-assisted flows where a human should review first).
- **Rotate tokens on a cadence.**

## The three-level hierarchy

```text
Quote                   the envelope: state, account, contact, totals, signers
  └── QuoteChange       one per "thing you're doing" — a plan change, a coupon,
        │               a discount, a one-off adjustment
        └── QuoteCharge line items — each maps to a PriceListCharge (or a
              │         free-form adjustment) with quantity + price
              └── QuotePriceTier   (for tiered/volume/bands pricing models)
```

A single Quote can have **multiple QuoteChanges** — e.g. one `SUBSCRIBE`
change for a new plan plus one `DISCOUNT` change for an onboarding
coupon. On apply, each QuoteChange drives its own outcome
(Subscription / charge update / adjustment).

## Lifecycle state machine

```text
                ┌─────────────────┐
                │      DRAFT      │  ── quoteApplyChanges ──┐   server-driven,
                └────────┬────────┘                          │   no customer sign-off
                         │ (if approval rules match)         │
                         ▼                                   │
                ┌─────────────────┐                          │
                │  IN_APPROVAL    │  quoteApprovalStart      │
                └────────┬────────┘                          │
                         │ quoteApprove                      │
                         ▼                                   │
                ┌─────────────────┐                          │
                │    APPROVED     │                          │
                └────────┬────────┘                          │
                         │ quoteMarkAsShared                 │
                         ▼                                   │
                ┌─────────────────┐         ┌──────────┐     │
                │     SHARED      │ ──────▶ │  VIEWED  │     │   (customer opened it)
                └────────┬────────┘         └────┬─────┘     │
                         │ quoteAccept            │          │
                         ▼                        │          │
                ┌─────────────────┐               │          │
                │    ACCEPTED     │ ◀─────────────┘          │
                └────────┬────────┘                          │
                         │ quoteApplyChanges (or applyOnAccept)
                         ▼                                   ▼
                               Subscriptions + Invoice

Exits:  quoteReject  → REJECTED    (terminal from IN_APPROVAL / SHARED / VIEWED)
        (post-apply) → UNDONE      via quoteUndo (reverses an applied quote)
```

Two paths to apply:

- **DRAFT → APPLIED directly** via `quoteApplyChanges` — for server-driven flows
  (programmatic signup, ops tooling). The quote never enters the customer-visible
  SHARED state. The Quick Start below uses this path.
- **DRAFT → SHARED → VIEWED → ACCEPTED → APPLIED** — for sales-assisted or
  self-serve flows where the customer signs off before billing runs. See
  "Share / accept workflow" below.

Not every path hits every state — approval is only inserted when an
approval rule matches; `quoteMarkAsShared` can skip straight from DRAFT
when no approvals are required; `applyOnAccept` wires ACCEPTED → Apply
automatically without a separate mutation.

## Quick start: sell a new subscription

End-to-end: create the quote, attach a price list, apply it, get back
the subscription and its first invoice.

```graphql
# 1. Create the quote
mutation CreateQuote {
  quoteCreate(attributes: {
    accountId: "acc_xxx"
    contactId: "con_xxx"          # must have an email — checked on apply
    startDate: "2026-05-01"
    name: "Acme Corp — Starter Annual"
    applyOnAccept: false          # explicit; we'll apply server-side
  }) {
    quote { id state }
    errors
  }
}

# 2. Attach a plan's price list as a SUBSCRIBE change
mutation AddChange {
  quoteChangeCreate(
    quoteId: "qte_xxx"
    priceListId: "pli_xxx"
    evergreen: true               # auto-renew
    renewalTermMonths: 12
  ) {
    quoteChange { id kind }       # kind will be SUBSCRIBE
    errors
  }
}

# 3. Apply — creates the Subscription + first Invoice
mutation ApplyQuote {
  quoteApplyChanges(quoteId: "qte_xxx", persist: true, taxes: true) {
    quote { id state }
    subscriptions { id state plan { code } }
    invoice { id number amount }
    errors
  }
}
```

### Resolving `accountId` → `contactId` and `priceListCode` → `priceListId`

Most callers work in business-meaningful codes, not Bunny IDs. Two lookup
patterns you'll need before the Quick Start mutations:

```graphql
# Billing contact is a direct association on Account — use it for contactId
query ResolveContact { account(id: "acc_xxx") { billingContact { id email } } }

# Top-level `priceLists` is a Relay connection — wrap the selection in
# `nodes { ... }` (or `edges { node { ... } }`). Reading `{ id code }`
# directly off it will not compile.
query ResolvePriceList {
  priceLists(filter: "code = 'starter-monthly-usd'") {
    nodes { id code }
  }
}
```

The top-level `priceLists` query needs the **`product:read`** scope, not
just `standard:read`. Add it to the client when you're resolving codes
to IDs; drop it if you only need subscription/quote CRUD.

If an account doesn't have a `billingContact` set, resolve one via its
`contacts` association — `account(id: $id) { contacts { id email } }`
returns a plain list (not a connection), so `{ id email }` works
directly. The apply will fail with a clear error if the contact is
missing an email.

The apply output shape is `{ quote, subscriptions, invoice, errors }` —
**one invoice, potentially many subscriptions** (you could subscribe to
multiple plans in one quote). The invoice is `null` when nothing
billable was produced (e.g. a DISCOUNT-only change on an already-paid
period). Two defensive habits that pay off in practice:

- **The quote does not transition to an "APPLIED" state.** `QuoteState`
  has no `APPLIED` value — after a successful apply, the quote stays in
  whatever state it was in (typically ACCEPTED, or DRAFT if you applied
  directly from draft). The "this quote ran" signal is the produced
  Subscription + Invoice, not a state flip.
- **Fall back to `quote.quoteChanges[].subscription`** if
  `subscriptions: []` is empty despite a successful apply — the
  subscription is always reachable through the QuoteChange it came
  from. Query both paths in the selection set so your caller can
  handle either shape.

## `QuoteChangeKind` — pick the right kind for the job

`quoteChangeCreate` returns a change with a kind inferred from the
inputs. When you need finer control, use the specialised mutations
below — they set the kind for you and validate the preconditions:

| Kind | How you create it | What it does on apply |
| --- | --- | --- |
| `SUBSCRIBE` | `quoteChangeCreate(priceListId, quoteId)` | Creates a new subscription on this plan |
| `RENEW` | `quoteChangeCreateRenew(subscriptionId, priceListId, quoteId)` | Extends the subscription into the next term |
| `UPDATE` | `quoteChangeUpdate(id, charges)` | Modifies charges on an existing subscription |
| `QUANTITY_UPDATE` | `quoteChargeUpdate(quoteChargeId, quantity)` | Seat change; prorated on apply |
| `PRICE_UPDATE` | `quoteChargeUpdate(quoteChargeId, price)` | Explicit override; usually ops/support only |
| `UNSUBSCRIBE` | `quoteSubscriptionCancel` → quote | Cancels the subscription at period end or on a date |
| `REINSTATE` | `quoteSubscriptionReinstate(ids)` | Undoes a prior cancel |
| `DISCOUNT` / `PERCENTAGE_DISCOUNT` / `FREE_PERIOD_DISCOUNT` | `quoteChangeDiscountCreate` / `quoteChargeFreeMonthsCreate` | Applies a discount against the subscription |
| `COUPON` | `quoteChangeAddCoupon(couponCode, quoteChangeId)` | Attaches a catalog coupon to a change |
| `ADJUSTMENT` | `quoteChargeCreate` with an adjustment charge | One-time charge / credit on the invoice |

You can have multiple QuoteChanges with different kinds on the same
Quote — that's the main reason the hierarchy exists.

## Canonical flows

### Renewal

Use the shortcut — it creates a Quote + a RENEW QuoteChange with
charges copied from the current subscription and any configured price
adjustments applied:

```graphql
mutation RenewSubscription {
  quoteSubscriptionRenew(ids: ["sub_xxx"], priceListId: "pli_new") {
    quote { id state }
  }
}
# Then: quoteApplyChanges(quoteId, persist: true, taxes: true)
```

`priceListId` is optional — omit to renew on the same price list.

**Don't select `errors` on this payload.** `QuoteSubscriptionRenewPayload`
declares `errors: [String!]!` as non-null, but the resolver returns
`null` on success — requesting `errors` in the selection cascades the
whole payload to `null` via GraphQL's non-null propagation rule. Omit
the field; catch real failures via the surrounding HTTP-level error
envelope (see `bunny-graphql` for error-shape patterns).

Subscriptions that mix `ONE_TIME` charges with `RECURRING` ones cannot
be renewed via this shortcut — the underlying operation rejects with
`Cannot renew one-time charges`. For those, use the full quote flow
(`quoteCreate` → `quoteChangeCreateRenew` → `quoteApplyChanges`) and
drop the one-time charges from the renewal QuoteChange manually.

### Upgrade (plan change)

```graphql
mutation UpgradeSubscription {
  quoteSubscriptionUpgrade(subscriptionId: "sub_xxx", priceListId: "pli_pro") {
    quote { id state }
    errors
  }
}
# Apply to realize the upgrade
```

The upgrade produces a quote whose kind is inferred from the plan delta;
apply it to transition the subscription, prorate the current period, and
bill the difference on the next invoice.

### Add-on

```graphql
mutation AddOnSubscription {
  quoteSubscriptionAddon(subscriptionId: "sub_xxx", priceListId: "pli_addon") {
    quote { id state }
    errors
  }
}
```

Add-on price lists are distinct catalog entries — make sure the
plan is marked `addon: true` in the catalog (see `bunny-catalog`).

### Quantity change

For seats / units tied to a usage feature:

```graphql
mutation BumpSeats {
  quoteCreate(attributes: {
    accountId: "acc_xxx"
    contactId: "con_xxx"
    startDate: "2026-05-01"
  }) { quote { id } errors }
}

mutation ChangeQuoteCharge {
  quoteChargeUpdate(
    quoteChargeId: "qch_xxx"      # the seats charge from the subscription
    quantity: 25
  ) {
    quoteCharge { id quantity subtotal }
    errors
  }
}
```

`quoteSubscriptionUpdate(subscriptionIds: […])` is a higher-level shortcut
that creates a quote with UPDATE changes for all charges on the given
subscriptions — useful when you want the integrator to edit the whole
subscription in one place.

### Apply a coupon mid-quote

```graphql
mutation ApplyCoupon {
  quoteChangeAddCoupon(couponCode: "LAUNCH20", quoteChangeId: "qch_xxx") {
    quoteChange { id kind coupon { couponCode } }
    errors
  }
}
```

Coupons are top-level catalog objects; see `bunny-catalog` for how
they're created and scoped. `quoteChangeRemoveCoupon` reverses the
attachment.

### Sales-assisted flow with deal → quote

When a salesperson is driving the deal, create the quote alongside a
Deal record so the pipeline tracks it:

```graphql
mutation CreateQuoteWithDeal {
  quoteCreateWithDeal(
    accountId: "acc_xxx"
    dealAttributes: {
      name: "Acme Q3 expansion"
      stage: "negotiation"
      ownerId: "usr_xxx"
    }
    quoteAttributes: {
      contactId: "con_xxx"
      startDate: "2026-05-01"
      expiresAt: "2026-05-15"
    }
  ) {
    quote { id deal { id name stage } }
    errors
  }
}
```

The deal remains the umbrella — a single deal can span multiple quote
revisions (via `quoteDuplicate`) as negotiations continue.

## Share / accept workflow

For sales-driven flows (customer countersigns), use the share → accept
path instead of applying directly:

```graphql
# Make the quote viewable by the contact (sends the share email per
# emailSubject / emailBody on QuoteAttributes)
mutation ShareQuote { quoteMarkAsShared(id: "qte_xxx") { quote { id state } errors } }

# Customer clicks the portal link, reviews, and accepts with their
# name + title. (The portal UI handles this; integrators rarely call
# quoteAccept directly unless building a bespoke signing flow.)
mutation AcceptQuote {
  quoteAccept(quoteId: "qte_xxx", name: "Jane Doe", title: "CEO") {
    quote { id state acceptedAt }
    errors
  }
}
```

After ACCEPTED, either:

- **Server applies** via `quoteApplyChanges` (explicit, auditable)
- **Auto-applies** if `applyOnAccept: true` was set on the quote — the
  Apply fires inside the same transaction as Accept, so the customer
  gets their subscription + first invoice synchronously with the click

Use auto-apply for self-service signup; use explicit apply for
sales-assisted deals where a human should eyeball the accepted quote
before billing runs.

For digital-signature variants (`quoteSignatureCreate`,
`quoteSignersSendEmail`, `quoteContactSignerIds`), see the hosted
customer-portal flow in `bunny-customer-portal`.

## Price adjustments

Subscriptions carry `priceAdjustmentAction` + `priceAdjustmentTiming` +
`priceAdjustmentPercentage`. Apply them ahead of renewal:

```graphql
mutation AdjustSubscriptionPrice {
  quoteApplyPriceAdjustments(
    id: "qte_xxx"
    priceAdjustmentPercentage: 4.5     # 4.5% bump
    startDate: "2026-06-01"
  ) {
    quote { id }
    errors
  }
}
```

A per-QuoteChange variant (`quoteChangeApplyPriceAdjustment`) lets you
bump just one plan change when a quote spans multiple subscriptions.

See `bunny-catalog` for the underlying `PriceAdjustmentAction` /
`PriceAdjustmentTiming` enums that control when adjustments apply on
renewal.

## Approvals

When an `ApprovalRule` matches (on discount %, product, plan, total
amount, etc.), `quoteMarkAsShared` is blocked until the rule's approver
signs off:

```graphql
# DRAFT → IN_APPROVAL
mutation StartApproval { quoteApprovalStart(id: "qte_xxx") { quote { id state } errors } }

# IN_APPROVAL → APPROVED (only the configured approver can call this)
mutation Approve { quoteApprove(id: "qte_xxx", notes: "Discount justified — strategic account") { quote { id state } errors } }

# Or reject (terminal — clone with quoteDuplicate to revise)
mutation Reject { quoteReject(id: "qte_xxx", notes: "Discount exceeds delegation") { quote { id state } errors } }

# Cancel approval and return to DRAFT
mutation Cancel { quoteApprovalCancel(id: "qte_xxx") { quote { id state } errors } }
```

Approval rules themselves are configured in the admin portal. The
approval surface is a gate on top of the normal lifecycle — if no rule
matches, quotes proceed directly from DRAFT to SHARED via
`quoteMarkAsShared`.

## Ramp deals (stepped quantity or discount)

Ramp deals model "quantity grows over time" or "intro pricing steps
down" contracts without N separate quotes. Each ramp interval is a
distinct QuoteCharge with its own quantity + discount. The ramp input
reuses `QuoteChargeAttributes` (same input type as regular charges;
Bunny fills in the start/end dates per ramp interval):

```graphql
mutation RampContract($charges: [QuoteChargeAttributes!]!) {
  quoteChangeCreateRampUp(
    quoteId: "qte_xxx"
    priceListId: "pli_seats"
    rampIntervalMonths: 3               # each tier lasts 3 months
    charges: $charges
  ) {
    quoteChange { id kind rampIntervalMonths }
    errors
  }
}
```

```json
{
  "charges": [
    { "quantity": 1, "discount": 50 },
    { "quantity": 2, "discount": 25 },
    { "quantity": 3, "discount": 0 },
    { "quantity": 4, "discount": 0 }
  ]
}
```

Constraints (enforced by the API, worth knowing before you build a UI):

- Recurring charges only — one-time and usage charges cannot be ramped
- Single-charge price lists only (one charge in the price list)
- Same plan cannot appear twice in a quote
- Custom quantity start dates must align with ramp interval boundaries

Use `quoteChangeCreateRampUpPreview` to compute the ramped charges
without writing them, when you want to show the customer a proposed
schedule before committing to the ramp. Typical pattern: preview in a
sales-UI drawer, let the rep tweak the intervals, then call
`quoteChangeCreateRampUp` once they're happy. Whether to apply the
resulting quote is a separate decision — leave it in DRAFT when a
human needs to approve the schedule; apply directly when the script
is running against an already-approved contract.

## Undoing an applied quote

`quoteUndo` reverses an APPLIED quote's effects (cancels the subscription
it created, voids the invoice it generated, etc.) and transitions the
quote to UNDONE. Useful for operational reversals within a short window;
past the invoice-paid threshold you generally want a credit note instead
(see `bunny-billing`).

## References

- [docs.bunny.com/developer](https://docs.bunny.com/developer) — prose docs
- Sibling skills: `bunny-catalog` (price lists, coupons, price adjustments), `bunny-subscriptions` (post-apply subscription lifecycle, renewals-without-quotes), `bunny-customer-portal` (hosted accept UI), `bunny-components` (`Quote` / `Quotes` React embeds), `bunny-billing` (invoice / credit-note handling), `bunny-webhooks` (Quote workflow events for pipeline integration)

---

Last verified against bunnyapp/api release 2026-04-21-1
