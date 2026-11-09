---
name: bunny-agent-tools
description: How to operate a Bunny billing account through the Bunny MCP tools — which tool answers which request, the domain model behind them (warren, entity, account, subscription, quote, invoice), the lifecycle states records move through, what the tools cannot do, and the rules that keep an agent from inventing IDs, dates or amounts. Use whenever the Bunny MCP server is connected and the user asks about their customers, subscriptions, invoices, quotes, pricing, or revenue.
---

# Operating Bunny through MCP

Bunny is a subscription billing platform. These tools let you read and change a live billing account, so the cost of guessing is real: a wrong ID cancels the wrong subscription, an invented date bills the wrong period.

Everything below assumes the Bunny MCP server is connected. If the tools are not available, say so rather than describing what you would have done.

## The domain, in one diagram

```
Warren (the customer's Bunny tenant — everything belongs to one)
├── Entity (a legal/regional business unit: currency, tax, legal separation)
├── Product → Plan → PriceList → PriceListCharge     ← the catalogue
│                  └── Feature (what a charge meters or grants)
│
└── Account (a customer of theirs)
    ├── Contact (the first one is the billing contact)
    ├── Subscription → SubscriptionCharge
    ├── Quote → QuoteChange → QuoteCharge            ← how subscriptions change
    ├── Invoice → InvoiceItem
    ├── CreditNote → CreditNoteItem
    └── Payment → PaymentApplication
```

**Pricing flows one way:** `PriceListCharge → QuoteCharge → SubscriptionCharge`. Price, quantity, discount and dates are customised **only on a quote**. Once a subscription exists, changing its commercial terms means a new quote — see `bunny-agent-changes`.

### Words users say, and what they mean

| They say | It is |
|---|---|
| customer, client, company, organization | **Account** |
| plan, package, tier | **Plan** (the catalogue), **PriceList** (its pricing in one currency and cadence), or **Subscription** (what a customer has) — ask if ambiguous |
| their bill, what they owe | **Invoice** |
| proposal, order form, contract change | **Quote** |
| region, country, legal entity, business unit | **Entity** |
| usage, metered, consumption | **FeatureUsage** against a **Feature** |
| tenant | Ambiguous — in Bunny a **Tenant** is the customer's instance on a platform, while a **Warren** is the billing tenant itself |

## Where records are in their life

States decide what is possible, so read the state before acting. Spell them exactly as below in a `gql_query` filter — `canceled` has one L, and anything else is rejected. Results come back uppercase.

**Subscription** — `pending` (future start date) → `active`; or `trial` → `active` or `trial_expired`, depending on the trial expiration action. An active subscription ends as `canceled` (someone cancelled it) or `expired` (a non-evergreen term ran out). An evergreen subscription renews itself at term end. A subscription can be `canceled` with a cancellation date still ahead, so it is live until then.

**Quote** — `draft` → `shared` → `viewed` → `accepted`, with `in_approval` → `approved` / `rejected` when approval rules apply, and `undone` for an applied quote that was reversed. A quote can be applied straight from `draft` — it need not be shared or accepted first — but one that needs approval cannot apply until approved, and an undone quote can never be reapplied. Only drafts can be edited.

**Invoice** — `preparing` → `ready` → `not_due` → `due` → `unpaid` (overdue, retries exhausted) or `paid`. Side paths: `failed` (tax calculation failed), `draft` / `merged` (accounts on draft invoicing), `queued` / `consolidated` (consolidated billing), `processing_payment`, and `voided`. **Overdue means `due` or `unpaid` past the due date** — `draft`, `preparing` and `processing_payment` are not collections problems.

**Credit note** — `preparing` / `draft` → `ready` → `partially_applied` / `applied`.

## Which tool for which request

**Finding anything at all → `gql_query`.** It is the workhorse: resolve a name to an ID, list, search, browse, filter. There is always a way to look something up. Never reply that no list tool exists, and never guess an ID.

