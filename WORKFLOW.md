# Taste — Reverse-Engineer a Website's Design DNA

> **This is the portable core of the skill.** It is harness-neutral: it does
> not assume Claude Code, Gemini CLI, Codex, or any specific tool namespace.
> Each harness has a thin adapter in `adapters/` that says how to install the
> browser dependency, what the browser tools are actually named in that
> harness, and how the workflow is invoked. Read your harness's adapter first,
> then execute the phases below.

This skill turns a URL into two files:

- **`{outputDir}/{outputFilename}`** — human-readable. Two sections:
  - **Design Map** — practical tokens (colors, type scale, spacing, radii, shadows, grid)
  - **Taste DNA** — 3-4 opinionated trade-offs (Trigger → Decision → Reason → Evidence)
- **`{outputDir}/{domain}.json`** — structured. Same data, machine-parseable, for downstream tools (Cursor, Windsurf, design systems).

The pipeline rejects generic descriptions like "clean and modern". A finished
markdown file should read like a senior designer explaining *why* this
particular page made *these particular* choices — and what plausible
alternatives it rejected.

## Prerequisites — a browser-automation capability

This skill requires a **Playwright MCP server** (https://playwright.dev/docs/getting-started-mcp)
providing real-browser automation: navigate, resize, screenshot, wait, and
evaluate-JS. The screenshot is essential — do **not** fall back to a plain
HTTP fetcher (WebFetch or similar); Step 1 depends on seeing the rendered page.

How you install that server and what the browser tools are **named** depends on
your harness. Read the matching adapter before Phase 1:

| Harness | Adapter |
|---|---|
| Claude Code | `adapters/claude-code.md` |
| Gemini CLI | `adapters/gemini-cli.md` |
| Codex (desktop or CLI) | `adapters/codex.md` |
| Any other LLM / harness | `adapters/generic.md` |

Throughout Phase 1 this workflow refers to the Playwright tools by their
**canonical base names** — `browser_resize`, `browser_navigate`,
`browser_wait_for`, `browser_take_screenshot`, `browser_evaluate`. These base
names come from the `@playwright/mcp` server and are the same everywhere. Your
harness may expose them under a **namespace prefix** (for example, Claude Code
exposes `browser_navigate` as `mcp__playwright__browser_navigate`). Your
adapter states the exact prefix. Apply it to every tool call below.

Verify the Playwright tools are available before starting. If they are not,
stop and give the user the install steps from their adapter — do not improvise
a substitute.

## Workflow

Follow the phases below in order. Each phase has explicit success criteria — do
not advance until the current phase has produced what the next phase needs.

### Phase 0 — Parse the URL and export target

**Step 0 — Output folder**

Parse the invocation for these flags (checked in order — first match wins):

1. **`--output <path>`** (e.g. `/taste https://acme.com --output knowledge-base/acme/brand.md`)
   Internal flag set by other skills (e.g. content-composer). Write the markdown file to exactly `<path>` and the JSON file alongside it in the same directory. Set `{outputDir}` to that directory and `{outputFilename}` to the basename (`brand.md`). Skip all prompts and go straight to Step 1.

2. **`--client <name>`** (e.g. `/taste https://acme.com --client acme`)
   Client shorthand for interactive use. Resolve the project root by running `git rev-parse --show-toplevel` (fall back to `pwd` if not a git repo). Set `{outputDir}` to `{project-root}/knowledge-base/clients/{name}/` and `{outputFilename}` to `{domain}.md`. Create the directory if it does not exist (`mkdir -p`). Skip all prompts and go straight to Step 1.

3. **No flag** (bare `/taste <url>`)
   Default. Set `{outputDir}` to `.` (current working directory) and `{outputFilename}` to `{domain}.md`. No prompt, no folder creation.

Store `{outputDir}` and `{outputFilename}`. Use `{outputDir}/{outputFilename}` (markdown) and `{outputDir}/{domain}.json` wherever Phase 3, Phase 4, and Phase 6 reference output file paths.

**Step 1 — URL**

Extract the URL from the user's message. Accept these forms:
- `/taste https://linear.app`
- "analyze the design of vercel.com" (add `https://` if missing)
- A bare URL anywhere in the prompt

If the user provided no URL, ask for one. If they provided a file path or something that clearly isn't a URL, ask whether they meant a hosted page. Don't proceed until you have a real URL.

Resolve common short forms before navigating: `linear.app` → `https://linear.app`, `https://www.stripe.com` is fine as-is.

**Step 2 — Extract domain for file naming**

From the URL, extract the hostname and strip any `www.` prefix. Examples:
- `https://linear.app` → `linear.app`
- `https://www.stripe.com` → `stripe.com`
- `https://vercel.com/home` → `vercel.com`

Store this as `{domain}`. All output files will be named after it: `{domain}.md`, `{domain}.json`, and any export file. Never use the generic name `taste.md` — that name collides when the user runs this skill on multiple sites in the same directory.

**Step 3 — Export target + Crawl scope**

If the user already specified both an export target AND crawl scope in their message, skip this question entirely. If either is missing, ask once in a single message:

> "Two quick questions before I start:
>
> **Export target** — which tool are you building with?
> ```
>  1  Cursor          → .cursor/rules/{domain}-taste.mdc
>  2  Windsurf        → .windsurf/rules/{domain}-taste.md
>  3  Claude Code     → CLAUDE.md (appends Design Taste section)
>  4  GitHub Copilot  → .github/copilot-instructions.md (appends)
>  5  Bolt            → .bolt/prompt
>  6  Antigravity     → GEMINI.md
>  7  v0 by Vercel    → taste-tokens.css + instructions
>  8  Figma Make      → taste-figma.css + instructions
>  9  Lovable         → print text to paste in Project Knowledge
> 10  Skip            → keep {domain}.md + {domain}.json only
> ```
>
> **Crawl scope** — this page only, or explore 2–3 linked pages?
> ```
>  1  This page only       → faster, focused on exactly what you gave me
>  2  Explore linked pages → I'll visit 2–3 nav pages and merge the data
> ```"

Store answers as `{exportTarget}` and `{crawlScope}`. Defaults: `{exportTarget}` = "skip", `{crawlScope}` = "single". Do not ask again.

### Phase 1 — Capture page data (browser automation)

Run these steps in order. Prefix each tool name with your harness's Playwright
namespace (see Prerequisites). If any step errors, stop the pipeline and report
the error to the user — do not silently continue with degraded data.

1. **Resize the viewport** to 1440×900 so measurements are consistent across runs:

   ```
   browser_resize  width=1440  height=900
   ```

2. **Navigate** to the URL:

   ```
   browser_navigate  url=<the URL>
   ```

3. **Wait briefly** for client-side hydration. Heavy SPAs (figma.com, notion.so) need a few seconds after `domcontentloaded` to render real content. Use:

   ```
   browser_wait_for  time=3
   ```

   If after 3 seconds the page snapshot still looks like a loading skeleton, give it another 3 seconds. If it's still skeleton after that, note it in the final output ("page may be incompletely rendered; some measurements approximate").

4. **Take screenshots** — viewport first, then a conditional full-page capture:

   **4a. Viewport screenshot** (1440×900, un-scaled — this is your primary visual reference for Step 1):
   ```
   browser_take_screenshot  type=jpeg  filename={domain}-viewport.jpeg
   ```

   **4b. Conditional full-page screenshot** — first check page height:
   ```
   browser_evaluate  function=() => document.documentElement.scrollHeight
   ```

   - If **≤ 6× viewport height** (≤ 5400px at 1440×900): take a normal full-page screenshot:
     ```
     browser_take_screenshot  fullPage=true  type=jpeg  filename={domain}-fullpage.jpeg
     ```
   - If **> 6× viewport height**: skip full-page (too tall, wastes tokens). Instead take two targeted viewport shots:
     ```js
     // Mid-page (50% scroll)
     browser_evaluate  function=() => window.scrollTo(0, document.documentElement.scrollHeight * 0.5)
     browser_take_screenshot  type=jpeg  filename={domain}-mid.jpeg
     // Footer
     browser_evaluate  function=() => window.scrollTo(0, document.documentElement.scrollHeight)
     browser_take_screenshot  type=jpeg  filename={domain}-footer.jpeg
     // Scroll back to top
     browser_evaluate  function=() => window.scrollTo(0, 0)
     ```

   Use the **viewport screenshot** as the primary visual reference in Step 1 — it represents the first impression the designer intended. Use the full-page or mid+footer screenshots to verify patterns that only appear lower on the page (footer typography, late-section color variations, etc.). JPEG keeps token cost down vs. PNG.

5. **Run the DOM extractor**. Read `references/extract.js` and pass its **entire contents** as the `function` argument:

   ```
   browser_evaluate  function=<contents of references/extract.js>
   ```

   The tool returns a JSON object. Save it mentally as `domData` — you'll feed it into Step 1.

6. **Client logo capture** — run this step **only in client-context runs**:
   `--client <name>` was passed, or `{outputDir}` resolves inside a
   `knowledge-base/clients/` folder (the `--output` form other skills use).
   Skip it entirely for bare `/taste <url>` runs.

   Downstream consumers (e.g. content-composer's Q0b brand-context step) look
   for `logo.png` / `logo.svg` / `logo.jpg` in the client folder and use it
   silently when present — capturing it here removes an interactive question
   later. Rules:

   - **Never overwrite**: if any `{outputDir}/logo.*` already exists, skip
     this step silently — an existing logo may be curated.
   - This step is **best-effort and never blocks the pipeline**: on any
     failure, note it for the Phase 6 report and continue.

   **6a. Locate the logo** on the already-loaded primary page:

   ```js
   (function() {
     const scope = document.querySelector('header') || document.querySelector('nav') || document.body;
     // 1) an <img> that self-identifies as a logo, or sits inside a home link
     const imgs = [...scope.querySelectorAll('img')].filter(im => {
       const hay = ((im.alt||'') + ' ' + (im.className||'') + ' ' + (im.id||'') + ' ' + (im.src||'')).toLowerCase();
       let home = false;
       try { const a = im.closest('a'); home = a && new URL(a.href, location.href).pathname === '/'; } catch(e) {}
       return hay.includes('logo') || home;
     });
     const im = imgs.find(i => i.offsetWidth > 16 && i.offsetHeight > 8);
     if (im) {
       const src = im.currentSrc || im.src;
       if (src) return { type: src.startsWith('data:') ? 'data' : 'url', src: src, alt: im.alt || '' };
     }
     // 2) an inline <svg> inside a home link or logo-classed element
     const svgHost = scope.querySelector('a[href="/"] svg, [class*="logo" i] svg, a[aria-label*="home" i] svg');
     if (svgHost) {
       const s = svgHost.cloneNode(true);
       if (!s.getAttribute('xmlns')) s.setAttribute('xmlns', 'http://www.w3.org/2000/svg');
       // Inline sprite references — a standalone file can't reach the page's
       // <symbol>/sprite definitions, so <use href="#id"> would render empty.
       const defs = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
       let unresolved = false;
       s.querySelectorAll('use').forEach(u => {
         const ref = u.getAttribute('href') || u.getAttribute('xlink:href') || '';
         if (!ref.startsWith('#')) { unresolved = true; return; }
         const node = document.getElementById(ref.slice(1));
         if (node) defs.appendChild(node.cloneNode(true)); else unresolved = true;
       });
       if (unresolved) return { type: 'svg-unresolved' };
       if (defs.childNodes.length) s.insertBefore(defs, s.firstChild);
       return { type: 'svg', markup: s.outerHTML };
     }
     return null;
   })()
   ```

   **6b. Save it** into the client folder. Use exactly one of three canonical
   extensions — `svg`, `png`, `jpg` — so downstream consumers that look for
   `logo.{png,svg,jpg}` always find it. **Normalise `.jpeg` → `.jpg`.**

   - `type: "svg"` → write `markup` to `{outputDir}/logo.svg`.
   - `type: "url"` → take the extension from the URL path (`.svg`, `.png`,
     `.jpg`/`.jpeg` — save `.jpeg` as `logo.jpg`; anything else → skip with a
     note, don't guess at conversions) and download via shell:
     `curl -fsSL "<src>" -o "{outputDir}/logo.<ext>"` — verify the file is
     non-empty afterwards.
   - `type: "data"` → decode the data-URI with `python3` (extension from the
     MIME type, `image/jpeg` → `jpg`; skip non-svg/png/jpeg with a note).
   - `type: "svg-unresolved"` → the SVG references a sprite the snippet could
     not resolve (external `use` href or missing symbol) — treat as
     unextractable, same as `null`.
   - `null` → the logo is a CSS background, font icon, or otherwise
     unextractable — note it; the downstream workflow will ask the user
     instead (its option 1/2/3 logo prompt).
   - Sanity check a saved `logo.svg`: it must contain a `viewBox` or
     `width`/`height`, and no remaining `<use` whose target is not inside the
     file. If the check fails, delete the file and report "not captured".

7. (Optional cleanup) The browser stays open for the rest of the session. No need to close it unless the user asks.

**Multi-page capture** (only if `{crawlScope}` is `"multi"`):

After capturing the primary page, extract navigation links and visit up to 2 additional pages from the same site.

**8a. Extract nav links** from the already-loaded primary page:

```js
(function() {
  const links = [];
  const seen = new Set([location.pathname]);
  const skip = /\/(login|logout|signup|sign-in|register|auth|account|checkout|cart|password|reset)\b/i;
  document.querySelectorAll('nav a, header a, [role="navigation"] a').forEach(a => {
    try {
      const u = new URL(a.href);
      if (u.hostname !== location.hostname) return;
      if (skip.test(u.pathname)) return;
      if (seen.has(u.pathname)) return;
      if (u.pathname === '/' && location.pathname !== '/') return;
      seen.add(u.pathname);
      links.push({ href: u.origin + u.pathname, text: (a.innerText || a.textContent).trim().slice(0, 40) });
    } catch(e) {}
  });
  return links.slice(0, 6);
})()
```

**8b. Pick 2 pages** from the returned list. Prefer:
- Shorter paths (`/product` over `/product/features/detail`)
- Distinct section names (Product, Community, Pricing, Docs — not just different slugs under the same section)
- Avoid the root `/` if you already captured it

**8c. For each additional page**, run a lighter capture loop (navigate → wait 3s → viewport screenshot only → DOM extractor). Skip full-page/mid/footer screenshots for pages 2-3 — the viewport is enough to confirm design consistency, and the DOM extractor captures the full page regardless. Name screenshots `{domain}-viewport-2.jpeg`, `{domain}-viewport-3.jpeg`. Store each page's data as `pages[1]`, `pages[2]` (with `pages[0]` being the primary URL).

If a linked page fails to load or is blocked, skip it silently and note it in the Phase 6 report.

**8d. Merge data** before Phase 2. Combine color candidates across all pages (union), typography across all pages (union). When a value appears on 2+ pages it is a **system signal** — mark it. Values that appear on only one page are **local** — still capture them, but weight them lower.

In Steps 1–3, always state how many pages were analyzed ("Across 3 pages: …") and distinguish system patterns from local ones. Multiple-page agreement is the strongest available evidence for a taste principle.

**Success criterion for Phase 1**: you have at least one screenshot AND a populated `domData` JSON (from the primary page) with `colors`, `typography`, `layout`, `effects`, `cards`, `images`, `spacingSamples`, `sectionGaps`, and `grid` fields. If any of these are completely empty *and* the screenshot shows real content, the extractor needs improvement — note it and proceed anyway.

**Degraded-data path**: if `domData.errors` is non-empty, the extractor partially failed. In Step 1, mark affected categories with `~approx (extractor section failed)` and rely on the screenshot for those measurements. In Phase 6, list the failed sections as a caveat in the report.

### Phase 2 — Run the 4-step analysis

> **Screenshot is ground truth.** Before reading any reference file: the viewport screenshot shows what a human actually sees. The DOM data supplies exact numbers. When they conflict, trust the screenshot. A color that appears in DOM but covers <5% of visible surface area is decorative — do not list it as a brand color. Section dividers that don't appear in the screenshot don't exist, regardless of what the DOM reports. Imperceptible shadows (≤3% opacity) are CSS artifacts, not design decisions.

The four reference files in `references/` contain the full prompts for each step. Read each one when you reach it (don't preload all four at once — the steps are deliberately sequential and earlier steps don't need later instructions).

For each step:
1. Read the corresponding reference file.
2. Generate the output it asks for, in your own response.
3. Start the output with the marker line specified in the reference (e.g., `### Step 1 Output — Measurements`).
4. The next step gets your prior outputs as input — keep them in the conversation, don't summarize them away.

#### Step 1 — Measure
- Read `references/step1-measure.md`
- Inputs: `domData` (the JSON from `browser_evaluate`) + the screenshot
- Produce: 20 categories of measurements, every value cited with a specific px / hex / ratio

#### Step 2 — Pattern
- Read `references/step2-pattern.md`
- Inputs: Step 1 output
- Produce: 5-8 systematic patterns, each with Pattern / Evidence / Design Goal

#### Step 3 — Taste
- Read `references/step3-taste.md`
- Inputs: Step 1 + Step 2 outputs
- Produce: 4 taste principles (each Trigger / Decision / Reason / Evidence), at least one Restraint trade-off

#### Step 4 — Observer
- Read `references/step4-observer.md`
- Inputs: Step 1 + 2 + 3 outputs + the original URL (you have to substitute it into the JSON's `source` field)
- Produce: a fenced ```json``` block followed by the Markdown Design Map + Taste DNA

### Phase 3 — Write the output files

Step 4's response contains both formats. Extract them:

1. **JSON**: find the first ```json``` fenced code block in Step 4's output. Parse it (it must be valid JSON — if it isn't, that's a Step 4 failure; re-run Step 4 with the parse error pointed out).
2. **Markdown**: take everything in Step 4's output *after* the JSON block ends. Strip leading whitespace.

Then write two files into `{outputDir}` (from Phase 0; use `pwd` if you need to confirm where the current working directory is), using the `{domain}` you extracted in Phase 0:

- `{outputDir}/{outputFilename}` — the Markdown portion
- `{outputDir}/{domain}.json` — the JSON, pretty-printed with 2-space indent

If either file already exists, overwrite without asking. Users invoking this skill twice on the same site expect the latest analysis to win.

### Phase 4 — Self-audit (Anti-Slop pass)

Run this grep via a shell — do not scan mentally:

```bash
grep -icE 'clean|modern|sleek|visually appealing|user-friendly|intuitive|seamless|\belegant\b|minimalist|polished|refined|sophisticated|delightful|stunning|\bbeautiful\b|\bpremium\b' {outputDir}/{outputFilename}
```

If the count is **0**, the file passes. If **>0**, show the offending lines:

```bash
grep -inE 'clean|modern|sleek|visually appealing|user-friendly|intuitive|seamless|\belegant\b|minimalist|polished|refined|sophisticated|delightful|stunning|\bbeautiful\b|\bpremium\b' {outputDir}/{outputFilename}
```

For each hit:
- Inside a quoted foil ("they could have gone for a 'clean and modern' look, but instead…") — fine, counts as passing.
- Used as a positive descriptor of *this* design — slop. Edit the principle, re-save, re-run grep until count is 0.

Also verify with a shell:

```bash
grep -c "Design Map\|Taste DNA" {outputDir}/{outputFilename}
```

Must return **2**. If less, a section is missing — re-run Step 4 with the missing section called out.

Validate the JSON:

```bash
python3 -m json.tool {outputDir}/{domain}.json > /dev/null && echo "valid" || echo "INVALID"
```

If any check fails, fix and re-run before shipping.

### Phase 5 — Export for your build tool

You already captured `{exportTarget}` in Phase 0. Do not ask again.

If `{exportTarget}` is "skip", go straight to Phase 6.

If `{exportTarget}` is "all", generate every file-based format (options 1–8).

Otherwise, use `{exportTarget}` to determine the export. Accept natural-language equivalents: "cursor", "windsurf", "claude" / "claude code", "copilot", "bolt", "antigravity" / "gemini", "v0", "figma" / "figma make", "lovable".

Read `references/export-formats.md` for the exact file paths, frontmatter structure, and content spec for each target. All formats draw from `{domain}.json` and must follow the same Anti-Slop rules — no vague vibes, cite specific px/hex.

### Phase 6 — Report

Print to the user (substitute actual `{domain}`, line count, and byte count):

```
✓ Wrote {outputDir}/{outputFilename} (<line-count> lines) and {outputDir}/{domain}.json (<byte-count> bytes)
[if export file(s) written]: ✓ Also wrote <path(s)>
[client runs — one of]:
  ✓ Saved site logo to {outputDir}/logo.<ext>
  · Logo already present ({outputDir}/logo.<ext>) — kept as-is
  ✗ Logo not captured (<reason>) — the content workflow will ask for it at its brand-context step

  Sharpest principle: "<name of the strongest taste principle>"
  → <one-sentence quote of its Decision>

  Open {outputFilename} to read the full Design Map + Taste DNA.
```

Keep it tight. The files do the talking.

## Error handling

| Failure | Response |
|---|---|
| Playwright browser tools not installed/connected | Stop with the exact install command from the user's adapter (`adapters/*.md`). Do not try to fall back to a plain HTTP fetch — the screenshot is essential for Step 1. |
| URL is unreachable / 404 / DNS fail | Tell the user the page didn't load. Don't ship an analysis based on an error page. |
| Cloudflare / bot-detection block | The screenshot will show a "verify you are human" page. Detect this in Step 1 (huge whitespace, single button labeled "Verify"). Tell the user the site blocks automated access; suggest they manually open the URL in their own browser if they want a screenshot, or try a different page on the same domain. |
| SPA still rendering after the wait | After two 3-second waits, proceed with what's there. Add a one-line caveat to the report: "Page may have been mid-hydration; some measurements approximate." |
| User gave a file path, not a URL | Ask whether they meant a hosted version. Don't try to open local files — this skill is for live pages. |
| User gave a URL behind a login | Detect the login page (form with password input). Tell the user this skill doesn't handle authenticated pages, suggest using the public marketing page instead. |
| Logo capture fails (client runs) | Never fatal. Skip, keep the taste pipeline going, and report the reason in Phase 6 — downstream workflows have their own logo prompt as fallback. |
| Step 4 returns malformed JSON | Re-run Step 4 once with the specific parse error pointed out. If it fails again, save the raw output as `{outputDir}/{domain}-raw.md` and tell the user to inspect manually. |

## Notes on philosophy (read once, internalize, then mostly forget)

This skill is built on a strong opinion: design tokens alone are useless for an AI agent. Tokens say "spacing is 8px"; taste says "spacing is 8px *because* the designer chose readable rhythm over data density, and 8px is the smallest unit that still feels intentional at 1× zoom". The Reason is what makes the same agent generate good design for a *different* page, instead of just copying numbers.

The four prompts in `references/` are a port of a Gemini-based pipeline that was validated in production. The single biggest failure mode in early versions was generic output — paragraphs of "this design uses a clean, modern aesthetic with intuitive navigation" that contain zero information. The Anti-Slop checks (banned phrases, mandatory Restraint trade-off, evidence requirement) exist because every weak principle deleted is worth ten extra well-evidenced ones added.

When in doubt about whether a taste principle is good: ask "could I have written this without ever seeing this specific website?" If yes, it's slop. Throw it out.
