# 03 - المعمارية التقنية الكاملة (System Architecture)

---

## 1. الصورة الكبيرة

```
┌──────────────────────────┐         ┌───────────────────────────────────────────────┐
│   Driver App (Android)    │         │                 BACKEND                         │
│ ───────────────────────── │  HTTPS  │ ┌────────────┐   ┌──────────────┐               │
│ Foreground Service        │ batches │ │ Ingestion  │──▶│ Queue/Stream │               │
│ Fused Location + Activity │────────▶│ │   API      │   │ (Redis/NATS) │               │
│ Motion State Machine      │         │ └────────────┘   └──────┬───────┘               │
│ Local SQLite (outbox)     │         │                         ▼                       │
│ Sync Worker (WorkManager) │◀────────│  config / ack   ┌──────────────────┐            │
│ Local Stop Detector       │         │                 │ Stream Processor │            │
│ Integrity checks          │         │                 │ - Stop detector  │            │
│ SMS fallback (opt.)       │         │                 │ - Rule engine    │            │
└──────────────────────────┘         │                 │ - Trip builder   │            │
                                      │                 └───┬─────────┬────┘            │
                                      │                     ▼         ▼                 │
                                      │  ┌──────────────────────┐ ┌───────────────┐     │
                                      │  │ PostgreSQL+PostGIS   │ │ Notification  │     │
                                      │  │ + TimescaleDB        │ │ Service       │──┐  │
                                      │  └──────────┬───────────┘ └───────────────┘  │  │
                                      │             │   ┌──────────────┐              │  │
                                      │             └──▶│ Watchdog cron │             │  │
                                      │                 │ Nightly jobs  │             │  │
                                      │                 └──────────────┘              │  │
                                      │  ┌──────────────┐  WebSocket                  │  │
                                      │  │ Dashboard API│◀─────────────┐              │  │
                                      └──┴──────────────┴──────────────┼──────────────┼──┘
                                                                       │              │
                                        ┌──────────────────────────────┴───┐   ┌──────┴─────────┐
                                        │ Web Dashboard (React, RTL, Map)  │   │ FCM / Telegram │
                                        │ + Supervisor mobile / PWA        │   │ WhatsApp / SMS │
                                        └──────────────────────────────────┘   └────────────────┘
```

---

## 2. التقنيات المقترحة (Tech Stack) مع البدائل

### 2.1 تطبيق السواق

| الخيار | المميزات | العيوب | التوصية |
|---|---|---|---|
| **Native Kotlin (Android)** | أقصى تحكم في الخلفية، Foreground service، WorkManager، أقل استهلاك | Android بس | ✅ **الأفضل للمرحلة الأولى** |
| Flutter + `flutter_background_geolocation` (Transistorsoft) | كود واحد Android/iOS، المكتبة جاهزة فيها: SQLite offline، motion detection، heartbeat، start on boot، HTTP batch upload | الترخيص مدفوع (~$399 لتطبيق واحد، دايم) | ✅ **بديل ممتاز وأسرع في التنفيذ** لو عاوزين iOS بعدين |
| React Native + نفس مكتبة Transistorsoft | نفس الفكرة | نفس | مقبول |
| Flutter + مكتبات مجانية (geolocator, background_locator, Tracelet) | مجاني | محتاج شغل كتير ومشاكل مع OEMs | ⚠️ مخاطرة |

> **قرار مقترح:** لو الفريق عنده خبرة Kotlin → Native. لو عاوزين سرعة + iOS مستقبلاً → Flutter + Transistorsoft (الـ $399 أرخص بكتير من شهور شغل على حالات الخلفية).

### 2.2 السيرفر

| الطبقة | المقترح | البديل |
|---|---|---|
| لغة/Framework | **Node.js (NestJS)** أو **Go** | Python (FastAPI), Laravel |
| قاعدة البيانات | **PostgreSQL + PostGIS + TimescaleDB** | MySQL (أضعف في الجغرافيا) |
| Cache/حالة لحظية | **Redis** (آخر موقع، حالة كل سواق، تايمرز الوقوف) | — |
| Queue/Stream | **Redis Streams** (بداية) → NATS/Kafka (لما نكبر) | RabbitMQ, BullMQ |
| Realtime للداشبورد | **WebSocket (Socket.IO)** | SSE, MQTT |
| Jobs مجدولة | BullMQ / cron | — |
| إشعارات | **Firebase Cloud Messaging** + Telegram Bot API + WhatsApp Business API + SMS gateway مصري | — |
| خرائط | **Google Maps** (تجربة أفضل في مصر) أو **OpenStreetMap + MapLibre** (مجاني) | Mapbox |
| Map matching | OSRM / Valhalla (مجاني self-hosted) | Google Roads API (~$10 لكل 1000 طلب) |
| Reverse geocoding | Google Geocoding + Cache، أو Nominatim | — |
| الاستضافة | سيرفر داخل مصر أو VPS (عشان قانون نقل البيانات) | AWS/GCP (محتاج تصريح نقل بيانات بره مصر) |

