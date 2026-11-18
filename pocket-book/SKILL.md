---
name: "pocket-book"
description: "Turn any set of pages (a PDF, document, notes, recipes) into a printable 4.25x5.5 pocket book: half-sheet cut, saddle-stitched, long-edge duplex on letter or A4."
---

# Pocket book imposition

Use when the user wants pages printed as a small booklet they cut, stack, fold and staple — "make this a pocket book", "print this as a mini booklet", "half-sheet booklet", or a cover for one.

## The format

- US Letter portrait (or A4), printed **double-sided, flip on long edge, 100% / Actual size**, black and white unless asked otherwise.
- Each sheet is cut across the middle (perpendicular to the long edge) into two half-sheets. Each half folds in the middle, so pages are **4.25 x 5.5 in** (letter) and **one sheet = 8 pages**.
- The **bottom half is printed rotated 180°**, so every page's top sits on an uncut sheet edge.
- Collation: top halves form stack A, bottom halves stack B. Page order is chosen so stack A (sheet 1 on top) goes straight onto stack B (bottom halves turned upright, sheet 1 on top), and the combined stack folds into one saddle-stitched book (fold the left half under, staple the fold).
- Tiny gray A1/B1… markers print beside the cut line on the front of each sheet so the user can check stack order.
- Page count is padded to a multiple of 8 with blank pages (at the end, or before the last page with `--blanks before-last` when the last page is a back page).
- Cover: a separate cardstock sheet gives two covers (one per half). Side 1 = outside (back cover left of the fold, front cover right); an optional side 2 = inside covers.

## Steps

