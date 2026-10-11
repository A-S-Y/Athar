# مواصفات وعقود واجهات البرمجة (API Contracts & Specifications)
## منصة «أثر» — خوادم Node.js / Fastify (الإصدار v1)

---

## 1. المعايير العامة والترويسات الإلزامية (Global Standards & Headers)

تعتمد منصة «أثر» معايير **RESTful APIs** الصارمة مع استخدام **JSON** كصيغة قياسية للبيانات، وترميز `UTF-8`.

### الترويسات المطلوبة في كل طلب (Request Headers):
| الترويسة (Header) | الإلزامية | مثال | الوصف |
| :--- | :--- | :--- | :--- |
| `Authorization` | إلزامي (للنقاط المحمية) | `Bearer eyJhbGciOi...` | رمز التحقق JWT الموقع بمفتاح RSA-256 |
| `Idempotency-Key` | إلزامي لعمليات `POST/PUT/PATCH` المالية والحساسة | `d8b5c464-9cf8-4c8d-b08e-c5db149e6f32` | مفتاح تجنب التكرار لحماية العمليات |
| `X-Request-Id` | اختياري (يُولده Fastify تلقائياً إن غاب) | `req-89a1-43ef` | رمز تتبع دورة حياة الطلب في سجلات الخادم (Structured JSON Logs) |
| `Accept-Language` | اختياري | `ar` أو `en` | لغة رسائل الأخطاء والتنبيهات (الافتراضي `ar`) |

### هيكل استجابة الأخطاء الموحد (RFC 7807 Problem Details):
```json
{
  "type": "https://athar.aden.gov.ye/errors/weight-anomaly-detected",
  "title": "Weight Anomaly Detected",
  "status": 422,
  "detail": "الوزن المسجل (55.4 كجم) يتجاوز الحد الأقصى المسموح به للمسار الجاف للزيارة الواحدة.",
  "instance": "/v1/collection/weigh",
  "requestId": "req-89a1-43ef",
  "invalidParams": [
    {
      "name": "grossWeightKg",
      "reason": "Exceeds threshold limit of 50.00 kg"
    }
  ]
}
```

---

## 2. واجهات المصادقة والجلسات (Authentication & Session Endpoints)

### `POST /v1/auth/request-otp`
إرسال رمز التحقق المؤقت عبر رسالة نصية قصيرة SMS.
* **الحد المسموح (Rate Limit):** 3 طلبات كل 15 دقيقة لكل رقم هاتف.
* **جسم الطلب (Request Body):**
  ```json
  {
    "phone": "+967771234567"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "success": true,
    "message": "تم إرسال رمز التحقق بنجاح",
    "expiresInSeconds": 180
  }
  ```

### `POST /v1/auth/verify-otp`
التحقق من الرمز وإصدار الرموز المميزة للجلسة.
* **جسم الطلب:**
  ```json
  {
    "phone": "+967771234567",
    "otpCode": "481920"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1Ni...",
    "refreshToken": "7f8b9a12c34d5e6f...",
    "expiresIn": 900,
    "user": {
      "id": "018f25a7-9e42-7000-8431-419b48f98011",
      "fullName": "أحمد عبدالله باوزير",
      "role": "RESIDENT",
      "status": "ACTIVE"
    }
  }
  ```

### `POST /v1/auth/refresh-token`
تدوير الرمز وإصدار Access Token جديد.
* **جسم الطلب:** `{ "refreshToken": "7f8b9a12c34d5e6f..." }`
* **الاستجابة الناجحة (200 OK):** `{ "accessToken": "...", "refreshToken": "..." }`

---

## 3. واجهات تطبيق المواطن السكني (Resident Mobile Endpoints)

### `POST /v1/resident/service-requests`
رفع طلب تجميع وفرز نفايات منزلية جديد.
* **الترويسة:** `Idempotency-Key: <UUID>`
* **جسم الطلب:**
  ```json
  {
    "addressId": "018f25b2-3210-7000-98ef-123456789abc",
    "categoryIds": [
      "018f25b5-4421-7000-a111-987654321def"
    ],
    "estimatedWeightKg": 12.5,
    "scheduledWindow": "2026-10-12T09:00:00Z",
    "notes": "الأكياس موضوعة بجوار البوابة الخارجية"
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "requestId": "018f25b8-5001-7000-8000-000000000001",
    "trackingNumber": "ATHAR-MNSR-2026-10492",
    "status": "SUBMITTED",
    "zoneName": "المنصورة - حي ريمي",
    "createdAt": "2026-10-11T04:45:00Z"
  }
  ```

