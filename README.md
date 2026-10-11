# 🌿 نظام «أثر» للإدارة الذكية للنفايات والتحفيز المجتمعي
### ATHAR — Smart Waste Management & Community Incentivization Platform
#### العقدة التشغيلية الأولى: مديرية المنصورة — العاصمة عدن (Node 1: Al-Mansoura, Aden)

---

<div align="center">

[![Node.js Version](https://img.shields.io/badge/Node.js-v20_LTS-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-3178C6?logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Framework](https://img.shields.io/badge/Fastify-v4.x-000000?logo=fastify&logoColor=white)](https://fastify.dev/)
[![Database](https://img.shields.io/badge/PostgreSQL-16_PostGIS_3.4-4169E1?logo=postgresql&logoColor=white)](https://postgis.net/)
[![Cache & Queue](https://img.shields.io/badge/Redis_7-BullMQ-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![Frontend Mobile](https://img.shields.io/badge/Flutter-3.x-02569B?logo=flutter&logoColor=white)](https://flutter.dev/)
[![Frontend Admin](https://img.shields.io/badge/React_18-Vite_SPA-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![Docker Ready](https://img.shields.io/badge/Docker-Multi--Stage-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![License](https://img.shields.io/badge/License-Proprietary_/_Gov-blue.svg)](#)

<p align="center">
  <b>منظومة رقمية متكاملة لتحويل فرز النفايات المنزلية إلى حافز اقتصادي وتنموي ملموس عبر إنترنت الأشياء (IoT)، الحسابات المزدوجة المتوازنة، ومطابقة النطاقات الجغرافية اللحظية.</b>
</p>

</div>

---

## 📑 الفهرس العام للمستند (Table of Contents)

1. [نظرة عامة على المشروع (Project Overview)](#-نظرة-عامة-على-المشروع-project-overview)
2. [مخطط معمارية النظام العام (System Architecture)](#-مخطط-معمارية-النظام-العام-system-architecture)
3. [مخطط العلاقات وقاعدة البيانات (Entity Relationship Diagram - ERD)](#-مخطط-العلاقات-وقاعدة-البيانات-entity-relationship-diagram---erd)
4. [مخطط التتابع الزمني للمصادقة والجمع الميداني (Sequence Flow)](#-مخطط-التتابع-الزمني-للمصادقة-والجمع-الميداني-sequence-flow)
5. [مخطط آلة الحالة لدورة حياة الطلب (State Machine Diagram)](#-مخطط-آلة-الحالة-لدورة-حياة-الطلب-state-machine-diagram)
6. [مخطط التدفق لمنطق كشف الشذوذ والأوزان (Business Logic Flowchart)](#-مخطط-التدفق-لمنطق-كشف-الشذوذ-والأوزان-business-logic-flowchart)
7. [دليل التوثيق الهندسي التفصيلي (Detailed Documentation Matrix)](#-دليل-التوثيق-الهندسي-التفصيلي-detailed-documentation-matrix)
8. [دليل التشغيل السريع للمطورين (Quick Start Guide)](#-دليل-التشغيل-السريع-للمطورين-quick-start-guide)

---

## 🌍 نظرة عامة على المشروع (Project Overview)

يعالج مشروع **«أثر»** تحديات تكدس النفايات الصلبة وضعف فرزها من المصدر في مديرية المنصورة (عدن)، من خلال حلقة مغلقة تربط بين:
1. **المواطنين (Residents):** فرز النفايات من المنازل وطلب الاستلام عبر تطبيق الجوال وكسب «نقاط أثر الخضراء» (Green Points).
2. **عمال النظافة والأسطول الميداني (Field Fleet):** جمع وفرز الشحنات ووزنها ميدانياً بدقة فائقة باستخدام موازين ذكية مقترنة عبر الـ Bluetooth.
3. **لوحة التحكم المركزية للبلدية (Back-Office Admin Portal):** غرفة قيادة شاملة لإدارة التوجيه الآلي، وتتبع الأسطول جغرافياً، والرقابة على الأوزان، وإدارة الشكاوى، والتدقيق المالي.
4. **الشركاء والمتاجر المحلية (Merchant Network):** استبدال النقاط المكتسبة بسلع استهلاكية وخصومات حقيقية عبر قسائم رقمية مشفرة (QR Codes).

---

## 🏛️ مخطط معمارية النظام العام (System Architecture)

يوضح المخطط التالي الترابط بين واجهات المستخدم، بوابات الحماية، خوادم **Fastify** المنظمة كـ **Modular Monolith**، طبقة المعالجة الخلفية، وقواعد البيانات:

```mermaid
flowchart TB
    subgraph Clients ["طبقة العملاء والواجهات (Client Layer)"]
        ResApp["📱 تطبيق المواطن (Flutter Mobile)"]
        WorkApp["🚴 تطبيق عامل الجمع (Flutter Mobile)"]
        MerchApp["🏪 بوابة التاجر ونقاط البيع (Flutter / Web)"]
        AdminPortal["🖥️ لوحة تحكم إدارة البلدية (React Vite SPA)"]
    end

    subgraph Gateway ["طبقة الأمان والموازنة (Ingress & Reverse Proxy)"]
        Nginx["🛡️ Nginx Reverse Proxy & WAF<br/>(TLS 1.3 Termination | Rate Limiter)"]
    end

    subgraph BackendCore ["النواة البرمجية (Fastify Modular Monolith - Node.js 20 LTS)"]
        AuthMod["🔑 وحدة الهوية والمصادقة (Auth & Sessions)"]
        DispatchMod["🗺️ وحدة التوجيه والعمليات (Dispatch & Fleet)"]
        WeighMod["⚖️ وحدة الأوزان والتحقق (Weighing & Verification)"]
        LedgerMod["💰 وحدة دفتر الأستاذ (Double-Entry Ledger)"]
        StoreMod["🎁 وحدة متجر المكافآت (Store & Vouchers)"]
        TicketMod["🎫 وحدة خدمة العملاء (Support & CRM)"]
        AuditMod["📋 وحدة الرقابة والتدقيق (Immutable Audit)"]
    end

    subgraph AsyncProcessing ["طبقة المعالجة الخلفية غير المتزامنة (Async & Background Workers)"]
        OutboxWorker["📤 مشغل صندوق الصادر (Transactional Outbox Worker)"]
        AnalyticsWorker["📊 مشغل التقارير الإحصائية (Cron Analytics)"]
    end

    subgraph DataStorage ["طبقة البيانات والتخزين (Data & Cache Layer)"]
        Postgres[("🐘 PostgreSQL 16 + PostGIS 3.4<br/>(Spatial Data & Financial ACID)")]
        RedisDB[("⚡ Redis 7 Alpine<br/>(Sessions, Locks & BullMQ Queues)")]
    end

    subgraph ExternalGateways ["الخدمات الخارجية (External Integrations)"]
        SMS["✉️ Yemen SMS Gateway (OTP)"]
        FCM["🔔 Firebase Cloud Messaging (Push Notifications)"]
        BLE["📶 Bluetooth Smart Scales (IoT Devices)"]
    end

    %% اتصالات الواجهات
    ResApp -->|HTTPS / REST| Nginx
    WorkApp -->|HTTPS / REST| Nginx
    MerchApp -->|HTTPS / REST| Nginx
    AdminPortal -->|HTTPS / WSS| Nginx
    WorkApp -.->|BLE Pairing| BLE

    %% التوجيه للخدمات
    Nginx --> AuthMod
    Nginx --> DispatchMod
    Nginx --> WeighMod
    Nginx --> LedgerMod
    Nginx --> StoreMod
    Nginx --> TicketMod
    Nginx --> AuditMod

    %% اتصالات البيانات
    BackendCore -->|Prisma / Kysely Queries| Postgres
    BackendCore -->|Distributed Locks & Caching| RedisDB

    %% معالجة صندوق الصادر
    BackendCore -.->|Write Event| Postgres
    OutboxWorker -->|Poll FOR UPDATE SKIP LOCKED| Postgres
    OutboxWorker --> SMS
    OutboxWorker --> FCM
```

> **الأساس المنطقي المعماري:**
> تم اختيار نمط **Modular Monolith** مع Node.js و TypeScript لتفادي التعقيد التشغيلي وشبكات الخدمات المصغرة (Microservices) في المرحلة الأولى، مع عزل النطاقات برمجياً. يتم تفريغ العمليات الطويلة (SMS, Push) عبر **Transactional Outbox Pattern** لضمان استجابة سريعة للـ API بأقل من 50 ميلي ثانية.

---

## 🗄️ مخطط العلاقات وقاعدة البيانات (Entity Relationship Diagram - ERD)

يوضح المخطط الكيانات المحورية، المفاتيح الأساسية والأجنبية، وضوابط القيد المزدوج:

```mermaid
erDiagram
    users ||--o{ auth_sessions : "يمتلك جلسات"
    users ||--o| admin_staff_profiles : "يمتلك ملف موظف"
    users ||--o{ household_addresses : "يسجل عناوين"
    users ||--o| ledger_accounts : "يمتلك حساب نقاط"
    users ||--o{ service_requests : "يرفع طلبات"
    users ||--o{ worker_tasks : "يُكلف بمهام"
    users ||--o| merchants : "يدير متجر شريك"
    users ||--o{ support_tickets : "يفتح بلاغات"

    geofence_zones ||--o{ household_addresses : "يحتوي منازل"
    geofence_zones ||--o{ service_requests : "يحدد نطاق الطلب"

    waste_categories ||--o{ collection_weighings : "يحدد تسعيرة النقاط"

    service_requests ||--o| worker_tasks : "يولد مهمة ميدانية"
    service_requests ||--o{ collection_weighings : "يسجل أوزان"
    service_requests ||--o{ support_tickets : "يرتبط بشكوى"

    merchants ||--o{ store_items : "يعرض سلع"
    store_items ||--o{ store_orders : "يُشترى في طلبات"

    ledger_accounts ||--o{ green_points_ledger : "حساب دائن (Source)"
    ledger_accounts ||--o{ green_points_ledger : "حساب مدين (Destination)"

    users {
        uuid id PK
        varchar phone_bindex UK "فهرس أعمى مشفر"
        text encrypted_phone "رقم الهاتف مشفر AES"
        varchar full_name
        varchar role "RESIDENT | WORKER | ADMIN"
        varchar status "ACTIVE | SUSPENDED"
    }

    geofence_zones {
        uuid id PK
        varchar code UK "MANSOURA-BLK-04"
        geometry boundary "Polygon SRID 4326"
        varchar status "ACTIVE"
    }

    service_requests {
        uuid id PK
        varchar tracking_number UK
        uuid resident_id FK
        uuid zone_id FK
        varchar status "SUBMITTED | WEIGHED | COMPLETED"
        varchar idempotency_key UK
    }

    collection_weighings {
        uuid id PK
        uuid request_id FK
        uuid category_id FK
        decimal gross_weight_kg
        decimal tare_weight_kg
        decimal net_weight_kg "محسوب آلياً"
        boolean anomaly_flag "مؤشر الشذوذ والاحتيال"
    }

    ledger_accounts {
        uuid id PK
        uuid user_id FK
        varchar account_type "SYSTEM_MINT | HOUSEHOLD | ESCROW"
        bigint balance "الرصيد الدقيق"
    }

    green_points_ledger {
        uuid id PK
        uuid transaction_ref
        uuid source_account_id FK
        uuid dest_account_id FK
        bigint amount "القيد المتوازن"
        varchar operation_type
        varchar idempotency_key UK
    }
```

> **الأساس المنطقي لقاعدة البيانات:**
> الاعتماد الكامل على `UUIDv7` للترتيب الزمني وتفادي تجزئة الفهارس، مع تطبيق امتداد `PostGIS` لفهرسة النطاقات الجغرافية مكانياً عبر مؤشرات `GiST`، وفصل حسابات النقاط في دفتر أستاذ متوازن غير قابل للتعديل (`Append-Only`).

---

## ⏱️ مخطط التتابع الزمني للمصادقة والجمع الميداني (Sequence Flow)

يوثق هذا المخطط التفاعل التبادلي أثناء تسجيل الوزن الميداني وصرف النقاط للمواطن:

```mermaid
sequenceDiagram
    autonumber
    actor Resident as المواطن
    actor Worker as عامل النظافة
    participant Scale as الميزان الذكي (BLE)
    participant WApp as تطبيق العامل
    participant API as خادم Fastify API
    participant DB as قاعدة بيانات PostgreSQL
    participant Outbox as صندوق الصادر (outbox_messages)
    participant WorkerJob as معالج الأحداث (BullMQ)
    participant Push as إشعارات الهاتف (FCM)

    Worker->>Scale: وزن النفايات عبر البلوتوث
    Scale-->>WApp: قراءة الوزن الصافي المعتمد (12.500 kg)
    Worker->>WApp: إرسال الوزن مع كود الطلب
    WApp->>API: POST /v1/worker/collection/weigh { requestId, netWeight, scaleId }
    
    activate API
    API->>API: التحقق من مفتاح عدم التكرار (Idempotency-Key)
    API->>API: فحص خوارزمية كشف الشذوذ (Anti-Fraud Check)
    
    API->>DB: BEGIN TRANSACTION (معاملة مصرفية آمنة)
    API->>DB: حفظ سجل الوزن في collection_weighings
    API->>DB: تحديث حالة الطلب إلى WEIGHED
    API->>DB: حساب النقاط: 12.5 كجم × 15 نقطة = 187 GP
    API->>DB: قيد محاسبي مزدوج متوازن في green_points_ledger
    API->>DB: إدراج حدث الإشعار في outbox_messages
    API->>DB: COMMIT TRANSACTION
    deactivate API
    
    API-->>WApp: 201 Created { status: "COMPLETED", pointsEarned: 187 }
    
    Note over Outbox,WorkerJob: تفريغ صندوق الصادر دون تأخير الـ API
    WorkerJob->>Outbox: استعلام عن الرسائل المعلقة (SELECT SKIP LOCKED)
    WorkerJob->>Push: إرسال إشعار فوري لهاتف المواطن
    Push-->>Resident: إشعار: تم إيداع 187 نقطة خضراء لحسابك بنجاح! 🌿
    WorkerJob->>Outbox: تحديث حالة الرسالة إلى PUBLISHED
```

---

## 🔄 مخطط آلة الحالة لدورة حياة الطلب (State Machine Diagram)

دورة حياة طلب الخدمة المنزلي (`service_requests`) محكومة بآلة حالة محددة تمنع أي انتقال غير مصرح به:

```mermaid
stateDiagram-v2
    [*] --> SUBMITTED: إنشاء الطلب من تطبيق المواطن
    
    SUBMITTED --> DISPATCHED: توجيه الطلب إلى عامل أو دراجة جمع
    SUBMITTED --> REJECTED: رفض الطلب (خارج النطاق الجغرافي)
    SUBMITTED --> CANCELLED: إلغاء الطلب من قِبل المواطن
    
    DISPATCHED --> IN_PROGRESS: استلام العامل للمهمة والتحرك للموقع
    DISPATCHED --> CANCELLED: إلغاء من الإدارة لظرف طارئ
    
    IN_PROGRESS --> WEIGHED: إتمام عملية الوزن بالميزان الذكي
    IN_PROGRESS --> FAILED: تعذر الوصول للعنوان أو عدم الرد
    
    WEIGHED --> COMPLETED: مطابقة الأوزان وصرف النقاط بنجاح
    WEIGHED --> DISPUTED: اعتراض المواطن أو رصد شذوذ وزني آلي
    
    DISPUTED --> COMPLETED: تسوية النزاع واعتماد وزن معدل من المشرف
    DISPUTED --> REJECTED: إلغاء الوزن بالكامل وإغلاق الطلب
    
    FAILED --> SUBMITTED: إعادة الجدولة لوردية قادمة
    
    COMPLETED --> [*]
    REJECTED --> [*]
    CANCELLED --> [*]
```

---

## ⚡ مخطط التدفق لمنطق كشف الشذوذ والأوزان (Business Logic Flowchart)

تمثيل خوارزمية مكافحة الاحتيال وفحص الأوزان الميدانية قبل الصرف المالي للنقاط:

```mermaid
flowchart TD
    WeightInput(["قراءة الميزان الميداني: Net Weight (kg)"]) --> CheckZero{"هل الوزن الصافي > 0 ؟"}
    
    CheckZero -- "لا (وزن غير صالح)" --> RejectZero(["رفض العملية: 422 Invalid Weight"])
    CheckZero -- "نعم" --> CheckAbsolute{"هل الوزن يتجاوز سقف الخامة الأقصى؟<br/>(مثال: بلاستيك > 50 كجم للمنزل)"}
    
    CheckAbsolute -- "نعم (تجاوز السقف)" --> FlagAnomaly["رصد شذوذ قطعي: Anomaly_Flag = TRUE"]
    CheckAbsolute -- "لا" --> CheckHouseholdBaseline{"هل الوزن > (المتوسط التاريخي للأسرة × 3.5)؟"}
    
    CheckHouseholdBaseline -- "نعم (قفزة شاذة)" --> FlagAnomaly
    CheckHouseholdBaseline -- "لا (وزن طبيعي)" --> Approve["اعتماد الوزن: Anomaly_Flag = FALSE"]
    
    FlagAnomaly --> HoldState["تجميد رصيد النقاط وتحويل الطلب للمشرف الرقابي"]
    Approve --> MintLedger["صرف النقاط فوراً في دفتر الأستاذ (Double-Entry Mint)"]
```

---

## 📚 دليل التوثيق الهندسي التفصيلي (Detailed Documentation Matrix)

تم تنظيم التوثيق التقني للمشروع في مسارات متخصصة لتسهيل القراءة والتنفيذ:

| المسار التشعيبي (Document Link) | موضوع الوثيقة | أبرز المحتويات والمخططات |
| :--- | :--- | :--- |
| 📘 [نظرة عامة على المعمارية](docs/01-architecture/system-overview.md) | معمارية النظام ومبادئ التصميم | مخططات C4 (Level 1 & Level 2)، نمط Outbox، ومواصفات SLOs. |
| 🖥️ [لوحة تحكم الإدارة المركزية](docs/01-architecture/back-office-portal.md) | المنصة الإدارية للموظفين | مصفوفة RBAC، غرف القيادة الست، والتصميم الهيكلي للواجهة. |
| 🗃️ [مخطط العلاقات والكيانات (ERD)](docs/02-database/erd.md) | بنية قاعدة البيانات الكاملة | تفصيل الـ 28 جدولاً، سياقات النطاقات، وفهارس PostGIS المكانية. |
| 📖 [قاموس بيانات قاعدة البيانات](docs/02-database/data-dictionary.md) | معجم الجداول والحقول | أنواع الحقول، القيود، الشروط، والقيم الافتراضية لكل جدول. |
| ⏱️ [مخططات التتابع الزمني للعمليات](docs/03-flows-and-logic/sequences.md) | تسلسل تدفق البيانات | تسجيل الدخول بالـ OTP، الجمع الميداني، ودورة حياة القسائم. |
| 🔄 [آلات الحالة المحددة (FSM)](docs/03-flows-and-logic/state-machines.md) | دورات حياة الكيانات | مصفوفات الانتقال، حالات طلبات الخدمة، المهام، والتذاكر. |
| 🧠 [منطق الأعمال والخوارزميات](docs/03-flows-and-logic/business-logic.md) | الخوارزميات الحسابية | فحص PostGIS `ST_Covers`، كشف الشذوذ، ومفاتيح منع التكرار. |
| 🔌 [مواصفات وعقود الـ APIs](docs/04-specifications/api-contracts.md) | مواصفات الـ REST Endpoints | عقود الـ Mobile والـ Worker والـ Admin وصيغ أخطاء RFC 7807. |
| 🛡️ [الأمان والامتثال والتحكم بالوصول](docs/04-specifications/security-and-compliance.md) | النموذج الأمني والتشفير | تحليل STRIDE، تشفير AES-256، الفهرسة العمياء، والمصادقة الثنائية. |
| 🗺️ [خارطة طريق المشروع (Roadmap)](docs/04-specifications/roadmap.md) | الجدول الزمني ومراحل الإطلاق | مخطط جانت (Gantt Chart)، معالم المشروع، وخطة تجربة حي ريمي. |
| 🚀 [النشر والعمليات والبنية التحتية](docs/04-specifications/deployment-and-ops.md) | أدلة الـ DevOps والتشغيل | ملف Dockerfile متعدد المراحل، docker-compose، وفحوصات الصحة. |

---

## 🚀 دليل التشغيل السريع للمطورين (Quick Start Guide)

### 1. المتطلبات المسبقة (Prerequisites)
* **Node.js:** الإصدار `v20.x LTS` أو أحدث.
* **مدير الحزم:** `pnpm` (الإصدار `9.x`).
* **Docker & Docker Compose:** لتشغيل قواعد البيانات والخدمات المساعدة محلياً.

### 2. استنساخ المستودع وتهيئة المتغيرات
```bash
# 1. استنساخ المستودع
git clone https://github.com/your-org/athar-platform.git
cd athar-platform

# 2. إنشاء ملف متغيرات البيئة
cp .env.example .env
```

### 3. تشغيل الخدمات الأساسية عبر Docker
```bash
# تشغيل PostgreSQL (مع PostGIS) وخادم Redis
docker compose up -d postgres redis

# التحقق من صحة عمل الحاويات
docker compose ps
```

### 4. تثبيت الحزم وتطبيق الترحيلات (Migrations & Seed)
```bash
# تثبيت الاعتماديات
pnpm install

# توليد عميل Prisma أو تشغيل ترحيلات SQL
pnpm db:migrate

# تغذية قاعدة البيانات بالبيانات الافتراضية (نطاقات المنصورة وخامات النفايات)
pnpm db:seed
```

### 5. تشغيل خادم التطوير ومشغل الصادر
```bash
# تشغيل خادم Fastify في وضع التطوير المباشر
pnpm dev

# في نافذة طرفية ثانية: تشغيل مشغل صندوق الصادر
pnpm worker:outbox
```

### 6. التحقق من سلامة النظام (Health Check)
```bash
curl http://localhost:3000/health/ready
# الاستجابة المتوقعة: {"status":"UP","checks":{"database":"UP","redis":"UP"}}
```

---

<div align="center">
  <sub>صُمم وطُوّر بمعايير هندسية عالمية لخدمة البيئة والمجتمع في العاصمة عدن 🇾🇪</sub>
</div>
