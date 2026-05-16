# k2pdfopt flag reference

Source: official help text (k2pdfopt v2.55, willus.com) + hands-on experience converting a 370-page ML book for an old Kindle Paperwhite 2. Trimmed to flags most useful for technical PDF → small-screen conversion.

## Device profiles (`-dev <name>`)

| Code | Device | Screen |
|------|--------|--------|
| `k2` | Kindle 2 | 600x800, 167 DPI |
| `k3` | Kindle Keyboard / Basic | 600x800, 167 DPI |
| `kp2` | Kindle Paperwhite 1/2 | 758x1024, 212 DPI |
| `kp3` | Kindle Paperwhite 3+ | 1072x1448, 300 DPI |
| `kpw5` | Kindle Paperwhite 5 (11th gen) | 1236x1648, 300 DPI |
| `kv` | Kindle Voyage | 1080x1430, 300 DPI |
| `koa` | Kindle Oasis | 1264x1680, 300 DPI |
| `ksc` | Kindle Scribe | 1860x2480, 300 DPI |
| `dx` | Kindle DX | 824x1200, 150 DPI |
| `kbg` | Kobo Glo | 758x1024, 212 DPI |
| `kba` | Kobo Aura | 758x1024, 212 DPI |
| `kbh2` | Kobo H2O | 1080x1430, 265 DPI |
| `nookst` | Nook Simple Touch | 600x800, 167 DPI |
| `ipad` | iPad | 1536x2048, 264 DPI |
| `pb` | Pocketbook | varies |

Custom: `-dev 1072x1448,300` for any width × height @ DPI. Or use explicit `-h <px> -w <px> -dpi <ppi>` — preferred when you've identified the exact device generation, because it avoids the trap of `-dev kp3` over-sizing for an older PW1/PW2 owner.

**If user says just "Kindle Paperwhite" without generation, ASK** (preferably with a photo). Don't default to `kp3` blindly — guessing wrong wastes a 10-minute conversion. Bezel/logo cues:
- PW1/PW2 (2012-2013): no front logo, flush plastic bezel, uneven front-light bleed, no buttons.
- PW3-PW4 (2015-2018): "kindle" wordmark on front bezel, still flush.
- PW5 / Signature (2021+): larger 6.8" screen, USB-C, flush glass front.

## Layout modes (`-mode <name>`) — the most important section