1. **Get the pages as a PDF, one page per book page.** If the user gave a PDF, use it (any page size is scaled to fit the 4.25x5.5 slot, aspect kept, centred). If they gave text, notes, markdown or a document, lay it out at 4.25x5.5 in (306x396 pt) yourself first — e.g. reportlab with ~0.3 in margins and 6.5–8 pt type, or HTML with `@page { size: 4.25in 5.5in; margin: 0.3in }` rendered by a headless browser or weasyprint. Design for a small page: short lines, generous headings, page numbers in the outer bottom corner (even pages are left of the fold, odd pages right). Keep it grayscale if the print is black and white.
2. **Impose**: write the script below to `impose_pocketbook.py` (from your scripts directory, not the input's) and run
   `python3 impose_pocketbook.py pages.pdf book.pdf` (add `--sheet a4`, `--blanks before-last`, `--margin 6`, or `--no-marks` as needed). It prints the page and sheet count. Requires `pypdf` and `reportlab` (`pip install pypdf reportlab --break-system-packages` if missing).
3. **Cover** (if wanted): make a 1–4 page PDF at 4.25x5.5 — front, back, [inside front, inside back] — and run `python3 impose_pocketbook.py --cover cover_pages.pdf covers.pdf`. One cardstock sheet covers two books.
4. **Check**: render the imposed PDF with `pdftoppm -r 50 -png` and look at sheet 1 front and back: front top half should read [last page | page 1]; the bottom half is upside down and, turned around, reads [P−2M | 2M+1] (M = sheets). Every page n and n+1 must be back to back.
5. **Deliver** the PDF(s) with these printing instructions:
   "Print at 100% (Actual size), double-sided, flip on long edge — N sheets. Cut each sheet across the middle at 5.5 in. Turn the bottom (B) halves around so they read upright, then stack the A halves on the B halves (A1/B1 on top, page 1 face up on the right), fold the left half under and staple the fold." Add the cover line if a cover was made: "Cover: print on cardstock, one side (two sides if it has inside covers, also flip on long edge); cut in half; one sheet covers two books."

## impose_pocketbook.py

```python
#!/usr/bin/env python3
"""Impose a PDF of pages into a printable pocket book (saddle-stitched, half-sheet cut).

    python3 impose_pocketbook.py pages.pdf book.pdf [--sheet letter|a4] [--blanks end|before-last]
                                [--margin PT] [--no-marks]
    python3 impose_pocketbook.py --cover cover.pdf covers.pdf [--sheet letter|a4]

Physical format (letter): portrait 8.5x11 sheet, printed double-sided, flip on LONG edge, at 100%.
Each sheet is cut across the middle into two 8.5x5.5 halves; each half folds in the middle into
4.25x5.5 pages, so one sheet = 8 book pages. The bottom half is printed rotated 180 degrees so every
page's top sits on an uncut sheet edge. After cutting, turn the B halves upright, stack A (sheet 1
on top) on B (sheet 1 on top), fold the left half under and staple the fold.

Input pages of any size are scaled to fit the page slot (aspect kept, centred). Page count is padded
to a multiple of 8 with blank pages (at the end, or just before the last page so a back page stays last).

--cover takes 1-4 pages: front, back, [inside front, inside back]. It prints two identical covers per
sheet (one per book) on side 1 (front/back), plus side 2 (inside covers) only if pages 3-4 are given.
"""
import argparse, copy
from pypdf import PdfReader, PdfWriter, PageObject, Transformation

SHEETS = {"letter": (612.0, 792.0), "a4": (595.28, 841.89)}

def place(dst, src, x, y, w, h, rotate, margin):
    """Draw src page into slot (x, y, w, h) of dst; rotate=True turns it 180 degrees in the slot."""
    src = copy.copy(src)
    src.transfer_rotation_to_content()
    box = src.mediabox
    sw, sh = float(box.width), float(box.height)
    s = min((w - 2 * margin) / sw, (h - 2 * margin) / sh)
    ox = x + (w - sw * s) / 2 - float(box.left) * s
    oy = y + (h - sh * s) / 2 - float(box.bottom) * s
    t = Transformation().scale(s, s).translate(ox, oy)
    if rotate:  # rotate 180 about the slot centre
        cx, cy = x + w / 2, y + h / 2
        t = t.translate(-cx, -cy).rotate(180).translate(cx, cy)
    dst.merge_transformed_page(src, t)

def imposition(P):
    """[(front_slots, back_slots)] per sheet; slot = (page, 'L'|'R', 'top'|'bot') in reading order."""
    M = P // 8; out = []
    for i in range(M):
        front, back = [], []
        for k, half in ((i, "top"), (M + i, "bot")):  # top halves -> stack A, bottom -> stack B
            front += [(P - 2 * k, "L", half), (2 * k + 1, "R", half)]
            back += [(2 * k + 2, "L", half), (P - 2 * k - 1, "R", half)]
        out.append((front, back))
    return out

def slot_xy(side, half, W, H):
    """Physical position of a reading-order slot. Bottom halves are rotated, so L/R swap there."""
    pw, ph = W / 2, H / 2
    if half == "top":
        return (0.0 if side == "L" else pw), ph, False
    return (pw if side == "L" else 0.0), 0.0, True

def marks(writer_page, W, H, sheet_no):
    """Tiny A#/B# markers beside the cut line (front side only)."""
    from io import BytesIO
    from reportlab.pdfgen import canvas
    buf = BytesIO(); c = canvas.Canvas(buf, pagesize=(W, H))
    c.setFillGray(0.62); c.setFont("Helvetica", 4.5)
    c.drawCentredString(W / 2, H / 2 + 4, f"A{sheet_no}")
    c.saveState(); c.translate(W / 2, H / 2 - 8); c.rotate(180)
    c.drawCentredString(0, -4.5, f"B{sheet_no}"); c.restoreState()
    c.showPage(); c.save(); buf.seek(0)
    writer_page.merge_page(PdfReader(buf).pages[0])

def impose(src_path, out_path, sheet="letter", blanks="end", margin=0.0, with_marks=True):
    W, H = SHEETS[sheet]
    pages = list(PdfReader(src_path).pages)
    if not pages: raise SystemExit("input has no pages")
    pad = (-len(pages)) % 8
    blank = None  # None = leave the slot empty
    if blanks == "before-last" and len(pages) > 1:
        pages = pages[:-1] + [blank] * pad + pages[-1:]
    else:
        pages = pages + [blank] * pad
    P = len(pages)
    w = PdfWriter()
    for n, (front, back) in enumerate(imposition(P), 1):
        for side_i, slots in enumerate((front, back)):
            dst = PageObject.create_blank_page(width=W, height=H)
            for page_no, side, half in slots:
                src = pages[page_no - 1]
                if src is None: continue
                x, y, rot = slot_xy(side, half, W, H)
                place(dst, src, x, y, W / 2, H / 2, rot, margin)
            if with_marks and side_i == 0: marks(dst, W, H, n)
            w.add_page(dst)
    with open(out_path, "wb") as f: w.write(f)
    return P, P // 8, pad

def impose_cover(src_path, out_path, sheet="letter", margin=0.0):
    W, H = SHEETS[sheet]
    pages = list(PdfReader(src_path).pages)
    front, back = pages[0], (pages[1] if len(pages) > 1 else None)
    sides = [(back, front)]  # outside: back cover left of the fold, front cover right
    if len(pages) >= 3:      # inside (long-edge flip): inside front sits behind the front cover
        sides.append((pages[2], pages[3] if len(pages) > 3 else None))
    w = PdfWriter()
    for left, right in sides:
        dst = PageObject.create_blank_page(width=W, height=H)
        for half in ("top", "bot"):
            for side, src in (("L", left), ("R", right)):
                if src is None: continue
                x, y, rot = slot_xy(side, half, W, H)
                place(dst, src, x, y, W / 2, H / 2, rot, margin)
        w.add_page(dst)
    with open(out_path, "wb") as f: w.write(f)
    return len(sides)

if __name__ == "__main__":
    ap = argparse.ArgumentParser(description=__doc__, formatter_class=argparse.RawDescriptionHelpFormatter)
    ap.add_argument("src"); ap.add_argument("out")
    ap.add_argument("--cover", action="store_true", help="impose a cover sheet (2 covers per sheet)")
    ap.add_argument("--sheet", default="letter", choices=SHEETS)
    ap.add_argument("--blanks", default="end", choices=["end", "before-last"])
    ap.add_argument("--margin", type=float, default=0.0, help="extra inset (pt) when scaling pages into a slot")
    ap.add_argument("--no-marks", action="store_true")
    a = ap.parse_args()
    if a.cover:
        n = impose_cover(a.src, a.out, a.sheet, a.margin)
        print(f"cover: {n} printed side(s); one sheet covers two books")
    else:
        P, M, pad = impose(a.src, a.out, a.sheet, a.blanks, a.margin, not a.no_marks)
        print(f"{P} pages ({pad} blank added) on {M} sheet(s)")
```

## Notes

- Inside covers: with long-edge duplex, the inside front cover is printed behind the front cover (right half on screen when viewing side 2), which the script handles.
- If a printer shifts pages on duplex, suggest `--margin 6` to inset content slightly.
- An inside page of the input that's already landscape will be scaled down to fit the portrait slot; rotate such pages to portrait first if they should fill the page.