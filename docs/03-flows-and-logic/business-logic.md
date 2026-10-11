# منطق الأعمال والخوارزميات المحورية (Business Logic & Core Algorithms)
## منصة «أثر» — بلدية المنصورة (العقدة التشغيلية الأولى)

توثق هذه الوثيقة الخوارزميات والمعادلات الرياضية ومنطق اتخاذ القرار المحوري المطبق في خوادم Node.js / Fastify وقاعدة بيانات PostgreSQL لضمان نزاهة العمليات الميدانية والمالية.

---

## 1. خوارزمية التحقق المكاني والنطاقات الجغرافية (Spatial Geofencing Logic)

تضمن هذه الخوارزمية أن طلبات الجمع وتواجد العمال تقع حصرياً داخل النطاق الجغرافي المصرح به لمديرية المنصورة باستخدام الامتداد المكاني PostGIS مع حفظ رقم إصدار النطاق.

```mermaid
flowchart TD
    Start(["استلام طلب جديد أو موقع أسرة [Lat, Lon]"]) --> BuildPoint["إنشاء نقطة هندسية: ST_SetSRID(ST_MakePoint(Lon, Lat), 4326)"]
    BuildPoint --> QueryZone{"فحص اشتمال النطاق الفعّال:<br/>SELECT id, boundary_version FROM active_nodes<br/>WHERE status = 'ACTIVE'<br/>AND ST_Covers(boundary, user_point)"}
    
    QueryZone -- "الموقع داخل نطاق مديرية المنصورة" --> Accept(["إنشاء طلب خدمة رسمي (ServiceRequest)<br/>وحفظ رقم نسخة النطاق boundary_version"])
    
    QueryZone -- "الموقع خارج النطاق الفعّال" --> Reject(["حفظ محاولة خارج النطاق (RejectedServiceAttempt)<br/>وعرض رسالة: 'الخدمة غير متوفرة بعد في هذا النطاق'<br/>مع خيار الانضمام لقائمة الانتظار (DemandWaitlist)"])
```

### الاستعلام المكاني المعتمد في Fastify / PostGIS:
```sql
SELECT 
    an.id AS node_id,
    an.name AS node_name,
    an.boundary_version
FROM active_nodes an
WHERE an.status = 'ACTIVE'
  AND ST_Covers(an.boundary, ST_SetSRID(ST_MakePoint($1, $2), 4326))
LIMIT 1;
```

---

## 2. محرك فحص الأوزان وكشف الشذوذ (Weight Verification & Anomaly Engine)

تضمن هذه الخوارزمية انضباط الأوزان المدونة لمساري الفرز (عضوي وجاف) والمرفوض، ومنع تسجيل أوزان غير معقولة تتجاوز السقف الأقصى المعتمد للزيارة.

```mermaid
flowchart TD
    WeightIn(["تدوين أوزان الزيارة: عضوي، جاف، مرفوض، ومرجع الميزان"]) --> CheckNegative{"هل أوزان العضوي والجاف والمرفوض >= 0 والوزن الإجمالي > 0 ؟"}
    
    CheckNegative -- "لا (قيم سالبة أو صفرية)" --> ErrZero(["رفض العملية: أوزان غير صالحة"])
    
    CheckNegative -- "نعم" --> CheckThreshold{"هل وزن العضوي أو الجاف يتجاوز السقف الأقصى المعتمد؟<br/>(max_threshold_kg للمسار الواحد)"}
    
    CheckThreshold -- "نعم (تجاوز السقف الأقصى)" --> FlagAnomaly["رصد شذوذ وزني قطعي:<br/>تسجيل الزيارة بحالة UNDER_REVIEW<br/>مع تجميد قيد النقاط وإحالة المراجعة للمشرف"]
    
    CheckThreshold -- "لا (ضمن السقف الطبيعي)" --> ApproveWeight["اعتماد الزيارة مباشرة:<br/>حالة الزيارة = APPROVED"]
    
    FlagAnomaly --> SupervisorReview["مراجعة المشرف الميداني للزيارة وصورة الإثبات خلال 24 ساعة"]
    SupervisorReview -- "اعتماد المشرف" --> MintPoints["احتساب وصرف النقاط لمساري العضوي والجاف فقط (المرفوض 0 نقطة)"]
    SupervisorReview -- "رفض المشرف" --> RejectVisit(["رفض الزيارة وإغلاقها دون أي نقاط"])
    
    ApproveWeight --> MintPoints
```

---

## 3. آلية تجنب تكرار العمليات عبر مفاتيح عدم التكرار (ACID Idempotency Engine)