| The user wants | Tool |
|---|---|
| Find / list / search any record | `gql_query` |
| A number, trend, total, breakdown, MRR/churn/retention | `analytics_query`, or a `bunny://` resource via `mcp_read_resource` |
| What the next invoice will look like | `invoices_preview` |
| An exchange rate | `currencies_fetch_rate` — never scrape a rate site |
| Results as a file | `export_csv` |
| "How do I…", "what is…", "does Bunny support…" | `mcp_docs_search` |
| The exact shape of a Bunny GraphQL query or mutation | `mcp_docs_api_reference` |
| The contents of a URL they pasted | `mcp_fetch_url` |
| A clickable link to something in Bunny | `mcp_get_link` |
| Create or change a customer or contact | `accounts_create` / `accounts_update`, `contacts_create` / `contacts_update` |
| Build or change the catalogue | `products_create`, `plans_*`, `price_lists_*`, `price_list_charges_create`, `features_*` |
| Change, renew, discount or move a subscription | Quotes — see `bunny-agent-changes` |
| Cancel a subscription, or stop it renewing | `subscriptions_cancel` / `subscriptions_update` — see `bunny-agent-changes` |
| "How are we doing?", "what should I worry about?" | See `bunny-agent-insights` |

`gql_query` and `analytics_query` are different surfaces and not interchangeable: `gql_query` returns records, `analytics_query` returns aggregates over a semantic layer. "Show me Acme's subscriptions" is `gql_query`; "what was MRR last quarter" is `analytics_query`. Details in `bunny-agent-data`.

## What these tools cannot do

There is no tool for any of the following. Say so plainly, point the user to the Bunny app (link the record with `mcp_get_link`), and do not try to approximate it with another tool:

- void an invoice, issue or apply a credit note, refund, or record or apply a payment
- move a trial's end date (you can only choose what happens *when* it ends)
- remove a discount that is already on a subscription
- reinstate a cancelled subscription, or undo an applied quote
- send a quote to the customer, or start or decide an approval
- report metered usage, or delete any record

## Rules

1. **Look it up before you act.** No hardcoded IDs, no assumed defaults. If you need an account ID, query for it.
2. **Prefer exact matches.** Several results? Prefer an exact name or code match. Still ambiguous — ask, listing the candidates with their names and IDs.
3. **Use only values the user gave you.** "30% discount" means a percentage of 30 — pass it through, do not compute the resulting amount. Missing something required? Ask.
4. **Never invent a date.** You do not know today's date. For "today", "now", "immediately", "end of month", read the current date from `bunny://session` (via `mcp_read_resource`) first. Each entity has its own current date in its timezone; date arguments default to it when omitted.
5. **Return names, not bare IDs.** Copy `name` (and `code`) from query results so the user can tell which record you mean.
6. **Link records you name.** Use `mcp_get_link` for the URL. For a long list, fetch one link and reuse the URL pattern for the rest.
7. **Do not call a write tool with unresolved parameters.** If you could not resolve the target, stop and ask.
8. **Confirm before anything that moves money or state.** Applying a quote, cancelling, changing renewal or pricing terms, anything touching more than one record — say what you are about to do, to which named record, and wait.

## Reading vs writing

Reads execute immediately — run them and summarise. Writes change a live billing system, so state the effect in plain terms before calling, and never chain a write off an unverified assumption.

Every tool whose name ends in `_create`, `_update`, `_compose`, `_cancel` or `_apply_*` is a write, catalogue tools included. The reads are `gql_query`, `analytics_query`, `invoices_preview`, `export_csv`, `currencies_fetch_rate`, `mcp_read_resource` and the `mcp_docs_*`, `mcp_fetch_url` and `mcp_get_link` tools.

## When you cannot do something

Say so plainly and name the nearest thing you can do. Do not describe a tool you do not have, and do not fall back to writing GraphQL mutations for the user to run themselves unless they asked for that.
