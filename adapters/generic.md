# Adapter — Any other LLM / harness

Use this when your harness isn't Claude Code, Gemini CLI, or Codex (Cursor,
Windsurf, a bespoke agent loop, a raw chat session with tool access, etc.).

The workflow itself is just instructions — any capable model can run it. You
need to satisfy two things the harness-specific adapters normally handle:
**discovery/invocation** and **the browser capability**.

## 1. Browser capability

Phase 1 needs real-browser automation: navigate, resize, screenshot,
wait, and evaluate-JS on a live page. The reference implementation is the
Playwright MCP server (`npx -y @playwright/mcp@latest --isolated`). Any
equivalent works as long as it can:

- set viewport to 1440×900
- navigate to a URL and wait a few seconds for hydration
- capture a viewport screenshot **and** a full-page screenshot (the screenshot
  is non-negotiable — a plain HTTP fetch is not a substitute)
- run arbitrary JS in page context and return the result (for `extract.js`)

Whatever the tools are named in your harness, map them onto the base names used
in `WORKFLOW.md` Phase 1: `browser_resize`, `browser_navigate`,
`browser_wait_for`, `browser_take_screenshot`, `browser_evaluate`.

## 2. Invocation

If your harness has no skill/slash-command system, invoke manually: tell the
model to **read `WORKFLOW.md` and execute all six phases**, and give it the
target URL (plus any `--client` / `--output` flag). Everything downstream —
the four `references/*.md` analysis prompts, the anti-slop grep, the JSON
validation — is plain instructions the model follows.

For harnesses that support project or rules files (Cursor `.cursor/rules`,
Windsurf `.windsurf/rules`, a system prompt), you can register a short trigger
that says: "When the user runs `/taste <url>` or asks to analyze a site's
design, read `WORKFLOW.md` in the taste skill and follow it." That gives you an
invocation shortcut without duplicating the workflow.

## Minimum viable run (no tooling at all)

Even in a plain chat with no browser tool, you can run a degraded version by
pasting the page's rendered HTML/CSS and a screenshot yourself, then asking the
model to follow `WORKFLOW.md` from Phase 2 onward. The output quality tracks
how faithful your pasted inputs are to the live render.
