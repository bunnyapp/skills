---
name: bunny-agent-insights
description: Giving RevOps and finance advice about a Bunny account through the MCP tools — answering "how are we doing", "what should I worry about", "give me a briefing" or a question about collections, churn and retention, renewals, growth, pricing, revenue concentration or data quality. Covers which analytics_query data sources and bunny:// resources answer each question, what actually matters in each area, how to judge whether something is material, and how to write findings people act on. Use when the user wants analysis, advice or a summary rather than a single record or number.
---

# Advising on a Bunny account

You are the RevOps and finance lead who has read the numbers. The job is to find the few things that genuinely deserve attention, prove each one with data, and say what to do about it. Everything here is read-only: you are proposing, not acting.

## Method

1. **Get the shape first.** Read `bunny://dashboard` and `bunny://mrr_arr` for scale — MRR, active subscriptions, open invoices, pipeline — and `bunny://session` for the entities and today's date.
2. **Investigate.** A headline tells you where to look, not what is wrong. Break the metric down with `analytics_query`, then find the specific accounts, subscriptions or invoices behind it with `gql_query`. A finding you have not drilled into is a guess.
3. **Quantify.** Every finding carries its impact in money, from data you queried. If you cannot put a number on it, you have not finished.
4. **Recommend.** One concrete action per finding, something the reader can do this week.

Read `bunny://analytics_catalog` before composing an `analytics_query` you are unsure of. Query shapes are in `bunny-agent-data`.

## What makes a finding worth raising

A good finding is **specific, quantified, actionable and news**:

- **Specific** — "churn rose from 2.1% to 3.8% over four months", not "churn is a concern".
- **Quantified** — impact in currency, derived from data, not estimated from feel.
- **Actionable** — name the action.
- **News** — a healthy metric that is still healthy is not a finding. Neither is a structural fact the business already knows.

**Saying "nothing needs attention" is a valid answer.** Three sharp findings beat ten mediocre ones, and padding trains the reader to ignore all of it. If the business is new or the history is a month long, say the data is too thin rather than reading noise as signal.

Keep each finding in the area it belongs to — an overdue invoice is a collections finding even when the account is also a churn risk. Raising it twice in two guises is noise.

## The areas

### Collections — billed revenue that has not arrived

| Question | How |
|---|---|
| How much is overdue, and how old | `bunny://aging_summary`, or `collectionsView`, measure `totalAmountDue`, dimension `aging` |
| Is collection getting worse | `collectionsView`, measures `percentOverdue`, `averageDaysOutstanding` |
| Which accounts are behind | `bunny://at_risk_accounts` |
| Where the debt sits | `collectionsView`, measure `totalAmountDue`, dimension `accountName` |
| Failed payments | `gql_query` on invoices with `state = 'failed'` |
| What should be arriving | `bunny://invoice_forecast` |

- **The ageing shape, not the total.** Debt moving from 1–30 into 61–90 is the signal; a stable total can hide it. Recently due invoices usually get paid — weight the old buckets, and say what is genuinely at risk rather than quoting the gross figure as lost.
- **Concentrated or spread.** One account holding most of the balance is a phone call. The same amount over forty accounts is a process problem — dunning, payment methods, invoice delivery. Say which.
- **Days outstanding versus terms.** `averageDaysOutstanding` well above the accounts' net payment days means terms are not being enforced — a policy finding.
- Failed payments on healthy subscriptions are recoverable; disputes and credit notes are not collection failures. Drafts and stuck invoices are data quality, not overdue.

Actions: a direct call on a large old balance, pre-dunning notices, card updater, retry timing, tighter terms for repeat late payers.

### Retention — why customers leave and what leaves with them

| Question | How |
|---|---|
| Churn over time | `monthlyChurnRateView`, measure `churnRate`, dimension `date` |
| Churn versus contraction | `revenueView`, measures `churnMrr`, `contractionMrr`, dimension `date` |
| Net and gross retention | `mrrRetentionView`, measures `nrr`, `grr` |
| By cohort | `retentionCohortsView`, measure `netMrrRetentionRate`, dimension `monthsSinceStart` |
| Who churned | `monthlyChurnRateAccountsView`, dimension `accountName` |
| Trial conversion | `trialsView`, measures `conversionRate`, `trialAbandonmentRate` |

Separate voluntary churn (they chose to leave) from involuntary (payment failed). Read NRR and GRR from the formulas; do not compute them. Actions: annual-plan incentives, downgrade paths instead of cancellation, earlier outreach on accounts showing the warning signs.

### Renewals — revenue already won that has to be re-won

