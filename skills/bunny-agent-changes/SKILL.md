---
name: bunny-agent-changes
description: Making changes in Bunny through the MCP tools — why subscription changes go through quotes rather than direct edits, picking the right quote kind (subscribe, update, renew, price_list_change), the compose/charge/apply lifecycle and what applying actually does, why quantities are absolute and never deltas, discounts and price adjustments, cancelling now versus at term end, renewal settings, building the product catalogue with per-unit pricing, creating customers, and chaining dependent writes with $N references. Use when the user wants to add seats, change quantity or price, upgrade, discount, renew, migrate or cancel a subscription, or set up customers, products, plans and pricing.
---

# Changing things in Bunny

These tools alter a live billing system. A quote applied is a contract change; a cancellation stops invoicing. Confirm the target by name before every write.

## The rule that governs almost everything

**A subscription's commercial terms cannot be edited in place. Changes go through a quote.**

```
PriceListCharge  →  QuoteCharge  →  SubscriptionCharge
                    ↑
              the only place price, quantity,
              discount, tiers and dates are set
```

So "give Acme 20% off", "add 10 seats", "move them to the annual plan", "renew for two years" are all **quote** operations, not subscription edits. Attempting them as a direct update is the most common mistake an agent makes here.

What *can* change directly on a subscription:

- **`subscriptions_update`** — whether it renews (`evergreen`), the renewal term length (`renewal_term_months`), the price adjustment applied at renewal (`price_adjustment_action` / `_percentage` / `_timing`), what happens when a trial ends (`trial_expiration_action`: `cancel` or `convert`), and custom field values. Only the fields you pass change. Renewal settings cascade to add-ons and cannot be set on an add-on directly. On an account billed by an integration, only custom fields can change.
- **`subscriptions_cancel`** — end it. See *Cancelling* below.

## The quote lifecycle

```
quotes_compose            create a draft quote — pick the kind that matches the intent
        ↓
quote_charges_compose     add lines to the draft
quote_charges_update      adjust quantity, price, discount or dates of a line
quote_changes_update      renewal term, price adjustment settings, trial start
        ↓
quotes_update             quote dates, billing terms, email settings
        ↓
quotes_apply_changes      ← activates it. This is the moment it becomes real.
```

Nothing takes effect until `quotes_apply_changes`. Drafting changes no subscription, issues no invoice and charges nothing, so build the whole quote, show the user what it contains, and apply only once they confirm.

A quote holds one **quote change per subscription**, and a quote change is the shape of the subscription it will become. Charges hang off the quote change, not off the quote.

**Recomposing wipes edits.** Passing an existing draft as `quote` to `quotes_compose` deletes and rebuilds its changes and charges. To adjust a draft, edit its charges instead.

## Picking the quote kind

`quotes_compose` takes a `kind`, and the kind decides which other arguments are required. Getting this wrong is the first place an amendment goes astray.

| The user wants | `kind` | Required |
|---|---|---|
| A brand-new subscription | `subscribe` | `account` + `price_list` |
| More seats, a different quantity or price | `update` | `subscription` + `changes` (or `quantity`) |
| Another term on what they already have | `renew` | `subscriptions` (array, one account) |
| To move onto a different plan or cadence | `price_list_change` | `subscription` + `price_list` (the **target**) |

For `price_list_change` the target goes in `price_list` — there is no `new_price_list` argument. For `renew`, `price_list` is optional and means "renew onto this one instead"; a term length other than the current one ("renew for two years") is set afterwards with `quote_changes_update` `renewal_term_months: 24`. `start_date` defaults to the entity's current date. Passing `owner` wraps the quote in a deal.

## Quantities are absolute, never a delta

This is the single most expensive mistake available here.

> "Add 10 seats" on a subscription of 20 means **`quantity: 30`**, not `quantity: 10`.

Passing the delta silently *downgrades* the subscription to 10 and bills the customer accordingly. Every quantity field in the quoting tools — `quotes_compose`, `quote_charges_compose`, `quote_charges_update` — is the absolute new value.

So any relative instruction ("add", "increase by", "double", "drop a couple") requires reading the current quantity **before** you compose anything. You cannot compute 30 without knowing it was 20, and you must not guess it.

Single-charge subscription? The `quantity` shorthand on `quotes_compose` is enough. Multiple charges? The shorthand fails as **ambiguous** — use `changes`, a hash keyed by subscription charge ID, so it is unambiguous which line moves.

