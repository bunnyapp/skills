---
name: bunny-subscriptions
description: Manage subscription lifecycle in Bunny. Covers the state machine (PENDING → ACTIVE → CANCELED / EXPIRED / TRIAL / TRIAL_EXPIRED), direct creation via `subscriptionCreate` vs quote-based creation, the nightly cron driving automated renewals / expirations / trial transitions, the `evergreen` auto-renew gate, explicit lifecycle mutations (`subscriptionCancel`, `subscriptionReinstate`, `subscriptionSetAutoRenew`, `subscriptionTrialExtend`, `subscriptionTrialConvert`), quantity updates (`subscriptionQuantityUpdate`), tenant metadata (`subscriptionTenantUpdate`), metered-usage reporting with `featureUsageCreate` / `featureUsageUpdate` / `featureUsageDelete` (requires `billing:write`), add-on co-renewal rules, price-adjustment timing on renewal, and when to reach for a quote (`bunny-quoting`) vs a direct mutation. Use when creating, modifying, renewing, or cancelling subscriptions; when reporting metered usage; or when orchestrating trial conversions.
---

# Bunny Subscriptions

A **subscription** is the live billing relationship between an account and
a plan in Bunny. This skill covers the state machine, the automated-vs-manual
transitions, the shortcut mutations that bypass the full quote workflow,
and metered-usage reporting.

Quote-based lifecycle changes (plan upgrades, renewals with price-list
changes, one-time adjustments) are covered in `bunny-quoting` — prefer
that skill for any flow where the customer reviews or signs off before
the change lands.

