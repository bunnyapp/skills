# AGENTS.md

This file orients coding agents (Claude Code, Cursor, Codex, Gemini CLI, VS Code / Copilot) landing on this repo. It follows the [agents.md](https://agents.md/) convention.

## What this repo is

`bunnyapp/skills` ships **agent skills** for integrating [Bunny](https://bunny.com) — the subscription billing platform — into third-party apps. Each skill is a Markdown file that teaches an agent how to use one part of Bunny's API, SDK, or UI library. When an integrator asks an agent to do something Bunny-related, the right skill activates and the agent gets domain-specific guidance.

This repo is **not** the Bunny API source code — it's distribution metadata for AI tools. For Bunny's actual API, see the [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) SDK, the [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) gem, and [docs.bunny.com/developer](https://docs.bunny.com/developer).

## File layout

```text
bunnyapp/skills/
├── README.md                    # Human-facing install guide
├── AGENTS.md                    # You are here
├── CONTRIBUTING.md              # Skill-author rubric and PR conventions
├── LICENSE                      # MIT
├── template/SKILL.md            # Starter for new skills
├── <skill-name>/SKILL.md        # One per skill, top level (e.g. bunny-graphql/SKILL.md)
├── .github/
│   ├── CODEOWNERS               # Review routing per skill directory
│   └── workflows/
│       ├── validate.yml         # Runs on every PR
│       └── link-check.yml       # Weekly cron
├── .claude-plugin/              # Claude Code plugin manifest
├── .cursor-plugin/              # Cursor plugin manifest
├── .codex-plugin/               # OpenAI Codex plugin manifest
├── plugin.json                  # VS Code plugin manifest
├── gemini-extension.json        # Gemini CLI extension manifest
├── .mcp.json                    # Forward reference to the (future) bunnyapp/dev-mcp server
└── assets/                      # Logo and marketplace screenshots
```

**Skills live at the top level**, not under a `skills/` subdirectory. This matches the layout established by the initial `bunny-billing/` and `bunny-components/` skills.

## How to work in this repo

**If you're asked to add a new skill:**

1. Read `CONTRIBUTING.md` — it documents the SKILL.md shape, the "teach patterns not catalogues" rubric, and the credential-safety rules.
2. Copy `template/SKILL.md` to `<skill-name>/SKILL.md` and fill it in.
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

- `validate.yml`: YAML frontmatter parse, `name` ↔ directory match, description ≤ 1024 chars, markdown lint, JSON manifest parse, secret scan
- `link-check.yml`: weekly cron HEAD every external URL; opens an issue on 4xx/5xx

If `validate.yml` fails, read the error and fix locally before re-pushing. See `CONTRIBUTING.md` for the exact commands to run the same checks on your machine.

## Related repos

- [bunnyapp/bunny-node](https://github.com/bunnyapp/bunny-node) — official Node / TypeScript SDK
- [bunnyapp/bunny-ruby](https://github.com/bunnyapp/bunny-ruby) — official Ruby gem
- [bunnyapp/docs-developer](https://github.com/bunnyapp/docs-developer) — source of [docs.bunny.com/developer](https://docs.bunny.com/developer)
