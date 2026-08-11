---
name: bunny-agent-changes
description: Making changes in Bunny through the MCP tools — why subscription changes go through quotes rather than direct edits, picking the right quote kind (subscribe, update, renew, price_list_change), the compose/charge/apply lifecycle, why quantities are absolute and never deltas, what must be true before a quote can apply, discounts and price adjustments, cancelling, building the product catalogue, and chaining dependent writes with $N references. Use when the user wants to add seats, change quantity or price, upgrade, discount, renew, migrate or cancel a subscription, or set up products, plans and pricing.
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

So "give Acme 20% off", "add 10 seats", "move them to the annual plan" are all **quote** operations, not subscription edits. Attempting them as a direct update is the most common mistake an agent makes here.

What *can* change directly: `subscriptions_update` for non-commercial attributes, and `subscriptions_cancel` to end one.

## The quote lifecycle

```
quotes_compose            create a draft quote — pick the kind that matches the intent
        ↓
quote_charges_compose     add or change lines on the draft
quote_charges_update      adjust quantity, price or attributes of a line
quote_changes_update      price adjustment settings, renewal terms
        ↓
quotes_update             dates, billing terms, email settings
        ↓
quotes_apply_changes      ← activates it. This is the moment it becomes real.
```

Nothing takes effect until `quotes_apply_changes`. Everything before that is a draft you can keep editing, so build the whole quote, show the user what it contains, and apply only once they confirm.

A quote holds one **quote change per subscription**, and a quote change is the shape of the subscription it will become. Charges hang off the quote change, not off the quote.

Two shortcuts exist for common cases:

- **`quotes_create_subscription_discount`** — a discount quote against an active or pending subscription, without composing charges by hand.
- **`quotes_apply_price_adjustment`** — a percentage adjustment across every charge on a draft quote.

Prefer these to hand-building an equivalent quote; they encode rules you would otherwise have to reproduce.

## Picking the quote kind

`quotes_compose` takes a `kind`, and the kind decides which other arguments are required. Getting this wrong is the first place an amendment goes astray.

| The user wants | `kind` | Required |
|---|---|---|
| A brand-new subscription | `subscribe` | `account` + `price_list` |
| More seats, a different quantity or price | `update` | `subscription` + `changes` (or `quantity`) |
| Another term on what they already have | `renew` | `subscriptions` (array, one account) |
| To move onto a different plan or cadence | `price_list_change` | `subscription` + `price_list` (the **target**) |

For `price_list_change` the target goes in `price_list` — there is no `new_price_list` argument. For `renew`, `price_list` is optional and means "renew onto this one instead".

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

## Before `quotes_apply_changes` will succeed

Apply validates, and rejects the quote outright if any of these fail:

- it has at least one charge
- the account has a contact, and that contact has an email address
- it has not already been applied
- it has not been undone (an undone quote can never be reapplied)

The contact rule is the one that bites on new business: a freshly created account has no contacts, so a `subscribe` quote composed against it will compose happily and then fail at apply. Create the contact first — see below.

## Charge kinds on a draft

`quote_charges_compose` also takes a `kind`, and each demands a different pairing:

| `kind` | What it adds | Needs |
|---|---|---|
| `add` | A new line, or a carried-over subscription charge | Exactly one of `price_list_charge` / `subscription_charge` |
| `update` | A quantity or price override on an `update` quote | The charge it targets |
| `discount` | A discount line | Exactly one of `price_list_charge` / `coupon` |
| `adjustment` | A one-off against an existing charge | `subscription_charge` |
| `ramp` | Stepped pricing over time | `price_list_charge` + `ramp_interval_months` |
| `free_period` | Free periods at the start or end | `price_list_charge`, `name`, `free_periods`, `free_periods_position` |

`price` overrides are only valid on flat pricing; tiered and volume charges take `price_tiers_attributes` breakpoints instead.

## Traps worth knowing

**"Extend" is ambiguous, and one meaning is not available.** On an active subscription it means another term — a `renew` quote. On a trial it means moving the trial end date, and **these tools cannot do that**: `subscriptions_update` only sets what happens *when* the trial expires (`cancel` or `convert`), never when it expires. Read the subscription's state first, and if someone asks to extend a trial, say plainly that it has to be done in Bunny rather than reaching for a tool that will not do it.

**An account can have several subscriptions.** "Upgrade Acme" is not actionable until you know which one. List them with their plans and ask.

**Users say "plan" when they mean price list.** The plan is the sellable thing; the price list is the pricing for it in one currency and cadence. A move from monthly to annual is a `price_list_change`, not a new plan.

**Add-ons follow their parent.** Cancellation cascades to add-on subscriptions automatically, and evergreen and renewal term must be set on the parent — setting them directly on an add-on is rejected.

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
products_create      a container
   └── plans_create              the sellable thing
         └── price_lists_create        pricing in one currency
               └── price_list_charges_create   the actual line items and tiers
```

`features_create` defines what is measurable or entitled; features are then referenced by charges — a usage charge needs a feature as its unit of measurement.

`plans_update`, `price_lists_update`, `features_update` change existing records in place. Only provided fields change.

## Customers

`accounts_create` makes a customer. Follow it with `contacts_create` — the first contact becomes the billing contact automatically, and an account with no contact cannot be invoiced or emailed.

`accounts_update` and `contacts_update` change attributes in place.

## Before you write

State the effect in the user's terms, naming the record:

> "Cancelling **Enterprise** on **Acme Corp** (subscription 456), effective immediately. Invoicing stops and any usage to date bills on the final invoice. Confirm?"

Not "calling subscriptions_cancel with id 456". The user is approving a business outcome, not a function call.

Specifically confirm before: applying a quote, cancelling a subscription, anything touching money, and anything affecting more than one record.

## After you write

Say what happened, name the record, and link it with `mcp_get_link`. Do not re-propose an action you have already completed — if the user says "yes" again, check the history rather than repeating the write.

If a write fails, report the error as given. Do not retry with altered parameters hoping it lands; a billing write that half-succeeded is worse than one that clearly failed.
