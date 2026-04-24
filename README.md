# Bunny AI Toolkit

Connect your AI coding assistant to Bunny. The toolkit bundles [Agent Skills](https://docs.anthropic.com/en/docs/claude-code/skills) for Bunny's GraphQL API, the official Node and Ruby SDKs, the `@bunnyapp/components` React library, webhooks, the customer portal, and the subscription / quote / invoice lifecycle.

For more context see [docs.bunny.com/developer](https://docs.bunny.com/developer).

## Install

- **For Claude Code**: run these two commands in a chat:

  ```text
  /plugin marketplace add bunnyapp/skills
  /plugin install bunny-plugin@bunny-ai-toolkit
  ```

- **For Cursor**: install from the Cursor Marketplace.

- **For Gemini CLI**: run this command in your terminal:

  ```bash
  gemini extensions install https://github.com/bunnyapp/skills
  ```

- **For OpenAI Codex**: in the Codex CLI, run `/plugins`, search for **Bunny**,
  and select **Add to Codex**.

- **For VS Code**: open the Command Palette (`CMD+SHIFT+P`) and run
  **Chat: Install Plugin From Source**, then paste:

  ```text
  https://github.com/bunnyapp/skills
  ```

- **For any agentskills.io-compatible tool**: install individual skills with:

  ```bash
  npx skills add bunnyapp/skills --skill <skill-name>
  ```

## What you get

- **Docs and schema-aware help**: search Bunny's developer documentation and
  GraphQL schema from inside your editor
- **SDK-guided integration**: correct setup and usage patterns for the Node
  (`@bunnyapp/api-client`) and Ruby (`bunny_app`) SDKs
- **React components**: guided integration of `@bunnyapp/components`
  (BillingDetails, Invoice, Signup, Subscriptions, Quote, Quotes, Transactions)
- **Webhook handlers**: correct patterns for platform and workflow webhooks
- **Lifecycle flows**: quote → apply → subscription; renewals, upgrades,
  cancellation; invoices, payments, credit notes

## Available skills

| Skill | Covers |
| --- | --- |
| `bunny-graphql` | Direct GraphQL API: endpoint, auth, pagination, common operations |
| `bunny-node-sdk` | `@bunnyapp/api-client` (Node / TypeScript) |
| `bunny-ruby-sdk` | `bunny_app` gem (Ruby / Rails) |
| `bunny-components` | `@bunnyapp/components` React library |
| `bunny-webhooks` | Platform and workflow webhook handling |
| `bunny-customer-portal` | Popup, standalone, and signup portal variants |
| `bunny-catalog` | Products, plans, features, price lists, coupons |
| `bunny-quoting` | Quote lifecycle (create → share → accept → apply) |
| `bunny-subscriptions` | Subscription lifecycle: renewals, upgrades, cancellation, usage |
| `bunny-billing` | Invoices, payments, credit notes, reconciliation |
| `bunny-analytics` | MRR / ARR / churn, RevRec schedules, reporting exports |

## Credential safety

Every code sample in this toolkit reads tokens from environment variables. When
integrating Bunny, **never** hardcode, log, or commit access tokens. Rotate
tokens regularly and scope them to the minimum set of permissions you need.

## Feedback

Open an issue in this repository. For broader questions about Bunny's API, see
[docs.bunny.com/developer](https://docs.bunny.com/developer).

## Contributing

This repo is maintained by the Bunny team. See [CONTRIBUTING.md](CONTRIBUTING.md) for the authoring rubric, skill shape, and PR conventions. External pull requests may be considered on a case-by-case basis — please open an issue first to discuss.
