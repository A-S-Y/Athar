# إرشادات وقواعد وكيل الذكاء الاصطناعي لمشروع «أثر» (Agent Instructions & Rules)

تحدد هذه الوثيقة القواعد والمعايير الإلزامية التي يجب على أي وكيل ذكاء اصطناعي (AI Agent) الالتزام بها بدقة عند التعامل مع هذا المستودع، وبخاصة عند إنشاء أو تعديل مستندات Microsoft Word (`.docx`).

---

## 1. قواعد تحرير مستندات Word باللغة العربية (قواعد إلزامية وغير قابلة للتجاوز)

عند تعديل أي مستند Word (`.docx`) عربي في هذا المشروع:

1. **الالتزام بثوابت Word COM الثنائية:**
   * في Microsoft Word COM/VBA:
     * `wdReadingOrderRtl = 0` (الاتجاه من اليمين لليسار)
     * `wdSectionDirectionRtl = 0`
     * `wdTableDirectionRtl = 0`
   * **تحذير حاسم:** لا تقم إطلاقاً بوضع القيمة `1`، لأن القيمة `1` تعني `wdReadingOrderLtr` (من اليسار لليمين) وتجبر Word على تفعيل زر LTR في واجهة المستخدم.

2. **منع الخطوط الحمراء المتعرجة للتدقيق الإملائي:**
   * يجب وسم المقاطع اللاتينية/الإنجليزية بـ `<w:noProof/>` و `<w:lang w:val="en-US"/>`.
   * يجب ضبط خصائص المستند في Word COM:
     `doc.ShowSpellingErrors = False` و `doc.ShowGrammaticalErrors = False`.

3. **حماية المصطلحات اللاتينية من التشتت وانكسار الأقواس:**
   * عند تضمين مصطلحات إنجليزية تقنية مثل `(API Contracts)` أو `(Modular Monolith)`:
     * استبدل المسافات بمسافات غير قابلة للكسر: `\u00A0`.
     * استبدل الشرطات بشرطات غير قابلة للكسر: `\u2011`.
   * اعتمد المحاذاة الطبيعية لليمين: `WD_ALIGN_PARAGRAPH.RIGHT` وتجنب الضبط الكامل `JUSTIFY` مع النصوص المختلطة لمنع الفجوات البيضاء القبيحة.

4. **الجداول والمقاطع (RTL Layout):**
   * يجب أن تتضمن جميع الجداول عنصر `<w:bidiVisual/>` داخل `tblPr`.
   * يجب أن تتضمن جميع المقاطع عنصر `<w:bidi/>` داخل `sectPr`.

---

## 2. المرجع البرمجي السريع المعتمد

راجع الملف التفصيلي: [DOCX_ARABIC_RTL_GUIDE.md](file:///c:/Users/Alsiynii/Desktop/Work/Athar/DOCX_ARABIC_RTL_GUIDE.md) أو استعن بالمهارة المسجلة: `word-arabic-rtl-mastery`.

الكود الأساسي الواجب استدعاؤه بعد أي تعديل:
```python
import os, re, win32com.client
import docx
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml import OxmlElement
from docx.oxml.ns import qn


def apply_athar_rtl_standard(docx_path):
  doc = docx.Document(docx_path)
  for p in doc.paragraphs:
    p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
    pPr = p._p.get_or_add_pPr()
    if pPr.find(qn('w:bidi')) is None:
      pPr.insert(0, OxmlElement('w:bidi'))
    for r in p.runs:
      rPr = r._r.get_or_add_rPr()
      text = r.text
      if bool(re.search(r'[A-Za-z]', text)) and not bool(
          re.search(r'[\u0600-\u06FF]', text)
      ):
        r.text = text.replace(' ', '\u00A0').replace('-', '\u2011')
        if rPr.find(qn('w:noProof')) is None:
          rPr.append(OxmlElement('w:noProof'))
        lang = rPr.find(qn('w:lang'))
        if lang is None:
          lang = OxmlElement('w:lang')
          rPr.append(lang)
        lang.set(qn('w:val'), 'en-US')
        rtl_el = rPr.find(qn('w:rtl'))
        if rtl_el is not None:
          rPr.remove(rtl_el)
      else:
        if rPr.find(qn('w:rtl')) is None:
          rPr.append(OxmlElement('w:rtl'))
        lang = rPr.find(qn('w:lang'))
        if lang is None:
          lang = OxmlElement('w:lang')
          rPr.append(lang)
        lang.set(qn('w:bidi'), 'ar-SA')

  for t in doc.tables:
    if t._tbl.tblPr.find(qn('w:bidiVisual')) is None:
      t._tbl.tblPr.append(OxmlElement('w:bidiVisual'))
    for row in t.rows:
      for cell in row.cells:
        for p in cell.paragraphs:
          p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
          if p._p.get_or_add_pPr().find(qn('w:bidi')) is None:
            p._p.get_or_add_pPr().insert(0, OxmlElement('w:bidi'))

  for sec in doc.sections:
    if sec._sectPr.find(qn('w:bidi')) is None:
      sec._sectPr.insert(0, OxmlElement('w:bidi'))

  doc.save(docx_path)

  # أتمتة Word COM
  word = win32com.client.Dispatch('Word.Application')
  word.Visible = False
  try:
    d = word.Documents.Open(os.path.abspath(docx_path))
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

## 3. بروتوكول التحقق الصارم قبل إعلان إنجاز المهمة (Quality Gate)

> [!CAUTION]
> **ممنوع منعاً باتاً للوكيل إعلان نجاح المهمة أو تسليم الملف للمستخدم دون إجراء التحقق البصري المباشر.**

الخطوات الإلزامية:
1. تصدير الملف كـ PDF عبر `ExportAsFixedFormat`.
2. تحويل الصفحات لصور عالية الدقة عبر `pypdfium2`.
3. استعراض الصور بواسطة أداة `view_file` والتأكد مما يلي:
   * زر `<-¶` نشط ومفعل.
   * خلو تام من الخطوط المتعرجة الحمراء.
   * الأقواس `(...)` معتدلة وغير معكوسة أو متناثرة.
   * مسافات الكلمات طبيعية ومستقرة.
   * الجداول مرتبة من اليمين لليسار.
