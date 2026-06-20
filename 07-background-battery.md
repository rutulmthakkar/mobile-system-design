# 07 — Background Work & Battery

> **Round type: Both (HLD + LLD)**
> **HLD:** Designing a background task strategy — which API for which job, how battery restrictions affect your design
> **LLD:** WorkManager setup, chaining, constraints, foreground service types, Doze testing

---

## 1. Why This Matters in Interviews

Background work is where Android's constraints bite hardest. The wrong choice (background Service, unconstrained AlarmManager) drains battery and gets your app killed by the OS — or worse, flagged by Play Store. The right choice shows you understand the system's power management model.

**Common interview angles:**
- "How would you implement a periodic sync that works even when the app is killed?"
- "What's the difference between WorkManager and a Foreground Service?"
- "How does Doze mode affect your background tasks?"
- "A user reports their uploads stop when the screen turns off — why and how do you fix it?"
- "How would you implement a music player that keeps playing when the app is backgrounded?"
- "When would you use an expedited job vs a foreground service?"

---

## 2. The Decision Tree

```
Does the task need to run while the UI is visible?
│
├── YES → Coroutines in viewModelScope / lifecycleScope
│          (cancelled when screen leaves; that's fine)
│
└── NO (must survive backgrounding/kill)
    │
    ├── Must run IMMEDIATELY + user is aware (download, playback, call)?
    │     └── Foreground Service (with required notification)
    │
    ├── Must run SOON but can be deferred minutes?
    │     └── WorkManager Expedited Job
    │
    ├── Can be deferred? Needs constraints (network, charging)?
    │     └── WorkManager (one-time or periodic)
    │
    └── Exact time required (alarm clock, calendar reminder)?
          └── AlarmManager (setAlarmClock or setExactAndAllowWhileIdle)
```

---

## 3. Background Work APIs — Comparison

| API | Survives app kill | Survives reboot | Constraints | Immediate | Battery friendly |
|---|---|---|---|---|---|
| Coroutines (viewModelScope) | ❌ | ❌ | ❌ | ✅ | ✅ |
| **WorkManager** | ✅ | ✅ | ✅ | 🟡 (expedited) | ✅ |
| JobScheduler | ✅ | ✅ | ✅ | ❌ | ✅ |
| Foreground Service | ✅ | ❌ (need restart) | ❌ | ✅ | ❌ (always running) |
| AlarmManager | ✅ | 🟡 (setExact) | ❌ | ✅ | ❌ (wakes device) |
| Background Service | ❌ (killed by OS) | ❌ | ❌ | ✅ | ❌ |

**Background Service (plain `Service`):** Deprecated for background work since Android 8.0. The OS kills background services when memory is low. Don't use for anything that must complete.

---

## 4. WorkManager — Deep Dive

WorkManager is the recommended solution for **deferrable, guaranteed background work**. It:
- Persists work across app restarts and device reboots
- Respects system constraints (battery, network, storage)
- Handles retries with configurable backoff
- Works correctly with Doze mode (schedules during maintenance windows)
- Supports one-time and periodic tasks
- Supports chaining and parallel tasks

### 4.1 Core Concepts

```kotlin
// A Worker — the unit of work
class SyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            val repo = EntryPoint.get(applicationContext)  // or inject via HiltWorkerFactory
            repo.syncPendingChanges()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()  // triggers backoff
            else Result.failure(workDataOf("error" to e.message))
        }
    }
}
```

**Use `CoroutineWorker` over `Worker`** — it runs on `Dispatchers.Default` by default, you can switch dispatchers inside, and it cancels cleanly if WorkManager decides to stop the job.

### 4.2 Constraints

```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)      // any network
    // OR .setRequiredNetworkType(NetworkType.UNMETERED) // WiFi only
    .setRequiresBatteryNotLow(true)                     // skip if battery < 15%
    .setRequiresCharging(false)                         // don't require charging
    .setRequiresStorageNotLow(true)                     // skip if storage < threshold
    .setRequiresDeviceIdle(false)                       // don't wait for idle
    .build()
```

### 4.3 One-Time Work