### 2.3 الداشبورد
- **React + TypeScript + Vite**، مكتبة UI بتدعم RTL (Ant Design / MUI).
- خرائط: `@react-google-maps/api` أو `MapLibre GL` + Clustering + Heatmap layer.
- رسوم: ECharts / Recharts.

### 2.4 بديل "Build on open source": Traccar
- **Traccar** سيرفر مفتوح المصدر (Java) فيه جاهز: Devices، Positions، Trips/Stops، Geofences، Notifications، Reports، REST API.
- ممكن نستخدم Traccar كـ **Backend** ونكتب: تطبيق سواق خاص بينا (يبعت بروتوكول OsmAnd/HTTP) + Alert engine إضافي (للوقوف 30 دقيقة) + داشبورد عربي.
- **المميزات:** توفير 40-50% من وقت الـ Backend. **العيوب:** تخصيص محدود، Java، منطق الحضور والانصراف والـ KPI لازم يتبني بره.
- **التوصية:** نبني Backend خاص (الملفات دي مصممة كده) لكن **ننقل منطق Trips/Stops من Traccar** لأنه مجرّب.

---

## 3. نموذج البيانات (Database Schema)

```sql
-- الكيانات الأساسية
CREATE TABLE companies   (id uuid PK, name text, timezone text DEFAULT 'Africa/Cairo', settings jsonb);
CREATE TABLE branches    (id uuid PK, company_id uuid, name text);
CREATE TABLE drivers     (id uuid PK, branch_id uuid, name text, phone text, employee_code text,
                          status text, vehicle_id uuid, created_at timestamptz);
CREATE TABLE devices     (id uuid PK, driver_id uuid, install_id text UNIQUE, model text, os_version text,
                          app_version text, fcm_token text, last_seen_at timestamptz, bound_at timestamptz);
CREATE TABLE shifts      (id uuid PK, branch_id uuid, name text, start_time time, end_time time,
                          days int[], grace_minutes int DEFAULT 15);

-- الحضور والانصراف
CREATE TABLE attendance  (id uuid PK, driver_id uuid, shift_id uuid,
                          check_in_at timestamptz, check_in_loc geography(Point), check_in_photo text,
                          check_out_at timestamptz, check_out_loc geography(Point),
                          source text, -- online / offline_synced
                          status text); -- open / closed / auto_closed

-- النقاط (Hypertable في TimescaleDB)
CREATE TABLE positions (
  device_id    uuid NOT NULL,
  driver_id    uuid NOT NULL,
  attendance_id uuid,
  recorded_at  timestamptz NOT NULL,     -- وقت الموبايل (الأهم)
  received_at  timestamptz NOT NULL DEFAULT now(),
  seq          bigint NOT NULL,           -- رقم تسلسلي من الموبايل
  geom         geography(Point,4326) NOT NULL,
  accuracy_m   real, speed_mps real, bearing real, altitude real,
  activity     text,  -- still / in_vehicle / on_foot
  is_moving    boolean,
  is_mock      boolean,
  battery      smallint, charging boolean,
  provider     text,   -- gps / network / fused
  network      text,   -- none / 2g / 3g / 4g / wifi
  PRIMARY KEY (device_id, seq)            -- Idempotency: نفس النقطة مش هتتكرر
);
SELECT create_hypertable('positions', 'recorded_at');
CREATE INDEX ON positions (driver_id, recorded_at DESC);
CREATE INDEX ON positions USING GIST (geom);

-- الأحداث من الجهاز (heartbeat, gps_off, mock, boot, shutdown...)
CREATE TABLE device_events (id bigserial, device_id uuid, driver_id uuid, type text,
                            recorded_at timestamptz, payload jsonb, seq bigint,
                            UNIQUE(device_id, seq));

-- الرحلات والوقفات (ناتج المعالجة)
CREATE TABLE stops  (id uuid PK, driver_id uuid, attendance_id uuid,
                     start_at timestamptz, end_at timestamptz, duration_s int,
                     center geography(Point), radius_m real,
                     geofence_id uuid NULL, address text,
                     classification text, -- customer / depot / fuel / unauthorized / unknown / home
                     driver_reason text, supervisor_note text,
                     detected_offline boolean);
CREATE TABLE trips  (id uuid PK, driver_id uuid, attendance_id uuid,
                     start_at timestamptz, end_at timestamptz, distance_m real, duration_s int,
                     max_speed real, avg_speed real, start_stop_id uuid, end_stop_id uuid,
                     path geography(LineString), harsh_events int, overspeed_s int);

-- الجيوفينس
CREATE TABLE geofences (id uuid PK, company_id uuid, name text,
                        type text, -- depot / customer / fuel / restricted / operating_area / home
                        area geography(Polygon), max_dwell_min int, is_active boolean);

-- التنبيهات
CREATE TABLE alert_rules (id uuid PK, company_id uuid, type text, params jsonb,
                          severity text, channels text[], escalate_after_min int,
                          applies_to jsonb, active_hours jsonb, enabled boolean);
CREATE TABLE alerts (id uuid PK, rule_id uuid, driver_id uuid, type text, severity text,
                     started_at timestamptz,  -- وقت حدوث المشكلة الحقيقي
                     detected_at timestamptz, -- وقت اكتشاف السيستم
                     resolved_at timestamptz,
                     status text, -- open / acknowledged / resolved / false_positive
                     ack_by uuid, ack_at timestamptz, resolution_reason text,
                     location geography(Point), payload jsonb);

-- ملخصات يومية (Continuous aggregate أو Job ليلي)
CREATE TABLE daily_driver_stats (driver_id uuid, day date, shift_minutes int, moving_minutes int,
       stopped_minutes int, idle_unauthorized_minutes int, offline_minutes int, distance_km real,
       stops_count int, long_stops_count int, customer_visits int, overspeed_count int,
       first_move_at timestamptz, last_move_at timestamptz, alerts_count int, score smallint,
       PRIMARY KEY(driver_id, day));
```

