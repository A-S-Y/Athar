---
name: word-arabic-rtl-mastery
description: >-
  Expert instructions and production automation scripts for editing, generating,
  and formatting Arabic Microsoft Word (.docx) documents. Ensures strict Right-to-Left
  (RTL) reading order in Word UI, prevents red spellcheck squiggly underlines on English
  terms, eliminates scattered words/inverted parentheses, and guarantees correct table and section bidirectional layout.
---

# Word Arabic RTL Mastery Skill

This skill provides the definitive engineering methodology to create and modify Arabic Microsoft Word (`.docx`) documents on Windows without encountering RTL layout inversion, distorted word spacing, broken parenthetical terms, or spellcheck error squiggles.

---

## 1. Core Architecture & Root Causes

When modifying Arabic `.docx` documents programmatically with `python-docx` or Word COM automation, three critical issues often occur:

### A. The Word COM Enum Trap (`wdReadingOrderRtl = 0`)
* In Microsoft Word's COM/VBA object model:
  * `wdReadingOrderRtl = 0` (Right-to-Left)
  * `wdReadingOrderLtr = 1` (Left-to-Right)
  * `wdSectionDirectionRtl = 0`
  * `wdTableDirectionRtl = 0`
* **Common Bug:** Assuming `1` means RTL (boolean True). Passing `1` explicitly forces Word to activate the **Left-to-Right** button (`¶->`) on the ribbon!
* **Rule:** Always set `ReadingOrder = 0`, `SectionDirection = 0`, and `TableDirection = 0`.

### B. Spellcheck Red Squiggly Lines on English Terms
* Word applies the Arabic dictionary (`ar-SA`) to Latin text in Arabic paragraphs by default, flagging all technical terms with red squiggles.
* **Solution:**
  1. Split text into pure Arabic runs and pure English runs.
  2. On Latin runs, inject `<w:noProof/>` and `<w:lang w:val="en-US"/>`.
  3. Set `doc.ShowSpellingErrors = False` and `doc.ShowGrammaticalErrors = False`.

### C. Scattered Typography, Inverted Parentheses, and Margin Breaks
* In mixed Arabic/English text, if an English parenthetical term like `(API Contracts)` or `(Client-side Form Validation)` breaks across the left margin:
  * Word splits the term across two lines, resulting in orphaned brackets and visually reversed chunks.
  * Setting full justification (`WD_ALIGN_PARAGRAPH.JUSTIFY`) balloons inter-word spacing drastically.
* **Solution:**
  1. Use natural right alignment: `WD_ALIGN_PARAGRAPH.RIGHT`.
  2. Inside English parenthetical runs, replace regular spaces with non-breaking spaces (`\u00A0`) and regular hyphens with non-breaking hyphens (`\u2011`).
  3. This binds the technical term into an atomic token that wraps together without splitting or inverting brackets.

---

## 2. Standard Production Automation Template

Use this comprehensive Python routine whenever editing or finalizing Arabic documents:

```python
import os
import re
import docx
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml import OxmlElement
from docx.oxml.ns import qn
import win32com.client


def enforce_arabic_rtl_and_mixed_script(docx_path):
  doc = docx.Document(docx_path)

  def process_paragraph(p):
    p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
    pPr = p._p.get_or_add_pPr()

    # 1. OpenXML paragraph RTL
    if pPr.find(qn('w:bidi')) is None:
      pPr.insert(0, OxmlElement('w:bidi'))

    # 2. Process runs
    for r in p.runs:
      rPr = r._r.get_or_add_rPr()
      text = r.text
      has_en = bool(re.search(r'[A-Za-z]', text))
      has_ar = bool(re.search(r'[\u0600-\u06FF]', text))

      if has_en and not has_ar:
        # Pure English/Technical run:
        # Atomic tokens: prevent line breaks inside English terms
        r.text = text.replace(' ', '\u00A0').replace('-', '\u2011')

        # Disable spellcheck squiggles
        if rPr.find(qn('w:noProof')) is None:
          rPr.append(OxmlElement('w:noProof'))

        # Set English language
        lang = rPr.find(qn('w:lang'))
        if lang is None:
          lang = OxmlElement('w:lang')
          rPr.append(lang)
        lang.set(qn('w:val'), 'en-US')

        # Remove w:rtl if present so Latin brackets don't mirror improperly
        rtl_el = rPr.find(qn('w:rtl'))
        if rtl_el is not None:
          rPr.remove(rtl_el)
      else:
        # Pure Arabic run:
        if rPr.find(qn('w:rtl')) is None:
          rPr.append(OxmlElement('w:rtl'))

        lang = rPr.find(qn('w:lang'))
        if lang is None:
          lang = OxmlElement('w:lang')
          rPr.append(lang)
        lang.set(qn('w:bidi'), 'ar-SA')

  # Process all body paragraphs
  for p in doc.paragraphs:
    process_paragraph(p)

  # Process all tables and their cells
  for table in doc.tables:
    tblPr = table._tbl.tblPr
    if tblPr.find(qn('w:bidiVisual')) is None:
      tblPr.append(OxmlElement('w:bidiVisual'))
    for row in table.rows:
      for cell in row.cells:
        for p in cell.paragraphs:
          process_paragraph(p)

  # Process all sections
  for sec in doc.sections:
    secPr = sec._sectPr
    if secPr.find(qn('w:bidi')) is None:
      secPr.insert(0, OxmlElement('w:bidi'))

  doc.save(docx_path)

  # 3. Apply Word COM properties (0 = RTL)
  abs_path = os.path.abspath(docx_path)
  word = win32com.client.Dispatch('Word.Application')
  word.Visible = False
  try:
    d = word.Documents.Open(abs_path)
    d.ShowSpellingErrors = False
    d.ShowGrammaticalErrors = False

    for i in range(1, d.Sections.Count + 1):
      d.Sections(i).PageSetup.SectionDirection = 0  # wdSectionDirectionRtl

    for i in range(1, d.Paragraphs.Count + 1):
      d.Paragraphs(i).ReadingOrder = 0  # wdReadingOrderRtl

    for i in range(1, d.Tables.Count + 1):
      d.Tables(i).TableDirection = 0  # wdTableDirectionRtl

    d.Save()
    d.Close()
  finally:
    word.Quit()
```

---

## 3. Verification Protocol

Before declaring completion:
1. Export document to PDF via Word COM: `doc.ExportAsFixedFormat(pdf_path, 17)`.
2. Render pages to high-resolution PNG using `pypdfium2`.
3. Visually verify with `view_file`:
   - [ ] RTL button active (`<-¶`) and right margins respected.
   - [ ] No red spellcheck squiggly underlines under English terms.
   - [ ] Parentheses `(...)` correctly positioned and un-inverted.
   - [ ] English parenthetical phrases completely unbroken across lines.
   - [ ] Tables visually right-to-left (first column on the right).
