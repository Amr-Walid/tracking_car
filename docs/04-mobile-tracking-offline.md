# 04 - تطبيق السواق: التتبع في الخلفية + التخزين الأوفلاين + توفير البطارية

> ده أصعب وأهم جزء في المشروع كله. 80% من مشاكل أنظمة التتبع بالموبايل بتيجي من هنا: الموبايل بيقتل التطبيق، البطارية بتخلص، النت بيقطع، أو السواق بيلعب في الإعدادات.

---

## 1. تجربة السواق (UX) — بسيطة جداً

```
[أول مرة فقط]
  تسجيل دخول (رقم + OTP)  →  Wizard الصلاحيات (5 خطوات)  →  اختبار التتبع ✅

[كل يوم]
  ┌──────────────────────────┐
  │   صباح الخير يا أحمد      │
  │   وردية: 8:00 - 5:00      │
  │                            │
  │   [ 🟢 تسجيل حضور ]        │   ← ضغطة واحدة
  └──────────────────────────┘
          ↓ (التتبع اشتغل في الخلفية، إشعار ثابت "الوردية شغالة")
  ┌──────────────────────────┐
  │   ⏱ في الوردية من 3:12 س  │
  │   ✅ التتبع شغال           │
  │   📶 آخر مزامنة: منذ 1 د   │
  │   📦 نقاط بانتظار الرفع: 0  │
  │                            │
  │   [ 🔴 تسجيل انصراف ]      │
  └──────────────────────────┘
```
مفيش أي زرار تاني. مفيش Pause. مفيش خريطة (مش محتاجها السواق).

---

## 2. الصلاحيات والإعدادات المطلوبة على Android

| الصلاحية/الإعداد | ليه | ملاحظات |
|---|---|---|
| `ACCESS_FINE_LOCATION` | GPS دقيق | |
| `ACCESS_BACKGROUND_LOCATION` ("Allow all the time") | التتبع والشاشة مقفولة | لازم تتطلب **بعد** الـ Fine، وAndroid 11+ بيودي السواق للإعدادات يختارها يدوي |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_LOCATION` | Android 14+ إجباري تحدد نوع الـ service = `location` | |
| `POST_NOTIFICATIONS` | Android 13+ عشان الإشعار الثابت | |
| `ACTIVITY_RECOGNITION` | معرفة (واقف/راكب عربية/ماشي) | |
| `RECEIVE_BOOT_COMPLETED` | يرجع يشتغل بعد Restart | |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | استثناء من Doze | **مسموح** لتطبيقات التتبع، بس لازم نبرره لجوجل بلاي |
| `SCHEDULE_EXACT_ALARM` (اختياري) | Watchdog محلي | |
| **إعدادات الشركة المصنعة** (Autostart في Xiaomi/Oppo/Vivo/Realme، "Never sleeping apps" في Samsung، App launch في Huawei) | دي اللي بتقتل التطبيقات فعلاً | لازم Wizard بخطوات مصورة لكل ماركة — مرجع: dontkillmyapp.com |

### نشر التطبيق
- **الخيار 1 (أسهل وأقوى):** توزيع داخلي (APK/MDM) مش على Google Play → مفيش مراجعة سياسة Background Location.
- **الخيار 2:** Google Play → لازم **Permissions Declaration Form** + فيديو يوضح الاستخدام + Privacy policy + إظهار "Prominent disclosure" قبل طلب الصلاحية. التطبيقات الخاصة بالموظفين بتتقبل عادي لو موضح.
- **الخيار 3 (للتحكم الكامل):** موبايلات الشركة + **Android Enterprise / MDM** (مثل Google Endpoint management, Hexnode, Scalefusion) → تقدر **تمنع** السواق يمسح التطبيق، يقفل الـ GPS، أو يشيل الصلاحيات، وتدي Battery whitelist من بعيد. **ده أقوى حل للتحكم** لو الموبايلات ملك الشركة.

---

## 3. Motion State Machine (السر بتاع البطارية)

```
                  ┌───────────────────────────────────────┐
                  │                                       │
   check-in ──▶ [ACTIVE_MOVING] ──(stillness ≥ 3 min)──▶ [STATIONARY] 
                  ▲  GPS high accuracy                    │  GPS off / low power
                  │  every 10-15s or 20-50m               │  Geofence 100-150m حوالين المكان
                  │                                       │  Heartbeat كل 5 د (balanced/low power)
                  │                                       │  Activity recognition شغال
                  └──(exit geofence OR IN_VEHICLE/ON_FOOT  │
                      OR accelerometer motion)  ◀──────────┘

   check-out / shift end ──▶ [OFF] (كل حاجة مقفولة غير الـ config sync)
