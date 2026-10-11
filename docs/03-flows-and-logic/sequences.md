# مخططات التتابع الزمني للعمليات (System Sequence Diagrams)
## منصة «أثر» — بلدية المنصورة (العقدة التشغيلية الأولى)

توثق هذه الصفحة التتابعات الزمنية وتدفق البيانات بين مختلف طبقات النظام (تطبيقات الجوال، لوحة التحكم، خوادم Fastify، قاعدة بيانات PostgreSQL، وطوابير BullMQ) لأهم أربع عمليات حرجة في المنصة.

---

## 1. مخطط تسجيل الدخول بالرمز المؤقت والتدوير الآمن للرموز (Passwordless OTP & Token Rotation)

يوضح هذا المخطط آلية الدخول دون كلمة مرور عبر الرسائل القصيرة (SMS OTP) مع تحديد معدل الطلبات (Rate Limiting) وتدوير الرموز المميزة (Refresh Token Rotation).

```mermaid
sequenceDiagram
    autonumber
    actor User as المستخدم (مواطن / عامل)
    participant App as تطبيق الهاتف (Flutter)
    participant Gateway as بوابة الـ API (Fastify)
    participant Redis as خادم التخزين المؤقت (Redis 7)
    participant DB as قاعدة البيانات (PostgreSQL)
    participant SMS as مزود الرسائل (Yemen SMS Gateway)

    User->>App: إدخال رقم الهاتف (+967-7xxxxxxx)
    App->>Gateway: POST /v1/auth/request-otp { phone }
    
    Gateway->>Redis: فحص الـ Rate Limit (3 محاولات / 15 دقيقة)
    alt تجاوز الحد المسموح
        Redis-->>Gateway: Exceeded Rate Limit
        Gateway-->>App: 429 Too Many Requests (Retry after 900s)
    else ضمن الحد المسموح
        Gateway->>Gateway: توليد رمز سري آمن مكون من 6 أرقام (CSPRNG)
        Gateway->>Redis: SETEX otp:phone 180 (تخزين الرمز لمدة 3 دقائق)
        Gateway->>SMS: إرسال الرسالة النصية بالرمز
        SMS-->>User: وصول رسالة SMS: رمز تأكيد أثر هو [123456]
        Gateway-->>App: 200 OK { message: "OTP Sent Successfully" }
    end

    User->>App: إدخال الرمز السري المستلم
    App->>Gateway: POST /v1/auth/verify-otp { phone, otp_code }
    Gateway->>Redis: GET otp:phone
    alt الرمز غير مطابق أو منتهي الصلاحية
        Gateway-->>App: 401 Unauthorized (Invalid or Expired OTP)
    else الرمز صحيح
        Gateway->>Redis: DEL otp:phone (حذف الرمز فوراً لمنع إعادة الاستخدام)
        Gateway->>DB: البحث عن المستخدم بالرقم المشفر (Blind Index)
        alt مستخدم جديد
            Gateway->>DB: إنشاء سجل جديد في UserAccount وتوثيق الهاتف
        end
        Gateway->>Gateway: إصدار رمز وصول موثوق بالصلاحيات (RBAC Access Token)
        Gateway->>DB: تحديث طابع التحقق phone_verified_at في UserAccount
        Gateway-->>App: 200 OK { accessToken, userProfile }
        App->>App: تخزين الرموز بأمان في Flutter Secure Storage
    end
```

### القرارات المعمارية المطبقة:
* **تخزين الرموز المؤقتة في الذاكرة (Redis TTL):** تنتهي صلاحية الرمز تلقائياً بعد 180 ثانية، ويُحذف فور التحقق لمنع هجمات الإعادة (Replay Attacks).
* **الفهرسة العمياء للخصوصية:** لا يُبحث عن رقم الهاتف كنص صريح بل عبر قيمة `phone_bindex` المجزأة بتشفير HMAC لضمان سرية هوية المواطنين.

