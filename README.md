# Bunny Skills for Claude Code

This repository contains [skills](https://docs.anthropic.com/en/docs/claude-code/skills) for [Claude Code](https://claude.ai/code) that help LLMs integrate with [Bunny.com](https://bunny.com) subscription billing.

## Skills

- **bunny-billing** — Backend billing integrations using the Node.js or Ruby SDKs. Covers subscription creation, billing portals, metered usage, webhooks, and more.
- **bunny-components** — Embedding Bunny's pre-built React billing UI components for self-service subscription management inside your app.

## Installing for Claude Code

Skills are installed by pointing Claude Code at a directory containing skill files. You can install these skills globally (available in all projects) or per-project.

### Global install

Add the following to `~/.claude/settings.json`:

```json
{
  "skills": [
    "/path/to/bunny/skills"
  ]
}
```

### Per-project install

Add the following to `.claude/settings.json` in your project root:

```json
{
  "skills": [
    "/path/to/bunny/skills"
  ]
}
```

Once installed, Claude Code will automatically invoke the relevant skill when you describe what you want — for example:

> "Integrate Bunny billing into my Node.js app"

> "Add a subscription management page using Bunny components"

No special commands are needed; Claude detects the right skill from your description.
