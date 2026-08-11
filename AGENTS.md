# AGENTS.md

This file orients coding agents (Claude Code, Cursor, Codex, Gemini CLI, VS Code / Copilot) landing on this repo. It follows the [agents.md](https://agents.md/) convention.

## What this repo is

`bunnyapp/skills` ships **agent skills** for integrating [Bunny](https://bunny.com) — the subscription billing platform — into third-party apps. Each skill is a Markdown file that teaches an agent how to use one part of Bunny's API, SDK, or UI library. When an integrator asks an agent to do something Bunny-related, the right skill activates and the agent gets domain-specific guidance.

This repo is **not** the Bunny API source code — it's distribution metadata for AI tools. For Bunny's actual API, see the [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) SDK, the [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) gem, and [docs.bunny.com/developer](https://docs.bunny.com/developer).

## File layout

**The repository root is the plugin.** It is a valid [Agent Plugin](https://agent-plugins.org) (spec 1.0.0), and the vendor-specific manifests sit alongside it as extra entry points into the same content.

```text
bunnyapp/skills/                 # ← this directory IS the plugin root
├── plugin.json                  # Agent Plugins manifest ($schema + name)
├── mcp.json                     # Agent Plugins MCP config (typed server entries)
├── skills/
│   └── <skill-name>/SKILL.md    # One per skill (e.g. skills/bunny-graphql/SKILL.md)
├── assets/                      # Logo and marketplace screenshots
│
├── .mcp.json                    # Claude Code reads this fixed name; mirrors mcp.json
├── .claude-plugin/              # Claude Code plugin + marketplace manifests
├── .cursor-plugin/              # Cursor plugin + marketplace manifests
├── .codex-plugin/               # OpenAI Codex plugin manifest
├── gemini-extension.json        # Gemini CLI extension manifest
│
├── README.md                    # Human-facing install guide
├── AGENTS.md                    # You are here
├── CONTRIBUTING.md              # Skill-author rubric and PR conventions
├── MCP.md                       # Connecting the Bunny MCP server
├── LICENSE                      # MIT
├── template/SKILL.md            # Starter for new skills
└── .github/
    ├── CODEOWNERS               # Review routing per skill directory
    └── workflows/
        ├── validate.yml         # Runs on every PR
        └── link-check.yml       # Weekly cron
```

### Rules the layout has to satisfy

- **Skills live in `skills/<name>/SKILL.md`.** The spec discovers skills only at that fixed location, and clients **must not** recurse deeper — a `SKILL.md` nested any further is invisible. (They lived at the top level until v0.2.0.)
- **`plugin.json` is closed.** Its schema sets `additionalProperties: false` and a violation is fatal to the *entire* plugin, not just one component. The only permitted keys are `$schema`, `name`, `version`, `description`, `author`, `homepage`, `repository`, `license`, `keywords`, `extensions`. Anything client-specific goes under `extensions`, keyed by reverse-domain namespace, or into that client's own directory.
- **Every `mcp.json` server entry declares `type`** (`stdio`, `streamable-http`, or `sse`). Plugin-relative paths must start with `./` and resolve inside the plugin root.
- **`mcp.json` and `.mcp.json` must stay identical.** The spec names the file `mcp.json`; Claude Code discovers `.mcp.json`. Both ship, and CI fails if they drift.

## How to work in this repo

**If you're asked to add a new skill:**

1. Read `CONTRIBUTING.md` — it documents the SKILL.md shape, the "teach patterns not catalogues" rubric, and the credential-safety rules.
2. Copy `template/SKILL.md` to `skills/<skill-name>/SKILL.md` and fill it in. One level deep — no nesting.
3. Verify every mutation signature / payload shape / enum value against the Bunny GraphQL schema (not from memory) before shipping.
4. Open a PR on a branch named `add-skill/<name>`.

**If you're asked to edit an existing skill:**

1. Don't. An edit mid-migration invalidates the dogfood-verified state. File an issue describing the drift and let the codeowner decide whether to re-author and re-verify.
2. If the change is *really* cosmetic (typo, link fix), open a `fix/<short>` branch and leave a note in the PR body that no re-verification is needed.

**If you're asked to set up a live environment:**

This repo has no runtime. It's Markdown + JSON manifests. `npm install` does nothing useful; there's no dev server.

## What not to do

- **Don't inline secrets in code samples.** Every example reads tokens from env vars (`process.env.BUNNY_ACCESS_TOKEN`, `ENV['BUNNY_ACCESS_TOKEN']`). CI's secret-scan rejects the PR otherwise.
- **Don't split a SKILL.md into sub-files** (no `references/*.md`). Our skills are self-contained. Agents load the full SKILL.md on activation; fragmentation either misses context or inflates the token budget.
- **Don't rewrite a skill's description without care.** The description controls activation — a vague description means the skill doesn't fire when it should, and a broad description means it fires when it shouldn't.
- **Don't merge your own PR** — CODEOWNERS routes skill PRs to their owners for review.

## CI gates

Every PR runs:

- `validate.yml`: YAML frontmatter parse, `name` ↔ directory match, description ≤ 1024 chars, JSON manifest parse, Agent Plugins 1.0.0 conformance (manifest keys, MCP server types, `.mcp.json` mirror, skill discoverability), secret scan
- `link-check.yml`: weekly cron HEAD every external URL; opens an issue on 4xx/5xx

If `validate.yml` fails, read the error and fix locally before re-pushing. See `CONTRIBUTING.md` for the exact commands to run the same checks on your machine.

## Related repos

- [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) — official Node / TypeScript SDK
- [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) — official Ruby gem
- [bunnyapp/docs-developer](https://github.com/bunnyapp/docs-developer) — source of [docs.bunny.com/developer](https://docs.bunny.com/developer)