---

## 2. دورة حياة مسح رمز البطاقة، الجمع الميداني، وقيد النقاط (Field Collection & QR Card Flow)

توثق هذه العملية كيفية إتمام استلام النفايات من قبل عامل النظافة الميداني، ومسح بطاقة الـ QR المطبوعة على المنزل بكاميرا التطبيق، وتسجيل أوزان المسارين (عضوي وجاف) والمرفوض يدوياً مع مرجع الميزان، وتسجيل النقاط بالقيد المزدوج المضمون.

```mermaid
sequenceDiagram
    autonumber
    actor Resident as المواطن (رب الأسرة)
    actor Worker as عامل التجميع الميداني
    actor Supervisor as المشرف الميداني
    participant WApp as تطبيق العامل (Worker App)
    participant API as خادم Fastify API
    participant DB as قاعدة البيانات (PostgreSQL)
    participant Queue as طوابير BullMQ (Redis)
    participant SMS as مزود الرسائل (SMS Gateway)

    Note over Worker,WApp: 1. مسح بطاقة رمز الأسرة (QR Card)
    Worker->>WApp: فتح كاميرا التطبيق ومسح رمز البطاقة الفيزيائية المطبوعة على المنزل
    WApp->>API: POST /qr/resolve { qrCode: "MN-48192", taskId }
    
    activate API
    API->>DB: مطابقة رمز بطاقة الأسرة المعبر عن كود الأسرة (HouseholdQrToken) مع المهمة النشطة
    DB-->>API: معرّف الأسرة الصالح (Household ID)
    API-->>WApp: 200 OK { entityCode: "48192", address: "حي ريمي - عمارة 14" }
    deactivate API

    Note over Worker,WApp: 2. وزن المواد وتدوين الأرقام يدوياً
    Worker->>Worker: وزن أكياس العضوي والجاف بالميزان الميداني
    Worker->>WApp: إدخال الأوزان يدوياً: عضوي 8.5 كجم، جاف 3.2 كجم، مرفوض 0.4 كجم
    Worker->>WApp: إدخال مرجع الميزان (SCALE-04) وصورة الإثبات (اختياري)
    WApp->>API: POST /visits { householdId, organicKg, dryKg, rejectedKg, scaleReference }
    
    activate API
    API->>API: فحص الشذوذ الوزني بالتحقق الصارم من السقف الأقصى المعتمد للخامة
    
    alt رصد وزن شاذ أو مريب (> السقف الأقصى المعتمد)
        API->>DB: تسجيل الزيارة بحالة UNDER_REVIEW (تعليق النقاط)
        API->>DB: إنشاء تنبيه مراجعة للمشرف في لوحة العمليات
        API-->>WApp: 201 Created { status: "UNDER_REVIEW", message: "الزيارة بانتظار مراجعة المشرف" }
        Supervisor->>API: POST /visits/:id/approve (اعتماد المشرف بعد التحقق)
    else الأوزان طبيعية ومطابقة
        API->>DB: BEGIN TRANSACTION (بدء معاملة بنكية صارمة)
        API->>DB: حفظ سجل collection_visits و collection_items
        API->>DB: تحديث حالة service_requests إلى COMPLETED
        API->>DB: حساب النقاط: (8.5 × معدل العضوي) + (3.2 × معدل الجاف) — المرفوض 0 نقطة
        API->>DB: قيد محاسبي مزدوج: من SYSTEM_MINT إلى HOUSEHOLD_ACCOUNT
        API->>DB: COMMIT TRANSACTION (حفظ تام لا رجعة فيه)
        
        API->>Queue: إضافة مهمة إشعار إلى BullMQ: addJob('send-points-sms', { phone, points })
        API-->>WApp: 201 Created { status: "APPROVED", pointsEarned: 117 }
    end
    deactivate API
    
    Note over Queue,SMS: إرسال الإشعار للمواطن عبر طابور BullMQ
    Queue->>SMS: إرسال رسالة SMS للمواطن: "تم اعتماد 117 نقطة خضراء لحسابك!"
    SMS-->>Resident: وصول الرسالة النصية فورياً
```

