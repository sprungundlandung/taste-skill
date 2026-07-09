# Adapter — Codex (OpenAI)

The Codex **desktop app** has its own Agent Skills system that uses the same
`SKILL.md` format as Claude Code and Gemini CLI. It scans `$CODEX_HOME/skills/`
(default `~/.codex/skills/`; built-ins live under `.system/`). So the skill
installs the same way it does everywhere else — no custom prompt file needed.

## 1. Install the skill

Put the skill directory at `~/.codex/skills/taste/`. Either clone it there, or
— to keep a single copy on disk shared with Claude Code — symlink it:

```bash
# Shared single source of truth (edits propagate to both harnesses):
ln -s ~/.claude/skills/taste ~/.codex/skills/taste

# …or a standalone clone:
git clone https://github.com/senlindesign/taste-skill ~/.codex/skills/taste
```

Codex discovers it from the `name` + `description` frontmatter in `SKILL.md`
and exposes `/taste`.

## 2. Browser dependency (Playwright MCP)

Add to `~/.codex/config.toml` (Codex supports `[mcp_servers.*]` — it already
ships one for `node_repl`):

```toml
[mcp_servers.playwright]
command = "npx"
args = ["-y", "@playwright/mcp@latest", "--isolated"]
startup_timeout_sec = 120
```

Restart Codex so the server connects. Same `--isolated` / bundled-Chromium
rationale as the other adapters. Requires `npx`/Node on PATH.

## Tool namespace

Codex namespaces MCP tools by the server key you set in `config.toml` (here,
`playwright`). Confirm the exact separator your Codex build uses (list the
available tools once the server connects) and map the `WORKFLOW.md` base names
to it — commonly `playwright__browser_*`. The base names (`browser_resize`,
`browser_navigate`, `browser_wait_for`, `browser_take_screenshot`,
`browser_evaluate`) are stable; only the prefix differs.

Run shell steps (grep, python3) with Codex's shell/exec capability.

## Invocation

`/taste <url>` (with optional `--client <name>` / `--output <path>`), or a
natural-language trigger. Codex loads `SKILL.md`, which drives `WORKFLOW.md`.

## CLI note

If you use the Codex **CLI** rather than the desktop app and it doesn't scan a
skills directory, fall back to a custom prompt: create
`~/.codex/prompts/taste.md` whose body says "Read `~/.claude/skills/taste/WORKFLOW.md`
and execute all six phases; arguments: $ARGUMENTS". That yields the same
`/taste` command via prompt expansion.
