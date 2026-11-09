# Bunny Agent Plugin

Connect your AI assistant to Bunny and work with your live billing account
from chat: look up customers and subscriptions, answer revenue questions,
preview invoices, build products and pricing, and raise quotes to change or
cancel a subscription.

This repository is an [Agent Plugin](https://agent-plugins.org) (spec 1.0)
bundling the **Bunny MCP server** with the **skills** that teach an agent to use
its tools well.

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
reads or changes real data — need you to sign in once, against a single
endpoint that is the same for every account:

```text
https://auth.bunny.com/api/mcp
```

That URL ships working in `mcp.json`, so for most clients there is nothing to
edit. The server supports **dynamic client registration**, so a client that
speaks it enrols itself and takes you straight to the Bunny login and consent
screen — no API client to create, no client ID or secret to copy, no redirect
URI to pre-register. Your account is carried by the token you consent with,
which is why the URL needs no subdomain.

Hosts that still want a client ID and secret typed in are covered in
[MCP.md](MCP.md), along with the OAuth redirect URI that most often trips people
up.

Until you connect, the skills have no tools to call.

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

- **Answers from your data**: customers, subscriptions, invoices and quotes;
  MRR, ARR, churn, retention and AR aging; next-invoice previews; CSV exports
- **Changes, confirmed first**: add seats, change price, discount, renew,
  migrate or cancel a subscription through quotes; create customers and
  contacts; build products, plans, price lists and charges
- **Advice**: briefings and findings on collections, retention, renewals,
  growth, pricing, revenue concentration and data quality
- **Documentation**: search Bunny's product and developer docs from chat

## Available skills

| Skill | Covers |
| --- | --- |
| `bunny-agent-tools` | The domain model, record states, which tool answers which request, what the tools cannot do |
| `bunny-agent-data` | Reading data: `gql_query` filters, `analytics_query` metrics, `bunny://` resources, exports |
| `bunny-agent-changes` | Making changes: quotes, discounts, cancellation, renewals, the catalogue, customers |
| `bunny-agent-insights` | Advice and briefings: collections, retention, renewals, growth, pricing, concentration, data quality |

Each lives at `skills/<name>/SKILL.md`.

## Privacy and access

The plugin reads no credentials from your machine. The MCP server signs you in
with OAuth, and the token it issues is limited to your own Bunny permissions
and the account you consented for. Write tools ask for confirmation before
they change anything. See the [Bunny privacy policy](https://bunny.com/privacy/).

## Building an integration?

To integrate Bunny into your own application — the GraphQL API, the Node and
Ruby SDKs, the React components, webhooks or the customer portal — see
[docs.bunny.com/developer](https://docs.bunny.com/developer). The
`mcp_docs_search` tool searches the same documentation from chat.

## Feedback

Open an issue in this repository. For broader questions about Bunny's API, see
[docs.bunny.com/developer](https://docs.bunny.com/developer).

## Contributing

This repo is maintained by the Bunny team. See [CONTRIBUTING.md](CONTRIBUTING.md) for the authoring rubric, skill shape, and PR conventions. External pull requests may be considered on a case-by-case basis — please open an issue first to discuss.
