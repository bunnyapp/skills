---
name: bunny-catalog
description: Build and maintain a product catalog in Bunny — the subscription billing and management platform. Covers the object hierarchy (ProductCategory → Product → Plan → PriceList → PriceListCharge), Features for entitlement and metered billing, pricing models (flat, tiered, volume, bands) with tier configuration, charge types (recurring, one-time, usage) and billing periods, trial and renewal-term configuration on price lists, coupons (amount vs percentage, plan-scoped vs global), safe price versioning via price-list deprecation and duplication, and one-way catalog-sync patterns for mirroring an external source of truth into Bunny. Use when creating or updating products, plans, features, prices, or coupons; when modelling a pricing table; when implementing a catalog-sync job; or when versioning prices without breaking active subscriptions.
---

# Bunny Catalog

Bunny's catalog is what customers can subscribe to. This skill covers the
object model, the mutations that build and update it, and the patterns for
keeping a catalog in sync with an external source of truth.

For the prose docs, see
[docs.bunny.com/developer](https://docs.bunny.com/developer).

## Credential safety

Catalog mutations are write operations against live billing data. A stray
`planUpdate` or `priceListChargeUpdate` can change what every new
subscriber pays.

- **Load access tokens from environment variables only**
  (`process.env.BUNNY_ACCESS_TOKEN`, `ENV['BUNNY_ACCESS_TOKEN']`). Never
  hardcode, never log, never commit.
- **Scope tokens to `product:write`** (plus `product:read`) for catalog
  jobs. Don't hand catalog tooling a `billing:write` or `admin:write`
  token it doesn't need.
- **Rotate on a cadence** and rotate immediately on suspected leak.
- **Dry-run first** when wiring up a catalog-sync job — log the diff the
  job would apply, then turn on writes.

## The catalog mental model

```text
ProductCategory          "SaaS", "Add-ons", "Professional services"
  └── Product            "Pro", "Enterprise"        ← metadata + plan grouping
        ├── Feature      "Seats", "API calls"       ← entitlement + usage
        └── Plan         "Pro Monthly", "Pro Annual"← what a customer picks
              └── PriceList         one per currency / renewal term variant
                    └── PriceListCharge  the actual line items with prices
```

Two things that trip people up:

1. **Products don't have prices.** Prices live on `PriceListCharge`, two
   levels down. A product is a grouping construct.
2. **A plan can have multiple price lists.** USD vs EUR, monthly vs annual,
   a promotional price list for a campaign — each is its own `PriceList`.
   Subscribers are pinned to the price list they signed up on, which is
   what makes safe price changes possible (see "Versioning prices" below).

`ProductCategory` is optional and purely organisational — use it once you
have more than ~5 products.

## Quick start: create a plan with one price

End-to-end: a "Pro" product, one plan, a monthly USD price list, a single
$49/mo recurring charge.

```graphql
# 1. Create the product
mutation CreateProduct {
  productCreate(attributes: {
    code: "pro"
    name: "Pro"
    description: "For growing teams"
  }) {
    product { id code }
    errors
  }
}

# 2. Create a plan under that product
mutation CreatePlan {
  planCreate(attributes: {
    productId: "prd_xxx"      # from step 1
    code: "pro-monthly"
    name: "Pro Monthly"
    isVisible: true
  }) {
    plan { id code }
    errors
  }
}

# 3. Create a price list under that plan (one per currency/term)
mutation CreatePriceList {
  priceListCreate(attributes: {
    planId: "pln_xxx"         # from step 2
    currencyId: "cur_usd"     # look up with { currencies { id code } }
    code: "pro-monthly-usd"
    name: "Pro Monthly USD"
    renewalTermMonths: 1
    trialAllowed: false
  }) {
    priceList { id code }
    errors
  }
}

# 4. Attach a recurring charge — this is the actual price
mutation CreateCharge {
  priceListChargeCreate(attributes: {
    priceListId: "pli_xxx"    # from step 3
    code: "pro-monthly-base"
    name: "Pro subscription"
    chargeType: RECURRING
    billingPeriod: MONTHLY
    pricingModel: FLAT
    price: "49.00"
  }) {
    priceListCharge { id price }
    errors
  }
}
```

Every mutation returns `errors: [String!]` (array of plain strings — check
it on every response).

## Features: entitlement and metered billing

Features belong to a **Product** and get attached to **Plans** via the
plan's `featureIds`. They serve two purposes:

- **Entitlement flags** — "Pro includes SSO" (`BOOLEAN`)
- **Quantity limits** — "Pro allows 1,000,000 API calls" (`QUANTITY`)
- **Plan-level values** — "Pro API rate limit is 600/min" (`VALUE` — a
  string value surfaced to your app at runtime, not used for billing)
- **Groupings** — `GROUP` collects several features under one label for
  display

```graphql
mutation CreateSeatsFeature {
  featureCreate(
    productId: "prd_xxx"
    attributes: {
      code: "seats"
      name: "Seats"
      kind: QUANTITY                 # BOOLEAN | QUANTITY | VALUE | GROUP
      isUnit: true
      isProvisioned: true            # surfaces in provisioning webhook payload
      usageCalculationType: MAX      # AVERAGE | MAX | LAST | SUM
    }
  ) {
    feature { id code }
    errors
  }
}
```

`usageCalculationType` governs how Bunny aggregates `featureUsage` records
across a billing period:

| Value | How the period is billed |
| --- | --- |
| `SUM` | Total of all records in the period |
| `MAX` | Highest single value reported |
| `LAST` | Value of the most recent record |
| `AVERAGE` | Mean across all records |

`MAX` is a common choice for seat-based billing — customers can add and
remove seats during the period without gaming the bill.

Attach features to a plan. `featureIds` works on both `planCreate` and
`planUpdate` — pass the full list; it replaces what was there, so read
the current set before writing if you only want to add one:

```graphql
mutation UpdatePlanFeatures {
  planUpdate(id: "pln_xxx", attributes: {
    featureIds: ["ftr_seats", "ftr_sso", "ftr_api_calls"]
  }) {
    plan { id featureIds }
    errors
  }
}
```

Attaching a feature creates a `PlanFeature` join row with an empty
value. Set the **per-plan value** (e.g. "Pro includes 25 seats") as a
**separate second call** via `planFeatureUpdate`. The value field is a
string regardless of the feature's kind — a `BOOLEAN` feature takes
`"true"` / `"false"`, a `QUANTITY` feature takes the count as a string,
a `VALUE` feature takes any string:

```graphql
mutation SetSeatsOnPro {
  planFeatureUpdate(id: "plf_pro_seats", attributes: {
    value: "25"
  }) {
    planFeature { id value }
    errors
  }
}
```

Then price the feature per-unit with a `USAGE` or `RECURRING` charge on
the price list (see pricing models below).

## Pricing models

A `PriceListCharge` takes one of three `pricingModel` values:

| `pricingModel` | Shape | When to use |
| --- | --- | --- |
| `FLAT` | One `price` for the whole charge | Simple plans, add-ons |
| `TIERED` | Price varies by unit slab; each tier priced separately | "$0.01/call for first 10k, $0.008 after" |
| `VOLUME` | All units priced at the tier the total falls into | "100 seats? You pay tier-3 rate on all 100" |
| `BANDS` | Fixed price per tier band (flat fee per range, not per unit) | "$99/mo for 1–10 users, $199 for 11–50" |

Tiered / volume / bands charges use `priceListChargeTiers`. The input is
minimal — each tier declares only where it **starts** and its **price**.
The next tier's `starts` implicitly ends the previous one; the highest
`starts` tier is open-ended.

```graphql
mutation CreateTieredCharge {
  priceListChargeCreate(attributes: {
    priceListId: "pli_xxx"
    featureId: "ftr_api_calls"        # link to the usage feature
    code: "api-calls-tier"
    name: "API calls"
    chargeType: USAGE
    billingPeriod: MONTHLY
    pricingModel: TIERED
    priceListChargeTiers: [
      { starts: 1,       price: "0.01"  }   # 1–10,000 units
      { starts: 10001,   price: "0.008" }   # 10,001–100,000 units
      { starts: 100001,  price: "0.005" }   # 100,001+ (open-ended)
    ]
  }) {
    priceListCharge { id }
    errors
  }
}
```

`PriceListChargeTierAttributes` has exactly two fields, `starts` and
`price`. There is no `quantityMin`, `quantityMax`, or `ends` — those are
charge-level bounds on `PriceListChargeAttributes`, not tier fields.

**Tier updates are replace-wholesale.** Pass the full tier array on
`priceListChargeUpdate` and Bunny rebuilds the list; there is no
per-tier `id` you thread through. Sort by `starts` if you care about
avoiding spurious diffs.

## Charge types and billing periods

| `chargeType` | Billed | Typical use |
| --- | --- | --- |
| `RECURRING` | Every billing period | Base plan fee, seat licence |
| `ONE_TIME` | Once on the initial invoice | Setup fee, onboarding |
| `USAGE` | Every billing period, based on reported `featureUsage` | Metered API calls, consumed storage |

| `billingPeriod` | Meaning |
| --- | --- |
| `MONTHLY` | Charged monthly |
| `QUARTERLY` | Every 3 months |
| `SEMI_ANNUAL` | Every 6 months |
| `ANNUAL` | Yearly |
| `ONCE` | One-time charge that doesn't repeat (use with `chargeType: ONE_TIME`) |

The **price list's** `renewalTermMonths` determines the subscription's
overall cadence (e.g. an annual price list still renews once a year even
if it contains monthly-billed charges — those just invoice monthly within
the annual term).

## Trials

Trials are configured on the **price list**, not the plan:

```graphql
mutation EnableTrial {
  priceListUpdate(id: "pli_xxx", attributes: {
    trialAllowed: true
    trialLengthDays: 14
    trialExpirationAction: ACTIVATE  # ACTIVATE | CANCEL
  }) {
    priceList { id trialLengthDays }
    errors
  }
}
```

`trialExpirationAction`:

- `ACTIVATE` — after the trial ends, subscription becomes active and
  invoices start (card required unless customer-portal allows none).
- `CANCEL` — trial auto-cancels if the customer doesn't convert.

Different trial policies? Make a separate price list (e.g.
`pro-monthly-usd-14day` vs `pro-monthly-usd-no-trial`).

## Renewal term and price adjustment

On a price list:

- `renewalTermMonths` — e.g. `12` for an annual contract.
- `priceAdjustmentAction` — `CATALOG` (apply current catalog prices on
  renewal), `NO_CHANGE` (lock the subscriber's original prices), or
  `PERCENTAGE` (apply a flat percentage bump on renewal).
- `priceAdjustmentTiming` — when to apply the adjustment. Options are
  `AT_RENEWAL` or any month name (`JANUARY`, `FEBRUARY`, …, `DECEMBER`)
  for a fixed annual cutover.

`CATALOG` + `AT_RENEWAL` is the common "price changes take effect when
the customer renews" setup. `NO_CHANGE` pins subscribers to their
original prices forever — useful for grandfathering.

## Versioning prices without breaking subscriptions

**Existing subscribers are pinned to the price list they signed up on.**
Changing a `priceListCharge.price` will not affect existing subscribers
unless the price list is configured with `priceAdjustmentAction: CATALOG`.

Pattern for a clean price bump:

1. **`priceListDuplicate(id: "pli_old")`** — clone into a new price list
   with a new code (e.g. `pro-monthly-usd-v2`).
2. Edit charges on the duplicate.
3. Point new sign-ups at the new price list code.
4. **`priceListDeprecate(id: "pli_old", replacementPriceListId: "pli_new")`**
   — marks the old list deprecated. Existing subscribers stay put; new
   sign-ups are routed to the replacement.
5. `priceListUndoDeprecate` reverses it if needed.

Same lifecycle exists on individual charges:
`priceListChargeDeprecate(id, removeOnRenewal: true)` retires a charge
but optionally keeps billing existing subscribers until their next
renewal.

## Coupons

Coupons are top-level objects, optionally scoped to a plan:

```graphql
mutation CreateLaunchCoupon {
  couponCreate(
    planId: "pln_xxx"            # omit for a global coupon
    attributes: {
      couponCode: "LAUNCH20"
      description: "20% off first 3 months"
      discountKind: PERCENTAGE   # AMOUNT | PERCENTAGE
      discountAmount: 20
      active: true
    }
  ) {
    coupon { id couponCode }
    errors
  }
}
```

`discountKind: AMOUNT` uses `discountAmount` as a currency value and
requires `currencyId`. `PERCENTAGE` treats it as a percentage (0–100).

## Tax

Bunny does not have a top-level `TaxCategory` object. Tax is configured
on two axes:

- **`taxCode: String`** on `priceListChargeCreate` / `priceListChargeUpdate`
  — a free-text tax code the tax plugin interprets (e.g. Avalara SKU,
  Stripe Tax product tax code).
- **Tax plugins** — configured separately per warren (Avalara, Stripe
  Tax, etc.). Plugin setup is outside the catalog surface.

If you need Avalara AFC telecom categorisation, `priceListChargeCreate`
also accepts `avalaraAfcSaleType`, `avalaraAfcServiceType`, and
`avalaraAfcTransactionType`.

## Catalog-sync pattern (external source of truth)

If your pricing lives in a CMS, spreadsheet, or pricing engine and Bunny
is the biller, implement a one-way sync:

1. **List the source catalog** (from your system).
2. **List the Bunny catalog** — query `products { nodes { id code plans { nodes { id code priceLists { nodes { id code } } } } } }`.
3. **Diff on `code`** — stable, human-assigned, same across environments.
4. **Upsert** — create or `planUpdate` / `priceListUpdate` /
   `priceListChargeUpdate` as needed.
5. **Log the diff before writing** on first runs; turn on writes after a
   human review.

Catalog-list queries are Relay-style connections; paginate with `first:
N, after: <cursor>` and `pageInfo { hasNextPage endCursor }`. Many
list fields (`products`, `plans`, `priceLists`, `features`, `coupons`)
also accept a `filter: String` argument — pass the code (or a
substring) to narrow a fetch to the object you care about instead of
scanning the whole catalog. See `bunny-graphql` for the general
pagination contract.

Rules of thumb:

- **Use your `code` fields as the stable identity**, not Bunny's IDs.
  Codes survive environment promotion (dev → staging → prod), Bunny IDs
  do not.
- **Never delete, deprecate.** `priceListDeprecate` and
  `priceListChargeDeprecate` preserve billing continuity for existing
  subscribers. `*Delete` mutations are for mistakes caught before anyone
  subscribes.
- **Check `errors` on every response** and fail the sync run rather than
  leaving the catalog in a half-applied state.

### Helpers: duplicate and bulk import

- **`productDuplicate(id)`** / **`planDuplicate(id)`** — clone an entire
  product (with its plans and price lists) or a single plan. Useful when
  spinning up a variant (regional, partner-branded) off an existing
  structure.
- **`productImport(attributes: JSON!)`** — accepts a JSON blob describing
  multiple products, plans, features, and price lists in one call. Good
  for seeding a fresh warren from a pricing spreadsheet; the shape is
  Bunny-specific and outside the scope of this skill — see the product
  import docs at [docs.bunny.com/developer](https://docs.bunny.com/developer)
  for the schema.

## Sibling skills

- [`bunny-graphql`](../bunny-graphql/SKILL.md) — auth, pagination, error
  shape used by every mutation above.
- [`bunny-node-sdk`](../bunny-node-sdk/SKILL.md) /
  [`bunny-ruby-sdk`](../bunny-ruby-sdk/SKILL.md) — typed wrappers for
  these mutations.
- [`bunny-quoting`](../bunny-quoting/SKILL.md) — once the catalog exists,
  quotes reference price lists and charges.
- [`bunny-subscriptions`](../bunny-subscriptions/SKILL.md) — renewals,
  upgrades, usage reporting against the features defined here.
- [`bunny-webhooks`](../bunny-webhooks/SKILL.md) — catalog object changes
  (plan, feature, coupon) can drive workflow webhooks.

## References

- [docs.bunny.com/developer](https://docs.bunny.com/developer) — prose docs
- [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) — Node SDK
- [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) — Ruby gem

---

Last verified against bunnyapp/api release 2026-04-21-1
