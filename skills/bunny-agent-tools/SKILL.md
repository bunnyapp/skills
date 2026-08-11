---
name: bunny-agent-tools
description: How to operate a Bunny billing account through the Bunny MCP tools — which tool answers which request, the domain model behind them (warren, entity, account, subscription, quote, invoice), and the rules that keep an agent from inventing IDs, dates or amounts. Use whenever the Bunny MCP server is connected and the user asks about their customers, subscriptions, invoices, quotes, pricing, or revenue.
---

# Operating Bunny through MCP

Bunny is a subscription billing platform. These tools let you read and change a live billing account, so the cost of guessing is real: a wrong ID cancels the wrong subscription, an invented date bills the wrong period.

Everything below assumes the Bunny MCP server is connected. If the tools are not available, say so rather than describing what you would have done.

## The domain, in one diagram

```
Warren (the customer's Bunny tenant — everything belongs to one)
├── Entity (a legal/regional business unit: currency, tax, legal separation)
├── Product → Plan → PriceList → PriceListCharge     ← the catalogue
│
└── Account (a customer of theirs)
    ├── Subscription → SubscriptionCharge
    ├── Quote → QuoteChange → QuoteCharge            ← how subscriptions change
    ├── Invoice → InvoiceItem
    ├── CreditNote → CreditNoteItem
    └── Payment → PaymentApplication
```

**Pricing flows one way:** `PriceListCharge → QuoteCharge → SubscriptionCharge`. Price, quantity, discount and dates are customised **only on a quote**. Once a subscription exists, changing it means a new quote — see `bunny-agent-changes`.

### Words users say, and what they mean

| They say | It is |
|---|---|
| customer, client, company, organization | **Account** |
| plan, package, tier | **Plan** (the catalogue) or **Subscription** (what a customer has) — ask if ambiguous |
| their bill, what they owe | **Invoice** |
| proposal, order form, contract change | **Quote** |
| region, country, legal entity, business unit | **Entity** |
| usage, metered, consumption | **FeatureUsage** against a **Feature** |
| tenant | Ambiguous — in Bunny a **Tenant** is the customer's instance on a platform, while a **Warren** is the billing tenant itself |

## Which tool for which request

**Finding anything at all → `gql_query`.** It is the workhorse: resolve a name to an ID, list, search, browse, filter. There is always a way to look something up. Never reply that no list tool exists, and never guess an ID.

| The user wants | Tool |
|---|---|
| Find / list / search any entity | `gql_query` |
| A number, trend, total, breakdown, MRR/churn/retention | `analytics_query` |
| What the next invoice will look like | `invoices_preview` |
| An exchange rate | `currencies_fetch_rate` — never scrape a rate site |
| Results as a file | `export_csv` |
| "How do I…", "what is…", product documentation | `mcp_docs_search` |
| The contents of a URL they pasted | `mcp_fetch_url` |
| A clickable link to something in Bunny | `mcp_get_link` |
| Create a customer or contact | `accounts_create`, `contacts_create` |
| Build the catalogue | `products_create` → `plans_create` → `price_lists_create` → `price_list_charges_create` |
| Change or cancel a subscription | Quotes — see `bunny-agent-changes` |

`gql_query` and `analytics_query` are different surfaces and not interchangeable: `gql_query` returns records, `analytics_query` returns aggregates over a semantic layer. "Show me Acme's subscriptions" is `gql_query`; "what was MRR last quarter" is `analytics_query`. Details in `bunny-agent-data`.

## Rules

1. **Look it up before you act.** No hardcoded IDs, no assumed defaults. If you need an account ID, query for it.
2. **Prefer exact matches.** Several results? Prefer an exact name or code match. Still ambiguous — ask, listing the candidates with their IDs.
3. **Use only values the user gave you.** "30% discount" means `discount_value: 30` — pass it through, do not compute the resulting amount. Missing something required? Ask.
4. **Never invent a date.** You do not know today's date. For "today", "now", "immediately", "end of month", get the current date from the session context or a tool first.
5. **Return names, not bare IDs.** Copy `name` (and `code`) from query results so the user can tell which record you mean.
6. **Link entities you name.** Use `mcp_get_link` for the URL. For a long list, fetch one link and reuse the URL pattern for the rest.
7. **Do not call a write tool with unresolved parameters.** If you could not resolve the target, stop and ask.
8. **Confirm before anything that moves money or state.** Cancelling, applying a quote, issuing a refund — say what you are about to do, to which named record, and wait.

## Reading vs writing

Reads execute immediately — run them and summarise. Writes change a live billing system, so state the effect in plain terms before calling, and never chain a write off an unverified assumption.

Some tools read despite their name: `price_lists_create`, `price_lists_update` and `price_list_charges_create` are catalogue writes, and `mcp_read_resource` only reads. Judge by what the tool does, not what it is labelled.

## When you cannot do something

Say so plainly and name the nearest thing you can do. Do not describe a tool you do not have, and do not fall back to writing GraphQL for the user to run themselves unless they asked for that.
