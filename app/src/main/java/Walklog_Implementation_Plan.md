# Walklog — Full Implementation Plan (for Gemini in Android Studio)

> Read this whole file before writing any code. Follow it in every phase. If a chat message conflicts with this file, ask before deviating. This file is the single source of truth.

---

## 0. How you (the agent) must work

1. Work **one phase at a time** (Section 15). Do not start the next phase until the current one builds, runs, and meets its acceptance criteria.
2. Output **complete, compilable code for every file you create or change**. No `TODO`, no `...`, no placeholders, no "similar to above".
3. After each phase: run a Gradle sync and build, fix all errors and warnings you introduced, then list the files you changed and the manual steps I must do.
4. Never invent library versions. Use the latest **stable** version that Android Studio's dependency suggestions or Google Maven shows. No alpha, beta or RC versions.
5. Never add the `INTERNET` permission. The app is fully offline.
6. Never use the accelerometer to count steps. Use the hardware step sensor only.
7. If something in this plan is impossible or unsafe on the target Android version, stop and tell me, propose the closest alternative, and wait.
8. Follow the Git workflow in Section 16: work on the phase branch, make the commits listed for that phase, and at the end of every phase print the exact PowerShell git commands (add, commit, push, merge, tag) for me to run. Never run or suggest destructive git commands (`reset --hard`, `push --force`, `clean -fd`) without asking first.
9. Keep code idiomatic Kotlin, null-safe, with KDoc on public classes. No deprecated APIs where a current replacement exists.

---

## 1. Product summary

**App name:** Walklog
**Package / applicationId:** `com.lensora.walklog` (the package the existing Android Studio project already uses; never rename it)
**Users:** one person (me), personal use, installed by APK. Not published to Play Store (but keep it Play-compliant where cheap).
**Purpose:** count daily steps very accurately, track walks with GPS (speed, time, distance, route), keep a full history, remind me to move, and never lose data.

**Primary goals (in priority order):**
1. Step accuracy: error within roughly 2–5 steps on a controlled test walk (100 steps), and within about 1–3% over a full day.
2. Minimum GPS position error (target 1–3 m open sky on a dual-frequency phone, 3–5 m otherwise).
3. Never lose steps: counting survives app kill, reboot, midnight, timezone change.
4. Excellent UI/UX (Material 3, smooth, accessible).
5. Privacy: all data stays on the phone.

---

## 2. Non-negotiable constraints

- Language: Kotlin. UI: Jetpack Compose + Material 3. Architecture: **MVVM** with unidirectional data flow.
- `minSdk = 26`, `compileSdk = 36`, `targetSdk = 36`. (Android 17 / API 37 exists; keep targetSdk 36 unless I ask to move.)
- No network permission. If any dependency merges `INTERNET` into the manifest, remove it with `tools:node="remove"`.
- Hardware `Sensor.TYPE_STEP_COUNTER` is the source of truth for steps.
- All timestamps stored as UTC epoch millis, plus the zone id used for the local date.
- All persistent data in Room (+ DataStore for settings). No SharedPreferences, no browser-style storage.
- Must run on real phones; emulator has no real step data (provide a debug "inject steps" tool, see 13.6).

---

## 3. Tech stack and dependencies

Use the project setup that the current Android Studio "Empty Activity (Compose)" template generates (Kotlin Compose compiler plugin, version catalog `gradle/libs.versions.toml`, KSP for Room and Hilt). Put **every** dependency and plugin version in `libs.versions.toml`.

| Area | Library |
|---|---|
| Core | `androidx.core:core-ktx`, `androidx.core:core-splashscreen` |
| Lifecycle | `lifecycle-runtime-compose`, `lifecycle-viewmodel-compose`, `lifecycle-service` |
| UI | `androidx.activity:activity-compose`, Compose BOM (`ui`, `ui-tooling`, `ui-tooling-preview`, `material3`, `material-icons-extended`) |
| Navigation | `androidx.navigation:navigation-compose` (type-safe routes with kotlinx.serialization) |
| DI | Hilt (`hilt-android`, `hilt-compiler`, `androidx.hilt:hilt-navigation-compose`, `androidx.hilt:hilt-work`) |
| Database | Room (`room-runtime`, `room-ktx`, `room-compiler` via KSP, `room-testing`) |
| Settings | `androidx.datastore:datastore-preferences` |
| Background | `androidx.work:work-runtime-ktx` |
| Location + activity | `com.google.android.gms:play-services-location` |
| Widget | `androidx.glance:glance-appwidget`, `glance-material3` |
| Security | `androidx.biometric:biometric` |
| Serialization | `kotlinx-serialization-json` (JSON export/import) |
| Coroutines | `kotlinx-coroutines-android` |
| Test | `junit`, `kotlinx-coroutines-test`, `turbine`, `mockk`, `androidx.room:room-testing`, `androidx.compose.ui:ui-test-junit4`, `androidx.test.ext:junit` |

Build config: enable R8/minify for release, Java/Kotlin JVM target 17 (or what the template uses), `buildFeatures { compose = true }`, ViewBinding not used.

---

## 4. Architecture and project structure

MVVM + Repository + single source of truth (Room). UI observes `StateFlow<UiState>` and sends events to the ViewModel. Services and workers talk to the same repositories through Hilt.

```
com.kanak.walklog
├── WalklogApp.kt                    (Application, Hilt, notification channels init)
├── MainActivity.kt                  (single activity, edge-to-edge, nav host, app lock gate)
├── di/                              (Hilt modules: Database, DataStore, Sensors, Location, Clock)
├── core/
│   ├── time/                        (Clock abstraction, DateUtils, zone/DST helpers)
│   ├── util/                        (Result wrappers, formatters, unit conversion)
│   └── permissions/                 (PermissionState, OEM guide data)
├── data/
│   ├── local/                       (WalklogDatabase, entities, DAOs, converters, migrations)
│   ├── prefs/                       (SettingsDataStore, SettingsModel)
│   ├── repository/                  (StepRepository, WalkRepository, SettingsRepository,
│   │                                 AchievementRepository, BackupRepository)
│   └── model/                       (domain models mapped from entities)
├── engine/
│   ├── step/                        (StepEngine, DeltaCalculator, CadenceEstimator, VehicleFilter)
│   ├── distance/                    (StrideModel, CalorieCalculator, CalibrationEngine)
│   └── gps/                         (LocationEngine, GpsFilter, KalmanSmoother, RtsSmoother,
│                                     DeadReckoning, GnssQualityMonitor, WalkRecorder)
├── service/
│   ├── StepCounterService.kt        (foreground, type=health)
│   ├── WalkTrackingService.kt       (foreground, type=location|health)
│   └── receivers/                   (BootReceiver, TimeChangeReceiver, ReminderReceiver,
│                                     MidnightReceiver, NotificationActionReceiver)
├── work/                            (WatchdogWorker, DailyBackupWorker, SyncWorker)
├── notification/                    (Channels, builders, ReminderScheduler, InactivityChecker)
├── widget/                          (Glance widget + receiver)
├── tile/                            (Quick Settings TileService: start/stop walk)
└── ui/
    ├── theme/                       (Color, Type, Shape, Theme, dynamic color)
    ├── components/                  (ProgressRing, StatTile, BarChart, LineChart, RouteCanvas,
    │                                 EmptyState, GpsQualityChip, HoldToStopButton, etc.)
    ├── navigation/                  (routes, NavHost, bottom bar)
    └── screens/
        ├── onboarding/  home/  walk/  history/  daydetail/  walkdetail/  walks/
        ├── achievements/  settings/  calibration/  backup/  lock/  debug/  oemguide/
```

