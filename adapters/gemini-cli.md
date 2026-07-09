# Adapter — Gemini CLI

How the skill runs under Google's Gemini CLI.

## Install location

```bash
git clone https://github.com/senlindesign/taste-skill ~/.gemini/skills/taste
```

Gemini CLI discovers the skill from `~/.gemini/skills/taste/` via the same
`SKILL.md` frontmatter Claude Code uses.

## Browser dependency (Playwright MCP)

Add the server to `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest", "--isolated"]
    }
  }
}
```

Restart Gemini CLI so the tools load. Same `--isolated` and bundled-Chromium
rationale as the Claude Code adapter applies.

## Tool namespace

Gemini CLI exposes MCP tools namespaced by server name. Confirm the exact form
your version uses (list available tools), then map the `WORKFLOW.md` base names
to it — typically `playwright.browser_navigate` / `playwright__browser_navigate`
or similar. The base names (`browser_resize`, `browser_navigate`,
`browser_wait_for`, `browser_take_screenshot`, `browser_evaluate`) are stable;
only the prefix differs.

## Invocation

`/taste <url>` (or a natural-language trigger). Then execute all six phases of
`WORKFLOW.md` in order.
