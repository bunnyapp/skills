---
name: bunny-analytics
description: Pull revenue metrics and financial reporting from Bunny. Covers MRR / ARR / MUR scalars on `Account`, monthly `RecurringRevenue` time-series (plain list on Account, Relay connection at top level), `RevenueMovement` events tracking new / expansion / contraction / churn / reactivation deltas plus manual adjustments (`revenueMovementCreate` / `revenueMovementUpdate` / `revenueMovementDelete`, all `billing:write`), revenue recognition (ASC 606 / IFRS 15) via `revenueRecognitionTable` and `revenueRecognitionExport` (24 month columns + `period` tag), `revenueToDate`, client-side aggregation for failed/unpaid stuck revenue, the narrow filter-string grammar the server accepts, the `billing:read + standard:read` scope pair required for revenue reads, and patterns for MRR dashboards, churn waterfalls, and quarterly RevRec exports. Use for revenue dashboards, RevRec schedule exports, or subscription movement tracking.
---

# Bunny Analytics

Revenue metrics, recurring-revenue time series, and revenue recognition
schedules — the finance team's view of the billing engine that
`bunny-subscriptions` and `bunny-billing` drive.

This is the read-heavy skill in the set. The mutation surface is small
(three `revenueMovement*` mutations for manual adjustments); everything
else is query.

Public docs: [docs.bunny.com/developer](https://docs.bunny.com/developer).

## Credential safety

Analytics queries return sensitive revenue data — treat tokens with
the same care as production database credentials.

- **Load tokens from environment variables only.** Never log, never
  commit.
- **Scope tokens to `billing:read + standard:read`** for revenue
  dashboards. `billing:read` alone lets you see revenue fields but
  **silently filters `accounts` / `invoices` to empty lists** —
  those need `standard:read` too. The failure mode is empty results
  without any error, which is the single biggest time-sink when a
  dashboard comes back blank. The `revenueMovement*` mutations
  require `billing:write`.
- **Prefer service-account tokens** for dashboards and ETL jobs, not
  user tokens — reporting queries shouldn't be tied to an individual
  staff member's session.
- **Rotate** on a cadence.

## The three metrics on `Account`

Every account carries three live scalar fields recomputed as
subscriptions change:

| Field | Meaning |
| --- | --- |
| `mrr: Float` | Monthly Recurring Revenue — sum of all active subscription recurring charges normalised to a monthly amount |
| `arr: Float` | Annual Recurring Revenue — `mrr × 12` (shortcut for plans that want the annualised number) |
| `mur: Float` | Monthly Usage Revenue — monthly usage-based charges based on recent consumption |

```graphql
query AccountMetrics {
  account(id: "acc_xxx") {
    id name
    mrr arr mur
    revenueToDate                     # total revenue recognized to date
  }
}
```

Don't confuse `arr` (ARR, a `Float` derived from live subscription
state) with the sibling field `annualRevenue: Int` — that's a
lead-qualification field stored on the account, not a computed metric.
Grabbing the wrong one produces numbers that look plausible but aren't
your ARR.

These are **at-now** snapshots. For time-series (how MRR changed over
the last 12 months), use `recurringRevenues`.

### Stuck-revenue (failed / unpaid)

There's no pre-computed `revenueInFailedPayments` aggregate on Account;
build it client-side from the invoice list:

```graphql
query StuckRevenue {
  invoices(filter: "state = 'FAILED'", first: 200) {
    nodes { amountDue account { id name mrr } }
    pageInfo { hasNextPage endCursor }
  }
  # Separate query for UNPAID if your definition of "stuck" includes it
}
```