Rules:
- One ViewModel per screen, exposing one immutable `UiState` data class via `StateFlow`; one-off effects (snackbar, navigation) via `SharedFlow`/`Channel`.
- Composables are stateless where possible (state hoisting). No business logic in composables.
- Repositories expose `Flow`. Engines are plain Kotlin classes (unit-testable, no Android imports where avoidable).
- Inject a `Clock`/`TimeProvider` everywhere time is used so tests can control time.

---

## 5. Manifest, permissions, services, receivers

**Permissions**
```
ACTIVITY_RECOGNITION            (runtime; needed for step sensors and activity transitions)
POST_NOTIFICATIONS              (runtime, API 33+)
FOREGROUND_SERVICE
FOREGROUND_SERVICE_HEALTH
FOREGROUND_SERVICE_LOCATION
ACCESS_FINE_LOCATION            (runtime; request precise, not approximate)
ACCESS_COARSE_LOCATION          (required to be requested together with fine)
RECEIVE_BOOT_COMPLETED
SCHEDULE_EXACT_ALARM            (check canScheduleExactAlarms(); deep-link to the settings page; fall back to inexact if denied)
REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
WAKE_LOCK
VIBRATE
USE_BIOMETRIC
```
- Do **not** request `ACCESS_BACKGROUND_LOCATION` at first. Start the walk service while the app is visible; a foreground service of type `location` keeps location access with the screen off. Only add background location if real-device testing proves it is needed, and tell me first.
- Do **not** declare `INTERNET`.

**Application:** `android:allowBackup="false"` (location history stays on the phone; I use my own export), `android:enableOnBackInvokedCallback="true"`, `android:theme` uses the splash screen theme, `android:localeConfig` not needed.

**Services**
- `StepCounterService`: `android:foregroundServiceType="health"`, `android:exported="false"`.
- `WalkTrackingService`: `android:foregroundServiceType="location|health"`, `android:exported="false"`.
- Quick Settings `TileService` (exported with the required permission) to start/stop a walk.

**Receivers**
- `BootReceiver` (`BOOT_COMPLETED`, `LOCKED_BOOT_COMPLETED` not needed): schedule work, reschedule alarms, try to start `StepCounterService` inside try/catch (log if the system refuses). Counting must not depend on this succeeding, see Section 7.
- `TimeChangeReceiver` (`ACTION_TIMEZONE_CHANGED`, `ACTION_TIME_CHANGED`, `ACTION_DATE_CHANGED`): recompute local date, reschedule alarms.
- `ReminderReceiver`, `MidnightReceiver`, `NotificationActionReceiver` (not exported).

**Platform rules you must respect**
- A `health` foreground service may only start after `ACTIVITY_RECOGNITION` is granted, otherwise the system throws `SecurityException`. Check before starting.
- A `location` foreground service must start while the app is in the foreground and after location permission is granted.
- Handle `ForegroundServiceStartNotAllowedException` and `SecurityException` gracefully: log, show an in-app banner, never crash.
- Post the foreground notification within a few seconds of `startForegroundService()`.
- Edge-to-edge is enforced: call `enableEdgeToEdge()` and handle window insets in every screen.

---

## 6. Data layer

### 6.1 Room database (`WalklogDatabase`, exportSchema = true, start at version 1, write a migration for every change)

**CounterStateEntity** (single row, id = 1) — kept in Room so it is saved atomically with step totals:
`id, lastSensorValue: Long, lastBootCount: Int, lastEventEpochMs: Long, carryFraction: Double, initialized: Boolean`

**DailyStepEntity** (PK `date` = "yyyy-MM-dd" local):
`date, zoneId, rawSteps, steps, manualAdjustment, distanceMeters, firstStepTime, lastStepTime, activeMinutes, calories, excludedVehicleSteps, interrupted: Boolean`

**StepLogEntity** (hourly buckets, unique on `date + hour`):
`id, date, hour, rawSteps, steps, distanceMeters`

**WalkSessionEntity:**
`id, startTime, endTime, startDate, steps, distanceMeters, gpsDistanceMeters, stepDistanceMeters, avgSpeedMps, maxSpeedMps, durationSeconds, movingSeconds, pausedSeconds, calories, carryPosition, avgAccuracyMeters, usedDeadReckoning: Boolean, note`

**RoutePointEntity:**
`id, sessionId (FK cascade), timestamp, lat, lng, rawLat, rawLng, accuracy, speedMps, altitude, estimated: Boolean`

**CalibrationRecordEntity:**
`id, timestamp, type (STEP_COUNT | KNOWN_DISTANCE | GPS_AUTO), carryPosition, countedSteps, actualSteps, distanceMeters, resultFactor, resultStrideCm, accepted`

**LearnedStrideEntity:**
`cadenceBand (SLOW|NORMAL|BRISK|RUN), carryPosition, strideCm (EMA), sampleCount`

**AchievementEntity:** `id, unlockedAt (nullable), progress`

**ServiceEventEntity:** `id, timestamp, type (STARTED|STOPPED|KILLED_DETECTED|RESTARTED|BOOT|PERMISSION_LOST|SYNC), detail`

Provide DAOs with `Flow` queries for: today, date range, hourly for a date, walks list, walk with route, achievements, service events. Add indices on `date`, `sessionId`, `startTime`.

### 6.2 DataStore (preferences) `SettingsModel`
```
heightCm, weightKg, age, gender(MALE|FEMALE|OTHER), manualStrideCm?(nullable),
carryPosition(POCKET|HAND|BAG), stepCorrectionFactor per position (default 1.0, clamp 0.85..1.15),
strideCalibrationFactor per position (default 1.0),
dailyGoal(default 10000), reminderEnabled, reminderIntervalMinutes(default 120, range 30..360),
activeHoursStart(08:00), activeHoursEnd(22:00), inactivityAlertEnabled, inactivityMinutes(default 60),
goalNotificationEnabled, vehicleFilterEnabled(default true),
keepScreenOnDuringWalk, voiceFeedbackEnabled, voiceFeedbackEveryKm,
units(METRIC|IMPERIAL), timeFormat(12|24), weekStart(MON|SUN), theme(SYSTEM|LIGHT|DARK), dynamicColor,
appLockEnabled, lockTimeoutSeconds(30), dailyBackupEnabled, backupFolderUri?, onboardingDone,
debugMode
```
Expose as a `Flow<SettingsModel>`; all writes via a repository with validation and clamping.

