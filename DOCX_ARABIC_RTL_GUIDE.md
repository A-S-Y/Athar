# دليل التنسيق البرمجي المتقدم لمستندات Word باللغة العربية (RTL & Mixed-Script)

يوثق هذا الدليل القواعد الهندسية والمعايير البرمجية المتبعة في مشروع «أثر» لإنشاء وتعديل مستندات Microsoft Word (`.docx`) باللغة العربية دون الوقوع في مشاكل اتجاه النص، تشتت الكلمات، انعكاس الأقواس، أو خطوط التدقيق الإملائي الحمراء.

---

## 1. التحديات التقنية وأسبابها الجذرية (Root Cause Analysis)

عند تحرير ملفات Word برمجياً بواسطة مكتبة `python-docx` أو أتمتة COM (`win32com.client`)، تظهر 3 إشكاليات رئيسية:

### أ) فخ قيم الثوابت في Microsoft Word COM (`wdReadingOrderRtl = 0`)
* **السبب:** في بيئة COM/VBA لبرنامج Word، قيمة الثابت `wdReadingOrderRtl` هي **`0`**، بينما `wdReadingOrderLtr` هي **`1`**.
* **الخطأ الشائع:** افتراض أن `1` تمثل القيمة المنطقية `True` للاتجاه اليميني؛ حيث يؤدي تعيين `ReadingOrder = 1` إلى إجبار Word على تفعيل زر **الاتجاه من اليسار لليمين (`¶->`)** في شريط الأدوات الرئيسي (Ribbon).
* **القاعدة الثابتة:**
  ```python
  Paragraph.ReadingOrder = 0  # wdReadingOrderRtl
  Section.PageSetup.SectionDirection = 0  # wdSectionDirectionRtl
  Table.TableDirection = 0  # wdTableDirectionRtl
```

### ب) الخطوط المتعرجة الحمراء (Spellcheck Errors) تحت المصطلحات الإنجليزية
* **السبب:** يطبّق Word قاموس التدقيق الإملائي العربي (`ar-SA`) على كامل الفقرة بما فيها الكلمات اللاتينية، مما يعتبر جميع المصطلحات التقنية (مثل `FastAPI` أو `PostgreSQL`) أخطاء إملائية.
* **الحل:**
  1. فصل النصوص إلى مقاطع عربية خالصة ومقاطع إنجليزية خالصة.
  2. وسم المقاطع الإنجليزية بـ `<w:noProof/>` و `<w:lang w:val="en-US"/>`.
  3. تعطيل إظهار أخطاء التدقيق على مستوى المستند: `doc.ShowSpellingErrors = False`.

### ج) تشتت الكلمات وانكسار الأقواس عند حدود السطور (Line-Breaking & BiDi Inversion)
* **السبب:** عند وجود مصطلح إنجليزي بين قوسين مثل `(API Contracts)` قرب نهاية السطر:
  * إذا انكسر السطر بين الكلمات الإنجليزية، ينفصل القوس الأيمن عن الأيسر وتحدث فوضى في خوارزمية الاتجاه ثنائي الاتجاه (BiDi Algorithm).
  * محاذاة الضبط الكامل (`WD_ALIGN_PARAGRAPH.JUSTIFY`) تمدد المسافات بين الكلمات بشكل قبيح وفجوات بيضاء شاسعة.
* **الحل:**
  1. اعتماد المحاذاة الطبيعية لليمين: `WD_ALIGN_PARAGRAPH.RIGHT`.
  2. استبدال المسافات والشرطات العادية داخل المصطلحات التقنية بمسافات وشرطات غير قابلة للكسر:
     * مسافة غير قابلة للكسر (Non-breaking space): `\u00A0`
     * شرطة غير قابلة للكسر (Non-breaking hyphen): `\u2011`
  3. هذا يضمن التفاف المصطلح التقني ككتلة واحدة متماسكة دون تفكك أو انقلاب في الأقواس.

---

## 2. الدالة البرمجية المعتمدة والمعيارية للمشروع