| Mode | What it ACTUALLY does | When to use |
|------|----------------------|-------------|
| `def` | Default. Moderate reflow — breaks some 2D layout. | Mixed content where you're OK losing some structure. |
| `copy` | **Crops white margins only. Renders at original size.** Does NOT slice oversized pages into device-sized sub-pages. | Source PDF is ALREADY close to device size (e.g. A6 paperback PDF → 6" Kindle). NOT useful for 7-9" technical PDFs on 6" screens. |
| `fitwidth` (alias `fw`) | **Slices each source page horizontally into multiple device-width sub-pages.** Each source page becomes 2-3 Kindle pages. Code/matrices/equation rows stay intact horizontally; you scroll forward through sub-pages instead of within a page. | **THE right mode for technical books (Raschka, PRML, CLRS, etc.) on 6-7" e-readers.** Page count roughly doubles, output 40-80MB for a 400-pg book. |
| `fp` | Full reflow into a single text column. Destroys tables, matrices, code blocks — same disaster as Kindle's native reflow. | Prose-only novels with no 2D structure. |
| `2col` | Forces 2-column academic-paper interpretation, then reflows each column. | arXiv-style two-column papers. |
| `crop` | Just trim margins, no slicing or reflow. | Pre-cleaning before another tool processes the output. |
| `concat` | Concatenate all source pages into one tall page. | Niche. Stitching multi-page tables for printing. |

⚠️ **The `-mode copy` trap:** the name suggests "copy to device" but it does NOT slice pages to fit. Many guides (including older versions of this skill) recommend `copy` for technical books — this is wrong when source pages exceed device size. Always check source vs device dimensions first; if source is bigger, use `fitwidth`.

## Output text mode

- `-n` — keep native PDF text. Output is selectable + searchable on Kindle.
- (default) — output is bitmap. Smaller file, no text selection.

Use `-n` for technical books — being able to look up terms matters. Skip it if file size is critical (drops ~30-50%).

## Cropping & margins

- `-m <inches>` — output margin around content (default 0.03).
- `-mb <i,t,r,b>` — per-side input-page margin to crop.
- `-ws <0.0-1.0>` — word-spacing threshold. Lower = preserve tighter spacing. **0.01 is good for code** (helps the segmenter treat code lines as atomic).
- `-ls` — landscape mode (rotate output 90°).
- `-bp` — break each input page at the end (no page-spanning).
- `-bpc` — break at column boundaries.
- `-mc-` — disable markup/crop guides in output (cleaner).
- `-as -` — disable auto-skew correction (native PDFs aren't skewed; only enable for scans). The trailing ` -` (space + dash) is k2pdfopt's syntax for "off".

## Image / OCR

- `-cmax <0.0-2.0>` — contrast boost. 1.0 = none. Raise to 1.3-1.5 for scanned/faint PDFs.
- `-gtc <0.0-1.0>` — gamma correction for source text.
- `-ocr <t|m>` — run OCR. `t` = tesseract, `m` = mupdf. Adds searchable text layer to scanned PDFs.
- `-ocrvis t` — visualize OCR output for debugging.

## Performance / batch

- `-x` — overwrite output without prompting.
- `-ui-` — disable interactive prompts (run unattended).
- `-as` — auto-straighten skewed scans (`-as -` turns off for native PDFs).
- `-rt <auto|0|90|180|270>` — rotate input.

## Recommended presets

### Old Kindle Paperwhite 1/2 (758×1024, 212ppi) + technical book

```bash
k2pdfopt "input.pdf" \
  -ui- -x \
  -mode fitwidth \
  -h 1024 -w 758 -dpi 212 \
  -as - -mc- \
  -ws 0.01 \
  -n \
  -o "output_PW2.pdf"
```

### Modern Paperwhite 3-5 / Voyage (1072×1448 or 1236×1648, 300ppi) + technical book

```bash
k2pdfopt "input.pdf" \
  -ui- -x \
  -mode fitwidth \
  -h 1448 -w 1072 -dpi 300 \
  -as - -mc- \
  -ws 0.01 \
  -n \
  -o "output_PW3.pdf"
```

### Faint / scanned PDF on any device (add OCR + contrast)

```bash
k2pdfopt "input.pdf" -ui- -x -mode fitwidth -dev kp3 \
  -cmax 1.4 -ocr t -n -o "output_ocr.pdf"
```

### Prose-heavy novel where reflow is OK

```bash
k2pdfopt "input.pdf" -ui- -x -dev kp3 -mode fp -n -o "output_novel.pdf"
```

## Troubleshooting

- **Output looks identical to input, just slightly smaller margins** — you used `-mode copy`. It does not split pages. Switch to `-mode fitwidth`.
- **Output sub-pages are still too big for the device** — your `-dev` preset is for a newer device than the user owns. Switch to explicit `-h/-w/-dpi` matching the actual device generation (e.g. 758×1024 @ 212 for PW1/PW2).
- **Output is huge (>100MB)** — drop `-n`, use bitmap. Or raise compression with `-jpegq 75`.
- **Code blocks still wrap badly with `fitwidth`** — source code lines may genuinely exceed device width even at fitwidth split points. Try landscape (`-ls`) or accept a smaller font (`-fs <pt>` to force font size).
- **Pages look cut off** — input PDF probably has unusual margins; try `-mb 0,0,0,0` to disable input crop.
- **Output text not selectable on Kindle** — you forgot `-n`. Re-run.
- **First-run "cannot be opened" Gatekeeper dialog (macOS)** — manually-downloaded binary has quarantine xattr. Run `xattr -d com.apple.quarantine /path/to/k2pdfopt` then retry.
- **Takes forever** — that's normal. 400-page technical book: 5-15 minutes on Apple Silicon. OCR adds 2-3x. `fitwidth` is slower than `copy` because it actually does layout analysis.

## Transferring to Kindle

- **Modern Kindle (PW3+, post-2015)**: Send-to-Kindle email or the Send-to-Kindle Mac app work fine. Drag onto the dropzone.
- **Old Kindle (PW1/PW2, pre-2014)**: Send-to-Kindle is unreliable for these vintages. **Use USB.** Plug into Mac → Kindle mounts as a disk → drag the PDF into `documents/` → eject. On-device, open the book and set view to "Actual Size" / "Original" (NOT Reflow), since k2pdfopt already did the layout work.