### سياسة الاحتفاظ بالبيانات (Retention)
- `positions` الخام: 90 يوم (بعدها ضغط/أرشفة) — TimescaleDB compression بيوفر ~90% مساحة.
- `trips`, `stops`, `alerts`, `daily_driver_stats`: سنة أو أكتر (حسب السياسة).
- تقدير الحجم: 100 سواق × 10 ساعات × نقطة كل 15 ث ≈ 240 ألف نقطة/يوم ≈ ~50 ميجا/يوم قبل الضغط.

---

## 4. الـ API (أهم الـ Endpoints)

### Driver App API
```
POST /v1/auth/otp/request            {phone}
POST /v1/auth/otp/verify             {phone, code, install_id, device_info} → {access, refresh}
GET  /v1/me/config                   → إعدادات التتبع (intervals, thresholds, shift)
POST /v1/attendance/check-in         {client_id, recorded_at, lat, lng, accuracy, photo?, is_mock}
POST /v1/attendance/check-out        {client_id, recorded_at, lat, lng, ...}
POST /v1/locations/batch             {device_id, items:[{seq, recorded_at, lat, lng, acc, spd, brg,
                                      act, moving, mock, bat, chg, net}...]}  (gzip, حتى 500 نقطة)
     → {ack_up_to_seq: 18234}        // الموبايل يمسح لحد الرقم ده
POST /v1/events/batch                {items:[{seq, type, recorded_at, payload}]}
POST /v1/heartbeat                   {bat, gps_on, perm, net, pending_count, app_ver, last_seq}
PUT  /v1/me/fcm-token
POST /v1/stops/{id}/reason           {reason}
```

### Dashboard API
```
GET  /v1/live/drivers                 حالة كل السواقين (من Redis)
WS   /v1/live                         stream: positions, status changes, alerts
GET  /v1/drivers/{id}/timeline?date=
GET  /v1/drivers/{id}/track?from&to   (simplified polyline)
GET  /v1/alerts?status=open
POST /v1/alerts/{id}/ack              {reason, note}
CRUD /v1/geofences, /v1/rules, /v1/drivers, /v1/shifts, /v1/devices
GET  /v1/reports/{type}?from&to&branch&format=xlsx|pdf
GET  /v1/analytics/...
```

### قواعد مهمة في الـ Ingestion
1. **Idempotent:** `(device_id, seq)` مفتاح فريد → لو الموبايل بعت نفس الدفعة مرتين (النت قطع قبل الـ ack) مفيش تكرار.
2. **Ordering:** المعالجة دايماً بـ `recorded_at`. لو وصلت نقط أقدم من آخر نقطة متعالجة (late data) → تتعمل **reprocessing** للفترة دي.
3. **Clock skew:** الموبايل بيبعت `device_time` مع كل request والسيرفر يحسب الفرق، ولو الفرق كبير (> 2 دقيقة) يصحح أو يستخدم وقت الـ GPS (`location.time` من الأقمار الصناعية دقيق ومش بيتأثر بتغيير السواق لساعة الموبايل).
4. **Validation:** رفض النقط اللي accuracy > 100م من حسابات الوقوف (بس نخزنها)، وفلترة القفزات المستحيلة.

---

## 5. الحالة اللحظية في Redis

```
driver:{id}:state = {
  status: "moving|stopped|offline|off_shift|gps_off",
  last_pos: {lat,lng,acc,at},
  stop_anchor: {lat,lng,since},      // لو واقف: مركز الوقفة وبدايتها
  last_seen_at, battery, gps_on, app_ver,
  attendance_id, shift_end_at
}
ZSET stop_timers   score=since  member=driver_id   // عشان Job كل دقيقة يجيب اللي عدوا 30 دقيقة
ZSET last_seen     score=last_seen_at member=driver_id // للـ Watchdog
```

## 6. الأمان
- JWT قصير العمر + Refresh token مربوط بالجهاز (Device binding) → السواق مايقدرش يسجل من موبايل تاني.
- Certificate pinning في التطبيق.
- تشفير قاعدة البيانات المحلية (SQLCipher).
- Play Integrity API (اختياري) للتأكد إن التطبيق مش متعدل والجهاز مش Rooted.
- RBAC في الداشبورد + Audit log.
- Rate limiting على الـ API.
