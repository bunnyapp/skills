---
name: bunny-agent-changes
description: Making changes in Bunny through the MCP tools — why subscription changes go through quotes rather than direct edits, the compose/charge/apply quote lifecycle, discounts and price adjustments, cancelling, building the product catalogue, and chaining dependent writes with $N references. Use when the user wants to change, upgrade, discount, renew or cancel a subscription, or set up products, plans and pricing.
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

Two shortcuts exist for common cases:

- **`quotes_create_subscription_discount`** — a discount quote against an active or pending subscription, without composing charges by hand.
- **`quotes_apply_price_adjustment`** — a percentage adjustment across every charge on a draft quote.

Prefer these to hand-building an equivalent quote; they encode rules you would otherwise have to reproduce.

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
