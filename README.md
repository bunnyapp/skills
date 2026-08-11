# Bunny AI Toolkit

Connect your AI assistant to Bunny. The toolkit is an [Agent Plugin](https://agent-plugins.org)
bundling two things:

- **Developer skills** — Bunny's GraphQL API, the official Node and Ruby SDKs,
  the `@bunnyapp/components` React library, webhooks, and the customer portal.
- **Operator skills + MCP tools** — work with a live Bunny account from chat:
  look up customers, answer revenue questions, preview invoices, build pricing,
  and raise quotes.

For more context see [docs.bunny.com/developer](https://docs.bunny.com/developer).

## Install

Pick your assistant. Every route installs the same plugin from this repository.

### Claude Code

Run these two commands in a chat:

```text
/plugin marketplace add bunnyapp/skills
/plugin install bunny-plugin@bunny-ai-toolkit
```

### VS Code (GitHub Copilot)

Open the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`), run
**Chat: Install Plugin From Source**, and paste the repository URL:

```text
https://github.com/bunnyapp/skills
```

VS Code clones the repo and reads `plugin.json`, `skills/` and `mcp.json`
directly — it implements Agent Plugins 1.0.

### Gemini CLI

```bash
gemini extensions install https://github.com/bunnyapp/skills
```

### Cursor

Install **Bunny** from the Cursor Marketplace.

### OpenAI Codex

In the Codex CLI run `/plugins`, search for **Bunny**, and select
**Add to Codex**.

### Any other Agent Plugins host

Hosts that implement [Agent Plugins 1.0](https://agent-plugins.org) — Google's
Agents CLI and Data Agent Kit among them — load a plugin from a directory path.
The spec does not define a distribution mechanism, so clone the repo and point
the host at it:

```bash
git clone https://github.com/bunnyapp/skills bunny-plugin
```

### Individual skills only

For any [agentskills.io](https://agentskills.io)-compatible tool, when you want
one skill rather than the whole plugin:

```bash
npx skills add bunnyapp/skills --skill <skill-name>
```

## Connect your Bunny account

Installing the plugin gives you the skills. The MCP tools — everything that
reads or changes real data — need one more step, because **Bunny's MCP endpoint
is per account**:

```text
https://<your-subdomain>.bunny.com/api/mcp
```

There is no shared host, so the URL shipped in `mcp.json` is a placeholder.
Replace `YOUR-SUBDOMAIN` with your own, and create an API client in Bunny under
**Settings → API Clients** to authenticate. Full walkthrough, including the
OAuth redirect URI that most often trips people up, is in [MCP.md](MCP.md).

Without this step the developer skills still work; the operator skills have no
tools to call.

## How it is packaged

The repository root **is** the plugin, conforming to
[Agent Plugins 1.0](https://agent-plugins.org). Hosts implementing the spec
consume it directly — no build step, no per-platform repackaging:

```text
plugin.json          # manifest: $schema + name, plus optional metadata
mcp.json             # MCP servers, each with an explicit transport type
skills/<name>/       # one directory per skill, each holding a SKILL.md
```

Those locations are fixed by the spec, and the same tree feeds the
vendor-specific manifests kept alongside it (`.claude-plugin/`,
`.cursor-plugin/`, `.codex-plugin/`, `gemini-extension.json`) — one copy of
every skill, many distribution targets. CI validates conformance on every PR.

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
| `bunny-agent-tools` | Operating a live Bunny account through the MCP tool set |
| `bunny-agent-data` | Reading and interpreting Bunny data via MCP |
| `bunny-agent-changes` | Making changes via MCP: quote lifecycle, catalogue, cancellation |

Each lives at `skills/<name>/SKILL.md`.

## Credential safety

Every code sample in this toolkit reads tokens from environment variables. When
integrating Bunny, **never** hardcode, log, or commit access tokens. Rotate
tokens regularly and scope them to the minimum set of permissions you need.

## Feedback

Open an issue in this repository. For broader questions about Bunny's API, see
[docs.bunny.com/developer](https://docs.bunny.com/developer).

## Contributing

This repo is maintained by the Bunny team. See [CONTRIBUTING.md](CONTRIBUTING.md) for the authoring rubric, skill shape, and PR conventions. External pull requests may be considered on a case-by-case basis — please open an issue first to discuss.