```kotlin
val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(constraints)
    .setBackoffCriteria(
        BackoffPolicy.EXPONENTIAL,
        WorkRequest.MIN_BACKOFF_MILLIS,  // 10s minimum
        TimeUnit.MILLISECONDS
    )
    .setInputData(workDataOf("userId" to currentUser.id))
    .addTag("sync")
    .build()

// Enqueue — ExistingWorkPolicy prevents duplicate sync jobs
WorkManager.getInstance(context).enqueueUniqueWork(
    "user-sync",                    // unique name
    ExistingWorkPolicy.KEEP,        // KEEP: ignore if already queued
    // REPLACE: cancel existing, start new
    // APPEND: chain to existing
    syncRequest
)
```

### 4.4 Periodic Work

```kotlin
// Minimum interval is 15 minutes (OS enforced)
val periodicSync = PeriodicWorkRequestBuilder<SyncWorker>(
    repeatInterval = 1,
    repeatIntervalTimeUnit = TimeUnit.HOURS,
    flexTimeInterval = 15,                  // execute anytime in the last 15 min of the hour
    flexTimeIntervalUnit = TimeUnit.MINUTES
)
    .setConstraints(constraints)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "periodic-sync",
    ExistingPeriodicWorkPolicy.KEEP,        // don't restart if already scheduled
    periodicSync
)
```

**FlexInterval:** The work can run anytime in the last `flexTimeInterval` of the `repeatInterval`. This allows the OS to batch this work with other scheduled jobs for better battery efficiency.

### 4.5 Chaining Work

```kotlin
// Sequential chain: A → B → C
val compress = OneTimeWorkRequestBuilder<CompressWorker>().build()
val upload = OneTimeWorkRequestBuilder<UploadWorker>().build()
val notify = OneTimeWorkRequestBuilder<NotifyWorker>().build()

WorkManager.getInstance(context)
    .beginWith(compress)
    .then(upload)
    .then(notify)
    .enqueue()

// Parallel → then sequential:
// [resize, watermark] → upload
val resize = OneTimeWorkRequestBuilder<ResizeWorker>().build()
val watermark = OneTimeWorkRequestBuilder<WatermarkWorker>().build()
val upload = OneTimeWorkRequestBuilder<UploadWorker>().build()

WorkManager.getInstance(context)
    .beginWith(listOf(resize, watermark))  // run in parallel
    .then(upload)                           // upload waits for both
    .enqueue()
```

**Passing data between chained workers:**
```kotlin
class CompressWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val compressed = compressFile(inputData.getString("filePath")!!)
        return Result.success(workDataOf("compressedPath" to compressed))
    }
}

class UploadWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val path = inputData.getString("compressedPath")!!  // from previous worker
        upload(path)
        return Result.success()
    }
}
```

### 4.6 Expedited Work (Android 12+)

For important work that should run as soon as possible — not deferred, but also not requiring a foreground service notification:

```kotlin
val urgentSync = OneTimeWorkRequestBuilder<SyncWorker>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    // If device is out of expedited quota, fall back to normal work
    .build()

WorkManager.getInstance(context).enqueue(urgentSync)

// In the Worker, override getForegroundInfo for pre-Android 12 compat
// (WorkManager uses foreground service on API < 31 automatically)
override suspend fun getForegroundInfo(): ForegroundInfo {
    return ForegroundInfo(
        NOTIFICATION_ID,
        buildNotification("Syncing...")
    )
}
```

**Expedited quota:** The system grants each app a time budget for expedited jobs. Once exhausted, `OutOfQuotaPolicy` determines what happens. Apps in higher App Standby Buckets get more quota.

### 4.7 Observing Work Status

```kotlin
// Observe work by tag or unique name
WorkManager.getInstance(context)
    .getWorkInfosByTagLiveData("sync")
    .observe(viewLifecycleOwner) { workInfos ->
        val current = workInfos.firstOrNull()
        when (current?.state) {
            WorkInfo.State.RUNNING  -> showProgress()
            WorkInfo.State.SUCCEEDED -> showSuccess()
            WorkInfo.State.FAILED   -> showError(current.outputData.getString("error"))
            WorkInfo.State.ENQUEUED -> showQueued()
            else -> Unit
        }
    }

// Or with Flow
WorkManager.getInstance(context)
    .getWorkInfosByTagFlow("sync")
    .collect { workInfos -> /* handle */ }
```