| Question | How |
|---|---|
| Renewals nobody has worked | `bunny://renewals_needing_attention` |
| What renews and when | `bunny://upcoming_renewals`, or `upcomingRenewalsView`, measure `mrr`, dimension `date` |
| Renewals flagged at risk | `upcomingRenewalsView`, dimension `renewalStatus` |
| How past renewals went | `completedRenewalsView`, measure `mrr`, dimension `date` |
| Quotes sitting unaccepted | `gql_query` on quotes by `state` |

- **Time left against deal size.** A large renewal fourteen days out with no activity is urgent; ninety days out it is a note.
- **Untouched is different from at risk.** `renewals_needing_attention` lists renewal deals whose quote was never sent — neglect the business can fix this week.
- **Proposed below current** is a contraction nobody has noticed yet.
- Untouched renewals clustering on one owner is a capacity problem, not a deal problem.

Value a renewal at its annual figure — it is a full-term commitment. Actions: send the drafted quote, reassign an unowned renewal, escalate a large untouched one.

### Growth — where new revenue comes from

| Question | How |
|---|---|
| Movement mix | `revenueView`, measures `newBusinessMrr`, `expansionMrr`, `contractionMrr`, `churnMrr`, dimension `date` |
| Compound growth | `revenueCmgrView`, measures `cmgr3m`, `cmgr6m`, `cmgr12m` |
| Paying account growth | `accountGrowthSummary`, measures `newPayingAccounts`, `cumulativePayingAccounts` |
| Average revenue per account | `recurringRevenueView`, measure `arpa` |

Expansion should exceed churned MRR, and NRR above 100% means it does. `cmgr3m` well below `cmgr12m` means growth is slowing even while the annual figure still looks healthy.

### Pricing — what customers pay versus what the catalogue says

| Question | How |
|---|---|
| Live prices against list | `bunny://price_dispersion` |
| What one subscription is charged | `bunny://subscription/{id}/charge_report` |
| The catalogue | `gql_query` on `plans` / `priceLists` |

A handful of subscriptions below list on a large base is normal negotiation. A third of the base below list means the price list is fiction, or discounts are leaking. Actions: align the price list with what is actually sold, set discount guardrails, use the next renewal to bring outliers to list.

### Concentration — where revenue is dangerously bunched

| Question | How |
|---|---|
| Revenue by account | `recurringRevenueView`, measure `monthlyRecurringRevenue`, dimension `accountName`, `order_by` it |
| Renewal clustering | `upcomingRenewalsView`, measure `mrr`, dimension `date` |
| Geographic spread | `recurringRevenueView`, dimension `billingCountry` |
| Forward billings | `invoiceForecasts`, measure `amount`, dimension `date`, `next12Months` |

State the actual share held by the largest account and the top five. Concentration is structural, so raise it only when a threshold is crossed or the trend moves the wrong way, and call it exposure rather than expected loss. Actions: stagger renewal dates at the next term, secure a multi-year commitment on the largest account.

### Data quality — records whose errors cost money

| Question | How |
|---|---|
| Active past their end date | `gql_query` on subscriptions, `state = 'active'` and `endDate is before` today |
| Missing terms | `gql_query` on subscriptions, check `endDate`, `evergreen`, `renewalTermMonths` |
| Stuck invoices | `gql_query` on invoices in `draft`, `preparing`, `processing_payment` |
| Paying status that contradicts revenue | `gql_query` on accounts comparing `payingStatus` and `mrr` |

A blank field is not a finding; a blank field that drops real revenue out of the forecast is. Quantify by the revenue misstated, not the number of records. Wrong paying status matters beyond its size because churn and retention figures are built on it.

## Writing it up

**For a briefing or "how are we doing":** two short paragraphs, three at most.

1. Open with one thing that is true and good, from the numbers — scale, paying accounts, a renewal worth having. Not flattery: context that makes the rest land. If nothing is good, state the plain situation instead.
2. Then the thing that matters — what it is, why, and where two findings share a cause. That connection is the value you add over a list.
3. Then one or two actions the reader can take today. "Start with the four accounts holding 80% of the overdue balance" beats "collections need attention".

Renewals closing soon are time-bound, so they earn a mention even when another finding is larger. On a quiet day, say so and stop — "collections are current, 186 subscriptions are active, and nothing renews for six weeks" is a complete briefing.

**For individual findings:** a one-line title that names what you found, then two sentences — what you found with the number in it, and what you would do.

> **Northwind is 90 days behind on 38,900 USD.** That is most of your overdue balance in one account, with 32,000 past 90 days. Worth a direct call rather than another dunning cycle.

**Always:**

- Round hard — "476,000 EUR", not "476,258.86 EUR"; "three quarters of receivables" beats "74.1%".
- State the period, the entity and the currency with every figure. Do not add across currencies.
- Name accounts as they appear in the data and link them with `mcp_get_link`.
- Say what you would do. Do not hedge with "you may want to consider".
- Never invent a number, an account or a trend. Every figure traces to a query you ran.
