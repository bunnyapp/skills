# Contributing

This repo ships **agent skills** for Bunny — small Markdown files with YAML frontmatter that tell a coding agent (Claude Code, Cursor, Codex, Gemini CLI, VS Code / Copilot) how to integrate with Bunny's platform.

We follow the conventions established by [anthropics/skills](https://github.com/anthropics/skills) and [Shopify/Shopify-AI-Toolkit](https://github.com/Shopify/Shopify-AI-Toolkit). If a question isn't answered here, check those first; if it still isn't, open an issue.

## Skill shape

Every skill is a single `<name>/SKILL.md` file at the repo root (no `skills/` wrapper directory — we follow the existing `bunnyapp/skills` layout).

```yaml
---
name: bunny-<thing>
description: Keyword-dense single paragraph. ≤1024 chars. Covers what the skill teaches, the key types/mutations/patterns, and ends with "Use when …". Activation depends on this string — pack it with the vocabulary an integrator would use.
---

# Bunny <Thing>

<one-paragraph introduction>

## Credential safety

<mandatory — every skill has this section>

<!-- body: 200–500 lines. Teach patterns, not catalogues. -->

---

Last verified against bunnyapp/api release YYYY-MM-DD-N
```

### The SKILL.md must be self-contained

No `references/*.md` sub-files. Agents load the whole SKILL.md on activation; fragmenting content across siblings means the agent either misses context or blows its token budget loading everything. If a section grows large, compress — don't extract.

### Frontmatter rules

- `name` matches the directory name exactly.
- `description` is ≤ **1024 characters** (CI enforces this). Pack it: include the key types, enum names, mutation names, error shapes, and the scenarios the skill applies to.
- No other frontmatter keys unless there's a specific reason; keep the shape compatible with every agent product's skill loader.

### Line-count discipline

- Target **200–500 lines**. Shorter is fine for narrow skills; longer means you're cataloguing instead of teaching.
- Every skill ends with `Last verified against bunnyapp/api release YYYY-MM-DD-N`.

## Teach patterns, not method catalogues

Every SKILL.md teaches the integrator-facing **contract** — auth, scope reach, preconditions, error-shape quirks, canonical-flow composition — not an exhaustive list of SDK helpers or schema fields.

**Why:**

- Enumerating helpers and fields inflates what "verified" costs: every line has to be reviewed each time the SDK or schema shifts.
- The SDK README and the GraphQL schema are authoritative once a consumer is installed — agents read them directly.
- Catalogue content crowds out pattern content inside the 200–500 line ceiling.
- Rote-bug documentation becomes load-bearing — file upstream instead; don't freeze workarounds into the skill.

### Rubric for whether a helper / operation deserves its own section

Before adding anything to a skill, ask:

1. **Is the signature non-obvious?** (e.g. `subscriptionCreate` takes 20+ options with a precondition on the product's platform → yes, keep in full.)
2. **Does it teach a correctness-critical pattern?** (e.g. webhook signature validation against the raw body; HTTP status + body-array contract for `CheckoutValidation` → yes, keep.)
3. **Is it covered by a sibling skill?** (e.g. `portalSessionCreate` → `bunny-customer-portal`; `featureUsageCreate` → `bunny-subscriptions` → drop, link out.)
4. **Is it rote CRUD with an obvious method name?** (e.g. `tenantByCode`, `accountUpdateByTenantCode` → collapse into a single "helper catalogue" paragraph with a link to the SDK README.)

Prefer one worked end-to-end flow (create subscription → issue first invoice → handle webhook) to five isolated helper examples.

## Required section: `## Credential safety`

Every skill has this section. Mirror the wording:

- Code samples load secrets from environment variables (`process.env.BUNNY_ACCESS_TOKEN`, `ENV['BUNNY_ACCESS_TOKEN']`). Never inline strings.
- State the scope reach the skill's flows actually need — not more, not less.
- Warn against logging request bodies with auth headers, bundling tokens into client-side JS, or committing `.env` files.

CI rejects PRs introducing anything resembling a live token in a fenced code block.

## Source verification

Every SKILL.md claim about a mutation signature, payload shape, enum value, or scope should be verified against `api/schema.graphql` (or the SDK's public source). Run the mutation against a warren before claiming it works.

The `Last verified against bunnyapp/api release YYYY-MM-DD-N` footer records when that check last happened. Don't edit the skill without updating the footer (or acknowledging in the PR that a re-verification is needed).

## CI

Every PR runs:

- **Skills-format validator** — YAML frontmatter parses, `name` matches directory, `description` ≤ 1024 chars
- **Markdown lint** — trailing whitespace, code-fence language tags, heading style
- **JSON manifest parse** — every `*.json` in the repo is parseable
- **Secret scan** — rejects anything looking like a live Bearer / AWS / Stripe token in example code

Run the same checks locally before pushing:

```bash
pip install --user pyyaml
npx markdownlint-cli "**/*.md" --disable MD013 MD033 MD041
```

## PR conventions

- **Branch naming**: `add-skill/<name>` for new skills, `replace-skill/<name>` for replacements, `fix/<short>` for content fixes, `infra/<short>` for CI / manifests.
- **One skill per PR** (except the initial scaffold). Reviewers can focus; merged-green PRs don't block red ones.
- **PR body** states: the canonical flow the skill teaches, the source of truth (schema file / SDK README / docs page), and the release this was verified against. Cite `bunnyapp/api` commit SHA or release tag.
- **CODEOWNERS** automatically requests review from skill owners — don't self-approve those requests.

## Prior art we borrow from

- [anthropics/skills](https://github.com/anthropics/skills) — canonical SKILL.md template and conventions
- [Shopify/Shopify-AI-Toolkit](https://github.com/Shopify/Shopify-AI-Toolkit) — multi-skill repo layout + plugin manifests for every agent product
- [chargebee/ai](https://github.com/chargebee/ai) — prior competitor reference for the integrator-facing trigger-phrase style
- [agents.md](https://agents.md/) — repo-root orientation file for coding agents (see `AGENTS.md`)