```python
import os
import re
import docx
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.oxml import OxmlElement
from docx.oxml.ns import qn
import win32com.client


def enforce_full_rtl_and_bidi(docx_path):
  """تطبيق المعايير الشاملة للاتجاه من اليمين لليسار، وحماية المصطلحات المختلطة،

  وضبط الجداول والمقاطع، وحفظ إعدادات Word COM الثنائية.
  """
  doc = docx.Document(docx_path)

  def process_paragraph(p):
    p.alignment = WD_ALIGN_PARAGRAPH.RIGHT
    pPr = p._p.get_or_add_pPr()

    # 1. اتجاه الفقرة من اليمين لليسار في بنية OpenXML
    if pPr.find(qn('w:bidi')) is None:
      pPr.insert(0, OxmlElement('w:bidi'))

    # 2. معالجة المقاطع النصية
    for r in p.runs:
      rPr = r._r.get_or_add_rPr()
      text = r.text
      has_en = bool(re.search(r'[A-Za-z]', text))
      has_ar = bool(re.search(r'[\u0600-\u06FF]', text))

      if has_en and not has_ar:
        # مقطع تقني إنجليزي: حماية من الانكسار والتدقيق الإملائي
        r.text = text.replace(' ', '\u00A0').replace('-', '\u2011')

        if rPr.find(qn('w:noProof')) is None:
          rPr.append(OxmlElement('w:noProof'))

        lang = rPr.find(qn('w:lang'))
        if lang is None:
          lang = OxmlElement('w:lang')
          rPr.append(lang)
        lang.set(qn('w:val'), 'en-US')

        # إزالة w:rtl حتى لا يعكس Word أقواس النصوص اللاتينية
        rtl_el = rPr.find(qn('w:rtl'))
        if rtl_el is not None:
          rPr.remove(rtl_el)
      else:
        # مقطع عربي
        if rPr.find(qn('w:rtl')) is None:
          rPr.append(OxmlElement('w:rtl'))

        lang = rPr.find(qn('w:lang'))
        if lang is None:
          lang = OxmlElement('w:lang')
          rPr.append(lang)
        lang.set(qn('w:bidi'), 'ar-SA')

  # معالجة كافة فقرات المستند
  for p in doc.paragraphs:
    process_paragraph(p)

  # معالجة الجداول (اتجاه الأعمدة من اليمين لليسار)
  for table in doc.tables:
    tblPr = table._tbl.tblPr
    if tblPr.find(qn('w:bidiVisual')) is None:
      tblPr.append(OxmlElement('w:bidiVisual'))
    for row in table.rows:
      for cell in row.cells:
        for p in cell.paragraphs:
          process_paragraph(p)

  # معالجة المقاطع (Sections)
  for sec in doc.sections:
    secPr = sec._sectPr
    if secPr.find(qn('w:bidi')) is None:
      secPr.insert(0, OxmlElement('w:bidi'))

  doc.save(docx_path)

  # 3. تثبيت الخصائص في تطبيق Microsoft Word عبر COM
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

## 3. بروتوكول التحقق الصارم قبل التسليم (Quality Assurance)

لا يُعتمد أي تعديل على ملفات الوورد في هذا المشروع إلا بعد استيفاء خطوات الفحص التالية:
1. **تصدير بصري:** تصدير المستند إلى PDF عبر Word COM:
   `doc.ExportAsFixedFormat(pdf_path, 17)`.
2. **ريندر بدقة عالية:** تحويل الصفحات إلى صور بدقة 2x بواسطة مكتبة `pypdfium2`.
3. **الفحص البصري للمعاينة:**
   - [ ] تأكيد أن زر اتجاه النص في Word هو `<-¶` (نشط ومظلل).
   - [ ] خلو المصطلحات التقنية من أي خطوط حمراء متعرجة.
   - [ ] اعتدال الأقواس `(...)` وثباتها في مكانها الصحيح.
   - [ ] سلامة مسافات الكلمات وعدم وجود فجوات بيضاء أو كلمات مقطوعة.
   - [ ] سريان الجداول من اليمين إلى اليسار (العمود الأول في أقصى اليمين).