### 6.3 Save policy
- Persist step totals on every sensor batch into memory, and flush to Room **every 30 seconds** and on: service stop, app background, low memory (`onTrimMemory`), midnight sync, watchdog run, before reminder text is built.
- A flush is **one Room transaction** that updates `CounterStateEntity`, `DailyStepEntity`, and `StepLogEntity` together so a crash can never leave them inconsistent.

---

## 7. Step counting engine (accuracy critical)

### 7.1 Sensor setup
- `SensorManager.getDefaultSensor(Sensor.TYPE_STEP_COUNTER)`; if null show a clear full-screen error ("This phone has no hardware step counter") and disable counting. Do not fall back to the accelerometer.
- Register with `sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_FASTEST, 0)` — the last argument is `maxReportLatencyUs = 0` so events are not batched or delayed.
- Also register `TYPE_STEP_DETECTOR` (if present) for live cadence and for a debug cross-check log only. The counter remains the source of truth.
- The counter keeps counting in hardware even when the app is dead. Therefore **steps are computed from deltas**, never from "steps counted while the service ran".

### 7.2 Delta logic (reference implementation — implement and unit test exactly this)
```kotlin
data class CounterState(
    val lastSensorValue: Long,
    val lastBootCount: Int,
    val lastEventEpochMs: Long,
    val carryFraction: Double,
    val initialized: Boolean
)

data class RawDelta(val steps: Long, val rebooted: Boolean, val suspicious: Boolean)

fun computeDelta(state: CounterState, sensorValue: Long, bootCount: Int): RawDelta {
    if (!state.initialized) return RawDelta(0, false, false)          // first ever reading: set baseline only
    val rebooted = bootCount != state.lastBootCount || sensorValue < state.lastSensorValue
    val steps = if (rebooted) sensorValue else sensorValue - state.lastSensorValue
    return RawDelta(steps, rebooted, suspicious = steps > 60_000)      // log suspicious jumps, do not silently drop
}
```
- `bootCount` = `Settings.Global.getInt(contentResolver, Settings.Global.BOOT_COUNT, -1)`.
- On reboot: the new sensor value is steps since boot, so add it fully (steps taken before the app first read after reboot are included).
- Sensor values are floats: convert with `event.values[0].toLong()`.

### 7.3 Correct timestamps
Use the event's own timestamp, not "now", so batched or delayed events land in the correct hour:
`wallMs = System.currentTimeMillis() - (SystemClock.elapsedRealtimeNanos() - event.timestamp) / 1_000_000`

### 7.4 Applying the correction factor without rounding drift
```
corrected = rawDelta * stepCorrectionFactor[carryPosition] + carryFraction
whole = floor(corrected).toLong()
carryFraction = corrected - whole
```
Store `rawSteps` and corrected `steps` both in the daily row so factors can be re-tuned later.

### 7.5 Day and hour attribution
- Local date from `Instant.ofEpochMilli(wallMs).atZone(ZoneId.systemDefault()).toLocalDate()`; hour from the same zoned time. Use `java.time` only.
- **Midnight sync:** schedule an exact alarm at 00:00:00.5 local time every day (recompute after timezone/time change and after boot). When it fires, read the sensor once and attribute that delta to the **previous** date explicitly, then continue normally. This stops steps taken just before midnight from landing on the new day.
- If events arrive after a long gap (phone was dead, no midnight sync), attribute the delta to the date of the event and flag the day `interrupted = true`.
- DST/timezone: never use fixed 24-hour arithmetic; always use `ZonedDateTime`. On `ACTION_TIMEZONE_CHANGED`/`ACTION_TIME_CHANGED`, flush, recompute, reschedule alarms. A local day may have 23 or 25 hours; charts must handle that.

### 7.6 Manual open/resume sync
On every app open, service start, watchdog run and boot: read the current sensor value (one-shot listener if the service is not running), compute the delta against `CounterState`, and apply it. This recovers steps taken while the service was dead.

### 7.7 Vehicle filter (setting, default ON)
- Use the Activity Recognition Transition API (`ActivityTransitionRequest` for `IN_VEHICLE`, `ON_BICYCLE` ENTER/EXIT; needs `ACTIVITY_RECOGNITION`).
- While in a vehicle/bicycle state **and** (GPS speed > 25 km/h when available), put deltas into `excludedVehicleSteps` instead of totals.
- If Play Services activity detection is unavailable, skip the filter silently.
- Show the excluded count in the Debug screen so I can verify it does not drop real steps.

### 7.8 Cadence and active minutes
- Cadence (steps per minute) from a sliding 10-second window of corrected deltas. Bands: `<90 SLOW`, `90–110 NORMAL`, `110–130 BRISK`, `>130 RUN`.
- A minute counts as active if it has ≥ 60 steps.
- `firstStepTime` / `lastStepTime` = first and last event with a positive delta that day.

### 7.9 Service behavior
- `StepCounterService` (health type) holds the sensor listeners, updates the persistent notification (steps, distance, goal progress) at most once every 2 seconds (avoid notification spam), and flushes per 6.3.
- Return `START_STICKY`. Handle `onTimeout`/`onDestroy` by flushing and logging a `ServiceEvent`.
- `StepEngine` is a Hilt singleton owning the listeners; both services and the UI observe its `StateFlow<LiveStepState>` (today steps, cadence, sensor status, last event time). Do not register duplicate listeners.
- Never hold a partial wake lock all day. Wake lock only during a walk session.

---

## 8. Distance, stride, calibration, calories

### 8.1 Stride length
- Default stride: `heightCm × 0.415` (male), `× 0.413` (female), `× 0.414` (other), or the manual stride if set.
- Effective stride = `baseStride × strideCalibrationFactor[carryPosition]`, or the **learned stride** for the current cadence band and carry position when it has ≥ 3 samples.
- `distance = correctedSteps × stride` per delta (choose stride by the cadence band at that moment). Daily distance = sum. Display km (or miles when imperial).

