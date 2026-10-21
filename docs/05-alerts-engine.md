# 05 - محرك التنبيهات اللحظية (Real-time Alert Engine) وخوارزمية اكتشاف الوقوف

> الملف ده بيحل المشكلة الرئيسية: "السواق يقف نص ساعة → يوصلني إشعار فوراً، مش بعد ساعتين ومش بعد 3 أيام".

---

## 1. خوارزمية اكتشاف الوقوف (Stop Detection)

### 1.1 التعريف
**وقفة** = السواق فضل جوه دايرة نصف قطرها **R = 150م** لمدة **≥ T_min = 3 دقايق**.
**وقفة طويلة** = وقفة مدتها **≥ T_alert = 30 دقيقة** (قابلة للتعديل لكل فرع/سواق/نوع مكان).

### 1.2 الخوارزمية (Online / Streaming) — بتشتغل على كل نقطة توصل

```python
# الحالة لكل سواق محفوظة في Redis
def on_position(driver, p):
    if p.accuracy > 75: 
        return  # نقطة ضعيفة: تتخزن بس متأثرش على القرار
    s = state[driver]

    if s.anchor is None:
        s.anchor = Anchor(center=p.latlng, since=p.t, points=[p])
        return

    d = haversine(s.anchor.center, p.latlng)
    # هامش للـ GPS drift حسب دقة النقطة
    if d <= R + min(p.accuracy, 50):
        s.anchor.add(p)                       # لسه في نفس المكان
        s.anchor.center = weighted_median(s.anchor.points)  # تحديث المركز
        dwell = p.t - s.anchor.since
        if dwell >= T_min and s.status != 'stopped':
            s.status = 'stopped'; emit('stop_started', s.anchor)
            zadd('stop_timers', s.anchor.since, driver)
    else:
        # خرج من الدايرة: لازم يتأكد (نقطتين متتاليتين بره أو مسافة > 2R) عشان ميبقاش drift
        if confirm_exit(s, p):
            if s.status == 'stopped':
                close_stop(driver, s.anchor, end=last_inside_time)
                emit('stop_ended')
                zrem('stop_timers', driver)
            s.status = 'moving'
            s.anchor = Anchor(center=p.latlng, since=p.t, points=[p])
```

### 1.3 Timer Job (كل 30-60 ثانية) — عشان التنبيه يطلع **في الدقيقة 30 بالظبط** حتى لو مفيش نقط جديدة
```python
every 30s:
    for driver in zrangebyscore('stop_timers', 0, now - T_alert(driver)):
        if not already_alerted(driver, stop_id):
            if not in_allowed_geofence(driver.anchor) or dwell > geofence.max_dwell:
                if in_shift(driver):
                    raise_alert('PROLONGED_STOP', driver, started_at=anchor.since)
    # تصعيد
    for alert in open_alerts('PROLONGED_STOP'):
        if dwell >= 60min and not escalated: escalate(alert, to='manager')
```

> **ليه Timer؟** لأن الموبايل في وضع STATIONARY بيبعت heartbeat كل 5 دقايق بس، فمنستناش نقطة جديدة عشان نقرر.

### 1.4 تصنيف الوقفة (Classification) — عشان نقلل الإزعاج
| المكان | التصرف |
|---|---|
| جوه Geofence **مخزن/فرع** | مسموح (مفيش تنبيه) أو حد أقصى خاص (مثلاً 90 د) |
| جوه Geofence **عميل** | مسموح لحد `max_dwell_min` للعميل (مثلاً 45 د) |
| جوه Geofence **بنزينة** | مسموح 20 د |
| جوه Geofence **ممنوع** (بيت السواق، قهوة معروفة) | تنبيه فوري بعد 10 د |
| مكان غير معروف | تنبيه بعد 30 د |
| وقت البريك الرسمي | مسموح (حسب إعداد الوردية) |
| **أماكن متكررة** | السيستم يتعلم: لو أكتر من 3 سواقين بيقفوا في نفس المكان كتير → يقترح على الأدمن يحوله Geofence (مطعم، مكان تحميل...) |

