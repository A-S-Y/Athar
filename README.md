# 🌿 مشروع «أثر» (ATHAR) — منظومة إدارة وفرز النفايات واسترداد الموارد والتحفيز المجتمعي
### ATHAR — Municipal Waste Sorting, Resource Recovery & Civic Incentivization Platform
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
  <b>منظومة رقمية متكاملة لتحويل فرز النفايات المنزلية إلى حافز اقتصادي وتنموي ملموس عبر بطاقات التعريف المشفرة، الحسابات المزدوجة المتوازنة، ومطابقة النطاقات الجغرافية اللحظية.</b>
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

---

## 🌍 نظرة عامة على المشروع (Project Overview)

يعالج مشروع **«أثر»** تحديات تكدس النفايات الصلبة وضياع قيمتها الاقتصادية في مديرية المنصورة (عدن)، من خلال حلقة تشغيلية مغلقة تربط بين:
1. **المواطنين (Residents):** فرز النفايات عند المصدر في مسارين (عضوي وجاف)، وإبراز بطاقة الـ QR المطبوعة على المنزل، وكسب «نقاط أثر الخضراء» (Green Points).
2. **عمال النظافة والأسطول الميداني (Field Fleet):** زيارات جمع مجدولة، مسح بطاقة المنزل بكاميرا التطبيق، ووزن الأكياس وتدوين أوزان العضوي والجاف والمرفوض مع كود الميزان يدوياً.
3. **لوحة التحكم المركزية للبلدية (Back-Office Admin Portal):** غرفة قيادة شاملة لإدارة خطط السير، والرقابة على الأوزان والاستثناءات، وإدارة متجر أثر، والتدقيق المالي.
4. **متجر أثر الإلكتروني (Two-Step Digital Store):** شراء المنتجات والسلع الداعمة للبيئة بنقاط أثر والريال اليمني عبر نظام الشراء الرقمي بخطوتين (إرفاق إثبات التحويل المالي والتحقق الإداري المباشر).

---

## 🏛️ مخطط معمارية النظام العام (System Architecture)

يوضح المخطط التالي الترابط بين واجهات المستخدم، بوابات الحماية، خوادم **Fastify** المنظمة كـ **Modular Monolith**، طبقة المعالجة الخلفية، وقواعد البيانات:

```mermaid
flowchart TB
    subgraph Clients ["طبقة العملاء والواجهات (Client Layer)"]
        ResApp["📱 تطبيق المواطن (Flutter Mobile & Web)<br/>استعراض الرصيد ومتجر أثر"]
        WorkApp["👷 تطبيق العامل الميداني (Flutter Mobile)<br/>مسح بطاقة QR بالكاميرا وتسجيل الأوزان يدوياً"]
        AdminPortal["🖥️ لوحة تحكم إدارة البلدية (React Vite SPA)<br/>المشرف والعمليات والتدقيق المالي"]
    end

    subgraph Gateway ["طبقة الحماية والـ API (Fastify Gateway)"]
        Edge["Fastify Modular Monolith (TypeScript)<br/>RBAC + OTP Rate Limiting + Idempotency"]
    end

    subgraph Modules ["الوحدات الوظيفية المعتمدة (Domain Modules)"]
        M1["الهوية والجلسات (Identity & Sessions)"]
        M2["إدارة الأسر وبطاقات الـ QR (Households & Cards)"]
        M3["عقدة المنصورة والحدود (Active Node & Scope)"]
        M4["طلبات الخدمة والمهام (Service Requests & Dispatch)"]
        M5["الزيارات والأوزان (Visits: Organic / Dry / Rejected)"]
        M6["دفتر الأستاذ المزدوج (Green Points Ledger)"]
        M7["شركاء الاسترداد ومبيعات الجملة (Resource Partners)"]
        M8["متجر أثر والتحقق المالي (Athar Store & Payment Review)"]
        M9["البلاغات والرقابة والتدقيق (Incidents & Audit Log)"]
    end

    subgraph Workers ["معالجات الطوابير الخلفية (BullMQ / Redis)"]
        W1["إخطارات مهلة الـ 10 دقائق للإسناد (Dispatch Timeout SMS)"]
        W2["إشعار المواطن برصيد النقاط (Points Minted Notification)"]
        W3["معالجة دفعات نهاية اليوم (Field Batch CSV Import)"]
    end

    subgraph DataStore ["قواعد البيانات والتخزين (Persistence Layer)"]
        DB[("🐘 PostgreSQL 16 + PostGIS<br/>25 جدولاً معتمداً")]
        REDIS[("⚡ Redis 7<br/>طوابير BullMQ والتخزين المؤقت")]
        STORAGE[("📁 التخزين المحلي على السيرفر<br/>Contabo Self-Hosted Private Storage")]
    end

    Clients --> Edge
    Edge --> Modules
    Modules --> DB
    Modules --> REDIS
    Modules --> STORAGE
    Modules -.-> |إدراج المهام| REDIS
    Workers <--> REDIS
    Workers <--> DB
```