### 8.2 Calibration wizard (Calibration screen, three tabs)
1. **Step-count test:** user picks carry position, taps Start, walks about 100 steps counting them by hand, taps Stop, enters the actual count. Record `counted` vs `actual`. Factor = mean(actual/counted) over the last **3+ accepted tests** of that position, reject outliers (more than 10% from the median), clamp 0.85–1.15. Show a clear "counted / actual / error" result and a "Save as calibration" button. Explain that this sets the step correction factor for that carry position.
2. **Known-distance test:** user walks a measured distance (default 400 m; editable). New stride = `distance / correctedSteps`. Save as stride calibration for that carry position.
3. **GPS auto suggestion:** after any walk with ≥ 200 steps and median GPS accuracy ≤ 10 m, compute `gpsDistance / correctedSteps` per cadence band and update `LearnedStrideEntity` with an exponential moving average (α = 0.2). Show a non-blocking suggestion card; nothing changes without my confirmation, except the learned-stride EMA which is applied automatically after 3 samples.

### 8.3 Carry position
Chip on Home and in Settings: Pocket / Hand / Bag. Changing it applies to future steps only. Each walk records its carry position.

### 8.4 Calories
- Daily estimate: `kcal = 0.57 × weightKg × distanceKm`.
- Walk sessions: `kcal = MET × weightKg × hours`, MET by speed: `<3.2 km/h 2.0`, `3.2–4.8 → 3.0`, `4.8–5.6 → 3.5`, `5.6–6.4 → 4.3`, `6.4–8.0 → 5.0`, `>8.0 → 8.0`. Label everything as an estimate. Age is stored for future use only.

---

## 9. GPS walk tracking (minimum position error)

### 9.1 Walk lifecycle
States: `Idle → Acquiring (waiting for a good fix) → Active ⇄ Paused → Finished`. Start/Pause/Resume/Stop from Home, notification actions, and the Quick Settings tile. **Stop requires a 1-second press-and-hold** in the app to prevent accidental stops.

### 9.2 Location request
```kotlin
LocationRequest.Builder(Priority.PRIORITY_HIGH_ACCURACY, 1000L)
    .setMinUpdateIntervalMillis(1000L)
    .setMinUpdateDistanceMeters(0f)
    .setMaxUpdateDelayMillis(0L)
    .setWaitForAccurateLocation(true)
    .build()
```
- Require **precise** location permission. If only approximate is granted, block walk tracking and explain how to fix it.
- Acquiring state: wait until horizontal accuracy ≤ 5 m (allow 10 m after 45 seconds, show a "GPS ready" chip). Drop the first 5 seconds of points after start.
- Adaptive interval: keep 1 s while moving; when stationary (no steps for 10 s) relax to 3 s to save battery, restore to 1 s on the first new step.
- Partial wake lock during a walk only. Optional "keep screen on" setting.

### 9.3 Point filtering (`GpsFilter`) — reject a point if any is true
- `accuracy > 15 m` (use 20 m only during dead-reckoning recovery)
- `isMock` is true (`Location.isMock`)
- timestamp older than the previous accepted point, or older than 5 s
- implied speed from the last accepted point > 15 km/h (walking mode) or > 3× the median speed of the last 10 s
- displacement smaller than the point's own accuracy radius while stationary
- fewer than 6 satellites used in fix when satellite info is available (do not reject when unavailable)

### 9.4 Stationary handling (biggest accuracy win)
If the step sensor shows **no steps in the last 3 seconds** and GPS speed < 0.5 m/s: freeze position, add **zero** distance, and do not draw route points. Auto-pause after 15 s of this; auto-resume on the first steps.

### 9.5 Speed
Prefer `location.speed` (Doppler) when `hasSpeedAccuracy()` and speed accuracy ≤ 1 m/s; otherwise use distance/time over a 5-second window. Smooth with a moving average of 3 s. Track current, average moving speed (excluding pauses), max speed, pace (min/km).

### 9.6 Two-pass smoothing
1. **Live:** constant-velocity Kalman filter in a local tangent plane (metres east/north of the first fix). State `[x, y, vx, vy]`; measurement noise `R = accuracy²`; process noise for walking (about 0.5 m/s² acceleration std). Feed filtered position to the UI and live distance.
2. **After Stop:** run a Rauch–Tung–Striebel backward smoother over the whole track and recompute distance and route from the smoothed path. Save both raw (`rawLat/rawLng`) and smoothed (`lat/lng`) points.
- Distance = sum of geodesic distances (`Location.distanceTo` or haversine) between consecutive accepted smoothed points, ignoring segments during frozen/stationary periods.

### 9.7 Dead reckoning during GPS loss
If no accepted fix for more than 5 s while steps continue: advance the position using `correctedSteps × learnedStride` along the heading from `TYPE_ROTATION_VECTOR` (convert to true north using `GeomagneticField`). Mark those points `estimated = true` (drawn dashed on the map, counted in `usedDeadReckoning`). When GPS returns, blend back over about 5 s to avoid a jump. Distance during estimated segments uses step-based distance.

### 9.8 GNSS quality
Use `GnssStatus.Callback` (permission already granted) to show satellites used and signal strength; detect **L5 / dual-frequency** support from carrier frequencies near 1176.45 MHz and show "Dual-frequency GPS: Yes/No/Unknown" in Settings and Debug. `GpsQualityChip` shows Good (≤ 8 m), Fair (8–15 m), Poor (> 15 m) with the current accuracy in metres.

### 9.9 Walk data
Per walk store steps (corrected, delta from walk start), GPS distance, step-based distance, fused distance (GPS when good, otherwise steps×stride), durations, speeds, calories, average accuracy, per-kilometre splits (computed on demand). Points saved in batches every 10 seconds and on stop. If the process dies mid-walk, on next start detect the unfinished session, finish it with the last saved point, and mark it "interrupted".

### 9.10 Voice feedback (optional setting)
`TextToSpeech` announces distance and pace every X km (default 1 km). Off by default. Handle TTS init failure silently.

---

## 10. Goals, reminders, notifications

**Channels:** `live_counter` (low, silent, ongoing), `walk_active` (low, ongoing), `reminders` (default), `goal_achieved` (default), `inactivity` (default), `system_alerts` (high, for "tracking stopped" problems). Create in `Application.onCreate`.

**Daily goal:** default 10,000, editable (500–100,000). Goal-reached notification once per day (guard with a stored "goalNotifiedDate"). Haptic + ring animation on reaching it while the app is open.

**Reminders**
- Default every **2 hours**, configurable 30 min–6 h, only inside the active-hours window (default 08:00–22:00). Configurable window.
- Use `AlarmManager.setExactAndAllowWhileIdle` when `canScheduleExactAlarms()`; otherwise `setAndAllowWhileIdle`, and show a settings banner explaining the exact-alarm permission.
- Each reminder shows steps so far, steps left, and a short varied message. **Skip** the reminder if the goal is already reached.
- Reschedule the next alarm each time one fires, after boot, after time/timezone change, and after any reminders setting change. Never stack duplicate alarms (use fixed request codes and `FLAG_UPDATE_CURRENT | FLAG_IMMUTABLE`).