Sum `amountDue` per account client-side. Separate queries for `FAILED`
and `UNPAID` (the filter grammar doesn't support `OR`; see below).

## Time-series recurring revenue

`recurringRevenues` returns one row per account × month. It comes in
two shapes:

- **Account-scoped** (`account(id:).recurringRevenues(filter, sort)`)
  is a **plain list** with no pagination — good for "last N months
  for this one account" queries.
- **Top-level** (`recurringRevenues(filter, sort, first, after, ...)`)
  is a **Relay connection** for warren-wide aggregation; page with
  `first + after + pageInfo { endCursor hasNextPage }`.

```graphql
query MrrSeriesForOneAccount {
  account(id: "acc_xxx") {
    recurringRevenues(sort: "date asc") {
      date                            # ISO month — 2026-04-01 means "April 2026"
      recurringAmount                 # MRR that month
      usageAmount                     # usage revenue that month
      totalAmount                     # recurring + usage
      currencyId
    }
  }
}
```

`date` is the calendar month (first-of-month).

### Filter grammar is narrow — plan for it

`filter` is **not SQL.** The server-accepted grammar is a single
equality clause against a string column:

```text
field = 'quoted-string'
```

What does **not** work, despite looking reasonable:

- Comparison operators (`>=`, `<=`, `<`, `>`, `!=`) — `Malformed filter`
- Conjunctions (`AND`, `OR`) — `Malformed filter`
- `ILIKE` / `LIKE` / `IN` / `BETWEEN`
- Bare enum values — quote them (`'FAILED'`, not `FAILED`)
- Non-string columns (`accountId`, `manual: Boolean`) — the generated
  SQL wraps the right-hand side with `lower(...)`, which blows up at
  runtime (`function lower(bigint) does not exist`)

For date windows, multi-clause, or inequalities, **fetch wider and
filter client-side** — don't try to build the predicate in the filter
string. Sort with `sort: "date asc"` (or `desc`) and cap with `first`
to bound the result set.

## Movement events — new / expansion / contraction / churn

`RevenueMovement` is an event record: "on date X, account Y's recurring
revenue changed by Z because of a subscription event." Movements are
emitted automatically by the billing engine on subscription apply /
renew / cancel / quantity-update; finance can also post manual
movements for adjustments that happen outside the standard flow.

```graphql
query MovementWaterfall {
  revenueMovements(first: 500) {             # fetch wider, filter client-side
    nodes {
      id                                     # ID
      date                                   # ISO date the movement occurred
      movementType                           # lowercase string: new | expansion | contraction | churn | reactivation
      usageMovementType                      # nullable string — set when the movement is usage-driven
      accountId
      recurringAmount                        # delta to MRR
      usageAmount                            # delta to usage revenue
      totalAmount
      manual                                 # true if posted via revenueMovementCreate
    }
    pageInfo { hasNextPage endCursor }
    totalCount                               # useful for progress telemetry
  }
}
```

`movementType` is typed `String` (not an enum) and the values are
**lowercase** on the wire. Bucket on strict string equality, not enum
comparison.

Typical uses:

- **MRR waterfall**: group by `movementType`, sum `recurringAmount` per
  bucket — gives you new + expansion + contraction + churn for the
  period. Filter the date window client-side after fetching, since the
  filter grammar can't express `AND` of date comparisons (see above).
- **Churn analysis**: `movementType === "churn"`, group by account.
- **Manual adjustment audit**: `manual === true` to surface
  non-automatic postings.

### Posting a manual movement

For corrections that don't come from a subscription event — e.g.
reclassifying a one-time charge as recurring for a specific customer
deal:

```graphql
mutation PostManualMovement {
  revenueMovementCreate(attributes: {
    accountId: "acc_xxx"
    currencyId: "cur_usd"
    date: "2026-04-01"
    movementType: "expansion"
    recurringAmount: 500.0            # the MRR delta
    usageAmount: 0.0
  }) {
    revenueMovement { id movementType totalAmount }
    errors
  }
}
```

**Scope requirement:** `billing:write` — annotated on all three
`revenueMovement*` mutations in the schema. `billing:read` is enough
to query.

Update / delete follow the same pattern (`revenueMovementUpdate(id,
attributes)`, `revenueMovementDelete(id)`). Manual movements are
marked `manual: true` on the record so they remain auditable.

## Revenue recognition (ASC 606 / IFRS 15)

For SaaS finance, Bunny computes revenue recognition schedules on a
per-invoice-line basis — when you bill a 12-month contract upfront, the
revenue is recognized evenly across the 12 months of service delivery.
Two queries surface this:

### `revenueRecognitionTable` — dashboard / drilldown shape

Grouped by account → invoice → line item, with monthly revenue +
deferred-balance columns. Use for building a UI that lets finance
drill from account to invoice to line item.

```graphql
query RevRecDashboard {
  revenueRecognitionTable(
    entityId: "ent_xxx"
    year: 2026
    limit: 50
    offset: 0
  ) {
    accounts {
      id name
      dateLabels { date label }        # column headers — 12 month labels
      totals { date revenue balance }  # per-month account totals
      documents {
        id number                      # invoice number
        totals { date revenue balance }
        items {
          id
          # line-level breakdown — see schema for full shape
        }
      }
    }
  }
}
```

Omit `filter` for unfiltered results; the narrow-grammar caveats from
"Filter grammar is narrow" apply here too. Filter account names or
invoice numbers client-side after the fetch when you need substring
matching.

### `revenueRecognitionExport` — flat CSV-ready shape

One row per invoice line, with 24 columns (12 months × [revenue,
balance]). Designed to drop straight into a spreadsheet or accounting
import:

```graphql
query RevRecExport {
  revenueRecognitionExport(
    entityId: "ent_xxx"                 # required — schema enforces non-null
    year: 2026                           # required — schema enforces non-null
  ) {
    rows {
      accountName invoiceNumber lineText period
      januaryRevenue  januaryBalance
      februaryRevenue februaryBalance
      marchRevenue    marchBalance
      # ... all 12 months
    }
  }
}
```

Each row has `accountName`, `invoiceNumber`, `lineText`, **`period`**
(the fiscal-period tag your finance team uses to bucket the row), and
the 24 month columns (`<month>Revenue` + `<month>Balance` for each
lower-case English month name). Don't drop `period` from your CSV —
it's how the export rows are disambiguated when the same invoice
line spans fiscal boundaries.

`entityId` is Bunny's billing-entity abstraction — one per legal
billing entity in a warren (typically one-to-one with warren for
single-entity setups, but multi-entity warrens split revenue by
entity for separate P&L reporting). Discover entity IDs with:

```graphql
query Entities { entities(first: 10) { nodes { id name } } }
```

`filter` on the table query supports the same narrow equality grammar
covered under "Time-series recurring revenue" above — not SQL. Omit
for unfiltered. `limit`/`offset` pagination on table + export is
classic offset-based, not cursor-based, and the response has no
`totalCount` or `pageInfo` — page until you receive fewer than `limit`
rows.

## Building common dashboards

### Current MRR + breakdown by plan

```graphql
query CurrentMrrByPlan {
  subscriptions(filter: "state = 'active'", first: 1000) {
    nodes {
      plan { id name }
      account { id mrr }
    }
  }
}
```

Group by plan in your app, sum the `account.mrr` contributions. (If
multiple subscriptions on one account, the account's `mrr` is the
total — you'll need to compute per-subscription MRR from
`SubscriptionCharge.totalAmount` + `billingPeriod` normalisation.)

### 12-month MRR waterfall

1. Fetch `revenueMovements(filter: "date >= '<12 months ago>'", first: 500)`, paging as needed
2. Group by calendar month + `movementType`
3. Sum `recurringAmount` per (month, type) bucket
4. Render stacked bars — new + expansion above the line, contraction + churn below

### Quarterly RevRec for finance

```graphql
# Pull the flat export, save as CSV, hand to the controller
{ revenueRecognitionExport(entityId: "ent_xxx", year: 2026) { rows { accountName invoiceNumber lineText /* 24 month columns */ } } }
```

Run this once per quarter-close; reconcile against GL balances before
publishing. For mid-quarter previews, use
`revenueRecognitionTable(year: 2026, limit: …)` and filter client-side.

### Failed-payment dashboard

Two queries (the filter grammar can't `OR` across states), then sum
`amountDue` per account on the client:

```graphql
query Failed  { invoices(filter: "state = 'FAILED'", first: 200) { nodes { id number amountDue account { id name mrr } } pageInfo { hasNextPage endCursor } } }
query Unpaid  { invoices(filter: "state = 'UNPAID'", first: 200) { nodes { id number amountDue account { id name mrr } } pageInfo { hasNextPage endCursor } } }
```

Decide upfront whether "stuck revenue" means strictly `FAILED` (payment
gateway rejection — needs operator attention) or also includes `UNPAID`
(overdue but no gateway attempt — often just a grace-period
timing window). They behave differently downstream; don't merge them
silently.

## Reconciliation patterns

- **Use `entityId`, not warren-level totals**, for finance reporting —
  warrens with multiple entities need per-entity P&L anyway.
- **Treat `manual: true` movements as adjustments**, not primary
  data — when reconciling against a subscription-level source of
  truth, exclude them or report them separately.
- **`revenueToDate` on Account is a running total**, not a period
  aggregate. For period-specific revenue, sum the matching
  `revenueMovements` entries or use the RevRec export.
- **Cache dashboards, don't hammer queries.** These queries can scan
  large result sets; pull nightly and serve from your own warm cache
  for internal dashboards.

## References

- [docs.bunny.com/developer](https://docs.bunny.com/developer) — prose docs
- Sibling skills: `bunny-subscriptions` (the source of MRR), `bunny-billing` (the invoices feeding RevRec), `bunny-catalog` (plans with `recognitionPeriod` drive the RevRec schedule shape), `bunny-graphql` (filter syntax, pagination, `billing:read` scope notes)

---

Last verified against bunnyapp/api release 2026-04-21-1