### 1.5 حالات خاصة
- **زحمة/إشارة:** T_min = 3 د + R = 150م بيمنع اعتبار الزحمة وقفة؛ والزحمة الشديدة جداً (> 30 د في 150م) هتطلع تنبيه، وده مقبول — المشرف يقفله بسبب "زحمة". (تحسين: لو السرعة المتوسطة > 0 وبيتحرك ببطء على طريق بيانات المرور → تصنيف "traffic").
- **السواق نزل ماشي:** لو activity=ON_FOOT والمسافة اتعدت R → الوقفة اتقفلت (ولو العربية عندها GPS tracker نقارن).
- **Gap في البيانات:** لو فيه فجوة > 10 د بين نقطتين ونفس المكان تقريباً قبل وبعد → **وقفة** (نفس منطق Traccar `minimalNoDataDuration`). لو المكان مختلف → "رحلة بدون بيانات" (offline trip).

---

## 2. تنبيهات البيانات المتأخرة (Late / Offline data)

السيناريو: السواق دخل منطقة مفيهاش شبكة الساعة 10:00، وقف من 10:05 لـ 11:20، النت رجع 11:40.

| الوقت | اللي بيحصل |
|---|---|
| 10:15 | السيرفر: مفيش heartbeat من 15 د → **تنبيه A3 "انقطاع اتصال"** (مع آخر مكان معروف) |
| 10:35 | الموبايل محلياً: عدى 30 د واقف → يولد stop_summary (أولوية 2) — ولو فيه شبكة SMS يبعت SMS |
| 11:40 | النت رجع → الموبايل يرفع stop_summary **أولاً** ← السيرفر يطلع **تنبيه A1 "وقوف طويل (اتكشف متأخر)"**: `بدأ 10:05 - انتهى 11:20 - المدة 75 د - اتكشف 11:40` |
| 11:41 | باقي النقط تترفع → Reprocessing للفترة → Timeline يتحدث تلقائياً |

**قاعدة:** أي تنبيه ليه `started_at` (الحقيقي) و `detected_at` (وقت الاكتشاف) → في التقارير بنحسب **Detection delay** ونعرف المناطق اللي فيها مشاكل شبكة.

**تجميع التنبيهات:** لو انقطاع الاتصال اتبعه وقوف طويل لنفس الفترة → تنبيه واحد مدموج "انقطع ووقف 75 د" بدل اتنين.

---

## 3. هيكل محرك القواعد (Rule Engine)

### 3.1 تعريف القاعدة (JSON)
```json
{
  "id": "rule_long_stop_default",
  "type": "PROLONGED_STOP",
  "params": { "min_minutes": 30, "radius_m": 150, "exclude_geofence_types": ["depot"] },
  "applies_to": { "branches": ["*"], "drivers": ["*"] },
  "active_hours": { "only_in_shift": true, "exclude_breaks": true },
  "severity": "high",
  "channels": ["dashboard", "push", "telegram"],
  "escalation": [ { "after_min": 30, "to": ["manager"], "channels": ["push","whatsapp"] } ],
  "cooldown_min": 60,
  "enabled": true
}
```

### 3.2 أنواع القواعد (Evaluators)
| Evaluator | يشتغل على | Trigger |
|---|---|---|
| `PROLONGED_STOP` | stop state + timer | dwell ≥ X |
| `NO_SIGNAL` | last_seen ZSET + watchdog | now - last_seen ≥ X وهو في الوردية |
| `GPS_OFF` / `PERMISSION_REVOKED` / `POWER_SAVE` | device events | حدث |
| `MOCK_LOCATION` | position.is_mock / speed jumps | فوري |
| `LOW_BATTERY` | heartbeat | < X% |
| `OVERSPEED` | positions | speed > X لمدة Y |
| `GEOFENCE_ENTER/EXIT` | positions × geofences (PostGIS `ST_Contains` أو in-memory R-tree) | تغيير الحالة |
| `DWELL_EXCEEDED` | stop داخل geofence | dwell > max_dwell |
| `LATE_CHECK_IN` / `NO_CHECK_IN` | cron | shift_start + grace |
| `EARLY_CHECK_OUT` | attendance | قبل نهاية الوردية بـ X |
| `MOVEMENT_OUTSIDE_SHIFT` | positions | (لو مسموح التتبع) |
| `ROUTE_DEVIATION` (مرحلة 2) | positions × planned route | بعد > X متر |