Public docs: [docs.bunny.com/developer](https://docs.bunny.com/developer).

## Credential safety

Subscription mutations change live billing state — a miscounted
`quantities` on `subscriptionQuantityUpdate` with
`invoiceImmediately: true` invoices the customer in seconds.

- **Load tokens from environment variables only.** Never hardcode, never
  log, never commit.
- **Scope tokens narrowly.** `standard:read standard:write` covers most
  flows; metered usage reporting requires `billing:write`
  (see "Metered usage" below).
- **Rotate tokens** on a cadence and immediately on suspected leak.

## State machine

```text
                  subscriptionCreate
                   (with trial: true)
                          │
                          ▼
                     ┌─────────┐
                     │  TRIAL  │ ── trialEndDate + ACTIVATE ──► ACTIVE
                     └────┬────┘ ── trialEndDate + CANCEL ────► TRIAL_EXPIRED
                          │
                          │ subscriptionTrialConvert
                          ▼
                     ┌─────────┐
  subscriptionCreate │ ACTIVE  │ ── end_date reached, evergreen ──► ACTIVE (renewed)
  (trial: false) ──► │         │ ── end_date reached, non-evergreen ► EXPIRED
                     │         │ ── subscriptionCancel + date ─────► CANCELED (at date)
                     └─────────┘
                          ▲
                          │ subscriptionReinstate
                     ┌─────────┐
                     │ PENDING │ ── start_date reached ──► ACTIVE
                     └─────────┘
```

Subscriptions created with a future `startDate` land in `PENDING` until
the cron activates them; those created with `trial: true` start in
`TRIAL` and follow the price list's `trialExpirationAction` at
`trialEndDate`.

Two state transitions run automatically via nightly cron **per entity**
(Bunny's billing-entity abstraction, typically one per warren):

1. Expire non-evergreen subscriptions whose `endDate <= today`
2. Renew evergreen subscriptions whose `endDate <= today`
3. Activate pending subscriptions whose `startDate <= today`
4. Convert trials per `trialExpirationAction` at `trialEndDate`
5. Cancel subscriptions where `cancellationDate <= today`

The cron is idempotent — re-running it on the same day is a no-op — so
you can trigger a catch-up safely from support tooling when needed.

## Two ways to create a subscription

### Direct — `subscriptionCreate`

Straight to ACTIVE (or TRIAL) with no quote, no customer acceptance,
no approval gate. Typical for:

- Self-service signup after a billing form your app already rendered
- Programmatic tenant provisioning (API client → subscription atomically)
- Ops tooling creating a subscription on behalf of a customer

```graphql
mutation CreateSub {
  subscriptionCreate(
    billingDay: 1                    # optional — align billing to day of month
    attributes: {
      accountId: "acc_xxx"           # or accountCode, or account: { … } to create inline
      priceListCode: "starter-monthly-usd"
      startDate: "2026-05-01"
      tenantCode: "acme-team"        # optional — provision tenant in the same call
      tenantName: "Acme Team"
      trial: false
      renewalTermMonths: 12
    }
  ) {
    subscription { id state plan { code } tenant { id code } }
    errors
  }
}
```

Using `tenantCode` requires tenant provisioning to be enabled on the
product's Platform (admin portal → Products → your product → Platform).
Without that, the call fails with `Tenant provisioning is not enabled`.

**`subscriptionCreate` will not accept an add-on price list.** Plans
marked `addon: true` require a parent quote change and fail with
`Parent quote change must be provided for an addon price list`. Create
add-on subscriptions through `quoteSubscriptionAddon` in `bunny-quoting`
instead — that mutation wires the add-on to its parent subscription
correctly.

### Quote-based — via `bunny-quoting`

When the customer needs to review pricing, accept terms, or go through
approvals, build a quote instead. The final `quoteApplyChanges` produces
the subscription(s) + first invoice atomically. See `bunny-quoting` for
the full flow.

Rule of thumb: **if a human has to click "Accept" somewhere, use a
quote**. If not, prefer `subscriptionCreate` for simpler integration.

## Cancellation

`subscriptionCancel` **schedules** a cancellation — it doesn't move the
subscription to CANCELED immediately (unless you pass today's date):

```graphql
mutation Cancel {
  subscriptionCancel(ids: ["sub_xxx"], cancellationDate: "2026-05-31") {
    subscriptions { id state cancellationDate }
    errors
  }
}
```

The cron moves the subscription from ACTIVE to CANCELED on the
`cancellationDate` (this is also when the customer loses access). Before
that date the subscription is still ACTIVE and continues to invoice on
its normal schedule.

**`cancellationDate` must fall within the subscription's active term**
— between `startDate` and `endDate`. A date past `endDate` (e.g. a
"safe far future" sentinel like `2099-12-31`) is rejected with
`Cancellation date must be within the subscription period`. To cancel
at term end, pass the subscription's `endDate` verbatim; to cancel
further out, bump `endDate` first via a renewal.

A cancel response can come back with an empty `subscriptions: []` on
success; re-query the subscription afterwards to confirm
`cancellationDate` is set rather than relying solely on the mutation
payload.

To cancel **at the end of the current term** (common self-service
behaviour):

```graphql
# cancellationDate = subscription.endDate — customer keeps access until
# the term they already paid for ends.
```

`subscriptionReinstate` undoes a cancellation before it takes effect (or
revives an already-CANCELED subscription if the customer comes back):

```graphql
mutation Reinstate {
  subscriptionReinstate(
    ids: ["sub_xxx"]
    effectiveDate: "2026-06-01"
    paymentId: "pay_xxx"             # re-use a stored payment method
    taxes: true
  ) {
    subscriptions { id state }
    errors
  }
}
```

## Auto-renew toggle

`evergreen: true` is the gate that lets the nightly cron renew a
subscription. Flip it on an existing subscription without changing
anything else:

```graphql
mutation DisableAutoRenew {
  subscriptionSetAutoRenew(ids: ["sub_xxx"], evergreen: false) {
    subscriptions { id evergreen }
    errors
  }
}
```

Disabling evergreen doesn't cancel — it just means the subscription
will EXPIRE at its `endDate` instead of being renewed. Use this when
the customer wants to lock in "no auto-renew" without the billing
impact of a full cancellation.

**Renewals with plan changes** (switching price lists, applying
price-adjustments, renegotiating terms) go through the quote workflow
— see `quoteSubscriptionRenew` in `bunny-quoting`. The nightly cron
only auto-renews evergreen subscriptions on their **current** price list
with whatever price adjustments are configured on the subscription row.

## Trials

Trials are a subscription-level state, configured via the price list's
trial settings (`trialAllowed`, `trialLengthDays`,
`trialExpirationAction`) and driven by the cron at `trialEndDate`.

Extend a trial before it expires (sales-accommodated cases):

```graphql
mutation ExtendTrial {
  subscriptionTrialExtend(id: "sub_xxx", trialEndDate: "2026-06-15") {
    subscription { id state trialEndDate }
    errors
  }
}
```

Convert a trial to paid explicitly (e.g. the customer adds a payment
method via the portal):

```graphql
mutation ConvertTrial {
  subscriptionTrialConvert(
    subscriptionId: "sub_xxx"
    priceListCode: "starter-monthly-usd"   # or priceListId
    paymentId: "pay_xxx"                   # stored payment method
  ) {
    subscription { id state }
    invoice { id number amount }
    paymentApplications { id amount }      # the payment auto-applies to the invoice
    errors
  }
}
```

Conversion produces the first real invoice, auto-applies the payment
(the `paymentApplications` array is how you reconcile which portion of
the payment landed on which invoice), and transitions the state from
TRIAL to ACTIVE. For the deprecated
`subscriptionTrialConvertPreview`, use `quoteSubscriptionActivate`
instead — it produces a preview quote you can inspect before applying.

## Quantity changes (no quote needed)

For seat or unit changes where you don't need customer review,
`subscriptionQuantityUpdate` short-circuits the quote workflow. Proration
is computed and (optionally) invoiced in the same call:

```graphql
mutation BumpSeats {
  subscriptionQuantityUpdate(
    subscriptionId: "sub_xxx"
    quantities: [
      { code: "seats", quantity: 25 }       # SubscriptionChargeQuantityAttributes
    ]
    startDate: "2026-05-15"                # prorate from this date
    invoiceImmediately: true               # generate the prorated invoice now
    name: "June headcount bump"
    allowQuantityLimitsOverride: false     # respect plan min/max
  ) {
    quote {
      id
      invoices { id number amount }        # the prorated invoice lives here
    }
    errors
  }
}
```

`code` is the `PriceListCharge.code` — plans whose charges were created
without a code can't be updated via this mutation (the input requires a
non-null `code`). If you're designing a new catalog, set `code` on every
`priceListChargeCreate` so subscriptions built from it remain editable.

`SubscriptionQuantityUpdatePayload` has only `{ quote, errors }` — no
top-level `invoice`. The prorated invoice is nested under `quote.invoices`
because the mutation produces the change as a quote under the hood.

`invoiceImmediately: true` is the production-correct setting most of
the time — without it, the delta gets folded into the next regular
invoice and the subscription temporarily shows a mismatch between its
current quantities and what's been billed.

For **price** changes on a subscription charge (not quantity), reach
for the quote workflow — see `quoteChargeUpdate` in `bunny-quoting`.

## Tenant-side metadata

If your product associates a SaaS tenant with the subscription, update
the tenant code or name in lockstep:

```graphql
mutation RenameTenant {
  subscriptionTenantUpdate(
    id: "sub_xxx"
    attributes: { code: "acme-renamed", name: "Acme Co Renamed" }
  ) {
    subscription { id tenant { id code name } }
    errors
  }
}
```

Updating tenant metadata fires a `TenantProvisioningChange` webhook
(see `bunny-webhooks`) so your app can stay in sync.

## Metered usage

For `USAGE`-type charges (see `bunny-catalog`'s pricing models),
report usage as events. Bunny aggregates them per the feature's
`usageCalculationType` (`SUM`, `MAX`, `LAST`, `AVERAGE`) over each
billing period, then bills the aggregated quantity at renewal or the
next invoice.

```graphql
mutation RecordApiCalls {
  featureUsageCreate(attributes: {
    subscriptionId: "sub_xxx"
    featureCode: "api_calls"               # or featureId
    quantity: 1234.0
    usageAt: "2026-05-10T13:45:00Z"        # optional; defaults to now
    notes: "burst from campaign"
  }) {
    featureUsage { id quantity usageAt }
    errors
  }
}
```

**Scope requirement:** `billing:write` — explicitly annotated on
`featureUsageCreate`, `featureUsageUpdate`, and `featureUsageDelete`
in the schema. A token with only `standard:write` cannot post usage.

**`errors` on this payload is a scalar `String`, not a list.**
`FeatureUsageCreatePayload.errors: String` (rest of Bunny's mutations
use `[String!]`). Writing `if (errors?.length) …` treats it as a
string-length check instead of an array-length check — harmless but
counter-intuitive. Code doing `errors.map(...)` or `errors.join(...)`
will break on this one payload. Typical failure is a plain string like
`"Invalid feature"` when the `featureCode` doesn't resolve.

Operational rules:

- **Report usage events as they happen** — don't batch at end-of-period.
  The aggregation function (`SUM` / `MAX` / `LAST` / `AVERAGE`) decides
  what the customer is billed for; delayed batching can misreport under
  `MAX` or `LAST`.
- **`usageAt` is the billing-relevant timestamp**, not your API call
  time. Backdate it if you're replaying events.
- **Correct, don't duplicate.** If you need to fix a wrong report,
  use `featureUsageUpdate(id)` (or `featureUsageDelete(id)`) against
  the specific record — re-posting a corrected event under the same
  period would double-count.
- **`featureUsageHistogram`** (query) returns a histogram of usage by
  period; useful for building internal dashboards without hitting your
  own storage.

See `bunny-catalog` for how to define a `USAGE` charge and wire it to
a `QUANTITY`-kind feature.

## Add-on co-renewal

Subscriptions on plans marked `addon: true` behave as attachments to a
parent subscription. On the nightly cron:

- Add-ons with `evergreen: true` AND the **same `endDate`** as their
  parent get co-renewed in the parent's renewal quote automatically.
- Add-ons with different end dates renew independently.
- Manual renewals via `quoteSubscriptionRenew` do **not** co-renew
  add-ons — the operator must explicitly include them in the `ids`
  array.

If you're issuing a renewal quote for a parent and want add-ons
included, query the account's add-on subscriptions and pass all IDs to
`quoteSubscriptionRenew` together.

## Price adjustments

Subscriptions carry `priceAdjustmentAction` / `priceAdjustmentTiming` /
`priceAdjustmentPercentage` (set when the subscription was created or
via `subscriptionUpdate`). The cron applies the adjustment automatically
on renewal when:

- `priceAdjustmentAction` is `CATALOG` (apply current catalog prices) or
  `PERCENTAGE` (apply the stored percentage bump)
- `priceAdjustmentTiming` is `AT_RENEWAL` (every renewal) or the
  matching month name (bumps only in that month each year)

Manual renewals via `quoteSubscriptionRenew` do **not** auto-apply
adjustments — use `quoteApplyPriceAdjustments` on the resulting quote,
or set the adjustment on the subscription before triggering the cron.

See `bunny-catalog` for the `PriceAdjustmentAction` / `Timing` enum
values.

## Direct updates — `subscriptionUpdate`

For field-level edits that don't change billing (poNumber, notes, a
sales owner, contact reassignment), `subscriptionUpdate(id,
attributes)` is the straight path. Anything billing-relevant (plan
change, quantity, price, renewal term, evergreen flag) has a
dedicated mutation — prefer those over passing the same fields through
`subscriptionUpdate` so the intent is explicit and the audit trail
names the operation.

## References

- [docs.bunny.com/developer](https://docs.bunny.com/developer) — prose docs
- Sibling skills: `bunny-quoting` (quote-based lifecycle changes — upgrade, renewal with price changes, cancellation with credit), `bunny-catalog` (plans, price lists, features that subscriptions reference), `bunny-billing` (invoices, payments, credit notes generated by subscription lifecycle), `bunny-webhooks` (Subscription + TenantProvisioningChange events), `bunny-node-sdk` / `bunny-ruby-sdk` (typed helpers including `subscriptionCreate`, `featureUsageCreate`)

---

Last verified against bunnyapp/api release 2026-04-21-1