**Inactivity alert (setting, default ON, 60 min):** inside active hours, if there are zero steps in the last N minutes, post "You've been sitting for an hour"; at most once per period; never during a walk; not when the goal notification just fired.

**Persistent notification actions:** Start walk / Pause / Stop (via `NotificationActionReceiver`).

---

## 11. Reliability

- **Watchdog:** `WatchdogWorker` (WorkManager periodic, 15 min): runs the manual sync (7.6), checks the service is alive and restarts it (try/catch), writes a `ServiceEvent`. If a gap > 30 min without sensor events is found while the phone was on, mark the day `interrupted` and show "Tracking was interrupted" on the History day row and Day detail.
- **Boot:** see Section 5; counting correctness comes from delta logic, not from the boot receiver.
- **Process death:** all state is in Room/DataStore; on start, reload `CounterState`; unfinished walk recovery per 9.9.
- **Battery optimization:** onboarding and Settings show current status. Request exemption with `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`; fall back to `ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS`.
- **OEM guide screen (offline, static text with best-effort deep links, always guarded by try/catch):** Xiaomi/Redmi/POCO (Autostart, Battery saver → No restrictions), Oppo/Realme/OnePlus (Auto-launch, Allow background activity), Vivo/iQOO (Background power consumption, Autostart), Samsung (Sleeping apps / Never sleeping apps), Huawei/Honor (App launch → manage manually), Google Pixel/others (Unrestricted battery). Detect brand via `Build.MANUFACTURER`.
- **Permission loss:** if `ACTIVITY_RECOGNITION` or notifications are revoked while running, show a `system_alerts` notification and an in-app banner with a fix button; never crash.
- **No sensor / low storage / Do Not Disturb:** handle each with a friendly message; reminders respect DND by default (do not bypass it).

---

## 12. UI / UX specification

### 12.1 Design system
- Material 3, dynamic color on Android 12+ (setting to disable), custom fallback palette seeded from green `#0B8F6B` with full light/dark schemes. Shapes: rounded 16–28 dp. Use tabular numerals (`fontFeatureSettings = "tnum"`) for all counters so digits don't jitter.
- Edge-to-edge, correct insets, predictive back everywhere, consistent 16 dp grid, min touch target 48 dp.
- Motion: ring sweep and number count-up (≈600 ms, ease-out), shared axis transitions between tabs, animated chart bars. Respect the system "remove animations" setting.
- Haptics: goal reached, walk start/pause/stop, calibration saved.
- Every screen has: loading placeholder (shimmer/skeleton), empty state with an illustration-free icon + helpful text, and an error state with a retry.
- Accessibility: `contentDescription`/semantics on the ring ("6,240 of 10,000 steps, 62 percent"), charts expose a text summary, works at 200% font scale, do not rely on color alone, TalkBack traversal order correct.
- Navigation: bottom bar with **Home, Walks, History, Settings**. Achievements, Debug, Calibration, Backup are reached from Settings/Home.

### 12.2 Screens
1. **Onboarding** (first run, skippable per step where allowed): welcome → profile (height, weight, age, gender) → daily goal → permissions one at a time with a plain-language reason before each system dialog (activity recognition → notifications → precise location → battery exemption → exact alarms) → OEM guide if relevant → done. Permission denial never blocks the app; features degrade with clear banners.
2. **Home:** top bar with date and streak chip; animated ring (≈260 dp, 20 dp stroke, gradient) with big step count and "of 10,000"; three stat tiles (Distance, Active time, Calories); carry-position chips; tracking status card (green "Tracking active" or a warning with a fix button); mini hourly bars for today; "Next reminder at HH:MM"; extended FAB **Start walk** placed for one-thumb reach.
3. **Walk (live):** large elapsed time; distance; current speed; pace; average and max speed; steps; calories; `GpsQualityChip`; live route canvas; Pause/Resume and hold-to-stop button; screen-on option. Also an "Acquiring GPS…" state.
4. **Walks list:** cards with date, duration, distance, avg speed, mini route thumbnail; filters (week/month/all); swipe to delete with undo.
5. **Walk detail:** route polyline auto-fitted (start/end markers, dashed for estimated segments, color-graded by speed), speed-over-time line chart, per-km splits table, stats grid, edit note, delete.
6. **History:** segmented control Week / Month / All; bar chart (Compose Canvas) with goal line and tap-to-select; monthly summary card (total steps, daily average, best day, total distance, days meeting goal); scrollable day list showing date, steps, distance, first–last step time range, goal check, interrupted marker.
7. **Day detail:** hourly bar chart, summary tiles, walks that day, tracking gap notes, manual adjustment (+/− steps with confirmation and an audit note).
8. **Achievements:** grid of badges with progress (first 10k day, 7-day streak, 30-day streak, 50 km total, longest walk, fastest km, early bird, 100k week). Personal records section.
9. **Settings:** grouped sections — Profile; Stride and calibration (manual stride, carry position, calibration wizard, learned strides list, GPS quality info incl. dual-frequency); Goals and reminders; Tracking (vehicle filter, keep screen on, voice feedback); Appearance (theme, dynamic color, units, 12/24 h, week start); Data (export CSV/JSON, import, daily backup folder, delete all data with double confirmation); Security (app lock); Battery and background (status + guides); About and Debug.
10. **Calibration** (see 8.2), **Backup**, **Lock screen**, **OEM guide**, **Debug**.

### 12.3 UiState contract examples (implement similar for every screen)
```kotlin
data class HomeUiState(
    val loading: Boolean = true,
    val steps: Int = 0,
    val goal: Int = 10_000,
    val progress: Float = 0f,
    val distanceKm: Double = 0.0,
    val activeMinutes: Int = 0,
    val calories: Int = 0,
    val streakDays: Int = 0,
    val carryPosition: CarryPosition = CarryPosition.POCKET,
    val hourly: List<HourBucket> = emptyList(),
    val trackingStatus: TrackingStatus = TrackingStatus.Unknown,
    val walkState: WalkState = WalkState.Idle,
    val nextReminderAt: Long? = null,
    val banners: List<Banner> = emptyList(),
    val error: UiError? = null
)
```

---

## 13. Extra features

