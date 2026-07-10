---
name: taste
description: >-
  Reverse-engineer the design taste of any website. Given a URL, captures DOM data
  and a screenshot via real browser, then runs a 4-step analysis to produce
  {domain}.md + {domain}.json with practical design tokens (colors, typography,
  spacing, radii, shadows, grid) AND taste DNA (Trigger → Decision → Reason →
  Evidence trade-offs explaining WHY the design works). Use whenever the user wants
  to extract a site's design system, study a competitor's visual language, port an
  aesthetic to a new project, or generate design guidance for an AI coding agent.
  Triggers on '/taste <url>', 'analyze the design of X', 'what makes X's site good',
  'extract design tokens from X', 'give me X's design DNA', 'build something in the
  style of X', 'I want my app to feel like X'. Output rejects AI slop ('clean',
  'modern', 'user-friendly') in favor of concrete px/hex values and restraint
  trade-offs. Do NOT use for scraping data, summarizing page content, or tasks
  unrelated to visual design.
compatibility: >-
  Harness-neutral. The full workflow lives in WORKFLOW.md; per-harness install
  and browser tool-naming live in adapters/. This file is the entry point for
  skill-based harnesses (Claude Code, Gemini CLI, Codex desktop). Browser
  dependency is Playwright MCP — see adapters/claude-code.md, adapters/gemini-cli.md,
  or adapters/codex.md.
metadata:
  version: "1.2.0"
  author: Senlin
---

# Taste — Reverse-Engineer a Website's Design DNA

This `SKILL.md` is the entry point for **skill-based harnesses** (Claude Code,
Gemini CLI, and the Codex desktop app — all three discover skills from the
`name` + `description` frontmatter above and expose `/taste`). The full,
harness-neutral workflow lives in **`WORKFLOW.md`** — a single source of truth
shared by every harness. Per-harness install steps and exact browser tool
names live in **`adapters/`**.

## When to activate

Trigger this skill whenever the user wants to study a website's design and capture it as guidance for future design work. Catch all of these:

- Explicit slash form: `/taste <url>`
- Natural-language requests: "analyze the design of vercel.com", "what makes Stripe's site so good", "extract the design tokens from linear.app", "give me the design DNA for are.na"
- Mimicry intent: "I want my dashboard to feel like Linear", "build me a landing page in the style of <url>", "port this site's design to my project"
- Competitive research: "compare our look to <competitor>", "what design conventions does <site> use"
- Anywhere the user mentions a URL and asks about its design choices, typography, colors, spacing, philosophy, or "feel"

If the user mentions a URL in connection with any visual/design topic, lean toward activating — Claude tends to under-trigger skills. When in doubt, use it. The cost of a wasted skill invocation is small; the cost of generating generic design advice when this skill could have produced a specific, evidence-backed analysis is large.

Do NOT activate for: "extract data from a website" (use a scraping skill), "summarize the content of <url>" (use WebFetch), "deploy this site" (unrelated).

## Prerequisites (Claude Code)

Requires the **Playwright MCP server**. Verify with `claude mcp list | grep playwright`; if it isn't connected, install it:

```bash
claude mcp add playwright -s user -- npx -y @playwright/mcp@latest --isolated
```

Then restart Claude Code so the `mcp__playwright__browser_*` tools appear. Full
detail (why `--isolated`, first-run Chromium download, and the Gemini/Codex
variants) is in `adapters/`. If Playwright is unavailable when the skill runs,
stop and give the user the install command — do not fall back to WebFetch; the
screenshot is essential.

## How to run

1. **Read `WORKFLOW.md`** in this skill directory and execute all phases in
   order, exactly as written. It is the complete workflow — Phase 0 argument
   parsing (`--client` / `--output`), Phase 1 capture, the four `references/`
   analysis steps, the anti-slop audit, export, and report.
2. **Tool names:** `WORKFLOW.md` refers to the browser tools by their base
   names (`browser_navigate`, `browser_resize`, `browser_take_screenshot`,
   `browser_evaluate`, `browser_wait_for`). Each harness exposes them under its
   own namespace prefix — read the adapter matching **your** harness for the
   exact form:
   - Claude Code → `adapters/claude-code.md` (prefix `mcp__playwright__`)
   - Gemini CLI → `adapters/gemini-cli.md`
   - Codex desktop → `adapters/codex.md`
   Run shell steps (grep, python3) with your harness's shell/Bash tool.

Everything else in `WORKFLOW.md` applies verbatim.