تطبق المنصة حماية متقدمة وموثوقة لمنع تكرار العمليات المالية والتشغيلية عبر ترويسة `Idempotency-Key` المستندة إلى معاملات وقفل PostgreSQL الذري (جدول 31 من الوثيقة المرجعية).

```mermaid
flowchart TD
    ReqIn(["استقبال طلب POST / PUT مصحوباً بـ Idempotency-Key"]) --> CheckHeader{"هل الترويسة موجودة وصالحة (UUID/Hash)؟"}
    
    CheckHeader -- "غير موجودة" --> GenHash["توليد تجزئة فريدة من: (User_ID + Method + Body)"]
    CheckHeader -- "موجودة" --> UseKey["اعتماد المفتاح الممرر"]
    
    GenHash --> BeginTx
    UseKey --> BeginTx
    
    BeginTx["بدء معاملة ذرية: BEGIN TRANSACTION (PostgreSQL)"] --> LockRow["فحص وحجز مفتاح عدم التكرار:<br/>SELECT response_status, response_body FROM idempotency_keys<br/>WHERE key = $1 FOR UPDATE"]
    
    LockRow --> CheckStatus{"هل المفتاح مسجل مسبقاً في قاعدة البيانات؟"}
    
    CheckStatus -- "مسجل ومكتمل بنجاح" --> ReturnCached["إرجاع نفس الاستجابة السابقة مباشرة (200 OK)<br/>دون إعادة تنفيذ أي عمليات مالية أو خصم"]
    
    CheckStatus -- "قيد المعالجة (تزامن متزامن)" --> Return409(["إرجاع: 409 Conflict<br/>Concurrent Request in Progress"])
    
    CheckStatus -- "طلب جديد أول مرة" --> ExecLogic["تنفيذ منطق الأعمال الذري (خصم، وزن، نقاط)"]
    ExecLogic --> SaveResp["حفظ النتيجة والاستجابة في جدول idempotency_keys"]
    SaveResp --> CommitTx["اعتماد المعاملة: COMMIT TRANSACTION"]
    CommitTx --> Success201(["إرجاع النتيجة 201 Created للمستخدم"])
```

---

## 4. معادلة القيد المزدوج للنقاط الخضراء (Double-Entry Balance Rules)

لا يتم إنشاء أي نقطة خضراء (`Green Point`) في النظام دون قيد متوازن دائن ومدين؛ مما يمنع رياضياً أي اختلال في ميزان المدفوعات.

### 4.1 معادلة الإنشاء على أوزان الفرز المعتمدة (Recycling Minting):
الفرز عند المصدر مساران: **عضوي وجاف**؛ والمرفوض وزن مستبعد لا تنشأ عنه أي نقاط:
$$\Delta P = \lfloor W_{\text{organic}} \times R_{\text{organic}} \rfloor + \lfloor W_{\text{dry}} \times R_{\text{dry}} \rfloor$$
$$\Delta P_{\text{rejected}} = 0$$

**القيد المحاسبي المتولد:**
* **الطرف الدائن (Source Account):** `SYSTEM_MINT_ACCOUNT` ($\Delta P -$)
* **الطرف المدين (Destination Account):** `HOUSEHOLD_ACCOUNT` ($\Delta P +$)

### 4.2 معادلة استبدال وشراء منتجات المتجر بالنقاط (Store Product Redemption):
عند استبدال المواطن لنقاطه مقابل منتجات وسلع صديقة للبيئة من المتجر بتكلفة $P_{\text{spent}}$:
* **الطرف الدائن (Source Account):** `HOUSEHOLD_ACCOUNT` ($P_{\text{spent}} -$)
* **الطرف المدين (Destination Account):** `SYSTEM_BURN_ACCOUNT` ($P_{\text{spent}} +$)

### 4.3 معادلة إلغاء الطلب واسترداد النقاط (Order Cancellation & Points Rollback):
في حال رفض الإشعار المالي أو إلغاء الطلب قبل التجهيز:
* **قيد الاسترداد العكسي المتوازن:**
  * **الطرف الدائن (Source Account):** `SYSTEM_BURN_ACCOUNT` ($P_{\text{spent}} -$)
  * **الطرف المدين (Destination Account):** `HOUSEHOLD_ACCOUNT` ($P_{\text{spent}} +$)
* يعود رصيد المواطن فوراً كما كان دون أي نقص، ويوثق ذلك في `green_points_ledger` ببيان `ORDER_CANCELLED_REFUND`.
