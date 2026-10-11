# قاموس بيانات قاعدة بيانات نظام «أثر» (Data Dictionary)
## منصة إدارة النفايات الذكية والتحفيز المجتمعي — بلدية المنصورة (العقدة التشغيلية الأولى)

---

## 1. نظرة عامة والمعايير الفنية (Schema Standards)

قاعدة بيانات نظام **«أثر»** مبنية على **PostgreSQL 16+** مع امتداد **PostGIS 3.4+** المكاني لتوفير دقة جغرافية فائقة، واتساق تام وفق معايير ACID، ومعالجة آمنة للمعاملات المالية والنقاط عبر القيد المزدوج.

### معايير التسمية والأنواع (Conventions & Types)
* **المعرفات الأساسية (Primary Keys):** من نوع `UUIDv7` لضمان الترتيب الزمني للأرقام وتفادي تشتت الفهارس العنقودية (B-Tree Fragmentations).
* **التوقيت الزمني (Timestamps):** جميع الحقول بصيغة `TIMESTAMPTZ` (توقيت غرينتش الموحد UTC `+00:00`).
* **البيانات المكانية (Spatial Data):** تعتمد نظام الإحداثيات العالمي المرجعي `SRID 4326` (WGS 84).
* **الأمان والخصوصية (PII & Encryption):** تُشفر الهويات وحقول البيانات الحساسة باستخدام `AES-256-GCM` وتُفهرس للبحث بواسطة التعمية العمياء `HMAC-SHA256 Blind Indexing`.
* **الحذف المنطقي (Soft Deletes):** الحقل `deleted_at TIMESTAMPTZ NULL` مفعل على الكيانات القابلة للأرشفة.

---

## 2. النطاق 1: الهوية والمصادقة وإدارة الوصول (Identity & Access Context)

### 2.1 جدول المستخدمين الأساسي (`users`)
يخزن الحسابات الأساسية لجميع أطراف النظام (مواطنون، عمال ميدان، موظفو إدارة ومستودع).