### 3.3 دورة حياة التنبيه
```
OPEN ──(المشرف شافه)──▶ ACKNOWLEDGED ──(سبب + ملاحظة)──▶ RESOLVED
  │                                                         ▲
  ├──(الشرط اتلغى: السواق اتحرك/النت رجع)──▶ AUTO_RESOLVED ──┘
  └──(مفيش Ack في 15 د)──▶ ESCALATED
```
- **أسباب الإغلاق (قائمة جاهزة):** عند عميل / تحميل-تفريغ / عطل عربية / زحمة / بنزينة / صلاة-بريك / بدون مبرر ⚠️ / خطأ GPS (False positive).
- الأسباب دي بتدخل في التحليلات (ملف 06) → "كام ساعة ضاعت بدون مبرر".

### 3.4 منع الإزعاج (Alert Fatigue)
- **Deduplication:** تنبيه واحد لكل وقفة.
- **Cooldown:** نفس النوع لنفس السواق مرة كل X دقيقة.
- **Grouping:** "5 سواقين انقطع عنهم النت في نفس المنطقة" = تنبيه واحد (مشكلة شبكة مش سواقين).
- **Quiet hours** للقنوات المزعجة (WhatsApp بليل).
- **Severity routing:** low → داشبورد بس، high → push، critical → push + WhatsApp + صوت.

---

## 4. قنوات الإشعارات

| القناة | الاستخدام | التكلفة/الملاحظات |
|---|---|---|
| **Dashboard (WebSocket)** | كل التنبيهات + صوت + Badge | مجاني |
| **FCM Push** (تطبيق المشرف/PWA) | high و critical | مجاني |
| **Telegram Bot** | جروب للمشرفين، سهل جداً وسريع ومجاني | مجاني — **أسرع حاجة تتعمل في MVP** |
| **WhatsApp Business API** | للمديرين | مدفوع لكل رسالة (Template messages) عن طريق Meta أو مزود |
| **SMS** | critical لو مفيش نت عند المشرف | مدفوع (مزودين مصريين) |
| **Email** | التقرير اليومي/الأسبوعي | مجاني تقريباً |

### شكل الإشعار
```
🔴 وقوف طويل — أحمد محمد (فرع مدينة نصر)
⏱ واقف من 10:05 (35 دقيقة)
📍 شارع مصطفى النحاس — خارج أي منطقة مسجلة
🔋 54% | 📶 4G
[فتح على الخريطة] [اتصال بالسواق] [تم الاطلاع]
```

---

## 5. الـ Watchdog (حارس الاتصال)
```python
every 60s:
    for driver in zrangebyscore('last_seen', 0, now - 10min):
        if in_shift(driver):
            send_fcm_high_priority(driver.device, {"type":"wake_and_report"})   # محاولة إيقاظ
    for driver in zrangebyscore('last_seen', 0, now - 15min):
        if in_shift(driver) and not alerted: raise_alert('NO_SIGNAL', driver,
                         payload={last_pos, last_battery, last_network})
```
- **تشخيص ذكي للسبب:** آخر بطارية < 5% → "غالباً الموبايل فصل شحن". آخر network = none وفي منطقة معروفة بضعف الشبكة → "غالباً منطقة بدون تغطية". gps_off قبلها → "السواق قفل الموقع".
- **خريطة ضعف التغطية:** من كل الانقطاعات بنبني Heatmap للأماكن اللي النت بيقطع فيها → المشرف يعرف إن الانقطاع هناك طبيعي.

---

## 6. الأداء المستهدف
| | |
|---|---|
| وصول نقطة → تحديث الخريطة | < 2 ثانية (بعد وصولها للسيرفر) |
| تحقق شرط الوقوف → إشعار | < 60 ثانية |
| Watchdog | < 16 دقيقة من آخر اتصال |
| تحمل | 1000 سواق × نقطة/10ث = 100 نقطة/ث — Redis + Worker واحد يستحملهم بسهولة |