### `GET /v1/resident/ledger/balance`
استعلام رصيد النقاط الخضراء وسجل العمليات الأخير.
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "totalBalance": 1450,
    "currency": "GP",
    "co2SavedKg": 43.5,
    "recentTransactions": [
      {
        "id": "018f25ba-1002-7000-8000-000000000002",
        "type": "MINT_RECYCLING",
        "amount": 210,
        "description": "جمع 14 كجم مخلفات جافة مفرزة",
        "createdAt": "2026-10-10T14:22:10Z"
      }
    ]
  }
  ```

---

## 4. واجهات تطبيق عامل الجمع الميداني (Field Worker Endpoints)

### `POST /v1/qr/resolve`
حل رمز البطاقة الفيزيائية الممسوح بكاميرا هاتف العامل وربطه بالمهمة.
* **جسم الطلب:**
  ```json
  {
    "qrCode": "MN-48192",
    "taskId": "018f25bc-9901-7000-8000-111122223333"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "householdId": "018f25b8-5001-7000-8000-000000000001",
    "entityCode": "48192",
    "addressText": "حي ريمي - عمارة 14",
    "status": "ACTIVE"
  }
  ```

### `POST /v1/worker/visits`
تسجيل أوزان الزيارة الميدانية يدوياً (عضوي، جاف، مرفوض) مع مرجع الميزان.
* **الترويسة:** `Idempotency-Key: <UUID>`
* **جسم الطلب:**
  ```json
  {
    "householdId": "018f25b8-5001-7000-8000-000000000001",
    "taskId": "018f25bc-9901-7000-8000-111122223333",
    "organicKg": 8.500,
    "dryKg": 3.200,
    "rejectedKg": 0.400,
    "scaleReference": "SCALE-MNSR-04",
    "evidenceReference": "proof-visit-10492.jpg"
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "visitId": "018f25c0-1111-7000-8000-222233334444",
    "pointsEarned": 117,
    "status": "APPROVED",
    "message": "تم اعتماد الزيارة وقيد النقاط لحساب الأسرة"
  }
  ```

### `POST /v1/worker/collection/batch-sync`
مزامنة وتفريغ الدفعات الميدانية المسجلة أثناء انقطاع الاتصال (Field Batch CSV Import).
* **جسم الطلب:**
  ```json
  {
    "batchReference": "BATCH-MNSR-20261011-04",
    "recordsCount": 15,
    "records": [
      {
        "householdCode": "48192",
        "organicKg": 8.500,
        "dryKg": 3.200,
        "rejectedKg": 0.400,
        "scaleReference": "SCALE-MNSR-04",
        "occurredAtUtc": "2026-10-11T03:15:00Z"
      }
    ]
  }
  ```
* **الاستجابة الناجحة (202 Accepted):**
  ```json
  {
    "batchReference": "BATCH-MNSR-20261011-04",
    "status": "QUEUED_FOR_PROCESSING",
    "message": "تم استلام الدفعة الميدانية وجارٍ معالجتها وترحيلها خلفياً عبر BullMQ"
  }
  ```

---

## 5. واجهات لوحة التحكم الإدارية (Back-Office Admin Endpoints)

### `GET /v1/admin/dispatch/cockpit`
غرفة القيادة التفاعلية للعمليات الميدانية وتتبع الأسطول.
* **الأدوار المصرح لها:** `SUPER_ADMIN`, `ADMIN_SUPERVISOR`, `ADMIN_DISPATCHER`
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "activeWorkersCount": 18,
    "pendingRequestsCount": 34,
    "completedTodayCount": 142,
    "fleetLiveLocations": [
      {
        "workerId": "018f25a7-9e42-7000-8431-419b48f98011",
        "workerName": "سالم عمر كرامة",
        "currentCoordinates": [12.8631, 45.0102],
        "activeTaskId": "018f25bc-9901-7000-8000-111122223333",
        "batteryPercentage": 82,
        "lastHeartbeat": "2026-10-11T04:46:12Z"
      }
    ]
  }
  ```

### `POST /v1/admin/dispatch/reassign`
إعادة توجيه قسري لمهمة ميدانية من عامل إلى آخر.
* **جسم الطلب:**
  ```json
  {
    "taskId": "018f25bc-9901-7000-8000-111122223333",
    "newWorkerId": "018f25a9-0001-7000-8000-999988887777",
    "reason": "تعطل دراجة العامل الأول في شارع الكثيري"
  }
  ```