### 4.8 Hilt Integration

```kotlin
// HiltWorkerFactory setup
@HiltAndroidApp
class App : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory

    override fun getWorkManagerConfiguration() =
        Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}

// Worker with injected dependencies
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: SyncRepository  // injected!
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        repository.sync()
        return Result.success()
    }
}
```

---

## 5. Foreground Services

### 5.1 What They Are

A Foreground Service is a `Service` that shows a persistent notification — signaling to the user that the app is actively doing something. The OS gives it higher priority and won't kill it under normal memory pressure.

**Required for:**
- Media playback (music player)
- Navigation (turn-by-turn)
- Long downloads where the user expects progress
- VoIP calls
- Fitness tracking (GPS + heart rate)

### 5.2 Foreground Service Types (Mandatory since Android 14)

Since Android 14 (API 34), you must declare and use a specific service type. The OS uses this to enforce permission requirements:

```xml
<!-- AndroidManifest.xml -->
<service
    android:name=".MediaPlaybackService"
    android:foregroundServiceType="mediaPlayback"  <!-- required -->
    android:exported="false" />

<!-- Other types: -->
<!-- dataSync, location, camera, microphone, phoneCall, connectedDevice, remoteMessaging, shortService, specialUse -->
```

```kotlin
class MediaPlaybackService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = buildNotification()

        // API 29+ requires type
        ServiceCompat.startForeground(
            this,
            NOTIFICATION_ID,
            notification,
            ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
        )
        return START_STICKY  // restart if killed
    }

    override fun onBind(intent: Intent?): IBinder? = null
}

// Start the service
val intent = Intent(context, MediaPlaybackService::class.java)
ContextCompat.startForegroundService(context, intent)  // works across API levels
```

**START_STICKY vs START_NOT_STICKY:**
- `START_STICKY` — system restarts service after killing it, passes `null` intent
- `START_NOT_STICKY` — system does NOT restart; use for one-shot services that can be safely abandoned
- `START_REDELIVER_INTENT` — restarts with the original intent; use for download/upload

### 5.3 Android 12+ Background Launch Restrictions

Since Android 12, apps targeting API 31+ **cannot start a foreground service from the background** (with exceptions: pending intents, high-priority FCM, etc.).

**The fix:** Use WorkManager Expedited Jobs instead. WorkManager handles the background launch restriction by scheduling via `JobScheduler` on API 31+ (which is exempt) and falling back to a foreground service on older versions.

```kotlin
// Before Android 12: you could call startForegroundService() from background
// After Android 12: use expedited WorkManager instead

val work = OneTimeWorkRequestBuilder<UploadWorker>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()
WorkManager.getInstance(context).enqueue(work)
```

---

## 6. Doze Mode & App Standby

### 6.1 Doze Mode

Activated when the **device is unplugged, stationary, and screen is off** for a period of time.

**What gets restricted in Doze:**
```
❌ Network access blocked
❌ Wake locks ignored
❌ JobScheduler / WorkManager jobs deferred
❌ AlarmManager setExact() deferred (except setAlarmClock)
✅ FCM high-priority messages still delivered
✅ Foreground services still run
✅ Phone calls still work
```

**Maintenance windows:** During Doze, the system periodically exits Doze briefly to allow deferred jobs to run. WorkManager schedules during these windows automatically.

**Testing Doze:**
```bash
# Force Doze on emulator/device
adb shell dumpsys battery unplug
adb shell dumpsys deviceidle force-idle deep

# Verify your app handles it
adb shell dumpsys battery reset  # exit Doze
```

### 6.2 App Standby Buckets (API 28+)

The system assigns each app to a bucket based on usage frequency. Higher bucket = more background work allowed.

| Bucket | Usage | Background jobs | Network access |
|---|---|---|---|
| **Active** | Currently in use | Unlimited | Unlimited |
| **Working Set** | Used daily | ~every 2 hours | ~every 2 hours |
| **Frequent** | Used weekly | ~every 8 hours | ~every 8 hours |
| **Rare** | Used < weekly | ~every 24 hours | ~every 24 hours |
| **Restricted** | Almost never used | ~every 24 hours | Very limited |

