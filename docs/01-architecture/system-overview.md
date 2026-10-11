# 🏛️ معمارية النظام والهيكل العام (System Architecture Overview)

مستند التوصيف المعماري الشامل لمنظومة «أثر» لإدارة الفرز واسترداد الموارد الحضرية — **العقدة الأولى: مديرية المنصورة، عدن**.

---

## 1. الرؤية الهندسية والمبادئ المعمارية (Architectural Principles)

تعتمد منظومة «أثر» على معمارية برمجية حديثة توازن بين البساطة التشغيلية، وموثوقية الأداء الميداني، وقابلية التوسع الأفقي المستقبلي، بالاستناد إلى المبادئ الهندسية التالية:

1. **الخادم المرجعي الصارم (Server-Authoritative System):**
   * الخادم هو المصدر الوحيد للحقيقة (`Single Source of Truth`).
   * لا تُحسم أي عملية مالية، أو وزن، أو احتساب نقاط، أو استحقاق إسناد على جانب العميل (Client-side) على الإطلاق.
   * تُجرى كافة العمليات ضمن معاملات ذرية (`ACID Transactions`) مع تطبيق التحقق الصارم من الصلاحيات (`RBAC`).

2. **معمارية المونوليث النمطي (Modular Monolith Architecture):**
   * بناء الخادم الأساسي ككتلة برمجية موحدة عالية الكفاءة بلغة **TypeScript** ومحرك **Node.js** عبر إطار عمل **Fastify**.
   * عزل الوحدات الوظيفية داخل طبقات ونطاقات منطقية مستقلة (`Bounded Contexts`) تتواصل عبر واجهات برمجية وعقود داخلية واضحة (`Internal Contracts & Domain Services`).
   * تملك كل وحدة جداولها ومنطقها الخاص، مما يمهد للانتقال السلس إلى الخدمات المصغرة (`Microservices`) مستقبلاً عند زيادة الحِمل الجغرافي دون الحاجة لإعادة كتابة الكود.

3. **الاتصال المباشر مع مرونة العمل الميداني دون اتصال (Online-First with Offline Resilience):**
   * المسار الطبيعي للعمليات هو الاتصال المباشر بالإنترنت (`Online-First`).
   * في الميدان وأثناء انقطاع تغطية شبكة الهاتف في أزقة المنصورة، يدعم تطبيق العامل التخزين المحلي المشفر للبيانات، مع الترحيل المجمع نهاية الوردية (`End-of-Day Batch Import`) بعد التحقق البرمجي التام وفحص التكرار.

4. **نمط صندوق الصادر الموثوق (Transactional Outbox Pattern):**
   * فصل المهام غير المتزامنة الثقيلة (إرسال رسائل التحقق SMS، الإشعارات الموجهة Push Notifications، مطابقة الدفعات الميدانية) عن تدفق الاستجابة اللحظي للـ API.
   * كتابة الرسائل والأحداث كصفوف في جدول `outbox_messages` ضمن نفس المعاملة الذرية للعملية، ليتولى معالجو المهام الخلفية (`TypeScript Workers`) سحبها وتنفيذها بشكل غير متزامن ومضمون.

5. **المحاسبة المالية المزدوجة للنقاط (Double-Entry Points Accounting):**
   * تُعامل النقاط الخضراء كالتزام مالي حقيقي يستند إلى قيود متوازنة ($\sum Debits = \sum Credits$) بين حساب الإصدار وحساب المواطن وحساب ضمان المتجر وحساب الإتلاف، مما يمنع رياضياً أي تضخم أو تزييف في الأرصدة.

---

## 2. المكدس التقني المعتمد (Technology Stack)