> **الأساس المنطقي المعماري:**
> تم اختيار نمط **Modular Monolith** مع Node.js و TypeScript لتفادي التعقيد التشغيلي وشبكات الخدمات المصغرة (Microservices)، مع عزل النطاقات برمجياً. يتم تفريغ العمليات غير المتزامنة (SMS, Push) عبر **BullMQ Background Workers** لضمان استجابة سريعة للـ API بأقل من 50 ميلي ثانية.

---

## 🗄️ مخطط العلاقات وقاعدة البيانات (Entity Relationship Diagram - ERD)

يوضح المخطط الكيانات المحورية المعتمدة في النظام، الروابط بينها، وضوابط القيد المزدوج:

```mermaid
erDiagram
    USER_ACCOUNT ||--o{ HOUSEHOLD : "يمتلك / يدير"
    USER_ACCOUNT ||--o{ WORKER_TASK_DISPATCH : "يكلف كعامل"
    USER_ACCOUNT ||--o{ STORE_ORDER : "يشتري كزبون"
    
    HOUSEHOLD ||--o{ HOUSEHOLD_QR_TOKEN : "يحمل بطاقات رمز"
    HOUSEHOLD ||--o{ INTAKE_SURVEY : "يوثق استبيان الضم"
    HOUSEHOLD ||--o{ SERVICE_REQUEST : "يطلب خدمة جمع"
    HOUSEHOLD ||--o{ COLLECTION_VISIT : "يستقبل زيارات جمع"
    
    ACTIVE_NODE ||--o{ SERVICE_REQUEST : "يغطي جغرافياً"
    ACTIVE_NODE ||--o{ REJECTED_SERVICE_ATTEMPT : "يسجل خارج النطاق"
    ACTIVE_NODE ||--o{ DEMAND_WAITLIST : "قائمة رغبات التوسع"
    
    SERVICE_REQUEST ||--o| WORKER_TASK_DISPATCH : "يسند بمهمة"
    WORKER_TASK_DISPATCH ||--o{ COLLECTION_VISIT : "ينفذ ميدانياً"
    
    COLLECTION_VISIT ||--|{ COLLECTION_ITEM : "يتضمن أوزان الفرز"
    COLLECTION_VISIT ||--o| GREEN_POINTS_LEDGER : "ينشئ قيد نقاط معتمد"
    
    WORKER_TASK_DISPATCH ||--o{ FIELD_BATCH_IMPORT : "يرحل عبر دفعة"
    FIELD_BATCH_IMPORT ||--o{ FIELD_BATCH_ROW_ERROR : "يسجل أخطاء الصفوف"
    
    RESOURCE_PARTNER ||--o{ PARTNER_DELIVERY : "يستلم مواد مفروزة"
    MATERIAL_PRICE ||--o{ PARTNER_DELIVERY : "يسعر المواد"
    PARTNER_DELIVERY ||--o{ PARTNER_PAYMENT : "يسدد قيمتها مالياً"
    
    STORE_PRODUCT ||--o{ STORE_ORDER : "يباع كسلعة"
    
    USER_ACCOUNT ||--o{ INCIDENT_REPORT : "يبلغ عن حادث"
    USER_ACCOUNT ||--o{ CANDIDATE_APPLICATION : "يقدم للتوظيف"
    USER_ACCOUNT ||--o{ VOLUNTEER_INTEREST : "يسجل للتطوع"
    USER_ACCOUNT ||--o{ AUDIT_LOG : "يوثق كفاعل"
```