```

### إعدادات مقترحة (قابلة للتعديل من السيرفر)
| البارامتر | قيمة | التفسير |
|---|---|---|
| moving.interval | 10 ثانية | |
| moving.distanceFilter | 25 متر (ديناميكي: يزيد مع السرعة) | على الطريق السريع نقطة كل 100م كفاية |
| moving.priority | HIGH_ACCURACY | |
| stopTimeout | 3 دقايق | مدة السكون عشان يتحول STATIONARY (مش هيعتبر إشارة المرور وقفة) |
| stationary.radius | 150 متر | |
| stationary.heartbeat | 5 دقايق | يأكد إن الجهاز عايش وواقف فين |
| batch.upload | كل 60 ثانية أو 50 نقطة (أيهما أسبق) لما فيه نت | |
| maxBatch | 500 نقطة | |
| minAccuracy للحسابات | 50 متر | النقط الأسوأ بتتخزن بس مابتأثرش على الوقفة |

### ليه Hybrid (Activity + GPS + Geofence)؟
- Activity Recognition لوحده **مش موثوق** (ممكن يقول STILL في الزحمة).
- GPS لوحده بيطلع "Drift" (النقطة بتتنطط 30-50م وإنت واقف) → يبان إنه بيتحرك.
- الحل: **الوقفة = النقط فضلت جوه دايرة 150م لمدة ≥ 3 دقايق** (Spatial clustering) — نفس منطق Traccar الجديد (minDistance 200م / minDuration 3 دقايق).

---

## 4. التخزين الأوفلاين (Offline-first) — "صفر فقد"

### المبدأ: **كل نقطة تتكتب في SQLite الأول، وبعدين نحاول نرفعها.** (Outbox pattern)

```
GPS fix ──▶ Filter ──▶ INSERT INTO outbox (seq++, ...) ──▶ commit ✅
                                       │
                 SyncWorker ◀──────────┘ (trigger: كل 60 ث / رجوع النت / 50 نقطة)
                     │
                     ├─ SELECT * FROM outbox WHERE seq > last_acked ORDER BY priority DESC, seq LIMIT 500
                     ├─ gzip + POST /locations/batch
                     ├─ 200 {ack_up_to_seq} → DELETE WHERE seq <= ack  (أو mark synced)
                     └─ فشل → exponential backoff (5s, 15s, 60s, 5m... max 15m)