1. **Home-screen widget (Jetpack Glance):** today's steps, progress bar, distance. Update on step flush (throttled to once per minute) and on a periodic schedule.
2. **Quick Settings tile:** start/stop walk.
3. **Export / import:** CSV (daily summary, walks, route points) and JSON full backup via the Storage Access Framework (`CreateDocument`/`OpenDocument`). Import validates a `schemaVersion`, shows a preview, and asks merge vs replace. **Daily auto-backup:** WorkManager writes a JSON backup to a folder I choose (persisted SAF URI), keeps the last 7.
4. **App lock:** optional biometric/device-credential lock (`BiometricPrompt` with `DEVICE_CREDENTIAL` fallback), locks after the configured timeout in background; hide content in the recents screen when locked (`FLAG_SECURE` while locked).
5. **Delete all data:** double confirmation, wipes Room, DataStore, cancels alarms/work, stops services.
6. **Debug screen (visible when debugMode is on, or from About):** raw sensor value, baseline, last saved value, boot count, reboot counter, last flush time, carry fraction, excluded vehicle steps, detector vs counter mismatch log, service event log, GPS accuracy/satellites/dual-frequency, alarm schedule list, a **"Inject test steps"** tool (adds N steps through the same delta pipeline, marked as test data) for emulator use, an **accuracy test mode** (walk exactly 100 steps, enter the truth, log the error over time), and a local **crash/exception log viewer** (catch uncaught exceptions with `Thread.setDefaultUncaughtExceptionHandler`, write to an internal file, no network).
7. **Streaks and records:** current streak, best streak, weekly/monthly summaries, personal records (fastest km, longest walk, best day).
8. **Small settings:** units, 12/24 h, week start, "reset today" (with confirmation).
9. **Optional stretch (only after everything else is done and I ask):** Health Connect read-only cross-check; SQLCipher-encrypted database.

---

## 14. Testing plan

**Unit tests (JUnit + coroutines-test + Turbine + MockK)** — must all pass before Phase 6 is accepted:
- `computeDelta`: first reading, normal increase, reboot by lower value, reboot by boot count change, suspicious jump.
- Correction factor with carry fraction: 1000 events of +1 with factor 1.03 yields the exact expected total, no drift.
- Midnight attribution: steps before/after midnight land on the correct dates; delayed event timestamps; DST spring-forward and fall-back days; timezone change.
- Stride and distance: male/female defaults, manual stride, calibration factors, cadence-band selection.
- Calibration averaging with outlier rejection and clamping.
- `GpsFilter` rules; Kalman/RTS on synthetic tracks (straight line with noise: smoothed error < raw error); stationary freezing; dead-reckoning heading math.
- Reminder scheduler: next fire time inside/outside active hours, skip when goal reached, no duplicate alarms.
- Streak and achievements logic; CSV/JSON export → import round trip.
- Room migration tests for every version bump.

**UI tests (Compose):** Home displays state correctly; permission banners; goal edit; walk hold-to-stop.

**Manual real-device checklist (I will run this — print it in the final README):**
1. Walk exactly 100 steps × 3, pocket and hand; error within 2–5 steps.
2. Screen off for 30 minutes of walking; totals correct.
3. Force-stop the app, walk 50 steps, reopen: steps recovered.
4. Reboot, walk, open: no loss, no double count.
5. Walk across midnight: steps split correctly.
6. Change timezone: no crash, dates sane.
7. Revoke each permission while running: banners appear, no crash.
8. 1 km walk on a measured route: GPS distance within about 1–3%.
9. Full-day battery check: note the drain and tune the GPS interval if high.
10. Reminders arrive every 2 hours inside the window and not after goal reached.

---

## 15. Phased plan and acceptance criteria

Complete each phase fully (code + build + short summary + the git checkpoint from Sections 16.6 and 16.7) before the next.

**Phase 1 — Project skeleton and core counting**
- Project setup, version catalog, Hilt, Room (`CounterState`, `DailyStep`, `StepLog`, `ServiceEvent`), DataStore skeleton, theme, single `MainActivity`, permissions flow for activity recognition + notifications, `StepEngine`, `StepCounterService` (health FGS + notification), delta logic + unit tests, 30-second flush, midnight alarm, boot receiver, manual sync on open.
- Accept: steps count live on a real phone with screen off; survives app kill and reboot; unit tests pass; Home shows a plain number.

**Phase 2 — UI shell and MVVM**
- Design system, navigation, bottom bar, Home (ring, tiles), History (list + weekly chart), Day detail (hourly), Settings (profile, goal, theme), onboarding, empty/loading/error states, accessibility semantics.
- Accept: all screens navigate, rotate/resize safely, dark mode works, TalkBack reads the ring.

**Phase 3 — Distance, calibration, calories**
- Stride model, cadence bands, carry positions, calibration wizard (step test + known distance), factors applied with carry fraction, calories, distance on Home/History, more unit tests.
- Accept: 100-step test produces a saved factor; distances match a measured route within about 3%.

**Phase 4 — GPS walk tracking**
- Location engine, filters, stationary logic, Kalman + RTS, dead reckoning, GNSS monitor, `WalkTrackingService`, walk screens (live, list, detail with route and charts), splits, learned stride from GPS, unfinished-walk recovery, Quick Settings tile, voice feedback.
- Accept: 1 km outdoor walk gives distance within about 1–3%, route is smooth, standing still adds no distance, tunnel/indoor dropout is bridged.

**Phase 5 — Goals, reminders, reliability**
- Reminder scheduler, inactivity alerts, goal notification, channels, watchdog worker, interrupted markers, battery optimization flow, OEM guide, permission-loss handling, vehicle filter.
- Accept: reminders every 2 h inside the window, none after goal reached; watchdog restarts a killed service; alarms survive reboot.

**Phase 6 — Extras and polish**
- Widget, achievements, records, streaks, monthly summaries, export/import, daily backup, app lock, delete-all, debug screen and accuracy test mode, crash log viewer, small settings, splash screen, launcher icon (adaptive + monochrome), final UI polish and full test pass.
- Accept: every item in Sections 12–14 is implemented; all automated tests pass; manual checklist ready.

**Phase 7 — Release build**
- Signing config (keystore path read from `local.properties` / env, never committed), R8 rules for Room/Hilt/kotlinx.serialization/Glance, signed release APK, versioning (`versionCode`, `versionName 1.0.0`), lint clean, final README with the manual checklist.
- Accept: release APK installs and behaves the same as debug.

---

## 16. Git and GitHub workflow

Goal: a clean, safe history where `main` always builds, every phase is a checkpoint I can return to, and no secret or build output ever reaches GitHub.

### 16.1 Repository facts
- Remote: `https://github.com/KanakSaini04/Walklog.git` (private), remote name `origin`, default branch `main`.
- OS and shell: Windows, PowerShell inside Android Studio's Terminal. Use `.\gradlew` (not `./gradlew`).
- You (the agent) do not push. You prepare the commits and print the exact commands; I run them. Never put tokens, passwords or credentials in any command or file.
- `SPEC.md` (a copy of this file) lives in the repo root so every session can re-read it.

### 16.2 What must and must not be committed

