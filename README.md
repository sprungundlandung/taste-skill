# /taste — Design DNA Extractor for Claude Code

A Claude Code skill that reverse-engineers a website's design taste. Give it a URL, get back concrete design tokens **and** the opinionated trade-offs behind them — not just "what", but "why".

## What it produces

Running `/taste https://linear.app` outputs two files in your current directory:

**`linear.md`** — human-readable, two sections:
- **Design Map** — exact CSS values: `#08090A` background, `Inter Variable` at weight `510`, `6px` card radius, `rgb(35,37,42) 0px 0px 0px 1px inset` as the primary depth signal, etc.
- **Taste DNA** — 4 opinionated principles in Trigger → Decision → Reason → Evidence format. Each one names the trade-off the designer made and what they gave up. No generic descriptions.

**`linear.json`** — same data, machine-parseable. Feed it to Cursor rules, Windsurf rules, CLAUDE.md, Figma tokens, or any design system pipeline.

### Example principle (from linear.app)

> **Brand lives in white, not in color** — RESTRAINT  
> Trigger: Deciding on an accent color.  
> Decision: Don't introduce one. Use white (#F7F8F8) as the accent against near-black.  
> Reason: On #08090A, white carries all the emphasis needed. A branded CTA button would make it feel like a template.  
> Evidence: Every nav link and CTA is white or near-white. accentCandidates from DOM are 100% grays. The pink gradient in the hero is a one-time decorative moment, not a system.  
> Trade-off: Can't use color to differentiate feature tiers. Linear resolves this with weight and size hierarchy alone.

## Why it's different from other design extractors

Most tools (Dembrandt, Stylify Me, browser DevTools) output design tokens — the "what". This skill extracts design **taste** — the "why". When an AI agent has taste, it can make consistent design decisions on a *different* page, not just copy numbers from the original.

The output rejects words like "clean", "modern", "user-friendly". Every principle must cite a specific px or hex value and name something the design deliberately didn't do.

## Requirements

- [Claude Code](https://claude.ai/code) with the Skill system
- Playwright MCP (for real browser capture + screenshots)

## Install

**1. Install the skill**

```bash
# Clone into your Claude skills directory
git clone https://github.com/senlindesign/taste-skill ~/.claude/skills/taste
```

**2. Install Playwright MCP** (one-time, if you haven't already)

```bash
claude mcp add playwright -s user -- npx -y @playwright/mcp@latest --isolated
```

Restart Claude Code after adding the MCP server.

## Usage

```
/taste https://linear.app
/taste https://stripe.com
/taste vercel.com
```

The skill will:
1. Ask which build tool you're using (Cursor / Windsurf / Claude Code / Copilot / Bolt / v0 / Figma Make / Lovable / skip)
2. Capture the page via real browser + two screenshots (viewport + full-page)
3. Run a 4-step analysis: Measure → Pattern → Taste → Observer
4. Write `{domain}.md` + `{domain}.json` to your current directory
5. Export the design taste to your chosen tool's format

### Export formats

| Tool | File |
|------|------|
| Cursor | `.cursor/rules/{domain}-taste.mdc` |
| Windsurf | `.windsurf/rules/{domain}-taste.md` |
| Claude Code | `CLAUDE.md` (appends section) |
| GitHub Copilot | `.github/copilot-instructions.md` |
| Bolt | `.bolt/prompt` |
| Antigravity | `GEMINI.md` |
| v0 | `taste-tokens.css` |
| Figma Make | `taste-figma.css` |
| Lovable | printed to paste in Project Knowledge |

## How the analysis works

```
Phase 0  Parse URL + domain name + export target
Phase 1  Playwright captures viewport screenshot, full-page screenshot, and DOM data
Phase 2  4-step pipeline:
           Step 1 (Measure)  — 20 measurement categories, exact px/hex/ratio
           Step 2 (Pattern)  — 5-8 system-level rules with Evidence + Design Goal
           Step 3 (Taste)    — 4 principles, each with Trigger/Decision/Reason/Evidence
           Step 4 (Observer) — quality gate, writes final Design Map + Taste DNA
Phase 3  Write {domain}.md + {domain}.json
Phase 4  Self-audit: grep for banned phrases, verify JSON, verify sections
Phase 5  Export to your build tool
Phase 6  Report
```

**Visual primacy**: the screenshot is ground truth. DOM data supplies exact numbers, but when they conflict, the screenshot wins. A color at <5% visual surface area is decorative — not a brand color.

## File structure

```
taste/
├── SKILL.md                    # Skill entry point + full workflow
└── references/
    ├── extract.js              # Browser-injected DOM extractor
    ├── step1-measure.md        # Step 1 prompt (20 measurement dimensions)
    ├── step2-pattern.md        # Step 2 prompt (system pattern extraction)
    ├── step3-taste.md          # Step 3 prompt (taste principle generation)
    ├── step4-observer.md       # Step 4 prompt (quality gate + final output)
    └── export-formats.md       # Spec for all 9 export formats
```

## License

MIT
