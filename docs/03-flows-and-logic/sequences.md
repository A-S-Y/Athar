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
            Gateway->>DB: إنشاء سجل جديد في users + فتح حساب نقاط ledger_accounts
        end
        Gateway->>Gateway: توليد Access Token (15 دقيقة) + Refresh Token (30 يوماً)
        Gateway->>DB: حفظ Refresh Token Hash في auth_sessions
        Gateway-->>App: 200 OK { accessToken, refreshToken, userProfile }
        App->>App: تخزين الرموز بأمان في Flutter Secure Storage
    end
```

### القرارات المعمارية المطبقة:
* **تخزين الرموز المؤقتة في الذاكرة (Redis TTL):** تنتهي صلاحية الرمز تلقائياً بعد 180 ثانية، ويُحذف فور التحقق لمنع هجمات الإعادة (Replay Attacks).
* **الفهرسة العمياء للخصوصية:** لا يُبحث عن رقم الهاتف كنص صريح بل عبر قيمة `phone_bindex` المجزأة بتشفير HMAC لضمان سرية هوية المواطنين.

---

## 2. دورة حياة الجمع الميداني، الوزن الذكي، وضبط الرصيد (Smart Field Collection & Weighing Flow)

توثق هذه العملية كيفية إتمام استلام النفايات من قبل عامل النظافة الميداني، والاتصال بالميزان الذكي عبر Bluetooth، والتحقق التلقائي من الشذوذ، وتسجيل النقاط بالقيد المزدوج المضمون.

```mermaid
sequenceDiagram
    autonumber
    actor Resident as المواطن
    actor Worker as عامل التجميع الميداني
    participant Scale as الميزان الذكي (BLE Scale)
    participant WApp as تطبيق العامل (Worker App)
    participant API as خادم Fastify API
    participant DB as قاعدة البيانات (PostgreSQL)
    participant Outbox as صندوق الصادر (outbox_messages)
    participant WorkerQueue as عمال المعالجة (BullMQ Worker)
    participant Push as خدمة الإشعارات (Firebase FCM)

    Worker->>WApp: تأكيد الوصول لموقع المنزل (At Location)
    WApp->>API: PATCH /v1/tasks/:id/arrive { coordinates }
    API->>DB: تحديث حالة المهمة إلى AT_LOCATION
    
    Worker->>Scale: تشغيل وقراءة الميزان الرقمي عبر Bluetooth
    Scale-->>WApp: إرسال الوزن الإجمالي عبر بروتوكول BLE (Gross: 14.250 kg)
    
    Worker->>WApp: اختيار الخامة (بلاستيك PET) وتأكيد الوزن الصافي
    WApp->>API: POST /v1/collection/weigh { requestId, categoryId, gross, tare, scaleId }
    
    activate API
    API->>API: حساب صافي الوزن: Net = 14.250 - 0.250 = 14.000 kg
    API->>API: خوارزمية كشف الشذوذ (فحص السقف اليومي والوزن المفرط)
    
    alt رصد وزن شاذ أو مريب (> 50 كجم للمنزل الواحد)
        API->>DB: تسجيل الوزن مع تفعيل anomaly_flag = TRUE
        API->>DB: إنشاء تنبيه رقابي لمدير العمليات في لوحة التحكم
    else الوزن سليم ومطابق
        API->>DB: BEGIN TRANSACTION (بدء معاملة بنكية صارمة)
        API->>DB: حفظ سجل collection_weighings
        API->>DB: تحديث حالة service_requests إلى WEIGHED
        API->>DB: حساب النقاط المكتسبة: 14 كجم × 15 نقطة = 210 GP
        API->>DB: قيد محاسبي مزدوج: من SYSTEM_MINT إلى HOUSEHOLD_ACCOUNT
        API->>DB: إدراج حدث PointsMintedEvent في outbox_messages
        API->>DB: COMMIT TRANSACTION (حفظ تام لا رجعة فيه)
    end
    deactivate API
    
    API-->>WApp: 201 Created { netWeight: 14.0, pointsEarned: 210, status: "SUCCESS" }
    
    Note over Outbox,WorkerQueue: معالجة الأحداث بشكل غير متزامن (Transactional Outbox)
    WorkerQueue->>Outbox: استعلام عن الرسائل المعلقة (SELECT FOR UPDATE SKIP LOCKED)
    WorkerQueue->>Push: إرسال إشعار فوري للمواطن (تم إيداع 210 نقطة خضراء لحسابك!)
    Push-->>Resident: ظهور الإشعار على هاتف المواطن مع رنين صوتي
    WorkerQueue->>Outbox: تحديث حالة الرسالة إلى PUBLISHED