### `PATCH /v1/admin/incidents/:id/resolve`
إغلاق وحل بلاغ ميداني أو استثناء وزني.
* **جسم الطلب:**
  ```json
  {
    "resolutionSummary": "تمت مراجعة وزن الأسرة واعتماد تعديل وزني بتعويض رسمي 30 نقطة خضراء.",
    "adjustPointsAmount": 30
  }
  ```

---

## 6. واجهات المتجر الرقمي والتحقق الثنائي (Store & Two-Step Verification Endpoints)

### `POST /v1/store/orders`
إنشاء طلب شراء لسلعة من المتجر متضمناً بيانات وإثبات التحويل البنكي في خطوة واحدة (Two-Step Step 1).
* **الترويسة:** `Idempotency-Key: <UUID>`
* **جسم الطلب (Multipart / JSON):**
  ```json
  {
    "productId": "018f25c5-3333-7000-8000-444455556666",
    "quantity": 1,
    "fulfillment": "DELIVERY",
    "deliveryAddress": "مديرية المنصورة - حي عبد العزيز",
    "paymentMethod": "BANK_TRANSFER",
    "transferProvider": "الكريمي جوال",
    "transferRef": "TRX-9988112",
    "receiptFileRef": "receipts/ord-0042-proof.jpg"
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "orderId": "018f25c8-7777-7000-8000-888899990000",
    "orderNumber": "ORD-2026-0042",
    "status": "PENDING_VERIFICATION",
    "cashAmountYer": 15000,
    "pointsSpent": 0,
    "message": "تم استلام الطلب وحجز المخزون بنجاح، وهو قيد التحقق المالي لدى الإدارة"
  }
  ```

### `PATCH /v1/admin/store/orders/:id/verify`
اعتماد أو رفض الطلب وسند السداد من قبل موظف الإدارة في لوحة التحكم (Back-Office Portal).
* **الأدوار المصرح لها:** `SUPER_ADMIN`, `ADMIN_ACCOUNTANT`, `ADMIN_SUPERVISOR`
* **جسم الطلب:**
  ```json
  {
    "action": "APPROVE",
    "notes": "تمت مطابقة رقم الحوالة في الحساب البنكي الرسمي لبنك الكريمي"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "orderId": "018f25c8-7777-7000-8000-888899990000",
    "status": "VERIFIED_PAID",
    "verifiedBy": "موظف التدقيق المالي",
    "verifiedAt": "2026-10-11T04:48:00Z"
  }
  ```

---

## 7. واجهات الشركاء التجاريين والتسليم (Resource Partners & Delivery Endpoints)

### `GET /v1/partners/prices`
استعلام لائحة أسعار شراء الخامات المفروزة الفعالة المخصصة للشريك.
* **الأدوار المصرح لها:** `RESOURCE_PARTNER`, `ADMIN_SUPERVISOR`, `ADMIN_FINANCE`
* **الاستجابة الناجحة (200 OK):**
  ```json
  [
    {
      "materialCode": "DRY_RECYCLABLES",
      "nameAr": "مخلفات جافة مفروزة",
      "pricePerKgYer": 320.00,
      "validFrom": "2026-10-01T00:00:00Z",
      "validTo": "2026-12-31T23:59:59Z"
    }
  ]
  ```

### `POST /v1/partner-deliveries`
إنشاء محضر تسليم شحنة مفروزة من نقطة التجميع إلى شريك معتمد.
* **الأدوار المصرح لها:** `ADMIN_SUPERVISOR`, `ADMIN_DISPATCHER`
* **الترويسة:** `Idempotency-Key: <UUID>`
* **جسم الطلب:**
  ```json
  {
    "partnerId": "018f25d0-1111-7000-8000-222233334444",
    "pointId": "018f25d2-5555-7000-8000-666677778888",
    "materialCode": "DRY_RECYCLABLES",
    "weightKg": 420.500,
    "sourceVisits": ["VISIT-001", "VISIT-002", "VISIT-003"]
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "deliveryId": "018f25d5-9999-7000-8000-000011112222",
    "status": "DELIVERED",
    "totalValueYer": 134560.00,
    "message": "تم توثيق محضر تسليم الشحنة بنجاح"
  }
  ```