---

## 3. شراء منتجات المتجر بالتحقق الثنائي والتدقيق المالي (Store Order & Two-Step Verification Flow)

يوضح هذا المخطط دورة شراء واستبدال السلع البيئية عبر المتجر، ورفع إشعار التحويل البنكي، والتدقيق المالي والاعتماد من قبل موظفي الإدارة في لوحة التحكم (Back-Office Portal).

```mermaid
sequenceDiagram
    autonumber
    actor Resident as المواطن المستفيد
    actor Staff as موظف الإدارة والتدقيق المالي
    participant ResApp as تطبيق المواطن / المتجر
    participant AdminPortal as لوحة تحكم الإدارة (React Portal)
    participant API as خادم Fastify API
    participant Storage as وحدة التخزين المحلي المشفر (Contabo Storage)
    participant DB as قاعدة البيانات (PostgreSQL)
    participant Queue as طوابير المهام (BullMQ)

    Resident->>ResApp: تصفح المنتجات واختيار سلعة وإرفاق إشعار الحوالة
    ResApp->>API: 1. رفع صورة إشعار الحوالة (POST /v1/uploads)
    API->>Storage: حفظ صورة الإشعار في مجلد receipts المشفر محلياً على سيرفر Contabo
    Storage-->>API: مرجع الملف المشفر (receiptFileRef)
    API-->>ResApp: 201 Created { receiptFileRef }

    ResApp->>API: 2. إنشاء الطلب: POST /v1/store/orders { items, transferRef, receiptFileRef }
    
    activate API
    API->>DB: BEGIN TRANSACTION
    API->>DB: فحص وحجز المخزون: SELECT stock_quantity FROM store_products FOR UPDATE
    alt المخزون غير كافٍ
        API-->>ResApp: 409 Conflict (الكمية المطلوبة غير متوفرة)
    else المخزون متاح
        API->>DB: خصم الكمية من المخزون
        API->>DB: خصم النقاط الخضراء بالقيد المزدوج (إن وجد شق نقاط)
        API->>DB: إنشاء سجل الطلب بحالة PENDING_VERIFICATION مع مرجع الحوالة والإشعار
        API->>DB: COMMIT TRANSACTION
        API-->>ResApp: 201 Created { orderId, orderNumber, status: "PENDING_VERIFICATION" }
    end
    deactivate API

    Note over Staff,AdminPortal: مراجعة واعتماد الطلب من قبل الإدارة
    Staff->>AdminPortal: فتح شاشة «تدقيق طلبات المتجر والحوالات المعلقة»
    AdminPortal->>API: GET /v1/admin/store/orders?status=PENDING_VERIFICATION
    API->>DB: استعلام الطلبات المعلقة مع بيانات المشتري وصورة الإشعار
    DB-->>API: قائمة الطلبات
    API-->>AdminPortal: عرض الطلبات وسندات الدفع للمدقق المالي

    Staff->>AdminPortal: مطابقة رقم الحوالة في الحساب البنكي والضغط على «اعتماد»
    AdminPortal->>API: PATCH /v1/admin/store/orders/:id/verify { action: "APPROVE" }
    
    activate API
    API->>DB: UPDATE store_orders SET status = 'VERIFIED_PAID', verified_by_staff_id = ?, verified_at = NOW()
    API->>DB: INSERT INTO audit_logs (actor_id, action: 'STORE_ORDER_VERIFIED', entity_id)
    API->>Queue: إضافة مهمة إشعار وتجهيز الطلب (OrderApprovedJob)
    API-->>AdminPortal: 200 OK { status: "VERIFIED_PAID" }
    deactivate API

    Queue-->>ResApp: إشعار للمواطن: "تم اعتماد طلبك وجارٍ تجهيز الشحنة للتسليم!"
```