| Commit (always) | Never commit (ignored) |
|---|---|
| Source code (`app/src`), `build.gradle.kts`, `settings.gradle.kts` | Build output: `build/`, `*.apk`, `*.aab`, `.cxx` |
| Gradle wrapper: `gradlew`, `gradlew.bat`, `gradle/wrapper/*` | `.gradle/` cache |
| `gradle/libs.versions.toml` (all versions) | `local.properties` (contains my SDK path) |
| Room schema JSON in `app/schemas/` (needed for migration tests) | Signing files: `*.jks`, `*.keystore`, `keystore.properties` |
| `SPEC.md`, `README.md`, `.gitignore`, `.gitattributes` | IDE files: `.idea/`, `*.iml` |
| `.github/workflows/*.yml` (if used) | OS junk, logs, heap dumps, exported user backups |
| Proguard/R8 rule files, resources, launcher icons | Any real user data: `walklog-backup*.json`, `walklog-export*.csv` |

### 16.3 `.gitignore` (create in the project root, exact content)

```gitignore
# ===== Build output (generated, huge, rebuilt any time) =====
build/
*.apk
*.aab
*.ap_
*.dex
*.class
bin/
gen/
out/
/captures
.externalNativeBuild
.cxx

# ===== Gradle (cache; the wrapper files are still committed) =====
.gradle/
!gradle/wrapper/gradle-wrapper.jar

# ===== Local machine configuration (paths differ per computer) =====
local.properties

# ===== Signing and secrets (NEVER COMMIT; losing or leaking these is serious) =====
*.jks
*.keystore
keystore.properties
signing.properties
secrets.properties
*.pem
*.p12
*.key
.env
.env.*

# ===== IDE files (personal editor state) =====
.idea/
*.iml
*.ipr
*.iws
.vscode/
.fleet/

# ===== Operating system files =====
.DS_Store
Thumbs.db
desktop.ini

# ===== Logs, dumps and temp files =====
*.log
*.tmp
*.hprof
tmp/

# ===== App data exports (contain my location and health history) =====
walklog-backup*.json
walklog-export*.csv
```

Why each group exists:
- **Build output:** regenerated by Gradle; committing it bloats the repo and causes merge noise.
- **Gradle cache:** machine-specific; the wrapper stays so anyone can build with one command.
- **`local.properties`:** holds my Android SDK path; wrong on any other computer.
- **Signing and secrets:** anyone with the keystore and its passwords can sign updates as me. Keep them out of Git and back them up separately (see 17).
- **IDE files:** personal window layout and caches; not part of the app.
- **App data exports:** my walking and GPS history is private even in a private repo.

### 16.4 `.gitattributes` (create in the project root, exact content)

```gitattributes
# Normalize line endings so Windows (CRLF) and Linux (CI) do not fight
* text=auto
*.kt text eol=lf
*.kts text eol=lf
*.xml text eol=lf
*.md text eol=lf
*.json text eol=lf
gradlew text eol=lf
*.bat text eol=crlf

# Binary files: never diff or convert
*.png binary
*.jpg binary
*.webp binary
*.jar binary
*.ttf binary
*.otf binary
```
Why: on Windows Git otherwise warns "LF will be replaced by CRLF" on every commit, and `gradlew` breaks on Linux CI if it gets CRLF endings.

### 16.5 Branching and commit message convention

**Branches**
- `main` always builds and passes unit tests. Nothing broken goes to `main`.
- One branch per phase: `phase-1-core-counting`, `phase-2-ui-shell`, `phase-3-distance-calibration`, `phase-4-gps-walks`, `phase-5-reminders-reliability`, `phase-6-extras-polish`, `phase-7-release`.
- Bug fixes after a phase: `fix/<short-name>` branched from `main`.
- Merge phase branches with `--no-ff` so each phase stays visible in history.

**Commit format (Conventional Commits):** `type(scope): subject`
- Types: `feat` (new feature), `fix` (bug fix), `refactor` (no behavior change), `test`, `docs`, `build` (Gradle, dependencies), `chore` (housekeeping), `perf`, `style`.
- Subject: imperative mood ("add", not "added"), lowercase after the colon, no trailing period, at most 72 characters.
- Optional body (blank line after the subject): explain **why**, not what. Wrap at 72 characters.
- One logical change per commit. Never mix a feature with unrelated formatting. Never commit code that does not compile.
- Never use vague messages like "update", "fix stuff", "wip".

Examples:
```
feat(engine): add delta calculator with reboot handling
fix(gps): ignore points older than the last accepted fix
test(engine): cover midnight rollover on DST days
build: bump Room to latest stable
```

### 16.6 Per-phase checkpoint procedure (PowerShell)

**Phase 0 (already on `main`, do once):**
```powershell
git add SPEC.md
git commit -m "docs: add SPEC.md implementation plan"
git add .gitignore .gitattributes
git commit -m "chore: add .gitignore and .gitattributes"
git push origin main
```

**Every phase N:**
```powershell
# 1. Start the phase
git checkout main
git pull origin main
git checkout -b phase-N-name

# 2. After EACH unit of work (messages in 16.7)
git add -A
git status
git commit -m "feat(scope): message from the list"

# 3. Before finishing: tests and build must pass
.\gradlew testDebugUnitTest assembleDebug

# 4. Push the branch (this is also my backup)
git push -u origin phase-N-name

# 5. Merge into main, tag, push
git checkout main
git merge --no-ff phase-N-name -m "merge: phase N short title"
git tag -a v0.N.0 -m "Phase N complete: short title"
git push origin main --tags
```
Rules: run `git status` before every commit; if anything from the "never commit" column of 16.2 appears, stop and fix `.gitignore` first. At the end of every phase, print these commands filled in with the real branch name, tag and commit messages.

### 16.7 Commit messages per phase (use these; add `fix(...)` commits as needed)

**Phase 1 — branch `phase-1-core-counting` — tag `v0.1.0`**
```
build: add version catalog, Hilt, Room, DataStore and WorkManager
feat(theme): add Material 3 theme and app scaffolding
feat(data): add Room database with counter state, daily steps, step log and service events
feat(data): add DataStore settings skeleton
feat(engine): add StepEngine with hardware step counter listener
feat(engine): add delta calculator with reboot handling and carry fraction
test(engine): add unit tests for delta, reboot and correction factor
feat(service): add StepCounterService health foreground service and notification
feat(permissions): request activity recognition and notification permissions
feat(engine): add 30-second atomic flush and manual sync on open
feat(alarm): add midnight sync alarm plus boot and time-change receivers
feat(ui): show live step count on Home
```

**Phase 2 — branch `phase-2-ui-shell` — tag `v0.2.0`**
```
feat(theme): add Material 3 design system with dynamic color and dark theme
feat(ui): add reusable components for ring, tiles, charts and states
feat(nav): add navigation graph and bottom bar with predictive back
feat(home): add Home screen with animated ring and stat tiles
feat(history): add History screen with weekly bar chart and day list
feat(history): add Day detail screen with hourly breakdown
feat(settings): add Settings screen with profile, goal and theme
feat(onboarding): add onboarding with permission rationale flow
feat(ui): add loading, empty and error states
feat(a11y): add semantics, content descriptions and font scale support
test(ui): add Compose UI tests for Home and permission banners
```

