# منطق الأعمال والخوارزميات المحورية (Business Logic & Core Algorithms)
## منصة «أثر» — بلدية المنصورة (العقدة التشغيلية الأولى)

توثق هذه الوثيقة الخوارزميات والمعادلات الرياضية ومنطق اتخاذ القرار المحوري المطبق في خوادم Node.js / Fastify وقاعدة بيانات PostgreSQL لضمان نزاهة العمليات الميدانية والمالية.

---

## 1. خوارزمية التحقق المكاني والنطاقات الجغرافية (Spatial Geofencing Logic)

تضمن هذه الخوارزمية أن طلبات الجمع المنزلي وتواجد عمال النظافة تقع حصرياً داخل النطاق الجغرافي المصرح به لحي المنصورة والمربعات السكنية التابعة له.

```mermaid
flowchart TD
    Start(["استلام طلب جديد أو موقع عامل [Lat, Lon]"]) --> BuildPoint["إنشاء نقطة هندسية: ST_SetSRID(ST_MakePoint(Lon, Lat), 4326)"]
    BuildPoint --> QueryZone{"استعلام النطاق:<br/>SELECT id FROM geofence_zones<br/>WHERE status = 'ACTIVE'<br/>AND ST_Covers(boundary, user_point)"}
    
    QueryZone -- "تم العثور على نطاق مطابق" --> CheckBuffer["فحص المسافة لأقرب حاوية/شارع رئيسي:<br/>ST_DWithin(user_point, street_line, 50)"]
    CheckBuffer --> ValidateService{"هل المربع السكني متاح للخدمة الآن؟"}
    
    ValidateService -- "نعم (ضمن جدول التشغيل)" --> Success(["قبول الطلب وتعيين Zone_ID تلقائياً"])
    ValidateService -- "لا (خارج ساعات الوردية)" --> Defer(["حفظ الطلب وجدولته لأقرب نافذة زمنية"])
    
    QueryZone -- "لا يوجد نطاق يغطي النقطة" --> Reject(["رفض الطلب: 422 Unprocessable Entity<br/>'الموقع خارج نطاق التغطية التشغيلية لمديرية المنصورة'"])
```

### الاستعلام المكاني المعتمد في Fastify / PostGIS:
```sql
SELECT 
    gz.id AS zone_id,
    gz.name AS zone_name,
    gz.code AS zone_code
FROM geofence_zones gz
WHERE gz.status = 'ACTIVE'
  AND ST_Covers(gz.boundary, ST_SetSRID(ST_MakePoint($1, $2), 4326))
LIMIT 1;
```

---

## 2. محرك فحص الأوزان وكشف الشذوذ (Weight Verification & Anomaly Engine)

تضمن هذه الخوارزمية انضباط الأوزان المدونة لمساري الفرز (عضوي وجاف) والمرفوض، ومنع تسجيل أوزان غير معقولة تتجاوز السقف الاستهلاكي للأسرة.

```mermaid
flowchart TD
    WeightIn(["تدوين أوزان الزيارة: عضوي، جاف، مرفوض، ومرجع الميزان"]) --> CheckNegative{"هل أوزان العضوي والجاف >= 0 والوزن الإجمالي > 0 ؟"}
    
    CheckNegative -- "لا" --> ErrZero(["رفض العملية: أوزان غير صالحة"])
    
    CheckNegative -- "نعم" --> FetchLimits["جلب سقف الوزن المعتمد للأسرة والمتوسط التاريخي"]
    FetchLimits --> CheckAbsoluteMax{"هل إجمالي الوزن > السقف الأقصى للزيارة؟<br/>(مثال: > 40 كجم للأسرة الواحدة)"}
    
    CheckAbsoluteMax -- "نعم (تجاوز السقف)" --> FlagAnomaly1["رصد شذوذ قطعي:<br/>حفظ الزيارة بحالة UNDER_REVIEW<br/>السبب: EXCEEDED_MAX_THRESHOLD"]
    
    CheckAbsoluteMax -- "لا" --> CheckSpike{"هل الوزن > (متوسط الأسرة × 3.5) ؟<br/>(قفزة غير مبررة مقارنة بالمتوسط)"}
    CheckSpike -- "نعم (قفزة شاذة)" --> FlagAnomaly2["رصد شذوذ إحصائي:<br/>حفظ الزيارة بحالة UNDER_REVIEW<br/>السبب: STATISTICAL_WEIGHT_SPIKE"]
    
    CheckSpike -- "لا" --> ApproveWeight["اعتماد الزيارة مباشرة:<br/>حالة الزيارة = APPROVED"]
    
    FlagAnomaly1 --> HoldPoints["تعليق صرف النقاط وإحالة الزيارة للمشرف الميداني للتحقق خلال 24 ساعة"]
    FlagAnomaly2 --> HoldPoints
    
    ApproveWeight --> MintPoints["احتساب وصرف النقاط لمساري العضوي والجاف فقط (المرفوض 0 نقطة)"]
```

---

## 3. آلية تجنب تكرار العمليات عبر مفاتيح عدم التكرار (Distributed Idempotency Engine)

نظراً لتذبذب شبكة الإنترنت في الميدان (3G/4G)، قد يضغط المستخدم أو العامل زر الإرسال مرتين، أو يعيد تطبيق Flutter المحاولة تلقائياً. تطبق المنصة حماية متقدمة عبر مفتاح `Idempotency-Key`.

```mermaid
flowchart TD
    ReqIn(["استقبال طلب POST / PUT مصحوباً بـ Idempotency-Key"]) --> CheckHeader{"هل الترويسة موجودة وصالحة (UUID/Hash)؟"}
    
    CheckHeader -- "غير موجودة" --> GenHash["توليد تجزئة فريدة من: (User_ID + Method + Body)"]
    CheckHeader -- "موجودة" --> UseKey["اعتماد المفتاح الممرر"]
    
    GenHash --> RedisCheck
    UseKey --> RedisCheck
    
    RedisCheck["فحص القفل الذري في Redis:<br/>SET lock:key token NX EX 30"] --> IsLocked{"هل نجح حجز القفل في Redis؟"}
    
    IsLocked -- "فشل (الطلب قيد المعالجة الآن)" --> WaitResponse["انتظار 500ms أو إرجاع:<br/>409 Conflict: Concurrent Request in Progress"]
    
    IsLocked -- "نجح (الطلب جديد)" --> CheckDB{"هل مفتاح Idempotency-Key مسجل مسبقاً في قاعدة البيانات؟"}
    
    CheckDB -- "نعم مسجل ومكتمل سابقاً" --> ReturnCached["إرجاع نفس الاستجابة المخزنة السابقة مباشرة (200 OK)<br/>دون إعادة تنفيذ أي عمليات مالية أو خصم"]
    
    CheckDB -- "لا (طلب أول مرة)" --> ExecuteBusiness["تنفيذ منطق الأعمال داخل ACID Transaction"]
    ExecuteBusiness --> SaveResult["حفظ النتيجة ومفتاح عدم التكرار في PostgreSQL"]
    SaveResult --> ReleaseLock["حذف القفل الذري من Redis وإرجاع النتيجة 201 Created"]
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
