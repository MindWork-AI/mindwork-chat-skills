
# Documents and conversions

Read this when the deliverable is a file opened in an office program or a PDF viewer. Sessions, `files_changed`, input files and publishing are covered in `SKILL.md`; tool parameters in [tools.md](tools.md).

The running example is an offer for a customer, ACME GmbH: data becomes a Word document and a PDF,
with a spreadsheet of the numbers beside it. Every file ends the same way: render it to PNG, look at the
PNGs with `sandbox_read_file`, fix, and only then publish.

## Which tool makes which file

| You need | Use | Language |
|----------|-----|----------|
| PowerPoint deck | `pptxgenjs` (new), `python-pptx` (edit an existing deck) | javascript / python |
| Word document | `docx` (new), `python-docx` (edit), `pandoc` (from Markdown), `docxtpl` (fill a template) | javascript / python / bash |
| Excel workbook | `openpyxl` (formulas, formats), `pandas.to_excel`, `exceljs` | python / javascript |
| PDF from scratch | `reportlab` (drawn), `weasyprint` (HTML/CSS) | python |
| PDF from HTML with JavaScript | Playwright Chromium `page.pdf()` | python / javascript |
| Any Office file to PDF | `soffice --headless --convert-to pdf` | bash |
| PDF to images or text | `pdftoppm -png -r 60`, `pdftotext` | bash |
| Merge, split, rotate PDFs | `qpdf`, `pypdf` | bash / python |
| Read a document as text | `markitdown file.docx` | bash |

## Word: the offer

```javascript
import fs from "node:fs";
import { Document, Packer, Paragraph, HeadingLevel, Table, TableRow, TableCell, TextRun } from "docx";
const doc = new Document({ sections: [{ children: [
  new Paragraph({ text: "Offer for ACME GmbH", heading: HeadingLevel.HEADING_1 }),
  new Paragraph({ children: [new TextRun("Total: "), new TextRun({ text: "EUR 12,340", bold: true })] }),
  new Table({ rows: [["Item", "EUR"], ["Consulting", "12,340"]].map(r =>
    new TableRow({ children: r.map(t => new TableCell({ children: [new Paragraph(t)] })) })) }),
] }] });
fs.writeFileSync("offer.docx", await Packer.toBuffer(doc));
```

Text-heavy documents are quicker as Markdown: `pandoc notes.md -o notes.docx`. To fill a template the user
supplied, pass it as an input file and use `docxtpl`.

## PDF from Office files

Convert several files in one command (one LibreOffice instance per session), then render pages:

```bash
set -eo pipefail
soffice --headless --convert-to pdf offer.docx deck.pptx >/dev/null
pdftoppm -png -r 60 offer.pdf offer      # offer-1.png, offer-2.png, ...
```

`Warning: failed to launch javaldx` is harmless if the PDF exists. Fonts match Calibri, Arial and Times
metrics, so line breaks match Office.

For HTML sources: **WeasyPrint** for static HTML/CSS such as invoices (`python -m weasyprint page.html page.pdf`,
supports `@page` and page numbers); **Chromium** when the page runs JavaScript:

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.launch(chromium_sandbox=False)
    pg = b.new_page(); pg.goto("file:///mnt/data/page.html")
    pg.pdf(path="page.pdf", format="A4", print_background=True)
    b.close()
```

## Excel: the numbers

Write data with pandas, then use openpyxl for formulas, widths and number formats. Prefer formulas to
values computed in Python, so the user can change an input and watch totals update.

```python
import pandas as pd
from openpyxl import load_workbook
pd.read_csv("quarterly.csv").to_excel("quarterly.xlsx", index=False, sheet_name="Data")
wb = load_workbook("quarterly.xlsx"); ws = wb["Data"]; n = ws.max_row
ws[f"A{n+1}"] = "Total"; ws[f"B{n+1}"] = f"=SUM(B2:B{n})"
ws.column_dimensions["A"].width = 12
wb.save("quarterly.xlsx")
```

openpyxl does not calculate formulas. Check a value on a **copy** that LibreOffice recalculates, and publish the original:

```bash
mkdir -p recalc && soffice --headless --convert-to xlsx --outdir recalc quarterly.xlsx >/dev/null
python -c "from openpyxl import load_workbook; ws=load_workbook('recalc/quarterly.xlsx',data_only=True).active; print(ws.cell(ws.max_row,2).value)"
```

## PowerPoint

Build new decks with `pptxgenjs` (native, editable charts via `addChart`). Keep one idea per slide, large
type, consistent margins. Edit an existing deck with `python-pptx`: load, change text runs rather than
replacing shapes (keeps formatting), save under a new name. Convert to PDF and render thumbnails to check
every slide.

## PDF forms, merging, splitting, scans

```python
from pypdf import PdfReader, PdfWriter
print(PdfReader("form.pdf").get_fields().keys())          # list field names first
w = PdfWriter(); w.append(PdfReader("form.pdf"))
w.update_page_form_field_values(w.pages[0], {"customer": "ACME GmbH"}, auto_regenerate=False)
w.set_need_appearances_writer(True); w.write("filled.pdf")
```

```bash
qpdf --empty --pages letter.pdf filled.pdf -- offer-pack.pdf   # merge
qpdf offer-pack.pdf --pages . 2-3 -- excerpt.pdf               # pages 2-3
```

Scanned PDFs have no text layer: `pdftoppm -png -r 300` then `tesseract page-1.png out` (English installed).

## Using another skill's helper scripts

Scripts from other skills (`thumbnail.py`, form fillers) are not in the sandbox but run unchanged once
uploaded. One script: `input_files=[{"name": "scripts/thumbnail.py", "content_text": "<file content>"}]`.
A folder: upload one `.tar.gz` as `content_base64` and unpack it with `tar xzf`. Unpacked files keep old
timestamps and may be missing from `files_changed`. Write outputs under `/mnt/data`.

## Before you publish

Render every page or slide and look at the PNGs: text overflowing boxes, overlapping shapes, empty
placeholders, wrong fonts, charts without data, wrong numbers. Fix and re-render until clean. Publish only
the finals, for example `paths=["offer.pdf", "offer.docx", "quarterly.xlsx"]`, and tell the user what each is.