| اسم الحقل | النوع البرمجي | القيود (Constraints) | الوصف والشرح التقني |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY DEFAULT gen_random_uuid()` | المعرف الفريد للمستخدم (UUIDv7) |
| `phone_bindex` | `VARCHAR(64)` | `UNIQUE NOT NULL` | فهرس أعمى مشفر (Blind Index) لرقم الهاتف للبحث السريع |
| `encrypted_phone` | `TEXT` | `NOT NULL` | رقم الهاتف مشفر برمجياً بواسطة AES-256-GCM |
| `full_name` | `VARCHAR(150)` | `NOT NULL` | الاسم الرباعي المعتمد للمستخدم |
| `role` | `user_role_enum` | `NOT NULL DEFAULT 'RESIDENT'` | دور المستخدم: `RESIDENT`, `FIELD_WORKER`, `ADMIN_SUPERVISOR`, `ADMIN_DISPATCHER`, `ADMIN_FINANCE`, `ADMIN_SUPPORT`, `SUPER_ADMIN` |
| `status` | `user_status_enum`| `NOT NULL DEFAULT 'PENDING_VERIFICATION'` | حالة الحساب: `PENDING_VERIFICATION`, `ACTIVE`, `SUSPENDED`, `BANNED` |
| `failed_login_attempts` | `INT` | `DEFAULT 0` | عداد محاولات الدخول الخاطئة لمنع هجمات التخمين |
| `lockout_until` | `TIMESTAMPTZ` | `NULL` | توقيت فك الحظر الزمني في حال تجاوز الحد المسموح |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | وقت إنشاء الحساب |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | آخر تحديث لسجل المستخدم |
| `deleted_at` | `TIMESTAMPTZ` | `NULL` | علامة الحذف المنطقي |

### 2.2 جدول جلسات المصادقة (`auth_sessions`)
تتبع الرموز المميزة وأجهزة تسجيل الدخول لمنع الاستخدام غير المصرح به.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | معرف الجلسة الفريد |
| `user_id` | `UUID` | `REFERENCES users(id) ON DELETE CASCADE` | المعرف المرجعي للمستخدم |
| `refresh_token_hash`| `VARCHAR(64)` | `NOT NULL UNIQUE` | تجزئة SHA-256 لرمز التحديث (Refresh Token) |
| `user_agent` | `VARCHAR(255)` | `NOT NULL` | بصمة المتصفح أو تطبيق الهاتف |
| `ip_address` | `INET` | `NOT NULL` | عنوان بروتوكول الإنترنت لجهاز الاتصال |
| `expires_at` | `TIMESTAMPTZ` | `NOT NULL` | موعد انتهاء صلاحية الجلسة |
| `revoked_at` | `TIMESTAMPTZ` | `NULL` | وقت إلغاء الجلسة اليدوي |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت بدء الجلسة |

### 2.3 جدول ملفات الإدارة والموظفين (`admin_staff_profiles`)
بيانات موظفي لوحة التحكم الداخلية وأقسامهم وصلاحياتهم.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد لملف الموظف |
| `user_id` | `UUID` | `UNIQUE REFERENCES users(id)` | رابط بحساب المستخدم الإداري |
| `department` | `VARCHAR(80)` | `NOT NULL` | القسم (العمليات الميدانية، الدعم الفني، الشؤون المالية، الرقابة) |
| `employee_code` | `VARCHAR(30)` | `UNIQUE NOT NULL` | الرمز الوظيفي الرسمي داخل البلدية |
| `two_factor_secret` | `TEXT` | `NOT NULL` | المفتاح السري المشفر لـ TOTP 2FA الإلزامي للإدارة |
| `is_2fa_enabled` | `BOOLEAN` | `DEFAULT TRUE` | حالة تفعيل التحقق بخطوتين |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت الإنشاء |

---

## 3. النطاق 2: الإحداثيات المكانية والنطاقات الجغرافية (Geofencing & Spatial Context)

### 3.1 جدول النطاقات الجغرافية المعتمدة (`geofence_zones`)
تحديد حدود المربعات السكنية والشوارع الرسمية لحي المنصورة.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للنطاق الجغرافي |
| `name` | `VARCHAR(100)` | `NOT NULL` | اسم المنطقة (مثلاً: ريمي، القاهرة، عبد العزيز، السنافر) |
| `code` | `VARCHAR(20)` | `UNIQUE NOT NULL` | الرمز التشغيلي للنطاق (مثال: `MANSOURA-BLK-04`) |
| `boundary` | `GEOMETRY(Polygon, 4326)` | `NOT NULL` | المضلع الجغرافي المغلق لحدود المنطقة |
| `centroid` | `GEOMETRY(Point, 4326)` | `NOT NULL` | المركز الهندسي المحسوب للمنطقة لتوجيه الخرائط |
| `status` | `zone_status_enum`| `DEFAULT 'ACTIVE'` | حالة التغطية: `ACTIVE`, `INACTIVE`, `PLANNED` |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت التسجيل |

> [!NOTE]
> يتم تطبيق فهرس مكاني متقدم:
> `CREATE INDEX idx_geofence_boundary ON geofence_zones USING GIST (boundary);`

### 3.2 جدول مواقع المنازل والمشتركين (`household_addresses`)
العناوين الجغرافية الدقيقة للمنازل ونقاط التجميع السكنية.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للعنوان |
| `user_id` | `UUID` | `REFERENCES users(id)` | المستخدم صاحب العقار أو الشقة |
| `zone_id` | `UUID` | `REFERENCES geofence_zones(id)`| المنطقة التشغيلية المعتمدة |
| `coordinates` | `GEOMETRY(Point, 4326)` | `NOT NULL` | إحداثيات GPS الدقيقة للمنزل |
| `street_name` | `VARCHAR(150)` | `NOT NULL` | اسم الشارع أو المعلم القريب |
| `building_number`| `VARCHAR(50)` | `NOT NULL` | رقم العقار / العمارة |
| `floor_number` | `VARCHAR(20)` | `NULL` | رقم الطابق أو الشقة |
| `is_verified` | `BOOLEAN` | `DEFAULT FALSE` | التحقق الميداني من صحة العنوان بواسطة المنسق |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت التسجيل |

---

## 4. النطاق 3: العمليات الميدانية والوزن والنفايات (Waste Operations & Weighing)

### 4.1 جدول مسارات الفرز المعتمدة وأسعار النقاط (`material_prices` / `waste_categories`)
وفق الوثيقة التشغيلية المعتمدة لمشروع أثر: مساران فقط لفرز المصدر (عضوي وجاف)، بالإضافة إلى المرفوض الذي لا يمنح أي نقاط.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للمسار |
| `stream_type` | `waste_stream_enum`| `NOT NULL UNIQUE` | مسار الفرز: `ORGANIC` (عضوي)، `DRY` (جاف)، `REJECTED` (مرفوض) |
| `name_ar` | `VARCHAR(80)` | `NOT NULL UNIQUE` | اسم المسار بالعربية (عضوي، جاف، مرفوض) |
| `points_per_kg` | `INT` | `NOT NULL CHECK (points_per_kg >= 0)` | عدد النقاط الخضراء الممنوحة لكل كيلوغرام صافٍ (المرفوض = 0 نقاط) |
| `max_threshold_kg`| `DECIMAL(8,2)`| `NOT NULL DEFAULT 50.00` | الحد الأقصى المتوقع للوزنة الواحدة في الزيارة لرصد الشذوذ |
| `is_active` | `BOOLEAN` | `DEFAULT TRUE` | حالة تفعيل استلام المسار في العقدة التشغيلية |

### 4.2 جدول طلبات استلام وفرز النفايات (`service_requests`)
سجل طلبات الجمع المنزلي المقدمة من التطبيق المحمول.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للطلب |
| `tracking_number`| `VARCHAR(30)` | `UNIQUE NOT NULL` | كود التتبع التسلسلي التنازلي المقروء للمستخدم |
| `resident_id` | `UUID` | `REFERENCES users(id)` | المستخدم مقدم الطلب |
| `address_id` | `UUID` | `REFERENCES household_addresses(id)`| عنوان التسليم |
| `zone_id` | `UUID` | `REFERENCES geofence_zones(id)` | النطاق التشغيلي المشتق تلقائياً |
| `status` | `service_status_enum`| `NOT NULL DEFAULT 'SUBMITTED'` | الحالات: `SUBMITTED`, `DISPATCHED`, `IN_PROGRESS`, `WEIGHED`, `COMPLETED`, `REJECTED`, `DISPUTED` |
| `estimated_kg` | `DECIMAL(6,2)` | `NULL` | الوزن التقديري المدخل بواسطة المواطن |
| `scheduled_for` | `TIMESTAMPTZ` | `NOT NULL` | نافذة التجميع الزمنية المفضلة |
| `idempotency_key`| `VARCHAR(64)` | `UNIQUE NOT NULL` | مفتاح تجنب التكرار من جهاز الهاتف |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | توقيت رفع الطلب |

### 4.3 جدول مهام التكليف الميداني (`worker_tasks`)
توزيع العمليات الميدانية على أسطول الدراجات وعمال الجمع.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للمهمة الميدانية |
| `request_id` | `UUID` | `REFERENCES service_requests(id)` | رابط بطلب الجمع |
| `worker_id` | `UUID` | `REFERENCES users(id)` | عامل الجمع المكلف |
| `status` | `task_status_enum`| `NOT NULL DEFAULT 'ASSIGNED'` | الحالات: `ASSIGNED`, `ACKNOWLEDGED`, `EN_ROUTE`, `AT_LOCATION`, `COMPLETED`, `FAILED` |
| `assigned_by` | `UUID` | `REFERENCES users(id)` | الموجه التشغيلي أو النظام الآلي |
| `dispatched_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت الإرسال |
| `acknowledged_at`| `TIMESTAMPTZ`| `NULL` | وقت استلام العامل للإشعار (مهلة 10 دقائق قبل إعادة التكليف) |
| `arrived_at` | `TIMESTAMPTZ` | `NULL` | وقت الوصول لموقع المواطن |
| `completed_at` | `TIMESTAMPTZ` | `NULL` | وقت إتمام المهمة بنجاح |

