---
name: kindle-pdf-skill
description: Reflow/optimize PDFs for small e-readers (Kindle 6", Kobo, etc.) so technical books with code/tables/equations stay readable. Covers k2pdfopt, Calibre, and Kindle device settings.
version: 1.1.0
author: 偷泥方 (irisfeng)
license: MIT
homepage: https://github.com/irisfeng/kindle-pdf-skill
metadata:
  hermes:
    tags: [pdf, kindle, ereader, k2pdfopt, ebook, conversion]
    related_skills: []
---

# E-reader PDF Optimization

## When to load this skill

Load when the user asks any of:
- "Why is my PDF unreadable on Kindle / Kobo / e-reader?"
- "How do I convert / optimize / reformat a PDF for [small e-reader]?"
- "PDF排版在Kindle上一塌糊涂" (Chinese: PDF layout on Kindle is a mess)
- Anything involving "PDF" + "Kindle" / "e-ink" / "6 inch" / "reflow" / "排版" / "重排"

Especially when the PDF is a technical book (ML, math, CS) with matrices, code blocks, tables, or formulas — these are the cases where naïve approaches fail catastrophically.

## Root cause: why technical PDFs look terrible on small Kindles

A 6-inch Kindle screen is ~90×120mm. A typical technical PDF page is 7-8 inches wide (Letter or B5). The user has two "obvious" options, **both bad**:

1. **Keep it as PDF, drop onto Kindle (1:1 / "Original" mode)** — text becomes too small to read. Kindle's PDF view is fixed-layout; pinch-zoom/pan works but is miserable for a whole book.
2. **Convert PDF → AZW3 / MOBI / EPUB** (Calibre, Send-to-Kindle email, ebook-convert, etc.) — text becomes the right size and reflows, **BUT every 2D structure is destroyed**: tables, matrices, code blocks, equations, side-by-side figures all get serialized into a single linear text stream. The user's diagnostic symptom: rows like `3 / the first token ID / over / 5 / dog / 1 / 1.2753` where each cell of a 2D table lands on its own line.

**The reflowable-format conversion disaster is the #1 complaint and the giveaway diagnostic.** If the user shows a screenshot with single-column-vertical-data fragments, that almost always means they tried to convert the PDF to AZW3/MOBI/EPUB (or used Kindle's experimental PDF reflow mode, which has the same effect). Don't let them go further down that path.

**The fix is NOT to find a better reflowable format.** The fix is to **keep the PDF as a PDF** and re-cut its pages to fit the 6-inch screen — which is exactly what k2pdfopt does.

## Diagnostic checklist (do this first)

Before recommending a tool, gather these facts via vision/terminal:

1. **PDF page size** — `mdls -name kMDItemPageHeight -name kMDItemPageWidth -name kMDItemNumberOfPages "/path/to.pdf"` (macOS) or `pdfinfo` (poppler). Compare to the device screen.
2. **PDF content type** — text-only novel? Or technical with tables/code/math? This decides which tool to use.
3. **Device screen size** — Kindle 6" (Paperwhite/basic), Kindle Oasis 7", Kindle Scribe 10.2", Kobo Libra 7", etc. Ask if unclear.
4. **What does the failure look like?** — request a photo if possible, run `vision_analyze` to confirm reflow-disaster vs. tiny-text vs. wide-margins.

## Solutions, ranked by quality for technical PDFs

### 🅰 k2pdfopt — best for technical books on 6-7" e-readers

Re-slices each PDF page into multiple smaller pages sized to the device, preserving the 2D structure of tables/code/equations while still being readable at the target screen size. **The right answer for technical PDFs on 6" Kindle.**

**Recommend k2pdfopt when:** technical content + 6-7" device. This is the durable best-practice recommendation; don't reach for Calibre first for technical books.

**Install (macOS):**
- `brew install k2pdfopt` — **NOT IN HOMEBREW as of 2024-2026**. Don't waste a turn trying.
- Official binary: https://www.willus.com/k2pdfopt/download/ — requires a 3-digit captcha + browser session. **The captcha is fragile inside automated browser tools** (Browserbase/stealth combos black out the page; vision can't reliably read the noisy 3-digit image). Best path: **ask the user to download it themselves** and pass you the binary path. Don't burn 10 tool calls fighting the captcha.
- Source build: GitHub mirror at `remram44/k2pdfopt` (official tarball mirror) or `mo3rfan/k2pdfopt` (has cmake fixes + Dockerfile). Requires cmake + several deps. Only attempt if the user explicitly allows `git clone` (some users have security policies that block it — see references/security-policy-pitfalls.md).

**Post-download setup on macOS (manual binary):**

```bash
# Strip Gatekeeper quarantine + provenance attrs, make executable, install globally
xattr -d com.apple.quarantine ~/Downloads/k2pdfopt 2>/dev/null
xattr -d com.apple.provenance ~/Downloads/k2pdfopt 2>/dev/null
chmod +x ~/Downloads/k2pdfopt
sudo cp ~/Downloads/k2pdfopt /usr/local/bin/k2pdfopt
file /usr/local/bin/k2pdfopt   # verify: Mach-O 64-bit executable arm64
```

Without removing `com.apple.quarantine`, the binary will be killed by Gatekeeper on first run with a vague "cannot be opened" dialog.

**Run (the actual correct commands for technical PDFs on 6" Kindle):**

```bash
# ⭐ RECOMMENDED for 6" Kindle reading 7-9" technical PDFs: fitwidth mode
# Slices each source page into 2-3 device-sized sub-pages, text gets bigger,
# code/matrices/equations stay intact horizontally.
k2pdfopt "input.pdf" -ui- -x -mode fitwidth -h 1024 -w 758 -dpi 212 \
  -as - -mc- -ws 0.01 -o "output_PW2.pdf"

# Same but for Paperwhite 3+ / Voyage / Oasis (higher-res screens)
k2pdfopt "input.pdf" -ui- -x -mode fitwidth -h 1448 -w 1072 -dpi 300 \
  -as - -mc- -ws 0.01 -o "output_PW3.pdf"

# 🎯 IF USER PREFERS HOLDING KINDLE PORTRAIT (most common): add -ls-
# Without -ls-, fitwidth rotates content sideways → user must hold Kindle landscape.
# With -ls-, content stays upright → user holds Kindle normally, reads ~390 pages
# (one source page = one device page; text smaller but standard orientation).
k2pdfopt "input.pdf" -ui- -x -mode fitwidth -h 1024 -w 758 -dpi 212 \
  -ls- -as - -mc- -ws 0.01 -o "output_PW2_portrait.pdf"

# 🎯 IF FONT IS TOO SMALL in portrait mode: bump font scale with -fs
# Tradeoff: larger font = more page breaks, but easier on the eyes.
k2pdfopt "input.pdf" -ui- -x -mode fitwidth -h 1024 -w 758 -dpi 212 \
  -ls- -fs 1.2 -as - -mc- -ws 0.01 -o "output_PW2_bigfont.pdf"

# For PROSE-heavy novels (no tables/code), full reflow looks best
k2pdfopt "input.pdf" -ui- -x -dev kp3 -mode fp -o "output_novel.pdf"
```

**Landscape vs portrait output — explain this tradeoff to the user upfront so you don't have to redo the conversion:**

| Aspect | Default (no `-ls-`) | With `-ls-` (portrait) |
|---|---|---|
| Kindle holding orientation | Landscape (rotated 90°) | Portrait (normal) |
| Output page count (e.g. 370-pg book) | ~730 (2x) | ~390 (~1x) |
| Font size on 6" screen | Larger | Standard |
| Page turn frequency | More frequent | Normal |
| Best for | Eyes that need biggest text | Standard reading posture |

**Default recommendation depends on content type and device size:**

- **6" device (PW1-PW4) + technical book with code/tables/equations** → **landscape (NO `-ls-`) is the default.** Portrait + `-fs` cannot deliver enough text size for ML/math books at this screen size — physics, not preference. See `references/physical-zoom-limits.md`. User holds Kindle sideways but reads comfortably.
- **6" device + prose-heavy book** → portrait (`-ls-`) is fine; reflow tools (Calibre AZW3) may actually be better here.
- **6.8"+ device (PW5, Signature, Oasis, Scribe)** → portrait (`-ls-`) usually works even for technical content. Try it first.

If they specifically refuse landscape posture on 6" device for technical content, be honest: tell them the upper bound of comfort is what `-fs 1.15` produces in portrait mode, and the durable fix is a bigger device. Don't promise a "one more flag will fix it" outcome — there isn't one.

**Critical flags — read carefully, this is where session-after-session gets burned:**

- `-mode fitwidth` — **THIS is the right mode for technical PDFs on small screens.** It splits each source page horizontally into multiple device-width sub-pages while keeping code/equation/table rows together. Each source page becomes 2-3 Kindle pages; final page count roughly doubles.
- `-mode copy` — **MISLEADINGLY NAMED. Does NOT split pages.** It only crops white margins and renders at original size. If the source PDF is 7×9 inches and device is 6 inches, `copy` mode output is still too big to read. **The common mistake** is assuming "copy" implies device-sized copy — it doesn't. Use `fitwidth` instead.
- `-mode fp` / `-mode def` — full reflow. Destroys 2D structure (same disaster as Kindle native reflow). Only OK for prose.
- `-mode 2col` — assumes 2-column academic paper layout.
- `-h <px> -w <px> -dpi <ppi>` — **explicit physical device params. Prefer these over `-dev` presets when you know the exact device**, because `-dev` presets bake in assumptions (e.g. `kp3` assumes 1072×1448 @ 300ppi; using it for an older PW1/PW2 — 758×1024 @ 212ppi — produces sub-pages that are still too large).
- `-dev <code>` — device preset shortcut: `kp2` (Paperwhite 2), `kp3` (Paperwhite 3), `kpw5` (Paperwhite 5/Signature), `koa` (Oasis), `kv` (Voyage), `k3` (basic Kindle), `dx` (DX), `kbg` (Kobo Glo). Use only when device is clearly modern and you don't want to look up its exact pixels.
- `-ui-` — no interactive prompt (run unattended).
- `-x` — exit when done (don't open viewer).
- `-as -` — disable auto-skew correction (native PDFs aren't skewed; only enable for scans).
- `-mc-` — no markup/crop guides in output.
- `-ws 0.01` — tight word-spacing threshold; helps the page-segmenter identify code blocks and matrices as atomic units.
- `-n` — keep native PDF text (selectable + searchable on Kindle). Without `-n` the output is bitmap (smaller file, no text selection).
- `-rt 0` — *intended* to force no rotation. **In `-mode fitwidth` with a large source page, this flag is effectively ignored** — k2pdfopt still rotates content to fit the narrow device. Don't bother re-running with `-rt 0` if your first output looks "rotated" in a viewer; see the rotation pitfall below. **Use `-ls-` instead** — that's the actual correct flag for "keep content upright on a portrait-held device."
- `-ls-` — **THE correct flag for portrait-orientation output.** Forces k2pdfopt to NOT rotate content into landscape. Without it, fitwidth rotates content sideways to maximize text width; with it, the content stays upright and each source page becomes ~1 device page (text is smaller, but Kindle is held normally). This is the parameter to reach for when the user complains about "having to hold Kindle sideways."
- `-fs <factor>` — font scaling. `-fs 1.2` = "20% bigger" *nominally*. **READ THE NEXT PITFALL** before promising the user this will fix small text — on a 6" Kindle reading a 7-8" wide technical PDF, `-fs` only buys you ~10-15% perceived gain before it hits a hard wall, because the page width is already fully consumed by fitting one full source row into 758px. Tradeoff: more page breaks. See `references/physical-zoom-limits.md` for the math and the symptom: "I added -fs 1.25 but the text didn't actually get bigger."

A 300-400 page book takes 5-15 minutes on Apple Silicon; expect output to be 700-900 pages and 40-80 MB after `fitwidth` doubles the page count.

**Device → physical params cheat sheet (for `-h/-w/-dpi`):**

| Device | Pixels | DPI | Year |
|--------|--------|-----|------|
| Kindle Paperwhite 1/2 | 758×1024 | 212 | 2012–2013 |
| Kindle Paperwhite 3/4 | 1072×1448 | 300 | 2015–2018 |
| Kindle Paperwhite 5 / Signature | 1236×1648 | 300 | 2021+ |
| Kindle Voyage | 1080×1430 | 300 | 2014 |
| Kindle Oasis 2/3 (7") | 1264×1680 | 300 | 2017–2019 |
| Kindle Scribe (10.2") | 1860×2480 | 300 | 2022+ |

When in doubt about device age, **ask the user for a photo and identify by bezel/logo** (Paperwhite 2 has no logo on front, plastic flush bezel, no physical buttons, uneven front-light bleed). Old-device + new-preset = pages still too large.

### 🅱 Calibre — second choice, OK for plain text, mediocre for technical

`brew install --cask calibre` then `ebook-convert input.pdf output.azw3 --output-profile kindle_pw3 --pdf-engine pdftohtml`.

**Limits:** Calibre's PDF→EPUB/AZW3 path treats PDF as a flat text stream; it will mangle most tables, matrices, and code blocks the same way Kindle's native reflow does. Use it for novels and prose-heavy books only.

### 🅲 Kindle device-side settings — zero-install fallback

When the user can't or won't install anything:
- Settings → turn OFF "Reflow PDF" / "Enhanced View" (exact label varies by firmware). This forces original-layout display.
- Rotate device to landscape (orientation lock off).
- Use the crop tool to remove white margins (gives ~15% more usable area).
- For 6" devices reading technical PDFs: **honestly tell the user this will never feel like reading on a laptop**. A 10" Boox / Kindle Scribe / iPad is the physical fix.

## Decision tree (give this to the user)

```
Is the PDF a novel / mostly prose?
├─ Yes → Calibre PDF→AZW3, or just use Kindle's reflow mode.
└─ No (technical, has tables/code/math):
   ├─ Device is 6-7" e-reader? → k2pdfopt with -dev <device> -mode copy
   ├─ Device is 10"+ (Kindle Scribe, Boox Note, iPad)? → Read PDF natively, no conversion needed.
   └─ User refuses to install tools? → Kindle landscape + original mode + crop.
```

## Pitfalls

- **THE BIG ONE: `-fs N` does not give you N× bigger text on a 6" Kindle reading a 7-8" technical PDF.** A 758-pixel-wide 6" screen showing a 7.4" wide source page is already fitting ~90 source-page characters into 758 pixels — every character is already at minimum-readable size. `-fs 1.25` cannot magically widen the device; in `-mode fitwidth -ls-` (portrait, one source page → one device page) it produces only a marginal increase (~10-15% perceived), not 25%. **If a user says "I added -fs 1.25 but the text didn't actually get bigger" — believe them, this is the physics, not user error.** The only ways to actually get +20-30% text on a 6" device reading a wide technical PDF are: (a) drop `-ls-` so each source page splits into 2-3 landscape sub-pages (user holds Kindle sideways, text is much bigger), (b) accept horizontal slicing that breaks code-line / table-row integrity, or (c) move to a 6.8"+ device. There is no fourth option — `-fs` alone won't get you there. Full math + measurements in `references/physical-zoom-limits.md`.
- **For technical books on 6" devices, default to recommending landscape output (NO `-ls-`).** The previous default in this skill was portrait + `-fs 1.2` to "keep normal posture." In practice users come back and say "still too small." Landscape posture trades reading ergonomics for ~35% bigger text and is the durable answer for ML/math books on PW1-PW4 (6" screens). Mention the tradeoff explicitly so they choose with eyes open — but don't bury landscape as "alternative." Make it the headline option for technical content on 6" devices.
- **`-mode copy` does NOT split pages — it only crops white margins.** Despite the name suggesting "copy to device size," it renders at original page size. If source is 7×9" and device is 6", the output is still too big. **Use `-mode fitwidth` for actual device-sized splitting.** This is the single most common k2pdfopt mistake.
- **`-dev` presets assume a modern device.** `-dev kp3` bakes in 1072×1448 @ 300ppi; on an older PW1/PW2 (758×1024 @ 212ppi) the resulting sub-pages are still oversized. Prefer explicit `-h/-w/-dpi` once you know the actual device generation.
- **macOS quarantine kills manually-downloaded binaries silently.** First run of `~/Downloads/k2pdfopt` fails with a Gatekeeper dialog. Always `xattr -d com.apple.quarantine` + `chmod +x` before running.
- **Don't recommend Calibre first for ML/math/CS books.** Users come back disappointed. k2pdfopt is the right answer.
- **Don't burn turns on the willus.com captcha** if you're operating through automated browser tools — it's fragile. Ask the user to download manually.
- **PDF page size ≠ device size.** Always check both with `mdls`/`pdfinfo` before recommending a conversion approach.
- **Brew has no k2pdfopt formula** (as of last check). Don't try `brew install k2pdfopt` — it 404s.
- **iCloud Drive paths**: user's books may live at `~/Library/Mobile Documents/com~apple~CloudDocs/...`. Quote the path; spaces and `~` expand correctly only when quoted.
- **The reflow-disaster screenshot is the diagnostic — but ask about origin before assuming cause.** If the user sends a Kindle photo showing single-column vertical data fragments, that almost always means they ran the PDF through a **PDF→AZW3/MOBI/EPUB converter** (Calibre, Send-to-Kindle email, ebook-convert) — NOT that Kindle's native PDF viewer mangled it. Kindle's native PDF mode is fixed-layout and renders the original PDF as-is (text just looks tiny on 6"); it does not auto-reflow into single columns. **Before launching into k2pdfopt, confirm: "Is this the original PDF on Kindle, or did you convert it to AZW3/MOBI first?"** The fix is the same (k2pdfopt on the original PDF), but the explanation you give the user must match reality — otherwise they'll think the problem is their Kindle and consider buying a new one. Full write-up in `references/diagnostic-attribution.md`.
- **Transferring to old Kindle (Paperwhite 1/2):** Send-to-Kindle email/cloud features are unreliable on these vintages. The durable path is USB — plug into Mac, the Kindle mounts as a disk named `Kindle`, drag the PDF into `documents/`, eject. Tell the user to set the PDF to "Actual Size" / "Original" view on-device (not Reflow) since k2pdfopt already did the work.
- **DO NOT tell the user to email the k2pdfopt-optimized PDF via Send-to-Kindle.** Amazon's Send-to-Kindle service auto-converts PDFs (to AZW/KFX on older firmware, or re-processes them), which will destroy the careful page slicing k2pdfopt did. The user will get the matrix-explosion / formula-fragmentation disaster all over again. **USB transfer only** for k2pdfopt outputs.
- **Don't recommend converting to .mobi / .azw3 / .epub for technical PDFs.** Users sometimes ask "do I need to convert PDF to mobi for Kindle?" The answer is NO — Kindle reads PDF natively. Mobi/AZW3/EPUB are reflowable formats; converting a technical book to them destroys code indentation, matrix alignment, and equation layout. The right answer for technical PDFs is "keep it as PDF, but use k2pdfopt to make it Kindle-sized."
- **`-rt 0` (force no rotation) does NOT keep the source page upright** when the source page is bigger than the device in both dimensions. In `-mode fitwidth`, k2pdfopt always rotates internally to maximize width usage on the narrow device — the output PDF will still have content rotated relative to the source. Don't waste a re-run trying to get a "portrait" variant with `-rt 0`; the output is already correct and the device renders it correctly. The output page dimensions will be narrow+tall (e.g. ~257×348 pt for a 6" Kindle), matching the screen aspect ratio.
- **DO NOT use `vision_analyze` to verify rotation of k2pdfopt output.** Vision models systematically misjudge narrow+tall reflowed pages as "rotated 90°" because they pattern-match tall-rectangle = portrait-oriented-content. They will confidently produce wrong evaluations even when the output is correct. **Correct verification paths:**
  1. `mdls -name kMDItemPageHeight -name kMDItemPageWidth <pdf>` — narrow+tall (height > width, ~3:4 ratio matching the device) means correctly oriented for the device.
  2. `pdftoppm -r 100 -f N -l N <pdf> /tmp/preview -png` then ask vision a *specific* question: "scanning left-to-right across a row, do you see complete English sentences or single stacked letters?" If complete sentences → correct orientation. The generic "is this rotated?" question is unreliable.
  3. Trust the device. k2pdfopt outputs are reliably oriented; if metadata says narrow+tall, ship it.

- **k2pdfopt errors out "File or folder - could not be opened" partway through a long book.** Symptom: partial output PDF, "0 pages saved" for some source pages, exit code 0 but error message printed. Root cause: source PDF contains objects (font subsets, image dictionaries, etc.) MuPDF can't render cleanly. **Fix: pass the source through ghostscript first**, then feed the cleaned PDF to k2pdfopt:
  ```bash
  gs -o clean.pdf -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress \
     -dCompatibilityLevel=1.5 source.pdf
  k2pdfopt clean.pdf [...other flags...]
  ```
  `qpdf --linearize` alone is NOT always enough — gs's full re-render is. See `references/physical-zoom-limits.md` for the session where this happened on Raschka's *Build a LLM From Scratch*.

## See also

- `references/k2pdfopt-flags.md` — full flag reference and device codes
- `references/physical-zoom-limits.md` — **read this when user says "text is still too small after running k2pdfopt"** on a 6" device. Includes math, real measurements from the Raschka LLM book, and the durable answer (landscape, not flag tuning).
- `references/diagnostic-attribution.md` — distinguishing real causes (PDF→AZW3 conversion damage) from misattributed ones (blaming Kindle's PDF viewer)
- `nano-pdf` skill — for editing PDF text content, not for reformatting
- `ocr-and-documents` skill — for extracting text out of PDFs (different goal)