### `POST /v1/partner-payments/:id/verify`
اعتماد وتدقيق سند تحصيل مستحقات الشحنة من الشريك.
* **الأدوار المصرح لها:** `ADMIN_FINANCE`, `SUPER_ADMIN`
* **جسم الطلب:**
  ```json
  {
    "amount": 134560.00,
    "reference": "BANK-DEP-99410",
    "proofRef": "receipts/partners/dep-99410.pdf"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "paymentId": "018f25d8-3333-7000-8000-444455556666",
    "status": "VERIFIED",
    "verifiedAt": "2026-10-11T05:00:00Z",
    "message": "تم تأكيد سند التحصيل المالي رسمياً"
  }
  ```

---

## 8. واجهات استبيان الضم المبدئي للأسرة (Intake & Survey Endpoints)

### `POST /v1/intake`
بدء إجراءات تسجيل وضّم أسرة جديدة مع إحداثيات الموقع.
* **جسم الطلب:**
  ```json
  {
    "streetName": "شارع الكثيري",
    "buildingNumber": "12",
    "coordinates": [12.8631, 45.0102]
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "householdId": "018f25da-7777-7000-8000-888899990000",
    "entityCode": "48192",
    "status": "PENDING_SURVEY",
    "message": "تم تسجيل موقع الأسرة وتخصيص معرّف الكيان، يرجى استكمال الاستبيان"
  }
  ```

### `POST /v1/households/:id/survey`
رفع واعتماد إجابات استبيان الأسرة الميداني أونلاين.
* **جسم الطلب:**
  ```json
  {
    "templateVersion": 1,
    "answers": {
      "familyMembersCount": 5,
      "estimatedDailyWasteKg": 3.5,
      "willingToSeparateOrganic": true
    }
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "surveyId": "018f25dc-1111-7000-8000-222233334444",
    "status": "SUBMITTED",
    "message": "تم حفظ استبيان الأسرة بنجاح"
  }
  ```

---

## 9. واجهات التوظيف والتطوع المجتمعي (Community & HR Endpoints)

### `POST /v1/applications`
تقديم طلب للعمل كعامل جمع ميداني في المنصورة.
* **جسم الطلب:**
  ```json
  {
    "roleApplied": "FIELD_COLLECTOR",
    "experience": "خبرة سنتين في النظافة الحضرية والفرز",
    "documentRefs": ["docs/id-front.jpg", "docs/id-back.jpg"],
    "consentTerms": true
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "applicationId": "018f25de-5555-7000-8000-666677778888",
    "status": "SUBMITTED",
    "message": "تم استلام طلب التوظيف وسيتم التواصل بعد المراجعة"
  }
  ```

### `POST /v1/volunteer-interest`
تسجيل رغبة المواطن في التطوع المجتمعي والتوعوي.
* **جسم الطلب:**
  ```json
  {
    "area": "حي ريمي",
    "consentTerms": true
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "volunteerId": "018f25e0-9999-7000-8000-000011112222",
    "status": "REGISTERED",
    "message": "شكراً لاهتمامك بالتطوع لخدمة مجتمعك في أثر"
  }
  ```

---

## 10. واجهات التكليف والإسناد الميداني الصريح (Task Dispatch Endpoints)

### `POST /v1/dispatch/:id/assign`
تكليف عامل جمع بمهمة محددة وبدء مهلة الـ 10 دقائق للإقرار.
* **الأدوار المصرح لها:** `ADMIN_DISPATCHER`, `ADMIN_SUPERVISOR`
* **جسم الطلب:**
  ```json
  {
    "workerId": "018f25a7-9e42-7000-8431-419b48f98011"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "dispatchId": "018f25e2-3333-7000-8000-444455556666",
    "status": "ASSIGNED",
    "ackDueAt": "2026-10-11T05:10:00Z",
    "message": "تم إسناد المهمة وإرسال إشعار Push للعامل (مهلة 10 دقائق)"
  }
  ```

### `POST /v1/dispatch/:id/ack`
إقرار العامل الميداني باستلام المهمة عبر تطبيقه ومنع إرسال SMS التلقائية.
* **الأدوار المصرح لها:** `FIELD_WORKER`
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "dispatchId": "018f25e2-3333-7000-8000-444455556666",
    "status": "ACKNOWLEDGED",
    "acknowledgedAt": "2026-10-11T05:03:15Z",
    "message": "تم تسجيل إقرار العامل بنجاح وإلغاء مهلة الـ SMS"
  }
  ```
