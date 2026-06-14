# magnific-mcp skill for Claude Code

A best-practices guide that teaches Claude how to use the **Magnific MCP**
server effectively — covering model selection, credit budgeting, character
consistency via the Library, image→video chaining, and approval rules for
expensive operations.

This is a **Skill**, not an MCP. It sits on top of the existing Magnific MCP
(which Magnific provides) and gives Claude the meta-knowledge to drive it
well.

## Prerequisites

- An active [Magnific](https://magnific.ai) subscription.
- The Magnific MCP connected to your Claude client (Claude Code, Claude
  Desktop, etc.). The skill auto-detects whether the MCP is registered as
  `magnific`, `magnific-personal`, or `magnific-work`.

## Install

Clone into your Claude skills directory:

```bash
# macOS / Linux
git clone https://github.com/<USER>/magnific-mcp-skill ~/.claude/skills/magnific-mcp

# Windows (PowerShell)
git clone https://github.com/<USER>/magnific-mcp-skill $env:USERPROFILE\.claude\skills\magnific-mcp
```

Restart your Claude client. The skill becomes active automatically — Claude
will load it whenever you mention Magnific, generation, credits, or related
tasks.

## First-run setup

The first time you ask Claude to generate something via Magnific, it will
ask three quick questions:

1. **Usage alerts** — warn at a credit threshold? (none / percent / absolute)
2. **Credit tracking** — auto-update a local `budget.json` after each call?
3. **Cost preference** — `quality-first`, `balanced`, or `careful`?

Your answers persist in `~/.claude/skills/magnific-mcp/user_config.json` and
shape every subsequent decision.

## What's inside

| File | Purpose |
|------|---------|
| `SKILL.md` | The guide itself — 10 sections of accumulated best practice. |
| `user_config.example.json` | Template for your preferences. |
| `budget.example.json` | Template for credit tracking. |

## Customising

`SKILL.md` is plain markdown — edit it freely. If you discover a better
default through use, propose it via PR. Keep changes provider-neutral
(no account-tier specifics; those belong in your local `budget.json`).

## License

MIT.