**Impact on WorkManager:** Periodic work is still executed, but the system may delay it based on the app's bucket. An app in the Rare bucket may see its 15-minute periodic work run only once every 24 hours in practice.

**Android 14+ adaptive battery:** The system uses ML to predict usage patterns and applies even stricter restrictions to apps it predicts won't be used soon.

### 6.3 Battery Saver Mode

When the user enables Battery Saver:
- Background network access blocked for apps not in foreground
- Location access reduced
- WorkManager jobs deferred further
- Sync adapters suspended

**WorkManager's `setRequiresBatteryNotLow(true)`** is the right constraint — don't run non-critical sync when battery is below ~15%.

### 6.4 App Standby vs Doze — Key Difference

| | Doze | App Standby |
|---|---|---|
| Trigger | Device stationary + screen off | App not used recently |
| Scope | All apps | Per-app |
| When active | Device level | Even when device is active |
| Reset | Device moved / screen on | User opens the app |

---

## 7. AlarmManager

Use for **exact timing requirements** — alarm clocks, calendar reminders, medication reminders.

```kotlin
val alarmManager = getSystemService(AlarmManager::class.java)
val intent = PendingIntent.getBroadcast(
    context, 0,
    Intent(context, AlarmReceiver::class.java),
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

// Exact alarm that works in Doze (API 23+)
alarmManager.setExactAndAllowWhileIdle(
    AlarmManager.RTC_WAKEUP,
    triggerAtMs,
    intent
)

// For alarm clocks — highest priority, shown in status bar
alarmManager.setAlarmClock(
    AlarmManager.AlarmClockInfo(triggerAtMs, showIntent),
    intent
)
```

**Permissions required (API 31+):**
```xml
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
<!-- Or: USE_EXACT_ALARM (auto-granted for alarm clock apps) -->
```

**Don't use AlarmManager for:** periodic network sync, data refresh, deferred work. WorkManager's constraints are better for those. AlarmManager wakes the CPU even in Doze — expensive.

---

## 8. Battery Best Practices

### 8.1 Minimize Wake-Ups

Every time your app wakes the CPU (network call, alarm, job) it burns battery. Strategies:

```kotlin
// Batch network requests — one wake-up for 10 operations vs 10 wake-ups
// Let WorkManager decide when to batch with other apps' jobs

// Use connectivity-triggered work over polling
connectivityObserver.isConnected
    .filter { it }
    .onEach { workManager.enqueue(syncRequest) }
    .launchIn(appScope)

// Avoid repeating work when not needed — idempotent sync with delta
```

### 8.2 Obey System Signals

```kotlin
// Listen for battery state changes
val batteryReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            Intent.ACTION_BATTERY_LOW -> {
                // Reduce background work, cancel non-critical sync
                WorkManager.getInstance(context).cancelAllWorkByTag("optional-sync")
            }
            Intent.ACTION_POWER_CONNECTED -> {
                // Good time for expensive operations
                WorkManager.getInstance(context).enqueue(heavySyncRequest)
            }
        }
    }
}
// Register in manifest or dynamically with lifecycleScope
```

### 8.3 Network Efficiency

```kotlin
// Use constraints — sync only on WiFi for large data
val largeDataSync = OneTimeWorkRequestBuilder<FullSyncWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.UNMETERED)  // WiFi
            .setRequiresCharging(true)                       // also charging
            .build()
    ).build()
```

---

## 9. React Native Background Work

React Native runs on the JS thread — backgrounding kills it. Options:

```typescript
// @react-native-community/async-storage — data persists, but no background execution

// react-native-background-fetch — periodic background fetch (iOS + Android)
import BackgroundFetch from 'react-native-background-fetch'
BackgroundFetch.configure(
    { minimumFetchInterval: 15 },  // minutes
    async (taskId) => {
        await syncData()
        BackgroundFetch.finish(taskId)
    }
)

// Headless JS (Android-only) — run JS when app is in background
// Register in index.js:
AppRegistry.registerHeadlessTask('SyncTask', () => require('./SyncTask'))

// Native module wrapping WorkManager (preferred for Android)
// Use react-native-work-manager or write a native module
```

**For media playback in RN:** Use `react-native-track-player` — wraps ExoPlayer (Android) and AVPlayer (iOS) with background playback support and notification controls.

