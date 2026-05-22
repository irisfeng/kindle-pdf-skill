# Physical zoom limits on small Kindles

A note from a real session where the user said "切分后字还是小了，看的太辛苦" ("after slicing the text is still small, eyes hurt") and asked to boost text size another 20-30%.

We failed at first. Then we understood why. This file is so the next session doesn't repeat the same diagnosis from scratch.

## The book and the device

- **Book**: Sebastian Raschka, *Build a Large Language Model From Scratch*. Source PDF: 531 × 666 pt (7.38 × 9.25 in), 370 pages, ~85-90 characters per line.
- **Device**: Kindle Paperwhite 2 — 758 × 1024 px, 6", 212 DPI. Device page in PDF points: 257 × 348 pt (3.58 × 4.83 in).
- **Previous output (portrait, `-ls- -mode fitwidth`)**: 392 pages, one-source-page-per-device-page, vision reports ~65-70 chars/line on device, code blocks ~75-80% of body font size. User: "too small."

## The user's request

> "切分后字还是小了，看的太辛苦，眼睛都要坏了。请评估是否可以优化改进，放大整体效果 20-30%。"

Translation: text is still too small after slicing, eyes hurt, please make it 20-30% bigger.

## What we tried and what actually happened

### Attempt 1: `-fs 1.25` in portrait mode

Command:

```bash
k2pdfopt input.pdf -ui- -x -mode fitwidth \
  -h 1024 -w 758 -dpi 212 \
  -ls- -fs 1.25 -as - -mc- -ws 0.01 -o out.pdf
```

First run errored ("File or folder - could not be opened" after partial output). Symptom: original PDF has objects k2pdfopt's MuPDF can't render. Fixed by passing the PDF through ghostscript first:

```bash
gs -o clean.pdf -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress \
   -dCompatibilityLevel=1.5 input.pdf
```

(qpdf --linearize alone wasn't enough — gs full re-render was.)

Re-ran k2pdfopt on `clean.pdf`. It now ran but vision reported the output looked **essentially the same as before** — text wasn't noticeably bigger.

### Attempt 2: pymupdf custom slicer

Wrote a Python script using pymupdf `show_pdf_page(target, src, pno, clip=clip)` to slice and place. Tried multiple `extra_zoom` values (1.3, 2.5, 2.7, 3.0). Two failure modes:

- Small `extra_zoom` (1.3): the script computed `total scale vs original = 0.58×`. Text was still small.
- Large `extra_zoom` (2.7-3.0): the script reported `total scale = 1.26x-1.40x` of original, BUT vision still reported "90 chars per line" and "content occupies only 10-15% of screen."

This is when the penny dropped.

## The math (this is the durable insight)

For a 6" Kindle Paperwhite 2 reading the Raschka book:

```
Source page width:  531 pt  ≈ 7.38 in   ≈ 90 chars per line in source
Device page width:  257 pt  ≈ 3.58 in   ≈ 758 px @ 212 dpi
                              (with 2% margin: 247 pt inner area)

Horizontal zoom to fit source width into device width:
    h_zoom = 247 / 531 = 0.465
    (i.e. text is rendered at 46.5% of its source size)

Source char width (avg): 531/90 ≈ 5.9 pt
Device char width: 5.9 × 0.465 = 2.74 pt
                                 ≈ 8.1 px on the 212-DPI screen
```

8.1 px per character on a 212 DPI screen is at the lower edge of comfortable e-ink reading.

To get +25% bigger characters in **portrait** mode (`-ls-` keeps one source page = one device page), we'd need:

```
required width per device row = 247 / (90 chars × 1.25) = 2.2 pt per char
required source-side char width = 2.2 / 0.465 = 4.7 pt per char
```

But the source PDF has 5.9 pt/char. Nothing in k2pdfopt's flags can change that — it's a property of the source. To get bigger characters in portrait you'd need to **show fewer source characters per device row**, which means **horizontal slicing** — and that breaks code lines, table rows, and equations.

**Result: `-fs 1.25` in portrait mode at `-mode fitwidth` cannot deliver +25% bigger text on this device + this source. It can deliver maybe +10-15% by tightening line spacing and shaving margins, then it saturates.** Adding `-fs 2.0` doesn't help either — it just causes k2pdfopt to choose more aggressive splits the user doesn't want.

## What actually works

### Option A: Drop `-ls-` (landscape output, hold Kindle sideways)

Without `-ls-`, k2pdfopt rotates content sideways and produces ~2× the page count (each source page → 2 landscape sub-pages). The device shows a 1024 × 758 view instead of 758 × 1024 — **width is now 1024 px, which lets each row of source content render at 1024/758 ≈ 1.35× more horizontal pixels**. Text is ~35% bigger immediately. Cost: user must hold Kindle sideways.

For ML/math/CS textbooks on PW1-PW4, this is the right answer 90% of the time. Make it the default recommendation.

### Option B: Accept horizontal slicing (one source line → multiple device lines)

In portrait, allow k2pdfopt to break source lines across device pages. Long code lines and table rows will split — user has to flip back and forth to read one line. Painful for technical content. Avoid.

### Option C: Upgrade the device

6.8" Paperwhite 5 / Signature has 1236 px width. The same Raschka book in portrait fitwidth shows text ~63% bigger than on PW2 at the same physical comfort level. 7" Oasis is similar. 10" Scribe is night and day.

When the user is squinting at a 6" PW2 reading a technical book and refusing landscape, this is the honest answer: the screen is the bottleneck, not the converter.

## Don't repeat these mistakes

1. **Don't promise the user "+25% bigger text"** when they're on a 6" device reading a 7-8" wide technical PDF in portrait. Tell them up front: portrait caps at ~+15%, landscape gets +35%, bigger screen is the only way past that.
2. **Don't trust vision output for character-count estimates** of small device-page renderings. Vision systematically over-counts characters per line because it pattern-matches against the source PDF's original line wrapping. Better signal: `pdfinfo output.pdf | grep "Page size"`, then compute math directly.
3. **Don't try to outsmart `-mode fitwidth -ls-` with custom pymupdf scripts** unless you've internalized that `show_pdf_page` preserves the source PDF's typography — it scales the visual block, not the per-glyph rendering. Source content with N chars per line will produce a device page with N chars per line, just at a different physical size.
4. **If k2pdfopt errors out with "File or folder - could not be opened" partway through a long book**, the source PDF has objects MuPDF can't render. Fix by running through `gs -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress`. qpdf alone is not always enough.

## Decision script for the next session

When a user says "字还是太小" / "text is still too small" after a portrait `-mode fitwidth` run on a 6" Kindle:

1. Check device generation (ask if unclear). PW1-PW4 = 6". PW5+ = 6.8".
2. Check source PDF width via `pdfinfo` or `mdls`. If source width > 1.5× device width AND it's a technical book → **portrait is at its limit**.
3. Offer in this order:
   - "Landscape mode (hold Kindle sideways) gets you ~+35% text immediately. Want that?" — this is the default recommendation for the situation described above.
   - "Otherwise, we can try `-fs 1.15` in portrait — buys you ~10%, no posture change." — soft offer.
   - "Beyond that the screen is the bottleneck — a 6.8"+ device is the durable fix." — honest, optional.
4. **Do NOT promise +20-30% in portrait. Do NOT write a custom slicer trying to beat the physics. Do NOT pretend a flag combo will do it.**