> **الأساس المنطقي لقاعدة البيانات:**
> الاعتماد على نموذج موحد مكون من 25 كياناً معتمداً، مع عزل حسابات النقاط في دفتر أستاذ متوازن غير قابل للتعديل (`Append-Only`) لمنع تضخم النقاط من العدم، وحفظ الحسابات الجغرافية وتطابق العقد ضمن إطار تشغيلي منضبط.

---

## ⏱️ مخطط التتابع الزمني للمصادقة والجمع الميداني (Sequence Flow)

يوثق هذا المخطط التفاعل التبادلي أثناء مسح بطاقة الـ QR، تسجيل الأوزان يدوياً، وصرف النقاط للمواطن:

```mermaid
sequenceDiagram
    autonumber
    actor Worker as العامل الميداني
    actor Supervisor as المشرف الميداني
    participant WApp as تطبيق العامل (Flutter)
    participant API as خادم Fastify API
    participant DB as قاعدة البيانات (PostgreSQL)
    participant Queue as طوابير BullMQ (Redis)
    participant SMS as مزود الرسائل (SMS Gateway)

    Note over Worker,WApp: 1. الوصول للعنوان ومسح بطاقة الأسرة
    Worker->>WApp: فتح كاميرا التطبيق لمسح بطاقة الـ QR المطبوعة على المنزل
    WApp->>API: POST /qr/resolve { qrCode: "MN-48192", taskId }
    API->>DB: فحص البطاقة والمهمة وتطابق العقدة
    DB-->>API: معرّف الأسرة الصالح (Household_ID)
    API-->>WApp: 200 OK (بيانات الأسرة المختصرة بلا خصوصيات)

    Note over Worker,WApp: 2. وزن النفايات وإدخال الأرقام يدوياً
    Worker->>Worker: وزن أكياس العضوي والجاف بالميزان العادي
    Worker->>WApp: إدخال الأوزان يدوياً: عضوي 8.5 كجم، جاف 3.2 كجم، مرفوض 0.4 كجم
    Worker->>WApp: إدخال كود الميزان (SCALE-04) وصورة الدليل إن وجدت
    WApp->>API: POST /visits { householdId, organicKg, dryKg, rejectedKg, scaleRef }

    activate API
    API->>API: فحص الشذوذ الوزني مقارنة بالسقف المحدد
    alt رصد قفزة غير طبيعية أو وزن مشبوه
        API->>DB: حفظ الزيارة بحالة UNDER_REVIEW (تعليق النقاط)
        API-->>WApp: 201 Created { status: "UNDER_REVIEW", message: "بانتظار مراجعة المشرف" }
        Supervisor->>API: POST /visits/:id/approve (اعتماد المشرف بعد التحقق)
    else الأوزان طبيعية وضمن الحدود
        API->>DB: حفظ الزيارة بحالة APPROVED
    end

    Note over API,DB: قيد النقاط: عضوي + جاف فقط (المرفوض 0 نقطة)
    API->>DB: إدراج قيد دائن ومدين في green_points_ledger
    API->>Queue: إضافة مهمة إشعار المواطن بالرصيد المكتسب
    API-->>WApp: 201 Created { status: "APPROVED", pointsEarned: 117 }
    deactivate API

    Queue->>SMS: إرسال رسالة SMS للمواطن بالنقاط المعتمدة
```

---

## 🔄 مخطط آلة الحالة لدورات حياة الكيانات (State Machine Diagram)