```

### جدول الـ Outbox على الموبايل
```sql
CREATE TABLE outbox (
  seq INTEGER PRIMARY KEY,         -- تسلسلي ومستمر حتى بعد restart (يتخزن في prefs)
  kind TEXT,                       -- loc / event / attendance / stop_summary
  priority INTEGER DEFAULT 0,      -- attendance=3, stop_alert=2, events=1, loc=0
  recorded_at INTEGER,             -- GPS time (ms)
  payload BLOB,                    -- مضغوط
  attempts INTEGER DEFAULT 0
);
```

### قواعد مهمة
1. **Priority queue:** لما النت يرجع بعد انقطاع طويل، يترفع الأول: الحضور/الانصراف → ملخص الوقفات (stop_summary) → الأحداث → النقط. كده المدير يعرف "وقف ساعة" في **ثواني** حتى لو فيه 3000 نقطة لسه بتترفع.
2. **السعة:** نقطة ≈ 60-100 بايت → أسبوع أوفلاين كامل ≈ 5-10 ميجا. نحط حد أقصى 50,000 نقطة، ولو اتملى نعمل **Downsampling** للقديم (نقطة كل دقيقة) بدل ما نمسح.
3. **الحضور الأوفلاين:** Check-in يشتغل عادي بدون نت، يتسجل بالوقت الحقيقي (GPS time)، ويتعلم `source=offline_synced`.
4. **الوقت:** نستخدم `location.time` (من الأقمار الصناعية) + `elapsedRealtimeNanos` بدل ساعة الموبايل → السواق مايقدرش يزوّر الوقت بتغيير الساعة.
5. **Compression:** Delta encoding + gzip للدفعات → تقليل الداتا 70-80%.
6. **أتمتة الاكتشاف:** `ConnectivityManager.NetworkCallback` → أول ما النت يرجع يبدأ الـ sync فوراً.

### الـ Local Stop Detector (على الموبايل)
- الموبايل نفسه بيحسب الوقفة: لو واقف ≥ 30 دقيقة (نفس قاعدة السيرفر، جاية من الـ config) → يولّد `stop_summary` بأولوية عالية.
- لو فيه نت → يتبعت فوراً (ده تأكيد إضافي فوق حساب السيرفر).
- لو مفيش نت → يستنى في الـ outbox بأولوية 2.
- **SMS Fallback (اختياري):** لو مفيش داتا لمدة > 30 دقيقة وفيه إشارة شبكة عادية → يبعت SMS لرقم الشركة (Gateway) فيه: `FP|drv123|STOP|30.0444,31.2357|since 10:05|bat 54`. ده بيغطي المناطق اللي فيها شبكة ضعيفة (2G) من غير إنترنت. (محتاج صلاحية SEND_SMS → أسهل في التوزيع الداخلي مش Google Play).

---

## 5. مقاومة القتل (Survive the OS)

| السيناريو | الحل |
|---|---|
| السواق شال التطبيق من الـ Recent | Foreground Service مع `stopWithTask=false` + `START_STICKY` |
| الموبايل عمل Restart | `BOOT_COMPLETED` receiver → لو فيه وردية مفتوحة يرجع يشغل التتبع |
| التطبيق اتحدّث | `MY_PACKAGE_REPLACED` receiver |
| Doze mode | Foreground service + Battery optimization exemption |
| OEM killers (Xiaomi...) | Wizard للإعدادات + `WorkManager` periodic (كل 15 د) يتأكد إن الـ service شغالة ويرجعها |
| السيرفر يكتشف إن الجهاز "مات" | Watchdog على السيرفر (مفيش heartbeat 15 د) → **FCM high-priority push** يصحّي التطبيق → لو مارجعش → تنبيه للمشرف |
| السواق قفل الـ GPS | `LocationManager.PROVIDERS_CHANGED` → حدث `gps_off` + إشعار للسواق "شغّل الموقع" + تنبيه للمشرف |
| السواق سحب الصلاحية | يتكتشف في الـ heartbeat → تنبيه حرج |
| السواق مسح التطبيق | انقطاع نهائي → Watchdog → تنبيه حرج (والـ MDM يمنعه أصلاً) |
| Battery Saver اتفعل | `PowerManager.isPowerSaveMode` → حدث + تنبيه |

---

## 6. مكافحة الغش (Anti-fraud)

| الغش | الاكتشاف |
|---|---|
| **Fake GPS apps** | `Location.isMock()` (API 31+) / `isFromMockProvider()` (أقدم) + فحص Developer options + قائمة تطبيقات Mock معروفة |
| Root / Hooking (Magisk, Xposed) | Play Integrity API (device integrity) + فحوصات root |
| قفزات مستحيلة | سرعة محسوبة بين نقطتين > 200 كم/س أو accuracy ثابتة بشكل غير طبيعي |
| GPS مزيف بس Wi-Fi/Cell بيقولوا حاجة تانية | مقارنة مكان الـ GPS مع مكان الـ Network provider (فرق > 2 كم = مشبوه) |
| حط الموبايل في البيت وراح بالعربية | الموبايل هيبان "واقف" → تنبيه الوقوف هيكشفه. (الحل الكامل: GPS tracker في العربية + مقارنة الاتنين) |
| موبايل تاني | Device binding: الحساب مربوط بـ install_id واحد |
| التلاعب بالساعة | استخدام GPS time |

---

## 7. الحالة الصحية للجهاز (Heartbeat payload)

```json
{
  "seq": 18240, "recorded_at": 1761030000000,
  "battery": 54, "charging": false, "power_save": false,
  "gps_enabled": true, "perm_fine": true, "perm_background": true,
  "battery_opt_ignored": true, "notifications_enabled": true,
  "network": "4g", "airplane": false,
  "pending_outbox": 0, "last_fix_age_s": 12,
  "state": "STATIONARY", "app_version": "1.3.0", "os": "Android 14", "model": "Redmi Note 12",
  "mock_suspected": false
}
```
السيرفر بيستخدم الداتا دي في **Device Health Report** (مين موبايله فيه مشكلة قبل ما تبقى مشكلة).

---

## 8. هيكل كود التطبيق (Kotlin) المقترح

```
app/
 ├─ auth/            (OTP login, token storage - EncryptedSharedPreferences)
 ├─ attendance/      (CheckInViewModel, camera selfie, offline queue)
 ├─ tracking/
 │   ├─ TrackingService.kt        (ForegroundService type=location)
 │   ├─ MotionStateMachine.kt     (MOVING / STATIONARY / OFF)
 │   ├─ LocationProvider.kt       (FusedLocationProviderClient)
 │   ├─ ActivityTransitions.kt    (Activity Recognition Transition API)
 │   ├─ StationaryGeofence.kt     (GeofencingClient)
 │   ├─ LocalStopDetector.kt
 │   ├─ IntegrityChecker.kt       (mock, root, gps on, perms)
 │   └─ BootReceiver.kt / PackageReplacedReceiver.kt
 ├─ data/
 │   ├─ db/ (Room + SQLCipher: OutboxDao, ConfigDao)
 │   └─ sync/ (SyncWorker - WorkManager, NetworkCallback trigger, backoff)
 ├─ push/            (FCM: wake-up, config update, "اكتب سبب الوقفة")
 ├─ onboarding/      (Permission wizard per OEM)
 └─ config/          (RemoteConfig from /me/config)
```

### بديل سريع: Flutter + Transistorsoft
المكتبة دي فيها **جاهز**: SQLite persistence قبل الرفع، HTTP batch + retry، motion state machine (stopTimeout, stationaryRadius)، heartbeat، startOnBoot، stopOnTerminate=false، headless mode، geofences، schedule (ساعات الوردية). يعني 60-70% من الجدول اللي فوق جاهز بـ ~$399 ترخيص دايم.

---

## 9. اختبار التطبيق (Test Matrix)
- ماركات: Samsung، Xiaomi/Redmi، Oppo/Realme، Vivo، Huawei، Infinix/Tecno (منتشرين جداً في مصر).
- سيناريوهات: شاشة مقفولة 8 ساعات، Restart، وضع طيران ساعتين ثم رجوع، تغيير الساعة، Fake GPS app، بطارية 5%، Battery saver، إزالة من Recent، تحديث التطبيق أثناء الوردية.
- مقاييس: % النقط الواصلة، زمن التأخير، استهلاك البطارية/ساعة، الداتا/يوم.
- أداة مساعدة: تطبيق **DontKillMyApp Benchmark** على Play Store لقياس أي موبايل بيقتل الخلفية.
