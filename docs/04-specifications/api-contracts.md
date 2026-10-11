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
| `X-Request-Id` | اختياري (يُولده Fastify تلقائياً إن غاب) | `req-89a1-43ef` | رمز تتبع دورة الطلب في سجلات Pino |
| `Accept-Language` | اختياري | `ar` أو `en` | لغة رسائل الأخطاء والتنبيهات (الافتراضي `ar`) |

### هيكل استجابة الأخطاء الموحد (RFC 7807 Problem Details):
```json
{
  "type": "https://athar.aden.gov.ye/errors/weight-anomaly-detected",
  "title": "Weight Anomaly Detected",
  "status": 422,
  "detail": "الوزن المسجل (55.4 كجم) يتجاوز الحد الأقصى المسموح به لخامة البلاستيك للمنزل الواحد.",
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
        "description": "جمع 14 كجم بلاستيك PET",
        "createdAt": "2026-10-10T14:22:10Z"
      }
    ]
  }
  ```

---

## 4. واجهات تطبيق عامل الجمع الميداني (Field Worker Endpoints)

### `GET /v1/worker/tasks`
استعلام المهام الميدانية المسندة للعامل في ورديته الحالية.
* **معاملات الاستعلام (Query Params):** `?status=ASSIGNED,EN_ROUTE`
* **الاستجابة الناجحة (200 OK):**
  ```json
  [
    {
      "taskId": "018f25bc-9901-7000-8000-111122223333",
      "trackingNumber": "ATHAR-MNSR-2026-10492",
      "status": "ASSIGNED",
      "residentName": "أحمد عبدالله",
      "address": "حي ريمي - شارع التسعين - عمارة 14",
      "coordinates": {
        "latitude": 12.862410,
        "longitude": 45.008920
      },
      "estimatedKg": 12.5
    }
  ]
  ```

### `POST /v1/worker/collection/weigh`
تسجيل الوزن الميداني عبر الميزان الذكي.
* **الترويسة:** `Idempotency-Key: <UUID>`
* **جسم الطلب:**
  ```json
  {
    "requestId": "018f25b8-5001-7000-8000-000000000001",
    "categoryId": "018f25b5-4421-7000-a111-987654321def",
    "grossWeightKg": 14.250,
    "tareWeightKg": 0.250,
    "scaleDeviceId": "BLE-SCALE-MNSR-08",
    "workerLocation": {
      "latitude": 12.862415,
      "longitude": 45.008918
    }
  }
  ```
* **الاستجابة الناجحة (201 Created):**
  ```json
  {
    "weighingId": "018f25c0-1111-7000-8000-222233334444",
    "netWeightKg": 14.000,
    "pointsGranted": 210,
    "anomalyFlag": false,
    "status": "COMPLETED"
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

### `PATCH /v1/admin/tickets/:id/resolve`
إغلاق وحل تذكرة دعم عملاء أو نزاع وزني.
* **جسم الطلب:**
  ```json
  {
    "resolutionSummary": "تم اعتماد وزن معدل للمواطن وإيداع فارق 30 نقطة خضراء بتعويض رسمي.",
    "adjustPointsAmount": 30
  }
  ```

---

## 6. واجهات الشركاء ونقاط البيع (Merchant POS Endpoints)

### `POST /v1/merchant/vouchers/redeem`
اعتماد وصرف قسيمة مشتريات للمواطن عبر مسح الـ QR Code.
* **الأدوار المصرح لها:** `MERCHANT`
* **جسم الطلب:**
  ```json
  {
    "voucherCode": "ATHAR-VCH-8921-9920",
    "terminalId": "POS-SUPERMARKET-03"
  }
  ```
* **الاستجابة الناجحة (200 OK):**
  ```json
  {
    "status": "REDEEMED_SUCCESSFULLY",
    "voucherDetails": {
      "itemTitle": "سلة تموينية أساسية (أرز + سكر + زيت)",
      "pointsDeducted": 500,
      "settlementCreditYer": 25000,
      "redeemedAt": "2026-10-11T04:47:00Z"
    }
  }
  ```