## Amending a subscription, end to end

The full shape of the hardest task — "add 10 seats to Acme":

```
1. gql_query          find the account, its subscription, and that
                      subscription's charges — with their IDs and
                      current quantities
                          → subscription 456, charge 789 "Users", quantity 20

2. (arithmetic)       20 + 10 = 30. Absolute, from a value you read.

3. quotes_compose     kind: "update", subscription: 456,
                      changes: { 789: { quantity: 30 } }
                          → draft quote 1011

4. gql_query          read the draft back and show the user what it
                      contains — lines, quantities, amounts

5. (confirm)          "Acme Corp, Users 20 → 30 on subscription 456,
                       $X more per month. Apply?"

6. quotes_apply_changes   quote: 1011
```

Steps 1 and 4 are not optional. Step 1 is where the absolute quantity comes from; step 4 is how you avoid telling the user something the quote does not actually say.

To add a *further* override to a draft that already has one — "+10 SMS" after "+5 Users" — use `quote_charges_compose` with `kind: "update"` against the same quote change, rather than composing a second quote.

## Discounts and price changes

**Pass the user's numbers through.** "30% off" is a percentage of 30 — never convert it to an amount. "$100 off" is a fixed amount of 100. "Give them a discount" with no figure: ask.

**Ask how long.** A discount with no duration is ambiguous — ask whether it is permanent or for a period. "For 6 months", "for 3 billing periods", "this period only" and "until 31 December 2026" are all answers; "until December" is not (which year?). Only "permanent" or "forever" means no end.

The routes, in order of preference:

| The user wants | Route |
|---|---|
| A discount on an existing subscription | `quotes_create_subscription_discount` (the subscription) → a draft discount quote with an empty quote change. Then add the discount line with `quote_charges_compose` `kind: "discount"` |
| A discount on a new or amended quote | `quote_charges_compose` `kind: "discount"` on that quote's change |
| A percentage off one line already on a draft | `quote_charges_update` `discount:` (a percentage) |
| Every charge on a draft up or down by a percentage | `quotes_apply_price_adjustment` — changes the prices themselves; negative lowers them |

A discount line takes **exactly one** of `price_list_charge` (discount that one charge) or `coupon` (a subscription-level coupon). Its length is `billing_period_months` or an `end_date` inside the subscription period; `evergreen` makes it renew with the subscription.

What gets rejected: a discount on a subscription that is not active or pending, or is fully billed; a discount on a one-time charge; a percentage discount on a usage charge; a second limited discount on a charge whose current one still has periods left. Read the draft back before applying — that is where a discount that landed on the wrong line shows up.

## Before `quotes_apply_changes` will succeed

Apply validates, and rejects the quote outright if any of these fail:

- it has at least one charge
- the account has a billing contact, and that contact has an email address
- it has not already been applied, and has not been undone
- it is approved, if approval rules require it

The contact rule is the one that bites on new business: a freshly created account has no contacts, so a `subscribe` quote composed against it will compose happily and then fail at apply. Create the contact first — see *Customers*.

## What applying actually does

Say this to the user before they confirm, because all of it happens at once:

- creates or changes the subscriptions and their billing schedules, and grants any credits the charges include
- **issues an invoice** if the quote bills now — one-time charges, or a start date today or earlier — and applies the account's unapplied payments and credit notes to it. No payment method is charged.
- overwrites the account's payment terms with the quote's, and updates the linked deal (renewal deals are marked won)

It is hard to reverse: a quote can only be undone while none of its invoices has been issued, and undoing is not available through these tools.

## Cancelling

Two different mechanisms, and the user's wording picks one:

| User says | Do | Effect |
|---|---|---|
| "cancel" with no timing, "at end of term", "at renewal", "don't renew" | `subscriptions_update` `evergreen: false` | Runs to the end of the current term, then expires. Nothing is cut short. |
| "cancel now", "immediately", "today" | `subscriptions_cancel` with `cancellation_date` = today from `bunny://session` | Ends it on that date. Unbilled charges up to then are invoiced. |
| "cancel on 30 June" | `subscriptions_cancel` with that `cancellation_date` | Ends it on that date |

Default to end of term when timing is not given — it is the reversible one — and say which you are doing. `subscriptions_cancel` takes one or more subscriptions from the same account; add-ons cancel with their parent. A subscription already `canceled` or `expired` cannot be cancelled again.

