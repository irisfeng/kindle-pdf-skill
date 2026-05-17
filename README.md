# kindle-pdf-skill

> Fix the technical-PDF-on-Kindle disaster. Matrices, code blocks, and equations stay intact. No reflow nightmare.

English | [简体中文](./README.zh.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

An [Agent Skill](https://docs.claude.com/en/api/agent-skills) that teaches Claude (and other compatible agents) how to properly convert oversized technical PDFs into Kindle-friendly form — without destroying the layout of code, tables, matrices, and equations.

## The Problem

You found a great technical book — say, *Build a Large Language Model (From Scratch)* by Sebastian Raschka. The PDF is a Letter-sized brick, ~7.4 × 9.2 inches per page. You want to read it on your 6-inch Kindle.

You have two "obvious" options. Both fail:

1. **Drop the original PDF onto the Kindle.** It opens. It's readable. But the text is *tiny* — the entire 7.4-inch page squeezed onto a 3.5-inch usable column. You squint, you pinch-zoom, you give up.
2. **Convert PDF → AZW3 / MOBI (e.g. Calibre, Send-to-Kindle email).** Now the text is the right size and reflows. But every 2D structure is **destroyed**:

<p align="center">
  <img src="./assets/before-kindle-disaster.jpg" alt="Before: PDF→AZW3 reflow disaster — a 2D table fragmented into single-column vertical text" width="300">
  &nbsp;&nbsp;&nbsp;
  <img src="./assets/after-kindle-readable.jpg" alt="After: same book, k2pdfopt-optimized PDF — figure, vectors, and caption all intact" width="300">
</p>
<p align="center"><em>Left: PDF converted to AZW3. Right: same book, this skill's k2pdfopt-optimized PDF.</em></p>

On the left (AZW3 conversion), a 2D table has been shredded into a single column. Each cell — `3`, `the first token ID`, `over`, `5`, `dog`, `1`, `1.2753` — is on its own line. Equations are torn apart. Code indentation is gone. **The book is unreadable.**

On the right (this skill's output, still a PDF), a multi-row figure with three labeled embedding vectors (`[1.23, -0.31, 0.89]` …) stays atomic, the caption flows under it, and the chapter heading and page number land exactly where they should.

The root cause: **AZW3 / MOBI / EPUB are reflowable formats**. The converter has to extract a single linear text stream from your PDF. Anything that lives in a 2D grid — code blocks, matrices, tables, equations, side-by-side figures — gets serialized into nonsense.

The fix isn't to convert to a different format. The fix is to **keep the PDF as a PDF**, and just re-cut its pages to fit a 6-inch screen.

## The Solution

There's a tool called **[k2pdfopt](https://www.willus.com/k2pdfopt/)** that re-slices each PDF page into multiple device-sized sub-pages, **keeping rows and code blocks atomic**. But k2pdfopt has 100+ flags and most online tutorials get the critical ones wrong.

This skill encodes the exact recipe — flag combinations, device parameters, traps to avoid, and verification steps — for getting a usable result on the first try.

| | Without skill | With this skill |
|---|---|---|
| Tutorials needed | 5+ blog posts, mostly wrong | 0 — agent handles it |
| First-try success | ~10% | ~90% |
| Common landmines (`-mode copy` doesn't split pages, `-rt 0` doesn't keep portrait, etc.) | Hit every one | All documented |
| Code/matrix/equation preservation | Random | Guaranteed |
| Old Kindle support (Paperwhite 1/2) | "Just buy a new Kindle" | Optimized parameter set |

## What's in the box

```
skills/kindle-pdf-skill/
├── SKILL.md                          # ~13k chars — the main playbook
└── references/
    ├── k2pdfopt-flags.md             # Full flag reference + device cheat sheet
    └── security-policy-pitfalls.md   # macOS Gatekeeper, blocked shell patterns
```

The skill covers:

- **Diagnostic checklist** — read the PDF metadata, identify the failure mode, pick the right tool
- **k2pdfopt installation** on macOS, including how to get past Gatekeeper quarantine
- **Recipes for every Kindle generation** — Paperwhite 1/2 (212ppi), PW3/4 (300ppi), PW5/Signature, Oasis, Scribe
- **Landscape vs portrait output tradeoff** — when to use `-ls-`, when to bump `-fs 1.2`
- **Verification** — how to confirm the output is correct (and why `vision_analyze` lies about page orientation)
- **Pitfalls section** — every mistake the author personally made, so you don't have to

## Install

### Option A: Claude Code Plugin (recommended)

```bash
/plugin marketplace add irisfeng/kindle-pdf-skill
/plugin install kindle-pdf-skill@kindle-pdf-skill
```

### Option B: Hermes Agent

[Hermes Agent](https://hermes-agent.nousresearch.com/) loads skills from `~/.hermes/skills/`. Clone or copy this skill:

```bash
mkdir -p ~/.hermes/skills/productivity
cp -r skills/kindle-pdf-skill ~/.hermes/skills/productivity/
```

Then in any new Hermes session, just ask:

> My PDFs look terrible on Kindle. Help me fix them.

The agent will load this skill automatically.

### Option C: Manual / Other Agents

The whole skill is in [`skills/kindle-pdf-skill/SKILL.md`](./skills/kindle-pdf-skill/SKILL.md). It's a self-contained markdown document. Drop it into any agent system that supports custom instructions (Cursor rules, Codex prompts, ChatGPT custom instructions, etc.).

## How to use it

Once installed, just describe the symptom in plain language. The agent will:

1. Inspect your PDF (page size, content type, scan-vs-native)
2. Identify your Kindle model (ask for a photo if unsure — Paperwhite 2 looks very different from Paperwhite 5)
3. Install k2pdfopt if needed (and tell you exactly how to get past the macOS Gatekeeper)
4. Run the conversion with the correct flags for your specific device
5. Verify the output (and warn you about the landscape-vs-portrait tradeoff)

Example prompts that trigger this skill correctly:

- "My PDFs look terrible on my Kindle"
- "How do I make technical PDFs readable on a 6-inch e-reader?"
- "PDF排版在Kindle上一塌糊涂" (Chinese)
- "Convert this PDF for my Paperwhite"

## Why this exists

I ([@irisfeng](https://github.com/irisfeng)) have an old Kindle Paperwhite 2 and a lot of ML / math / CS PDFs. Every "convert PDF to Kindle" tutorial online either:

- Recommends Calibre (destroys technical layout)
- Recommends mobi/AZW3 conversion (destroys technical layout in a different way)
- Uses wrong k2pdfopt flags (`-mode copy` famously *doesn't* split pages)
- Assumes you have a brand-new Kindle Scribe

This skill is the result of an iterative session with [Hermes Agent](https://hermes-agent.nousresearch.com/) where every wrong path was tried, every landmine stepped on, and the working recipe was captured before it could be forgotten. That's what skills are for — turning expensive lessons into cheap, repeatable knowledge.

## Contributing

PRs welcome — especially:

- Parameter recipes for non-Kindle devices (Kobo, Boox, reMarkable, etc.)
- Translations of the skill into other languages
- New pitfalls you discovered

Open an issue first for anything larger than a typo fix.

## Acknowledgments

- **[willus](https://www.willus.com/)** — author of k2pdfopt, an unsung hero of e-ink readers
- **[Sebastian Raschka](https://sebastianraschka.com/)** — whose excellent *Build a LLM from Scratch* book triggered this entire endeavour
- **[Anthropic](https://github.com/anthropics/skills)**, **[Matt Pocock](https://github.com/mattpocock/skills)**, **[JimLiu](https://github.com/JimLiu/baoyu-skills)**, **[multica-ai](https://github.com/multica-ai/andrej-karpathy-skills)** — for the open-source skill conventions this repo follows
- **[Hermes Agent](https://hermes-agent.nousresearch.com/)** — where this skill was forged

## License

[MIT](./LICENSE) © 2026 偷泥方 (irisfeng)