```

---

## 3. شراء واستبدال قسائم المكافآت الرقمية (Store Voucher Generation & POS Redemption)

يوضح هذا المخطط كيفية استبدال النقاط المكتسبة بقسائم خصم رقمية لدى المتاجر والشركاء في حي المنصورة بنظام الحجز المعلق (Escrow).

```mermaid
sequenceDiagram
    autonumber
    actor Resident as المواطن المستفيد
    actor Merchant as التاجر (نقطة البيع POS)
    participant ResApp as تطبيق المواطن
    participant MerchApp as بوابة التاجر (Merchant Portal)
    participant API as خادم Fastify API
    participant DB as قاعدة البيانات (PostgreSQL)

    Resident->>ResApp: تصفح المتجر واختيار قسيمة تسوق (500 نقطة)
    ResApp->>API: POST /v1/store/orders { itemId, idempotencyKey }
    
    activate API
    API->>DB: BEGIN TRANSACTION
    API->>DB: قفل سجل المخزون: SELECT stock FROM store_items WHERE id = ? FOR UPDATE
    alt نفاد المخزون
        API-->>ResApp: 409 Conflict (السلعة نفدت من المخزون)
    else المخزون متاح
        API->>DB: فحص رصيد المواطن (balance >= 500)
        alt الرصيد غير كافٍ
            API-->>ResApp: 400 Bad Request (رصيد النقاط غير كافٍ)
        else الرصيد كافٍ
            API->>DB: تقليل المخزون بمقدار 1
            API->>DB: قيد محاسبي: تحويل 500 GP من حساب المواطن إلى MERCHANT_ESCROW
            API->>DB: توليد كود قسيمة فريد مشفر بصيغة QR Code
            API->>DB: إنشاء سجل في store_orders بحالة VOUCHER_GENERATED
            API->>DB: COMMIT TRANSACTION
            API-->>ResApp: 201 Created { voucherCode, qrData, expiresAt }
        end
    end
    deactivate API
    
    Note over Resident,Merchant: المواطن يتوجه إلى متجر الشريك في المنصورة
    Resident->>Merchant: إبراز رمز الاستجابة السريعة (QR Code) على الهاتف
    Merchant->>MerchApp: مسح الرمز بواسطة كاميرا جهاز الـ POS
    MerchApp->>API: POST /v1/merchant/vouchers/redeem { voucherCode }
    
    activate API
    API->>DB: BEGIN TRANSACTION
    API->>DB: استعلام القسيمة مع القفل: SELECT * FROM store_orders FOR UPDATE
    alt القسيمة مستخدمة سابقاً أو منتهية
        API-->>MerchApp: 422 Unprocessable (القسيمة غير صالحة أو مستخدمة)
    else القسيمة صالحة
        API->>DB: تحديث حالة القسيمة إلى REDEEMED وتوثيق وقت الصرف
        API->>DB: تسوية الحساب: تحويل 500 GP من حساب الحجز المؤقت ESCROW إلى حساب التاجر المستحق
        API->>DB: COMMIT TRANSACTION
        API-->>MerchApp: 200 OK { status: "REDEEMED_SUCCESSFULLY", merchantCreditYer: 2500 }
    end
    deactivate API
    
    MerchApp-->>Merchant: تأكيد شاشة الكاشير بالخصم الفوري للسلعة
    Merchant-->>Resident: تسليم المواد الغذائية / السلع للمواطن
```

---

## 4. مشغل صندوق الصادر وضمان تسليم الإشعارات (Transactional Outbox Dispatcher Worker)

يوضح هذا المخطط الآلية المعمارية التي تضمن وصول الرسائل والإشعارات الميدانية دون أي احتمالية لفقدان البيانات حتى لو انقطعت شبكة الإنترنت أو تعطل مزود الرسائل الخارجي.

```mermaid
sequenceDiagram
    autonumber
    participant Fastify as خادم Fastify (API Context)
    participant OutboxTable as جدول outbox_messages (Postgres)
    participant CronWorker as المعالج الخلفي (BullMQ Outbox Worker)
    participant ExternalAPI as مزود الخدمة الخارجي (FCM / SMS Gateway)
    participant DLQ as طابور الرسائل الميتة (Dead-Letter Queue)

    Note over Fastify,OutboxTable: عملية أعمال تمت بنجاح داخل معاملة ACID
    Fastify->>OutboxTable: INSERT INTO outbox_messages (event_type, payload, status: 'PENDING')
    
    loop كل ثانية واحدة (Polling with Backoff)
        CronWorker->>OutboxTable: SELECT * FROM outbox_messages WHERE status = 'PENDING' ORDER BY scheduled_at LIMIT 50 FOR UPDATE SKIP LOCKED
        OutboxTable-->>CronWorker: قائمة بالرسائل المستحقة للإرسال
        
        loop لكل رسالة في الحزمة
            CronWorker->>OutboxTable: UPDATE outbox_messages SET status = 'PROCESSING'
            CronWorker->>ExternalAPI: إرسال الطلب عبر شبكة الإنترنت (HTTPS POST)
            
            alt نجاح الإرسال (HTTP 200 OK)
                CronWorker->>OutboxTable: UPDATE outbox_messages SET status = 'PUBLISHED', published_at = NOW()
            else فشل مؤقت في الشبكة (Timeout / HTTP 503)
                alt عدد المحاولات أقل من 5
                    CronWorker->>OutboxTable: UPDATE outbox_messages SET status = 'PENDING', retry_count = retry_count + 1, scheduled_at = NOW() + (2 ^ retry_count * 5s)
                else استنفاد الحد الأقصى للمحاولات (5 محاولات)
                    CronWorker->>OutboxTable: UPDATE outbox_messages SET status = 'FAILED', last_error = 'Max retries exhausted'
                    CronWorker->>DLQ: نقل الرسالة لطابور التدخل اليدوي DLQ
                    CronWorker->>CronWorker: إطلاق تنبيه في Pino Logger بمستوى ERROR
                end
            end
        end
    end
```

### مزايا هذا النمط:
1. **استقلالية الـ API:** لا ينتظر خادم الـ API استجابة مزود الرسائل الخارجي، مما يجعل وقت الاستجابة للمستخدمين أقل من 40 ميلي ثانية.
2. **منع الإرسال الوهمي:** لا تُرسل أي رسالة إشعار إطلاقاً إذا فشلت المعاملة المالية في قاعدة البيانات، لأن حفظ الرسالة في الصندوق جزء من نفس المعاملة المصرفية التلقائية.
3. **تجنب التنافس (SKIP LOCKED):** يمكن تشغيل 10 نسخ من الـ Workers بالتوازي دون أن تعالج نسختان نفس الرسالة في نفس الوقت.