```mermaid
flowchart TD
    subgraph Clients ["طبقة الواجهات والمستخدمين"]
        C1["📱 تطبيق الميدان والمواطن<br/>(Flutter Android)"]
        C2["🌐 بوابة المتجر والمواطن الخفيفة<br/>(Flutter Web / PWA)"]
        C3["💻 بوابة الإدارة والعمليات المركزية<br/>(React + TypeScript SPA)"]
    end

    subgraph Gateway ["طبقة البوابة والتحكم (Fastify Edge)"]
        G1["Fastify Reverse Proxy & Security Headers (Helmet)"]
        G2["Rate Limiting & Anti-Abuse (Redis/Sliding Window)"]
        G3["Zod Schema Validation & OpenAPI Generator"]
        G4["Idempotency Middleware (UUIDv4 Tracking)"]
    end

    subgraph CoreBackend ["الخادم الأساسي (Node.js / TypeScript Core)"]
        M1["وحدة الهوية والصلاحيات (Identity & RBAC)"]
        M2["وحدة النطاق والمكان (Geofence & PostGIS)"]
        M3["وحدة الإسناد والمهام (Task Dispatch)"]
        M4["وحدة الجمع والموازين (Weighing & Intake)"]
        M5["وحدة المحاسبة والنقاط (Double-Entry Ledger)"]
        M6["وحدة المتجر والمخزون (Store & Catalog)"]
        M7["وحدة الشكاوى والدعم (CRM & Tickets)"]
        M8["وحدة التدقيق الشامل (System Audit)"]
    end

    subgraph WorkerPool ["معالجات المهام الخلفية (TypeScript Workers)"]
        W1["Outbox Poller & Dispatcher"]
        W2["SMS & OTP Worker"]
        W3["Push Notifications Worker"]
        W4["10-Min Dispatch Timeout Checker"]
        W5["EOD Batch Reconciliation Worker"]
        W6["Inventory Reservation TTL Cleaner"]
    end

    subgraph DataStore ["قواعد البيانات والتخزين المؤقت"]
        DB[("PostgreSQL 16+<br/>+ PostGIS 3.4+")]
        REDIS[("Redis 7+<br/>(Cache, Rate Limits & BullMQ)")]
        S3[("S3 / MinIO<br/>(Encrypted Object Storage)")]
    end

    Clients --> Gateway
    Gateway --> CoreBackend
    CoreBackend --> DB
    CoreBackend --> REDIS
    CoreBackend -.-> |Transactional Outbox| DB
    WorkerPool <--> DB
    WorkerPool <--> REDIS
    CoreBackend --> S3
```

### تفصيل المكونات التقنية:
| الطبقة | التقنية المعتمدة | مبرر الاختيار التقني |
| :--- | :--- | :--- |
| **الخادم الأساسي (Backend Core)** | **Node.js (LTS v20+) + TypeScript** | سرعة معالجة استثنائية لعمليات الإدخال والإخراج (`Non-blocking I/O`)، وتجانس كامل للكود بين الواجهات والخادم (`Full-Stack TypeScript`). |
| **إطار عمل الـ API** | **Fastify Framework** | سرعة معالجة تزيد بمقدار 2 إلى 4 أضعاف عن Express، تدقيق مدمج للمخططات (`JSON Schema`)، ودعم فوري لتوثيق OpenAPI. |
| **قاعدة البيانات الأساسية** | **PostgreSQL 16+** | أقوى محرك قواعد بيانات علائقية مفتوح المصدر لدعم المعاملات الذرية المعقدة وعزل المستويات المحاسبية. |
| **الامتداد المكاني الجغرافي** | **PostGIS 3.4+** | معيار الصناعة العالمي للاستعلامات المكانية، والتحقق الصارم من احتواء النطاقات الجغرافية (`ST_Covers / ST_Contains`). |
| **محرك الطوابير والـ Workers** | **BullMQ / Pg-Boss** | إدارة قوائم انتظار المهام غير المتزامنة، مع دعم مدمج لإعادة المحاولة الأسية (`Exponential Backoff`) وطوابير الرسائل المتعثرة (`DLQ`). |
| **الذاكرة المؤقتة ومحدد المعدل** | **Redis 7+** | تخزين جلسات العمل المؤقتة، تنفيذ خوارزمية النافذة المنزلقة لتحديد معدل الطلبات، وإدارة أقفال التزامن السريعة. |
| **تطبيقات الموبايل والمواطن** | **Flutter (Dart)** | قاعدة شفرة برمجية موحدة تُصرّف لتطبيق أندرويد ميداني خفيف، وتطبيق ويب تفاعلي سريع. |
| **بوابة الإدارة والعمليات** | **React + TypeScript (Vite)** | تطبيق مكتبي أحادي الصفحة (`SPA`) غني بالمكونات والجداول التفاعلية المتقدمة المخصصة للشاشات الكبيرة. |

---

## 3. مخطط سياق النظام (C4 Model - Level 1: System Context)

يوضح المخطط التالي كيفية تفاعل الفئات المختلفة من المستخدمين والأنظمة الخارجية مع منظومة «أثر»:

```mermaid
flowchart TB
    Citizen["👤 المواطن (Resident)"]
    Worker["👷 العامل الميداني (Field Worker)"]
    Supervisor["👮 المشرف الميداني (Field Supervisor)"]
    Staff["👔 موظف الإدارة والدعم (Back-Office Staff)"]
    Partner["🏬 الشريك والمتجر (Merchant / Partner)"]

    AtharSystem["♻️ منظومة «أثر» المركزية<br/>(Athar Waste Management Platform)"]

    SmsGateway["📱 بوابة الرسائل القصيرة<br/>(Local Telecom SMS Gateway)"]
    FcmService["🔔 خدمات الإشعارات السحابية<br/>(Firebase Cloud Messaging)"]
    MapTileServer["🗺️ خوادم الخرائط الجغرافية<br/>(OpenStreetMap / Vector Tiles)"]

    Citizen --> |"طلب جمع، استبدال نقاط، تتبع"| AtharSystem
    Worker --> |"مسح بطاقات QR، تسجيل أوزان، إقرار مهام"| AtharSystem
    Supervisor --> |"اعتماد الأوزان، مراجعة استثناءات"| AtharSystem
    Staff --> |"مراقبة الإسناد، خدمة العملاء، إدارة المخزون"| AtharSystem
    Partner --> |"صرف السلع، تسليم المواد، التحصيل المالي"| AtharSystem

    AtharSystem --> |"إرسال رموز التحقق OTP وتنبيهات المهام"| SmsGateway
    AtharSystem --> |"إرسال إشعارات المهام والتحديثات"| FcmService
    AtharSystem <--> |"جلب طبقات الخرائط والبيانات المكانية"| MapTileServer
```

---

## 4. مخطط الحاويات البرمجية (C4 Model - Level 2: Container Diagram)

يوضح هذا المخطط الحاويات البرمجية الأساسية التي يتكون منها النظام وبروتوكولات التواصل بينها:

```mermaid
flowchart LR
    subgraph ClientContainers ["حاويات العرض (Frontends)"]
        FlutterApp["📱 Flutter Android App<br/>(Dart)"]
        FlutterWeb["🌐 Flutter Web Store<br/>(Dart / CanvasKit)"]
        AdminSPA["💻 Admin Portal SPA<br/>(React / TypeScript / Vite)"]
    end

    subgraph ApiContainer ["حاوية خادم التطبيقات (Application API)"]
        FastifyApp["⚡ Node.js Fastify API Service<br/>(REST / JSON over HTTPS)"]
    end

    subgraph WorkerContainer ["حاوية معالجة المهام الخلفية (Background Engine)"]
        WorkerService["⚙️ Node.js Background Workers<br/>(BullMQ Engine)"]
    end

    subgraph DataContainers ["حاويات البيانات والتخزين (Persistence)"]
        PostgresDB[("🐘 PostgreSQL 16+<br/>+ PostGIS Extension<br/>(Port 5432)")]
        RedisDB[("⚡ Redis 7+<br/>(Port 6379)")]
        ObjectStore[("📦 S3 / MinIO<br/>(Port 9000 HTTPS)")]
    end

    FlutterApp --> |"REST HTTPS / TLS 1.3"| FastifyApp
    FlutterWeb --> |"REST HTTPS / TLS 1.3"| FastifyApp
    AdminSPA --> |"REST HTTPS / TLS 1.3 (2FA)"| FastifyApp

    FastifyApp --> |"TCP / SQL Connection Pool"| PostgresDB
    FastifyApp --> |"TCP / RESP3 Protocol"| RedisDB
    FastifyApp --> |"AWS SDK / S3 API"| ObjectStore

    WorkerService --> |"TCP / SQL Connection Pool"| PostgresDB
    WorkerService --> |"Job Queue / PubSub"| RedisDB
    WorkerService --> |"HTTPS"| ExternalAPIs["بوابات SMS و FCM"]
```

---

## 5. معمارية صندوق الصادر الموثوق (Transactional Outbox Deep-Dive)

لضمان الاتساق النهائي للبيانات وعدم فقدان أي رسالة نصية أو إشعار عند تعطل الشبكة أو فشل المزود الخارجي، تطبق المنظومة نمط **Transactional Outbox**:

```mermaid
sequenceDiagram
    autonumber
    actor Worker as العامل الميداني
    participant API as خادم التطبيقات (Fastify)
    participant DB as قاعدة البيانات (PostgreSQL)
    participant OutboxWorker as معالج الصندوق الصادر (Worker)
    participant SMS as مزود الرسائل (SMS Gateway)

    Worker->>API: إنهاء زيارة جمع وتثبيت الأوزان
    Note over API,DB: بدء معاملة ذرية موحدة (DB Transaction)
    API->>DB: 1. حفظ سجل الزيارة (CollectionVisit)
    API->>DB: 2. قيد النقاط في دفتر الأستاذ (GreenPointsLedger)
    API->>DB: 3. كتابة رسالة الإشعار في جدول (outbox_messages)
    Note over API,DB: اعتماد المعاملة (COMMIT)
    API-->>Worker: استجابة فورية بنجاح العملية (200 OK)

    loop كل 1-2 ثانية (Worker Polling / Event Trigger)
        OutboxWorker->>DB: استعلام الرسائل المعلقة (status = 'pending' FOR UPDATE SKIP LOCKED)
        OutboxWorker->>SMS: إرسال رسالة SMS للمواطن بالنقاط المكتسبة
        alt نجاح الإرسال
            SMS-->>OutboxWorker: استجابة بالنجاح (Message SID)
            OutboxWorker->>DB: تحديث الحالة إلى 'sent' وتدوين الطابع الزمني
        else فشل الإرسال (خطأ مؤقت)
            SMS-->>OutboxWorker: فشل الاتصال / خطأ الشبكة
            OutboxWorker->>DB: زيادة retry_count وجدولة المحاولة التالية (Exponential Backoff)
        end
    end
```

### مزايا النمط:
1. **الذرية المطلقة:** يستحيل إرسال إشعار بنقاط لم تُحفظ في قاعدة البيانات، ويستحيل حفظ نقاط دون كتابة أمر إشعارها في الصندوق.
2. **استقلالية زمن الاستجابة:** زمن استجابة الـ API للعامل الميداني لا يتأثر إطلاقاً ببطء أو تعطل مزودي خدمة الـ SMS الخارجيين.
3. **التعافي الذاتي:** أي انقطاع مؤقت في بوابات الاتصالات المحلية يتم استدراكه تلقائياً بمجرد عودة الخدمة.

---

## 6. استراتيجية النشر والبيئات (Deployment & Infrastructure Blueprint)

تعتمد المنظومة على الحاويات المعيارية (`Docker Multi-Stage Builds`) لضمان خفة الحجم والأمان العالي في بيئات التشغيل:

```mermaid
flowchart TD
    subgraph Host ["خادم التشغيل الإنتاجي (Production Host / Ubuntu 24.04 LTS)"]
        subgraph Ingress ["طبقة الدخول العكسي (Reverse Proxy)"]
            Nginx["Nginx / Caddy Proxy<br/>(TLS Termination & HTTP/2)"]
        end

        subgraph Containers ["حاويات Docker المعزولة"]
            API_Container["Node.js Fastify API Container<br/>(Non-root Alpine User)"]
            Worker_Container["Node.js Worker Container<br/>(BullMQ Runner)"]
            Postgres_Container["PostgreSQL 16 + PostGIS Container<br/>(Persistent SSD Volume)"]
            Redis_Container["Redis 7 Container<br/>(AOF Persistence)"]
        end
    end

    Internet(("🌐 شبكة الإنترنت")) -->|Port 443 HTTPS| Nginx
    Nginx -->|Proxy Pass 127.0.0.1:3000| API_Container
    API_Container <-->|Internal Docker Network| Postgres_Container
    API_Container <-->|Internal Docker Network| Redis_Container
    Worker_Container <-->|Internal Docker Network| Postgres_Container
    Worker_Container <-->|Internal Docker Network| Redis_Container
```

### مؤشرات ومستويات الخدمة المستهدفة (SLIs / SLOs):
* **زمن استجابة الاستعلامات العادية:** 95% من الطلبات تُعالج خلال أقل من **50 مللي ثانية**.
* **زمن استعلامات PostGIS المكانية:** 99% من اختبارات النطاق الجغرافي تُعالج خلال أقل من **40 مللي ثانية**.
* **جاهزية النظام (Uptime SLO):** 99.9% جاهزية تشغيلية على مدار الشهر.
* **التعافي من الكوارث:** هدف استعادة البيانات (**RPO**) أقل من **15 دقيقة** عبر النسخ الاحتياطي المستمر لملفات WAL (`Write-Ahead Logging`)، وزمن استعادة الخدمة (**RTO**) أقل من **ساعتين**.