دورات حياة الكيانات التشغيلية محكومة بآلات حالة محددة تمنع أي انتقال غير مصرح به:

```mermaid
stateDiagram-v2
    direction LR

    state "دورة حياة طلب الخدمة (ServiceRequest)" as SR {
        [*] --> NEW: تقديم الطلب
        NEW --> PLANNED: جدولة المسار
        PLANNED --> ASSIGNED: تكليف العامل
        ASSIGNED --> IN_PROGRESS: بدء الزيارة
        IN_PROGRESS --> COMPLETED: إتمام الجمع
        IN_PROGRESS --> CANCELLED: تعذر الوصول
        COMPLETED --> [*]
        CANCELLED --> [*]
    }

    state "دورة حياة زيارة الجمع (CollectionVisit)" as CV {
        [*] --> RECORDED: تدوين الأوزان يدوياً
        RECORDED --> APPROVED: أوزان مطابقة
        RECORDED --> UNDER_REVIEW: رصد شذوذ وزني
        UNDER_REVIEW --> APPROVED: اعتماد المشرف
        UNDER_REVIEW --> REJECTED: رفض المشرف
        APPROVED --> [*]: قيد النقاط في الدفتر
        REJECTED --> [*]: لا نقاط للمرفوض
    }

    state "دورة حياة طلب المتجر (StoreOrder)" as SO {
        [*] --> PENDING_VERIFICATION: حجز السلعة ورفع إيصال الحوالة
        PENDING_VERIFICATION --> VERIFIED_PAID: اعتماد السند من الإدارة المالية
        PENDING_VERIFICATION --> REJECTED: رفض السند المالي
        VERIFIED_PAID --> PROCESSING: تجهيز الشحنة بالمستودع
        PROCESSING --> RELEASED: صرف الطلب
        RELEASED --> DELIVERED: تسليم السلعة للمواطن
        REJECTED --> [*]: إلغاء وتحرير الحجز
        DELIVERED --> [*]
    }
```

---

## ⚡ مخطط التدفق لمنطق فرز المسارين واعتماد الأوزان (Business Logic Flowchart)

تمثيل خوارزمية فرز المسارين (عضوي وجاف) وفحص الأوزان الميدانية قبل الصرف المالي للنقاط:

```mermaid
flowchart TD
    Start(["بدء زيارة الجمع الميداني"]) --> ScanQR["مسح رمز البطاقة (QR Card) بكاميرا هاتف العامل"]
    ScanQR --> ResolveCode{"التحقق من كود الأسرة (5-6 خانات)"}
    
    ResolveCode -- "كود غير صالح أو ملغي" --> HaltScan(["إيقاف العملية وإبلاغ المشرف"])
    ResolveCode -- "كود صالح ومطابق" --> ReadScale["وزن الأكياس بالميزان الميداني وتدوين المرجع"]

    ReadScale --> InputWeights["إدخال الأوزان يدوياً في التطبيق:<br/>- وزن العضوي (Organic Kg)<br/>- وزن الجاف (Dry Kg)<br/>- وزن المرفوض (Rejected Kg)"]

    InputWeights --> CheckLimits{"فحص الشذوذ الوزني:<br/>هل الأوزان ضمن سقف الاستهلاك المعتمد؟"}

    CheckLimits -- "تجاوز السقف أو قفزة شاذة" --> FlagReview["حفظ الزيارة بحالة: UNDER_REVIEW<br/>إشعار المشرف الميداني للتحقق خلال 24 ساعة"]
    CheckLimits -- "سليم وضمن الحدود" --> AutoApprove["حفظ الزيارة بحالة: APPROVED"]

    FlagReview --> SupervisorAction{"قرار المشرف الميداني"}
    SupervisorAction -- "رفض الزيارة" --> RejectVisit(["إلغاء الزيارة: REJECTED (بلا نقاط)"])
    SupervisorAction -- "اعتماد أو تعديل" --> AutoApprove

    AutoApprove --> CalcPoints["احتساب النقاط الخضراء:<br/>Points = (Organic × Rate₁) + (Dry × Rate₂)<br/>(المرفوض: صفر نقطة)"]

    CalcPoints --> LedgerEntry["تسجيل قيد متوازن في دفتر الأستاذ (GreenPointsLedger)"]
    LedgerEntry --> EnqueueSMS["جدولة إرسال رسالة SMS للمواطن عبر طابور BullMQ"]
    EnqueueSMS --> EndSuccess(["اكتمال العملية بنجاح"])
```