### 4.4 جدول بنود وأوزان الجمع الميداني (`collection_items`)
تسجيل قياسات وأوزان الجمع الميداني المدخلة يدوياً لكل مسار (عضوي، جاف، مرفوض) مع تسجيل رقم الميزان وصورة الإثبات.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد لبند الوزن |
| `visit_id` | `UUID` | `REFERENCES collection_visits(id)` | زيارة الجمع الميدانية المعنية |
| `stream_type` | `waste_stream_enum`| `NOT NULL` | مسار الفرز: `ORGANIC` (عضوي), `DRY` (جاف), `REJECTED` (مرفوض - 0 نقاط) |
| `net_weight_kg` | `DECIMAL(8,3)` | `NOT NULL CHECK (net_weight_kg >= 0)` | الوزن الصافي المدخل ميدانياً من قبل العامل |
| `scale_reference`| `VARCHAR(60)` | `NOT NULL` | المعرف أو الرقم التسلسلي للميزان الميداني المستخدم |
| `evidence_reference`| `VARCHAR(255)`| `NULL` | المعرف المرجعي أو رابط صورة الإثبات التوثيقية |
| `anomaly_flag` | `BOOLEAN` | `DEFAULT FALSE` | مؤشر وجود انحراف أو شذوذ غير طبيعي في الوزن |
| `anomaly_reason` | `VARCHAR(255)` | `NULL` | تفاصيل رصد الشذوذ (تجاوز سقف الوزن التقديري للمسار) |
| `points_awarded`| `INT` | `NOT NULL DEFAULT 0` | النقاط الممنوحة (0 للمرفوض، ومحسوبة للعضوي والجاف) |
| `recorded_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | لحظة القراءة والتسجيل الميداني |

---

## 5. النطاق 4: القيد المزدوج ومحفظة النقاط (Gamification & Double-Entry Ledger)

### 5.1 جدول حسابات الأستاذ العام (`ledger_accounts`)
حسابات دفتر الأستاذ المنضبطة وفق معايير المحاسبة المزدوجة لمنع تضخم النقاط من العدم.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | معرف الحساب المالي |
| `user_id` | `UUID` | `NULL REFERENCES users(id)` | المالك (مواطن، ويكون NULL لحسابات النظام) |
| `account_type` | `account_type_enum`| `NOT NULL` | النوع: `SYSTEM_MINT`, `HOUSEHOLD`, `SYSTEM_BURN` |
| `currency` | `VARCHAR(10)` | `NOT NULL DEFAULT 'GP'` | العملة: Green Points (نقاط خضراء) |
| `balance` | `BIGINT` | `NOT NULL DEFAULT 0` | الرصيد الحالي المتزامن (محمي بتشديد المعاملات) |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت فتح الحساب |

### 5.2 جدول قيود المعاملات المتوازنة (`green_points_ledger`)
دفتر اليومية غير القابل للحذف أو التعديل (Append-Only Balanced Ledger).

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | معرف القيد المحاسبي |
| `transaction_ref`| `UUID` | `NOT NULL` | معرف المعاملة المركبة المشترك بين طرفي القيد |
| `source_account_id`| `UUID`| `REFERENCES ledger_accounts(id)` | الحساب الدائن / المصدر (الذي خرجت منه النقاط) |
| `dest_account_id` | `UUID`| `REFERENCES ledger_accounts(id)` | الحساب المدين / الوجهة (الذي استلم النقاط) |
| `amount` | `BIGINT` | `NOT NULL CHECK (amount > 0)` | كمية النقاط المنقولة (موجبة دائماً) |
| `operation_type` | `operation_type_enum`| `NOT NULL` | نوع العملية: `MINT_RECYCLING`, `REDEEM_STORE_PRODUCT`, `EXPIRE_BURN`, `ADMIN_ADJUSTMENT` |
| `related_entity_id`| `UUID`| `NULL` | مرجع الكيان المرتبط (معرف الوزن، معرف طلب المتجر، إلخ) |
| `idempotency_key` | `VARCHAR(64)` | `UNIQUE NOT NULL` | منع تكرار العمليات الحسابية |
| `notes` | `VARCHAR(255)` | `NULL` | بيان توضيحي للمعاملة |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | وقت تسجيل القيد بدقة |

---

## 6. النطاق 5: المتجر الرقمي والطلبات (Digital Store & Two-Step Orders)

### 6.1 جدول منتجات المتجر (`store_products`)
السلع والمنتجات الصديقة للبيئة المتاحة للاستبدال أو الشراء الداخلي في مديرية المنصورة.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للمنتج |
| `title_ar` | `VARCHAR(150)` | `NOT NULL` | عنوان السلعة بالعربية |
| `title_en` | `VARCHAR(150)` | `NOT NULL` | العنوان بالإنجليزية |
| `points_price` | `INT` | `NOT NULL CHECK (points_price > 0)` | التكلفة بالنقاط الخضراء |
| `cash_price_yer` | `DECIMAL(12,2)` | `NOT NULL DEFAULT 0.00` | السعر النقدي بالريال اليمني (في حال الشراء المزدوج) |
| `stock_quantity`| `INT` | `NOT NULL CHECK (stock_quantity >= 0)` | الكمية المتاحة في المخزون (محمية من الـ Race Condition) |
| `category` | `store_category_enum`| `NOT NULL` | التصنيف: `GROCERY`, `EDUCATION`, `SOLAR_ENERGY`, `SERVICES` |
| `is_available` | `BOOLEAN` | `DEFAULT TRUE` | حالة الإتاحة للعرض والشراء |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | تاريخ الإضافة |

### 6.2 جدول طلبات الشراء والتحقق الثنائي (`store_orders`)
توثيق عمليات الشراء والتحقق الإداري من سند التحويل البنكي أو النقدي.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY` | المعرف الفريد للطلب |
| `order_number` | `VARCHAR(30)` | `UNIQUE NOT NULL` | رقم الطلب المرجعي |
| `buyer_id` | `UUID` | `REFERENCES users(id)` | المواطن المشتري |
| `product_id` | `UUID` | `REFERENCES store_products(id)` | السلعة المطلوبة |
| `quantity` | `INT` | `NOT NULL CHECK (quantity > 0)` | الكمية المطلوبة |
| `points_spent` | `INT` | `NOT NULL DEFAULT 0` | إجمالي النقاط المحسومة |
| `cash_amount_yer`| `DECIMAL(12,2)` | `NOT NULL DEFAULT 0.00` | المبلغ النقدي المطلوب (إن وجد) |
| `payment_method` | `payment_method_enum`| `NOT NULL` | طريقة الدفع: `POINTS_ONLY`, `BANK_TRANSFER`, `CASH_ON_DELIVERY` |
| `proof_image_url`| `VARCHAR(500)` | `NULL` | رابط صورة إشعار الحوالة البنكية / السند المرفوع |
| `transfer_reference`| `VARCHAR(100)`| `NULL` | رقم الإشعار أو الحوالة البنكية المدخل من المشتري |
| `status` | `order_status_enum` | `NOT NULL DEFAULT 'PENDING_VERIFICATION'` | الحالات: `PENDING_VERIFICATION`, `VERIFIED_PAID`, `FULFILLED`, `REJECTED` |
| `rejection_reason`| `VARCHAR(255)`| `NULL` | سبب رفض الإشعار المالي إن وجد |
| `verified_by_staff_id`| `UUID` | `NULL REFERENCES users(id)` | الموظف الإداري الذي دقق سند التحويل |
| `verified_at` | `TIMESTAMPTZ` | `NULL` | وقت التحقق والاعتماد المالي |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL` | وقت تقديم الطلب |

---

## 7. النطاق 6: الحوادث والبلاغات الميدانية (Incidents & Operational Governance)

### 7.1 جدول بلاغات الميدان والحوادث التشغيلية (`incident_reports`)
توثيق بلاغات الحوادث الميدانية ونزاعات الأوزان أو تعثر الجمع، ومراجعتها واعتمادها من المشرفين خلال 24 ساعة.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY DEFAULT gen_random_uuid()` | المعرف الفريد للبلاغ |
| `incident_type` | `incident_type_enum`| `NOT NULL` | التصنيف: `WEIGHT_ANOMALY`, `RESIDENT_DISPUTE`, `COLLECTION_FAILURE`, `SAFETY_VIOLATION`, `OTHER` |
| `household_id` | `UUID` | `NULL REFERENCES households(id)` | الأسرة المعنية بالنزاع أو البلاغ |
| `visit_id` | `UUID` | `NULL REFERENCES collection_visits(id)`| زيارة الجمع الميدانية المرتبطة |
| `reported_by_worker_id`| `UUID`| `NULL REFERENCES users(id)` | العامل الميداني رافع البلاغ |
| `description` | `TEXT` | `NOT NULL` | تفاصيل البلاغ والملاحظات الميدانية |
| `photo_evidence_url`| `VARCHAR(500)`| `NULL` | رابط صورة الإثبات الميدانية |
| `status` | `incident_status_enum`| `NOT NULL DEFAULT 'OPEN'` | الحالات: `OPEN`, `INVESTIGATING`, `RESOLVED`, `REJECTED` |
| `resolution_notes`| `TEXT` | `NULL` | ملاحظات وتوجيه المشرف الإداري لحل البلاغ |
| `resolved_by_supervisor_id`| `UUID`| `NULL REFERENCES users(id)`| المشرف الإداري المعتمد للحل |
| `resolved_at` | `TIMESTAMPTZ` | `NULL` | وقت إغلاق واعتماد الحل (خلال 24 ساعة) |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | وقت تسجيل البلاغ |