---

## 10. HLD vs LLD Framing

### HLD Questions
- "Design a background sync system for an offline-first messaging app"
- "How do you ensure uploads complete even on poor networks?"

**HLD answer covers:**
1. WorkManager for guaranteed persistent execution
2. Outbox pattern in Room for pending operations queue
3. Constraints: `CONNECTED`, `BATTERY_NOT_LOW`
4. Expedited jobs for urgent messages (send on tap)
5. FCM as wake-up signal when device is idle
6. Retry with exponential backoff
7. Doze-aware: WorkManager handles maintenance windows

### LLD Questions
- "Implement a WorkManager job that uploads images with retry"
- "How do you show upload progress to the user?"
- "How does your Worker pass results back to the UI?"

**LLD answer covers:**
1. `CoroutineWorker` with `setProgress()` for incremental updates
2. `WorkInfo.State` observation via Flow
3. Chain: compress → upload → notify
4. `getForegroundInfo()` for pre-API 31 compat
5. `workDataOf` for input/output between workers

---

## 11. Common Misunderstandings & Pitfalls

**❌ Using a plain `Service` for background work on modern Android**
Background services are killed by the OS under memory pressure since Android 8.0. Use WorkManager for deferrable work, foreground service for user-visible immediate work.

**❌ Assuming WorkManager runs immediately**
WorkManager is designed for deferrable work — it may run minutes later, especially in Doze or if the device is in a low App Standby bucket. For time-critical work (sending a user's message), use expedited jobs or a foreground service.

**❌ Setting minimum periodic interval < 15 minutes**
The OS enforces a minimum of 15 minutes for PeriodicWorkRequest. Setting a shorter interval silently snaps to 15 minutes. If you need sub-15-minute updates, use a Foreground Service or push-based approach (FCM/WebSocket).

**❌ Not declaring foreground service type on Android 14+**
API 34 requires `android:foregroundServiceType` in the manifest AND the matching type passed to `startForeground()`. Missing this causes a `SecurityException` crash at runtime.

**❌ Starting foreground service from background on Android 12+**
Calling `startForegroundService()` while the app is not in the foreground throws `ForegroundServiceStartNotAllowedException`. Use expedited WorkManager instead — it's exempt from this restriction.

**❌ Using AlarmManager for periodic background sync**
AlarmManager wakes the CPU on every trigger — expensive for battery. For sync, use WorkManager with `CONNECTED` constraint and let the OS batch it with other work.

**❌ `setRequiresCharging(true)` for critical tasks**
If a task is critical (must happen today), don't require charging — the user may never plug in. Use `setRequiresBatteryNotLow(true)` instead (runs if battery > ~15%).

**❌ Not testing with Doze simulation**
A task that works perfectly in development (device awake, screen on) can fail completely in production when Doze kicks in. Always test with `adb shell dumpsys deviceidle force-idle deep`.

**❌ Creating a new WorkManager task for every user action**
If the user taps sync 5 times, you don't want 5 queued sync jobs. Use `enqueueUniqueWork` with `ExistingWorkPolicy.KEEP` — only one sync in the queue at a time.

---

## 12. Best Practices

- **WorkManager for anything that must survive app death** — sync, upload, cleanup, notifications
- **`CoroutineWorker` over `Worker`** — coroutine cancellation is cleaner, better structured concurrency
- **`enqueueUniqueWork` / `enqueueUniquePeriodicWork`** — always name your work to prevent duplicates
- **Expedited jobs over foreground services** for short urgent tasks on API 31+ — no visible notification required
- **Declare foreground service type** — mandatory on API 34+, good practice on API 29+
- **Test with Doze simulation** — `adb shell dumpsys deviceidle force-idle deep` before every release
- **`setRequiresBatteryNotLow`** instead of `setRequiresCharging` for non-critical sync
- **Constraints on all WorkManager tasks** — at minimum `CONNECTED` for network work
- **Set `FlexInterval` on periodic work** — lets OS batch with other jobs, friendlier to battery
- **Observe work status via `getWorkInfosByTagFlow`** — reactive UI without polling
- **Inject dependencies via `HiltWorkerFactory`** — don't construct repositories inside the Worker
- **`Result.retry()` with attempt check** — `if (runAttemptCount < 3) retry() else failure()` — don't retry forever
- **Use `workDataOf` for small payloads only** — WorkManager data is capped at 10KB; for large data, pass a Room ID and fetch in the Worker

---

## 13. Interview Q&A

**Q1: What's the difference between WorkManager and a Foreground Service?**

> WorkManager is for deferrable work that must complete but doesn't need to happen immediately and doesn't need to be visible to the user — sync, upload, cleanup. The OS schedules it during appropriate windows, respecting Doze, battery state, and network constraints. A Foreground Service is for work that must run immediately and the user must know about — music playback, active navigation, ongoing download. It requires a persistent notification. The practical rule: if the user tapped something and expects immediate background execution, foreground service (or expedited WorkManager job on API 31+). If it can happen anytime in the next few hours, WorkManager.

---

**Q2: How does Doze mode affect WorkManager, and how do you handle it?**

> In Doze mode — device unplugged, stationary, screen off — the OS blocks network access and defers jobs. WorkManager jobs are paused and queued until the system exits Doze for a maintenance window, then executes them. WorkManager handles this automatically — you don't do anything special. The key is designing tasks to be idempotent and tolerant of delay. If a task is truly urgent (high-priority notification, outgoing message), use a high-priority FCM message to wake the device, which is exempt from Doze restrictions, then execute the work. I always test Doze behavior with `adb shell dumpsys deviceidle force-idle deep` — tasks that work in development often silently fail in production Doze conditions.

---

**Q3: A user reports that file uploads stop when the screen turns off. What's happening and how do you fix it?**

> The upload is running in a coroutine scoped to a ViewModel or Activity — when the screen turns off and the OS eventually pauses the app, the coroutine is cancelled. The fix is to move the upload to WorkManager. The upload Worker survives app backgrounding and device restarts. I'd set constraints for `CONNECTED` network, use `CoroutineWorker` for structured concurrency, implement `getForegroundInfo()` to show a progress notification on older Android versions, and use `setExpedited()` on API 31+ so it starts immediately without requiring a foreground service. I'd also chain Workers: compress → upload → update DB → notify user. Each step is independent, and failure in any step triggers retry only for that step.

---

**Q4: When would you use AlarmManager over WorkManager?**

> AlarmManager for exact timing that can't be deferred — an alarm clock that must fire at 7:00am exactly, a medication reminder at a specific scheduled time, a calendar event alert. `setAlarmClock()` or `setExactAndAllowWhileIdle()` fires even in Doze. WorkManager for everything else — it's more battery-efficient because the OS can batch it with other work. The anti-pattern is using AlarmManager for periodic sync or data refresh — that wakes the CPU on every trigger even in Doze. For that, WorkManager with constraints is the right tool.

---

**Q5: What is App Standby and how does it affect background work?**

> App Standby buckets classify apps by usage frequency — Active, Working Set, Frequent, Rare, and Restricted. The less frequently used an app, the fewer background resources it gets. A Rare-bucket app might see its 15-minute periodic WorkManager job run only once every 24 hours in practice. There's no API to directly control which bucket your app is in — it's determined by how often the user opens it. The indirect solution: design features that encourage regular engagement to keep the app in a higher bucket. Also, FCM high-priority messages are exempt from App Standby restrictions — a valid use case is receiving a FCM wake-up that triggers a WorkManager job, bypassing bucket throttling.

---

**Q6: How do you chain background tasks in WorkManager?**

> WorkManager supports sequential and parallel chains. Sequential: `beginWith(taskA).then(taskB).then(taskC).enqueue()` — task B only starts when A succeeds. Parallel then sequential: `beginWith(listOf(taskA, taskB)).then(taskC).enqueue()` — A and B run in parallel, C waits for both. Data flows between workers via `inputData` and `outputData` — each worker reads from `inputData` and returns `Result.success(workDataOf("key" to value))`. Downstream workers receive merged output data. If any worker returns `Result.failure()`, the chain stops — downstream workers are marked CANCELLED. If a worker returns `Result.retry()`, only that worker retries — upstream work isn't re-executed.

---

*Previous: [06 — Performance](./06-performance.md)*
*Next: [08 — Threading & Concurrency](./08-threading-concurrency.md)*