---

## 4. مزامنة الدفعات الميدانية غير المتصلة وطوابير المعالجة الخلفية (Field Batch Sync & Background Queue Workers)

يوضح هذا المخطط الآلية المعمارية المعتمدة لمعالجة البيانات عند انقطاع التغطية الميدانية في مناطق عدن، حيث يحفظ التطبيق الميداني العمليات محلياً (SQLite)، ثم يقوم بالمزامنة الدفيعية فور توفر الشبكة لتعالجها طوابير BullMQ دون تحميل خادم الـ API أي بطء.

```mermaid
sequenceDiagram
    autonumber
    actor Worker as جامع النفايات الميداني
    participant WApp as تطبيق العامل (مخزن SQLite محلي)
    participant Fastify as خادم Fastify (API Gateway)
    participant RedisQueue as طوابير المهام (BullMQ / Redis)
    participant BackgroundWorker as معالج الطوابير (BullMQ Background Worker)
    participant DB as قاعدة البيانات (PostgreSQL)

    Note over Worker,WApp: العمل بدون اتصال (Offline Mode) في أحياء المنصورة
    Worker->>WApp: إتمام عمليات وزن متعددة أثناء انقطاع شبكة الهاتف
    WApp->>WApp: حفظ السجلات محلياً في SQLite مع التوقيع الرقمي للدفعة الميدانية

    Note over WApp,Fastify: استعادة الاتصال بالإنترنت
    WApp->>Fastify: POST /v1/collection/batch-sync { batchReference, recordsCount, records: [...] }
    
    activate Fastify
    Fastify->>DB: تسجيل الدفعة في field_batch_imports بحالة RECEIVED
    Fastify->>RedisQueue: إضافة الدفعة إلى طابور BullMQ: queue.add('process-field-batch', { batchId })
    Fastify-->>WApp: 202 Accepted { batchReference, status: "QUEUED_FOR_PROCESSING" }
    deactivate Fastify
    
    WApp->>WApp: تحديث حالة السجلات المحلية إلى SYNC_PENDING_CONFIRMATION

    Note over RedisQueue,BackgroundWorker: المعالجة غير المتزامنة خلف الكواليس
    BackgroundWorker->>RedisQueue: سحب مهمة الدفعة الميدانية
    BackgroundWorker->>DB: UPDATE field_batch_imports SET status = 'PROCESSING'
    
    loop لكل عملية وزن داخل الدفعة
        BackgroundWorker->>DB: BEGIN TRANSACTION
        BackgroundWorker->>DB: التحقق من عدم التكرار (Idempotency Key Check)
        BackgroundWorker->>DB: إدراج سجل بنود وأوزان الجمع المعتمدة (collection_items)
        BackgroundWorker->>DB: حساب ونقل النقاط الخضراء في green_points_ledger
        BackgroundWorker->>DB: COMMIT TRANSACTION
    end
    
    BackgroundWorker->>DB: UPDATE field_batch_imports SET status = 'COMPLETED', error_summary = NULL
    BackgroundWorker->>RedisQueue: جدولة إشعارات المستفيدين في طابور الإشعارات (Push Notifications Queue)
```

### مزايا هذا النمط:
1. **استجابة فورية فائقة السرعة (Under 30ms):** يرد الخادم برمز `202 Accepted` مباشرة دون حظر تطبيق العامل أثناء تفريغ مئات الأوزان.
2. **عزل الأعطال:** إذا تعطلت معالجة قيد محاسبي لأحد الأوزان، تُعزل السجلات الفاشلة وتُسجل في `error_summary` دون إفشال باقي عمليات الدفعة.
3. **مرونة التوسع (Horizontal Scalability):** يمكن تشغيل عدة حاويات من `BullMQ Background Worker` لمعالجة دفعات عشرات الشاحنات المتزامنة دون المساس بسرعة تصفح المواطنين للنظام.
