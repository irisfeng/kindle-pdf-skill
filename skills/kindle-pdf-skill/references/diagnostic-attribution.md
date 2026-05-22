## Pitfall: misattributing the "shredded table" symptom

**Observed symptom**: User sends a Kindle photo showing a 2D table broken into single-column vertical fragments (e.g. `3 / the first token ID / over / 5 / dog / 1 / 1.2753` each on its own line).

### Wrong reflex (what we did initially)

Assume this is Kindle's native PDF reflow mode misbehaving. Write README/skill copy that says "Kindle's reflow mode destroys 2D structure".

### Reality (user-corrected in a 2026-05 session)

Kindle's native PDF viewer is **fixed-layout**. It renders the original PDF page as-is, just at small size. It does NOT auto-reflow into single columns. The shredded-table screenshot is **always** produced by a PDF→reflowable-format conversion step the user ran BEFORE copying the file to Kindle:

- `ebook-convert input.pdf output.azw3` (Calibre CLI)
- Calibre GUI "Convert books" → AZW3/MOBI/EPUB
- Send-to-Kindle email (Amazon auto-converts PDFs to AZW/KFX server-side, especially on older firmware)
- Any other tool that produces `.mobi` / `.azw3` / `.epub` from a PDF source

### Correct diagnostic question to ask early

> "Is the file on your Kindle the original `.pdf`, or did you convert it to `.azw3` / `.mobi` / `.epub` first?"

If they're not sure, ask them to mount the Kindle via USB and check the file extension in `documents/`:

- `.pdf` → original, the problem is just that it's typeset too wide for the screen. Fix: k2pdfopt-process the original PDF, copy the result back.
- `.azw3` / `.mobi` / `.kfx` → they converted. Tell them to delete the converted file, k2pdfopt-process the original PDF, and copy the k2pdfopt PDF to Kindle.

### Why this matters for user trust

If you tell the user "Kindle's reflow mode is the problem":

- They conclude their Kindle is broken or too old
- They may consider buying a new device
- The fix you give them works, but they walk away with the wrong mental model

The portable, generation-independent rule is: **don't run technical PDFs through reflowable-format converters**. Get this framing right in the explanation, not just in the commands.

### Fix is the same either way

Run k2pdfopt on the original PDF, copy the resulting **PDF** (not AZW3, not MOBI) to Kindle via USB. See main SKILL.md for the parameter recipes.