**Phase 3 — branch `phase-3-distance-calibration` — tag `v0.3.0`**
```
feat(distance): add stride model with gender defaults and manual stride
feat(engine): add cadence estimator and cadence bands
feat(distance): add carry position profiles and correction factors
feat(calibration): add step-count calibration test with outlier rejection
feat(calibration): add known-distance stride calibration
feat(distance): add calorie calculator
feat(db): add calibration and learned stride tables with migration
feat(ui): show distance and calories on Home and History
test(distance): add tests for stride, calibration and calories
```

**Phase 4 — branch `phase-4-gps-walks` — tag `v0.4.0`**
```
feat(db): add walk session and route point tables with migration
feat(gps): add location engine with high-accuracy request
feat(gps): add point filter rules and stationary handling
feat(gps): add Kalman live smoother and RTS post-walk smoother
feat(gps): add dead reckoning with rotation vector heading
feat(gps): add GNSS quality monitor and dual-frequency detection
feat(service): add WalkTrackingService with location and health types
feat(walk): add walk recorder with pause, auto-pause and crash recovery
feat(ui): add live walk screen with hold-to-stop
feat(ui): add walks list and walk detail with route, speed chart and splits
feat(distance): learn stride per cadence band from GPS walks
feat(tile): add Quick Settings tile and voice feedback
test(gps): add tests for filter, smoothing and dead reckoning
```

**Phase 5 — branch `phase-5-reminders-reliability` — tag `v0.5.0`**
```
feat(notify): add notification channels
feat(reminders): add exact-alarm reminder scheduler with active hours
feat(reminders): add smart reminder content and goal-reached notification
feat(inactivity): add inactivity alerts
feat(work): add watchdog worker and service restart logic
feat(history): mark interrupted days and tracking gaps
feat(battery): add battery optimization flow and OEM guide screen
feat(permissions): handle permission loss with banners and alerts
feat(engine): add activity-transition vehicle filter
test(reminders): add scheduler and inactivity tests
```

**Phase 6 — branch `phase-6-extras-polish` — tag `v0.6.0`**
```
feat(widget): add Glance home screen widget
feat(achievements): add achievements, streaks and personal records
feat(history): add monthly summaries
feat(backup): add CSV and JSON export and import
feat(backup): add daily automatic backup to chosen folder
feat(security): add biometric app lock
feat(data): add delete-all-data flow
feat(debug): add debug screen, test step injection and accuracy test mode
feat(debug): add local crash log viewer
feat(settings): add units, time format, week start and reset today
feat(brand): add splash screen and adaptive launcher icon
refactor: final UI polish and cleanup
test: complete unit, migration and UI test coverage
```

**Phase 7 — branch `phase-7-release` — tag `v1.0.0`**
```
build: add release signing config reading keystore.properties
build: add R8 rules for Room, Hilt, serialization and Glance
chore: set versionCode 1 and versionName 1.0.0
docs: add README with build steps and manual test checklist
chore: fix lint issues
```
Phase 7 signing: keep the keystore file and `keystore.properties` (store path, alias, passwords) **outside Git**. The Gradle config reads them if present and builds an unsigned release otherwise.

### 16.8 Safety checks (run before every push)
```powershell
git status
git diff --staged --stat
git ls-files | Select-String -Pattern "jks|keystore|local.properties|backup.*json|\.apk"
```
The last command must print nothing. To test one path: `git check-ignore -v app/release/app-release.apk` (shows which rule ignores it).

If a secret (keystore, password, token) was committed even once: treat it as leaked, remove it from history with `git filter-repo`, and create a new key or password. Do not just delete the file in a new commit.

### 16.9 Undo and rollback cheat sheet (PowerShell)
| Situation | Command |
|---|---|
| See recent history | `git log --oneline --graph -15` |
| Discard uncommitted changes to one file | `git restore path/to/File.kt` |
| Unstage everything (keep the changes) | `git restore --staged .` |
| Undo the last commit, keep the work | `git reset --soft HEAD~1` |
| Undo a commit already pushed | `git revert <commit-hash>` |
| Park unfinished work | `git stash` then later `git stash pop` |
| Abort a bad merge | `git merge --abort` |
| Go back to a phase checkpoint | `git checkout -b recover-phase-2 v0.2.0` |
The agent must ask me before suggesting `git reset --hard`, `git clean -fd`, `git push --force` or history rewriting of pushed commits.

### 16.10 GitHub Actions and releases (optional, do after Phase 1 works)
Create `.github/workflows/build.yml` so every push builds and runs the unit tests. Check for newer versions of these actions and match the Java version to the JDK Android Studio uses:
```yaml
name: Build
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
      - run: chmod +x gradlew && ./gradlew testDebugUnitTest assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: walklog-debug-apk
          path: app/build/outputs/apk/debug/*.apk
```
For the final signed APK, do not commit it. Attach it to a GitHub Release for tag `v1.0.0` (GitHub → Releases → Draft a new release).

### 16.11 Other housekeeping
- Launcher icon: adaptive icon with monochrome layer; foreground image supplied by me (from the Gemini app). Generate through Image Asset Studio and wire `ic_launcher` / `ic_launcher_round`. Notification small icon: a simple monochrome vector footprint.
- README with: build steps, permissions explained, manual test checklist, backup/restore guide, known device quirks.

---

## 17. Manual steps I must do (list these at the end of the relevant phase)

- Grant runtime permissions; choose **precise** location.
- Disable battery optimization; follow the OEM guide for my phone brand.
- Allow exact alarms when asked.
- Create and back up the release keystore (losing it means no updates).
- Supply the logo image for the adaptive icon.
- Run the manual real-device checklist in Section 14.
- Run the git commands the agent prints at the end of each phase (Section 16.6), in Android Studio's Terminal, and check `git status` before every commit.
- Keep the keystore and `keystore.properties` out of Git and backed up in two places (for example a USB drive and cloud storage).

---

## 18. Definition of done

- All phases complete and accepted; app builds and runs on a real Android phone.
- No `TODO`, no placeholder code, no `INTERNET` permission, no deprecated APIs where replacements exist.
- Unit and UI tests pass; no crashes in the manual checklist; no lint errors.
- Step accuracy and GPS accuracy criteria met on a real walk.
- The app reads and follows this file: any deviation was reported and approved.
- Git: every phase is merged to `main` with its listed commits and tag; no secrets or build output are in the repo; `.gitignore` and `.gitattributes` match Section 16.