---

## 8. النطاق 7: سجل التدقيق والرقابة (Audit Trail & Governance)

### 8.1 جدول سجل الرقابة الصارم (`audit_logs`)
جدول تراكمي دائم (Append-Only Log) لا يخضع لأي تعديل أو حذف مطلقا لحفظ الشفافية.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `BIGSERIAL` | `PRIMARY KEY` | معرف تسلسلي ضخم غير منقطع |
| `actor_id` | `UUID` | `NULL REFERENCES users(id)` | الموظف أو المستخدم الذي نفذ الإجراء (أو النظام) |
| `action` | `VARCHAR(100)` | `NOT NULL` | اسم العملية (مثلاً: `ADMIN_ZONE_OVERRIDE`, `POINTS_FORCE_ADJUST`, `STORE_ORDER_VERIFIED`) |
| `entity_type` | `VARCHAR(60)` | `NOT NULL` | الكيان المتأثر (`service_requests`, `ledger_accounts`, `store_orders` إلخ) |
| `entity_id` | `UUID` | `NOT NULL` | معرف السجل المتأثر |
| `before_state` | `JSONB` | `NULL` | لقطة البيانات الكاملة قبل التعديل |
| `after_state` | `JSONB` | `NOT NULL` | لقطة البيانات الكاملة بعد التعديل |
| `ip_address` | `INET` | `NOT NULL` | عنوان الـ IP لمصدر العملية |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | الطابع الزمني الصارم للحظة التعديل |

