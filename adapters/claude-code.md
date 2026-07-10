# Adapter — Claude Code

How `/taste` runs under Anthropic's Claude Code.

## Install location

Skill lives at `~/.claude/skills/taste/`. Claude Code auto-discovers it via the
`SKILL.md` frontmatter (`name` + `description`) and exposes `/taste`.

## Browser dependency (Playwright MCP)

```bash
claude mcp add playwright -s user -- npx -y @playwright/mcp@latest --isolated
```

Then restart Claude Code so the MCP tools load. Verify:

```bash
claude mcp list | grep playwright
```

You should see `playwright: ... - ✓ Connected`.

- `--isolated`: every run starts with fresh browser state, so cookies / login
  sessions / cache never leak into the analyzed page (running `/taste
  https://linear.app` while signed into Linear would otherwise analyze the app
  dashboard, not the public marketing page).
- `npx @playwright/mcp@latest` with no `--browser` flag: Playwright downloads
  and uses its own bundled Chromium (~100MB, one-time). More portable than
  relying on the user's system Chrome. First run may take 30–60s for that
  download; subsequent runs start in seconds. Pre-download with `npx playwright
  install chromium` if you want to avoid the wait.

If Playwright is unavailable when the skill runs, stop and tell the user:
"I need Playwright MCP to scrape the page. Run `claude mcp add playwright -s
user -- npx -y @playwright/mcp@latest --isolated` and restart Claude Code, then
try again."

## Tool namespace

Claude Code exposes each Playwright MCP tool under the `mcp__playwright__`
prefix. Map the base names in `WORKFLOW.md` Phase 1 as follows:

| WORKFLOW.md base name | Claude Code tool name |
|---|---|
| `browser_resize` | `mcp__playwright__browser_resize` |
| `browser_navigate` | `mcp__playwright__browser_navigate` |
| `browser_wait_for` | `mcp__playwright__browser_wait_for` |
| `browser_take_screenshot` | `mcp__playwright__browser_take_screenshot` |
| `browser_evaluate` | `mcp__playwright__browser_evaluate` |

Run shell steps (grep, python3) with the Bash tool.

## Invocation

`/taste <url>` — or any natural-language trigger described in `SKILL.md`. Then
execute all six phases of `WORKFLOW.md` in order.