---

## 📚 دليل التوثيق الهندسي التفصيلي (Detailed Documentation Matrix)

تم تنظيم التوثيق التقني للمشروع في مسارات متخصصة لتسهيل القراءة والتنفيذ:

| المسار التشعيبي (Document Link) | موضوع الوثيقة | أبرز المحتويات والمخططات |
| :--- | :--- | :--- |
| 📘 [نظرة عامة على المعمارية](docs/01-architecture/system-overview.md) | معمارية النظام ومبادئ التصميم | مخططات C4 (Level 1 & Level 2)، معالجة المهام الخلفية، ومواصفات SLOs. |
| 🖥️ [لوحة تحكم الإدارة المركزية](docs/01-architecture/back-office-portal.md) | المنصة الإدارية للموظفين | مصفوفة RBAC، غرف القيادة الست، والتصميم الهيكلي للواجهة. |
| 🗃️ [مخطط العلاقات والكيانات (ERD)](docs/02-database/erd.md) | بنية قاعدة البيانات الكاملة | تفصيل الجداول وسياقات النطاقات ودورة الشراء بخطوتين وفهارس PostGIS المكانية. |
| 📖 [قاموس بيانات قاعدة البيانات](docs/02-database/data-dictionary.md) | معجم الجداول والحقول | أنواع الحقول، القيود، الشروط، والقيم الافتراضية لكل جدول. |
| ⏱️ [مخططات التتابع الزمني للعمليات](docs/03-flows-and-logic/sequences.md) | تسلسل تدفق البيانات | تسجيل الدخول بالـ OTP، الجمع الميداني، ودورة الشراء الرقمي بخطوتين. |
| 🔄 [آلات الحالة المحددة (FSM)](docs/03-flows-and-logic/state-machines.md) | دورات حياة الكيانات | مصفوفات الانتقال، دورة حياة طلب الخدمة، المهام الميدانية، البلاغات، وطلبات المتجر. |
| 🧠 [منطق الأعمال والخوارزميات](docs/03-flows-and-logic/business-logic.md) | الخوارزميات الحسابية | فحص PostGIS `ST_Covers`، كشف الشذوذ، ومفاتيح منع التكرار. |
| 🔌 [مواصفات وعقود الـ APIs](docs/04-specifications/api-contracts.md) | مواصفات الـ REST Endpoints | عقود الـ Mobile والـ Worker والـ Admin وصيغ أخطاء RFC 7807. |
| 🛡️ [الأمان والامتثال والتحكم بالوصول](docs/04-specifications/security-and-compliance.md) | النموذج الأمني والتشفير | تحليل STRIDE، تشفير AES-256، الفهرسة العمياء، والمصادقة الثنائية. |
| 🗺️ [خارطة طريق المشروع (Roadmap)](docs/04-specifications/roadmap.md) | الجدول الزمني ومراحل الإطلاق | مخطط جانت (Gantt Chart)، معالم المشروع، وخطة تجربة حي ريمي. |
| 🚀 [النشر والعمليات والبنية التحتية](docs/04-specifications/deployment-and-ops.md) | أدلة الـ DevOps والتشغيل | ملف Dockerfile متعدد المراحل، docker-compose، وفحوصات الصحة. |

---

<div align="center">
  <sub>صُمم وطُوّر بمعايير هندسية عالمية لخدمة البيئة والمجتمع في العاصمة عدن 🇾🇪</sub>
</div>