---

## 9. النطاق 8: المزامنة الميدانية وطوابير الخلفية (Field Batch Sync & Background Workers)

### 9.1 جدول استيراد الدفعات الميدانية (`field_batch_imports`)
توثيق مزامنة العمليات غير المتصلة (Offline Sync) من تطبيق جامع النفايات الميداني عبر Fastify وطوابير BullMQ.

| اسم الحقل | النوع البرمجي | القيود | الوصف |
| :--- | :--- | :--- | :--- |
| `id` | `UUID` | `PRIMARY KEY DEFAULT gen_random_uuid()` | معرف دفعة المزامنة الميدانية |
| `collector_id` | `UUID` | `REFERENCES users(id)` | جامع النفايات منفذ الدفعة |
| `batch_reference` | `VARCHAR(60)` | `UNIQUE NOT NULL` | الرقم المرجعي للدفعة من جهاز الميدان |
| `records_count` | `INT` | `NOT NULL CHECK (records_count > 0)` | إجمالي السجلات المضمنة بالدفعة |
| `status` | `batch_status_enum` | `NOT NULL DEFAULT 'RECEIVED'` | الحالات: `RECEIVED`, `PROCESSING`, `COMPLETED`, `FAILED` |
| `error_summary` | `TEXT` | `NULL` | تفاصيل الأخطاء في حال تعثر بعض السجلات |
| `synced_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT clock_timestamp()` | وقت استقبال الدفعة ومعالجتها |

---

## 10. خطة الفهارس وتحسين الأداء (Indexing & Partitioning Strategy)

```sql
-- 1. فهرس مكاني GiST لتسريع مطابقة النطاقات الجغرافية
CREATE INDEX idx_geofence_boundary ON geofence_zones USING GIST (boundary);
CREATE INDEX idx_household_coords ON household_addresses USING GIST (coordinates);

-- 2. فهرس طلبات المتجر للتدقيق المالي السريع
CREATE INDEX idx_store_orders_status ON store_orders (status, created_at)
WHERE status = 'PENDING_VERIFICATION';

-- 3. فهارس دفتر الأستاذ للبحث التاريخي والتدقيق المالي
CREATE INDEX idx_ledger_accounts ON green_points_ledger (source_account_id, dest_account_id);
CREATE INDEX idx_ledger_idempotency ON green_points_ledger (idempotency_key);

-- 4. تجزئة جدول سجلات التدقيق سنوياً (Range Partitioning by Year)
-- CREATE TABLE audit_logs_2026 PARTITION OF audit_logs
-- FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```