## Traps worth knowing

**"Extend" is ambiguous, and one meaning is not available.** On an active subscription it means another term — a `renew` quote. On a trial it means moving the trial end date, and **these tools cannot do that**: `subscriptions_update` only sets what happens *when* the trial expires (`cancel` or `convert`), never when it expires. Read the subscription's state first, and if someone asks to extend a trial, say plainly that it has to be done in Bunny.

**An account can have several subscriptions.** "Upgrade Acme" is not actionable until you know which one. List them with their plans and ask.

**Users say "plan" when they mean price list.** The plan is the sellable thing; the price list is its pricing in one currency and cadence. A move from monthly to annual is a `price_list_change`, not a new plan. Price list names often combine the two ("Growth Annual").

**Add-ons follow their parent.** Cancellation cascades to add-on subscriptions, and evergreen and renewal term must be set on the parent — setting them on an add-on is rejected.

## Chaining dependent writes with `$N`

When one write needs the ID of a previous one in the same turn, reference it positionally: `$0` is the first write's result, `$1` the second, and so on.

```
products_create   →  $0
plans_create      →  product: "$0",  yields $1
price_lists_create → plan: "$1",     yields $2
price_list_charges_create → price_list: "$2"
```

This is how you build a catalogue from nothing in one pass. Do not invent IDs to bridge the steps, and do not claim a record was created before its write has actually run.

## Building the catalogue

Order matters, because each level belongs to the one above:

```
products_create      a container — not sellable on its own
   └── plans_create              the sellable thing (addon: true for add-ons)
         └── price_lists_create        pricing in one currency, plus trial and renewal terms
               └── price_list_charges_create   the actual line items and tiers
```

Look before you create: check whether the product, plan or feature already exists with `gql_query`, and add to it rather than making a duplicate.

**Charges.** `charge_type` is `recurring`, `one_time` or `usage`. Recurring and usage charges need a `billing_period` (`monthly`, `quarterly`, `semi_annual`, `annual`). `pricing_model` is one of:

- `flat` — a fixed fee; takes `price`
- `volume`, `tiered`, `bands` — priced per unit through `price_tiers` (`[{starts, ends, price}]`, last tier open-ended) and need a `feature`

**Anything counted per unit — seats, users, SMS, API calls — is `volume`, never `flat`.** It needs a feature as its unit:

1. find the product that owns the price list's plan
2. look for an existing unit feature on it (`isUnit` true) matching the name
3. if none, `features_create` with `product`, `name`, `kind: "quantity"`, `is_unit: true`, and a `code`
4. `price_list_charges_create` with `pricing_model: "volume"`, that `feature`, and tiers starting at 1 — a single `[{starts: 1, price: 10}]` is a plain per-unit rate

Postpaid usage charges also need `usage_calculation_type` (`sum`, `max`, `last`, `average`). Credits-mode usage (`usage_mode: "credits"`) takes a `depletion_policy`; with `top_up`, Bunny creates the top-up charge for you if you do not name one.

Adding a charge never changes existing subscriptions — they keep the charges they were sold. Adding to a deprecated price list fails.

`plans_update`, `price_lists_update`, `features_update` change existing records in place. Only provided fields change.

## Customers

`accounts_create` needs the `entity` the customer belongs to — read the entities from `bunny://session` and ask if there is more than one. Follow it with `contacts_create`: the first contact becomes the billing contact automatically and **must have an email**, and an account with no contact cannot be invoiced, emailed or have a quote applied.

`accounts_update` and `contacts_update` change attributes in place.

## Before you write

State the effect in the user's terms, naming the record:

> "Cancelling **Enterprise** on **Acme Corp** (subscription 456) today. Invoicing stops and anything unbilled up to today goes on a final invoice. Confirm?"

Not "calling subscriptions_cancel with id 456". The user is approving a business outcome, not a function call.

Specifically confirm before: applying a quote, cancelling a subscription, changing renewal or price-adjustment settings, anything touching money, and anything affecting more than one record.

## After you write

Say what happened, name the record, and link it with `mcp_get_link`. Do not re-propose an action you have already completed — if the user says "yes" again, check the history rather than repeating the write.

If a write fails, report the error as given. Do not retry with altered parameters hoping it lands; a billing write that half-succeeded is worse than one that clearly failed.
