# 09 — Observability

> **Round type: HLD (primary) + Both**
> **HLD:** Designing an observability stack for a production mobile app — crash reporting, performance monitoring, analytics, alerting
> **LLD:** Instrumentation code, custom traces, log levels, structured events, non-fatal reporting

---

## 1. Why This Matters in Interviews

Observability is what separates a hobby app from a production system. Interviewers ask about it to test whether you think beyond feature development — do you know what happens after the app ships? Senior engineers design systems to be observable from day one, not as an afterthought.

**Common interview angles:**
- "How do you know your app is healthy in production?"
- "A user reports a crash you can't reproduce locally. How do you debug it?"
- "How do you detect a performance regression after a release?"
- "What metrics would you monitor for a video streaming feature?"
- "How do you distinguish a spike in crashes from normal variance?"
- "How would you design an analytics system for a social media app?"

---

## 2. The Three Pillars of Mobile Observability

```
┌──────────────────────────────────────────────────────────────┐
│                   Mobile Observability                        │
├──────────────────┬──────────────────┬────────────────────────┤
│     CRASHES      │   PERFORMANCE    │      ANALYTICS         │
│                  │                  │                         │
│ Crash-free rate  │ Startup time     │ User behaviour          │
│ ANR rate         │ Frame rate / FPS │ Conversion funnels      │
│ Non-fatal errors │ Network latency  │ Feature adoption        │
│ Native crashes   │ Screen load time │ Retention / DAU         │
│ Stack traces     │ API error rates  │ Business metrics        │
└──────────────────┴──────────────────┴────────────────────────┘
```

All three answer different questions:
- **Crashes:** Is the app broken?
- **Performance:** Is the app slow?
- **Analytics:** Are users doing what we expect?

---

## 3. Crash Reporting

### 3.1 What to Capture

A crash report is useless without context. Minimum required:

```
Stack trace           → where did it crash (deobfuscated)
Device info           → manufacturer, model, OS version, RAM
App version + build   → which release introduced it
User identifier       → how many unique users affected
Breadcrumbs           → what the user did before the crash
Custom keys           → app state at crash time (screen name, feature flag state)
Threads               → full thread dump (especially for ANRs)
Session data          → was this a fresh launch or 30-minute session?
```

### 3.2 Firebase Crashlytics ✅ Default choice

```kotlin
// gradle
implementation(platform("com.google.firebase:firebase-bom:32.7.0"))
implementation("com.google.firebase:firebase-crashlytics-ktx")
implementation("com.google.firebase:firebase-analytics-ktx")  // required for Crashlytics

// Enable mapping file upload (deobfuscation)
// In app/build.gradle.kts — Gradle plugin handles this
plugins { id("com.google.firebase.crashlytics") }

android {
    buildTypes {
        release {
            // R8 mapping uploaded automatically by Crashlytics Gradle plugin
            isMinifyEnabled = true
        }
    }
}
```

**Non-fatal error reporting:**
```kotlin
// Report caught exceptions that don't crash but indicate something wrong
try {
    val result = parseResponse(rawJson)
    processResult(result)
} catch (e: JsonParseException) {
    // App recovered but this is unexpected — report for monitoring
    FirebaseCrashlytics.getInstance().recordException(e)
    showFallbackUI()
}
```

**Custom keys — enrich crash reports with app state:**
```kotlin
fun setUserContext(userId: String, screen: String) {
    val crashlytics = FirebaseCrashlytics.getInstance()
    crashlytics.setUserId(userId)                          // which user affected
    crashlytics.setCustomKey("current_screen", screen)     // which screen they were on
    crashlytics.setCustomKey("feature_flag_new_feed", featureFlags.isNewFeedEnabled)
    crashlytics.setCustomKey("network_type", getNetworkType())
    crashlytics.setCustomKey("memory_available_mb", getAvailableMemoryMB())
}

// Call on every screen navigation
navController.addOnDestinationChangedListener { _, destination, _ ->
    setUserContext(currentUserId, destination.route ?: "unknown")
}
```

**Breadcrumbs — custom log entries before crash:**
```kotlin
// Log significant events — they appear in the crash report timeline
FirebaseCrashlytics.getInstance().log("User tapped checkout button")
FirebaseCrashlytics.getInstance().log("Payment API call started")
// If it crashes during payment, you'll see the breadcrumb trail
```

### 3.3 ANR Reporting

ANR = Application Not Responding. Main thread blocked > 5 seconds.

Crashlytics automatically captures ANRs (since version 18.3.0). The report includes:
- The full main thread stack trace at the moment of ANR
- Device state (CPU, memory)
- What the user was doing

**Monitoring ANR rate:**
```
Target: ANR rate < 0.47% of daily sessions (Play Console "bad behavior" threshold)
Alert when: ANR rate increases > 10% week-over-week
```

**Testing ANR detection locally:**
```kotlin
// Force an ANR for testing — never in production
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    Thread.sleep(6000)  // ← causes ANR for testing only
}
```

### 3.4 Sentry — Alternative / Complement

```kotlin
// gradle
implementation("io.sentry:sentry-android:7.9.0")

// AndroidManifest.xml
<meta-data android:name="io.sentry.dsn" android:value="https://your-dsn@sentry.io/project" />
<meta-data android:name="io.sentry.traces-sample-rate" android:value="0.2" /> <!-- 20% trace sampling -->
<meta-data android:name="io.sentry.session-replay.on-error-sample-rate" android:value="1.0" />

// Manual capture
Sentry.captureException(exception)
Sentry.captureMessage("Checkout flow started")

// Transactions (performance tracing)
val transaction = Sentry.startTransaction("checkout", "user.action")
val span = transaction.startChild("payment_api_call")
try {
    val result = paymentApi.processPayment(request)
    span.finish(SpanStatus.OK)
} catch (e: Exception) {
    span.finish(SpanStatus.INTERNAL_ERROR)
    Sentry.captureException(e)
} finally {
    transaction.finish()
}
```

### 3.4 Bugsnag (SmartBear Insight Hub)

Bugsnag is used in production by Airbnb, Slack, Lyft, Netflix, and Pinterest. Its core philosophy differs from Crashlytics and Sentry — it frames error monitoring as a **stability management decision**: "Should I ship new features or fix bugs right now?"

```kotlin
// gradle
implementation("com.bugsnag:bugsnag-android:6.+")

// AndroidManifest.xml
<meta-data android:name="com.bugsnag.android.API_KEY" android:value="your-api-key"/>

// Optional: manual init with config
Bugsnag.start(this, Configuration.load(this).apply {
    releaseStage = if (BuildConfig.DEBUG) "development" else "production"
    enabledReleaseStages = setOf("production", "staging")
    addOnError { event ->
        event.addMetadata("account", "tier", userTier)
        event.addMetadata("app", "screen", currentScreen)
        true  // return false to discard the event
    }
})

// Non-fatal
try {
    parseResponse(rawJson)
} catch (e: JsonParseException) {
    Bugsnag.notify(e)
}

// Breadcrumbs
Bugsnag.leaveBreadcrumb("Checkout started", mapOf("cart_value" to cartValue), BreadcrumbType.USER)

// Session tracking — powers the stability score
Bugsnag.startSession()   // on foreground
Bugsnag.pauseSession()   // on background
```

**Bugsnag's signature feature — Stability Score:**
Calculates `(error-free sessions / total sessions) × 100` per release. A 99.5% stability score means 0.5% of sessions had an error. You set a target (e.g., 99.5%) and a critical threshold (e.g., 99.0%). The release dashboard compares each build against the previous — immediately surfaces regressions. Answers: "Is this release stable enough to keep rolling out?"

**Intelligent error grouping:** Bugsnag clusters errors by root cause, not just stack trace similarity — widely considered best-in-class for reducing noise on high-volume apps.

### 3.5 Crashlytics vs Bugsnag vs Sentry

| | Firebase Crashlytics | Bugsnag | Sentry |
|---|---|---|---|
| Cost | **Free** | Paid (limited free) | Free tier + paid |
| Setup | Gradle plugin (auto) | Simple SDK | sentry-cli for mapping |
| ANR monitoring | ✅ (v18.3+) | ✅ | ✅ |
| Performance tracing | Via Firebase Perf | ✅ Built-in | ✅ Built-in |
| Session replay | ❌ | ❌ | ✅ |
| Stability score | ❌ | ✅ Signature feature | ❌ (release health) |
| Error grouping | Good | **Best-in-class** | Good |
| Jira integration | One-way | Two-way ✅ | Two-way ✅ |
| NDK crashes | ✅ | ✅ | ✅ |
| React Native | ✅ | ✅ | ✅ |
| Full-stack (web+mobile) | ❌ | ✅ 50+ platforms | ✅ |
| Compliance (SOC2/HIPAA) | Via Google | ✅ Enterprise | ✅ Enterprise |
| Who uses it | Most mobile teams | Airbnb, Slack, Netflix | Atlassian, Disney |
| Best for | Zero-config mobile crashes | Release-gated stability, enterprise | Full observability web + mobile |

**Decision guide:**
- **Crashlytics** → default for mobile, free, Firebase already in stack, small-to-mid team
- **Bugsnag** → enterprise teams that need "ship vs fix" stability dashboards, full-stack error tracking, best error grouping
- **Sentry** → unified crash + performance + session replay, web and mobile in one platform

---

## 4. Performance Monitoring

### 4.1 Firebase Performance Monitoring

Automatically tracks:
- App startup time (cold, warm)
- HTTP/HTTPS network requests (latency, payload size, success rate)
- Screen rendering (slow/frozen frames)

```kotlin
implementation("com.google.firebase:firebase-perf-ktx")

// Custom trace — measure any operation
val trace = Firebase.performance.newTrace("feed_load")
trace.start()
try {
    val feed = repository.loadFeed()
    trace.putAttribute("item_count", feed.size.toString())
    trace.putMetric("cache_hits", cacheHitCount.toLong())
} finally {
    trace.stop()  // trace duration auto-calculated
}

// Custom HTTP metric (for non-OkHttp requests)
val metric = Firebase.performance.newHttpMetric(
    "https://api.example.com/feed",
    FirebasePerformance.HttpMethod.GET
)
metric.start()
// ... execute request
metric.setHttpResponseCode(200)
metric.setResponsePayloadSize(responseBytes.size.toLong())
metric.stop()
```

**OkHttp integration — automatic HTTP tracing:**
```kotlin
// Add FirebasePerfOkHttpClient to intercept all OkHttp requests
val client = OkHttpClient.Builder()
    .addInterceptor(FirebasePerfInterceptor())  // auto-traces all requests
    .build()
```

### 4.2 Custom Performance Metrics to Track

**Startup metrics:**
```kotlin
// Report TTFD (Time to Full Display) — when the user can actually interact
// Use reportFullyDrawn() to mark when content is ready
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    viewLifecycleOwner.lifecycleScope.launch {
        viewModel.uiState.first { it.data != null }  // wait for content
        activity?.reportFullyDrawn()  // tells system TTFD
        // Firebase Performance also captures this
    }
}
```

**Network request metrics (custom):**
```kotlin
class ObservabilityInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val startTime = System.currentTimeMillis()
        val response = chain.proceed(request)
        val latencyMs = System.currentTimeMillis() - startTime

        analytics.trackNetworkRequest(
            endpoint = request.url.encodedPath,
            statusCode = response.code,
            latencyMs = latencyMs,
            requestSizeBytes = request.body?.contentLength() ?: 0,
            responseSizeBytes = response.body?.contentLength() ?: 0
        )
        return response
    }
}
```

**Key metrics by feature type:**

| Feature | Key metrics |
|---|---|
| Feed | Load time P50/P95, scroll FPS, items loaded |
| Video player | Buffering time, rebuffer rate, playback errors, quality switches |
| Search | Query latency, results count, no-results rate |
| Checkout | Funnel conversion, payment API latency, error rate |
| Auth | Login latency, failure rate by error type |
| Image upload | Upload speed, success rate, compression ratio |

---

## 5. Analytics & Event Tracking

### 5.1 Event Schema Design

Events should answer: What happened, where, when, and by whom?

```kotlin
// Good event naming: noun_verb (past tense action)
// Bad: "click", "tap", "action"
// Good: "feed_post_liked", "checkout_step_completed", "video_playback_started"

// Structured event with properties
fun trackFeedPostLiked(postId: String, position: Int, isFollowing: Boolean) {
    analytics.logEvent("feed_post_liked") {
        param("post_id", postId)
        param("position_in_feed", position.toLong())
        param("author_is_following", isFollowing.toString())
        param("screen_name", "home_feed")
        param("app_version", BuildConfig.VERSION_NAME)
        // Don't include PII: user email, phone, full name
    }
}
```

### 5.2 Firebase Analytics

```kotlin
// Auto-collected events (no code needed):
// first_open, session_start, screen_view (with Navigation component),
// app_update, os_update, in_app_purchase

// Custom events
firebaseAnalytics.logEvent("video_playback_started") {
    param(FirebaseAnalytics.Param.CONTENT_TYPE, "episode")
    param(FirebaseAnalytics.Param.ITEM_ID, episodeId)
    param("episode_number", episodeNumber.toLong())
    param("playback_quality", "hd")
    param("connection_type", getConnectionType())
}

// Screen tracking — automatic with Jetpack Navigation
// Manual:
firebaseAnalytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
    param(FirebaseAnalytics.Param.SCREEN_NAME, "checkout_payment")
    param(FirebaseAnalytics.Param.SCREEN_CLASS, "CheckoutPaymentFragment")
}

// User properties — segment users
firebaseAnalytics.setUserProperty("subscription_tier", "premium")
firebaseAnalytics.setUserProperty("preferred_language", "en")
```

### 5.3 Analytics Abstraction Layer

Never call Firebase Analytics directly from ViewModels or business logic. Wrap it:

```kotlin
// Interface — swap implementation without changing call sites
interface Analytics {
    fun trackEvent(name: String, properties: Map<String, Any> = emptyMap())
    fun trackScreen(screenName: String)
    fun setUserProperty(key: String, value: String)
    fun setUserId(userId: String)
}

// Firebase implementation
class FirebaseAnalyticsImpl(
    private val firebase: FirebaseAnalytics
) : Analytics {
    override fun trackEvent(name: String, properties: Map<String, Any>) {
        firebase.logEvent(name) {
            properties.forEach { (key, value) ->
                when (value) {
                    is String -> param(key, value)
                    is Long   -> param(key, value)
                    is Double -> param(key, value)
                    is Int    -> param(key, value.toLong())
                }
            }
        }
    }
}

// No-op for tests — don't pollute analytics with test data
class NoOpAnalytics : Analytics {
    override fun trackEvent(name: String, properties: Map<String, Any>) = Unit
    override fun trackScreen(screenName: String) = Unit
    override fun setUserProperty(key: String, value: String) = Unit
    override fun setUserId(userId: String) = Unit
}
```

### 5.4 Event Design Principles

**Consistent naming convention:**
```
Screen: noun (feed, checkout, profile, search)
Action: screen_object_action (feed_post_liked, checkout_address_saved)
Error: feature_error (payment_error, login_error)
```

**What to track:**
```
✅ User actions (tap, swipe, submit)
✅ Feature exposure (did user see the paywall?)
✅ Funnel steps (step 1, step 2, completed, abandoned)
✅ Errors the user experienced (not just crashes)
✅ Performance the user experienced (video buffered, slow load)

❌ PII (email, phone, exact location without consent)
❌ Every keystroke (too noisy, privacy risk)
❌ Internal technical events (cache hit, thread pool size)
❌ Events with no planned analysis (collect only what you'll use)
```

---

## 6. Logging

### 6.1 Log Levels

```kotlin
// Android Log levels — use appropriately
Log.v("TAG", "Verbose")    // Everything — very noisy. Dev only.
Log.d("TAG", "Debug")      // Debug info. Dev + staging. Strip from release with ProGuard.
Log.i("TAG", "Info")       // Normal app events. Appropriate for production.
Log.w("TAG", "Warning")    // Unexpected but recoverable. Always log.
Log.e("TAG", "Error")      // Errors. Always log. Consider sending to crash reporter.

// Timber — wrapper with automatic tag, level filtering
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())   // logs to logcat in debug
        } else {
            Timber.plant(CrashlyticsTree())    // sends WARN+ to Crashlytics in release
        }
    }
}

// Custom Crashlytics tree
class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.WARN) return  // only WARN and above in production

        FirebaseCrashlytics.getInstance().log("$tag: $message")  // breadcrumb
        if (t != null && priority == Log.ERROR) {
            FirebaseCrashlytics.getInstance().recordException(t)  // report errors
        }
    }
}
```

### 6.2 What NOT to Log

```kotlin
// ❌ NEVER log sensitive data
Log.d("Auth", "Token: ${user.authToken}")    // token in logcat → security breach
Log.d("Payment", "Card: ${card.number}")     // PCI violation

// ✅ Log safe representations
Log.d("Auth", "Token acquired for user: ${user.id}")
Log.d("Payment", "Card ending in: ${card.lastFour}")
```

---

## 7. Alerting Strategy

### 7.1 Key Thresholds to Alert On

```
Crash-free rate:
  Alert: < 99.5% (goal: > 99.9% for most apps)
  Critical: < 99% (immediate investigation)

ANR rate:
  Alert: > 0.47% of sessions (Play Console bad behavior threshold)
  
Startup time (P95 cold start):
  Alert: > 5 seconds
  
Network error rate (per endpoint):
  Alert: > 2% error rate on critical paths (auth, checkout)
  
Custom metric — video buffering:
  Alert: > 5% of playback sessions experience buffering > 3s
```

### 7.2 Velocity Alerting

Alert when a crash **newly appears or suddenly spikes** — not just on absolute rate.

```
Velocity alert: crash X affected 0 users yesterday, now affects 500 users today
  → Likely introduced in today's release
  → Immediate rollback candidate
```

Firebase Crashlytics and Sentry both support velocity alerts natively.

### 7.3 Release-Gated Monitoring

```
After every release:
1. Monitor crash-free rate — compare this version vs previous
2. Monitor ANR rate — any increase?
3. Monitor P95 startup time — any regression?
4. Monitor top 3 API error rates — anything new?

Automated: set up Slack/PagerDuty alerts for any metric that degrades > X%
Gate: if crash-free rate drops > 0.5% vs previous version within 2 hours,
      trigger rollback review process
```

---

## 8. Android Vitals (Play Console)

Google Play automatically collects crash and ANR data from opted-in users. Available in Play Console → Android Vitals.

**Key metrics:**
- **Crash rate** — % of daily active users experiencing a crash
- **ANR rate** — % of daily active users experiencing an ANR
- **Bad behaviors** — crash rate > 1.09% or ANR rate > 0.47% triggers Play Store badge

**Limitations:**
- 24–48 hour data delay
- No stack trace grouping or custom alerting
- No custom metrics

Android Vitals is a **lagging indicator** — good for trends, not real-time alerts. Use Crashlytics/Sentry for real-time monitoring.

---

## 9. HLD — Observability Architecture

### "Design the observability stack for a social media app with 10M users"

```
Data Collection Layer (on device):
├── Crashlytics SDK          → crashes, ANRs, non-fatals
├── Firebase Performance     → startup, HTTP latency, custom traces
├── Firebase Analytics       → user events, funnels, retention
└── Custom OkHttp interceptor → API error rates, latency percentiles

Data Transmission:
├── Crash reports → sent immediately (high priority, small payload)
├── Performance traces → batched, sent when CONNECTED + UNMETERED (or on WiFi)
├── Analytics events → batched, sent every session or on background
└── All data: stored locally if offline, flushed on reconnect

Backend Processing:
├── Crashlytics / Sentry → grouping, deduplication, alerts
├── BigQuery (via Firebase) → raw event export for custom analysis
├── Grafana / Datadog → custom dashboards for business + engineering
└── PagerDuty / Slack → alert routing

Alerting Rules:
├── Crash-free rate < 99.5% → Slack warning
├── Crash-free rate < 99.0% → PagerDuty page
├── New crash affecting > 1000 users → immediate Slack + PagerDuty
├── P95 startup > 5s → Slack warning
└── Any API error rate > 5% → PagerDuty
```

---

## 10. React Native Observability

```typescript
// Sentry for React Native
import * as Sentry from '@sentry/react-native'

Sentry.init({
    dsn: 'https://your-dsn@sentry.io/project',
    tracesSampleRate: 0.2,   // 20% of transactions traced
    environment: __DEV__ ? 'development' : 'production',
    // Attach breadcrumbs automatically for navigation
    integrations: [new Sentry.ReactNativeTracing()]
})

// Wrap root component
export default Sentry.wrap(App)

// Manual capture
Sentry.captureException(error)
Sentry.captureMessage('Checkout started')

// Add context
Sentry.setUser({ id: userId })
Sentry.setTag('feature_flag', 'new_checkout_v2')

// Firebase Analytics in RN
import analytics from '@react-native-firebase/analytics'
await analytics().logEvent('feed_post_liked', {
    post_id: postId,
    position: index
})
await analytics().setUserId(userId)
await analytics().setUserProperty('subscription_tier', 'premium')
```

---

## 11. Common Misunderstandings & Pitfalls

**❌ Only monitoring crashes, ignoring non-fatals**
Caught exceptions that the app recovers from are still signals of problems — parse failures, unexpected API shapes, feature degradations. Report them to Crashlytics with `recordException()`. They appear in the non-fatals dashboard and can reveal systemic issues before they become crashes.

**❌ Not uploading R8/ProGuard mapping files**
A crash stacktrace in production without mapping file shows `a.b.c()` instead of `UserRepository.fetchUser()`. The Crashlytics Gradle plugin uploads mapping automatically — don't disable it. Save mapping files per release even if you don't use Crashlytics.

**❌ Logging PII in crash reports or analytics**
User email, phone number, payment details, or health data in crash reports creates GDPR/CCPA liability. Use user IDs (not names), mask card numbers, never include tokens or passwords. Audit all `setCustomKey()` and `logEvent()` calls before each release.

**❌ Tracking events without a plan**
Logging every possible event creates noise that's expensive to store and impossible to analyze. Define a measurement plan before implementing: what question does each event answer? What action will you take based on it?

**❌ Using raw Firebase Analytics without an abstraction layer**
Calling `FirebaseAnalytics.logEvent()` directly from ViewModels couples business logic to an analytics SDK. When you switch analytics providers (or add a second one), you rewrite everything. Use an interface — ViewModel calls `analytics.trackEvent()`, implementation decides where it goes.

**❌ Not testing observability in staging**
Crash reports and analytics events look different in production (obfuscated, batched, delayed) than in development. Use Firebase's debug mode (`adb shell setprop debug.firebase.analytics.app com.example`) to see events in real-time in staging.

**❌ Alerting on absolute numbers instead of rates**
A spike to 100 crashes sounds bad. 100 crashes out of 1M sessions (0.01%) is fine. Always alert on rates (crashes per session), not raw counts. Raw counts spike with DAU growth even when the product is healthy.

**❌ No release-gated monitoring**
Releasing without comparing crash rate to the previous version means regressions are discovered by users, not engineers. Every release should automatically trigger a monitoring period with baseline comparison.

---

## 12. Best Practices

- **Crashlytics in every production app** — zero-config crash detection is non-negotiable
- **Upload R8 mapping files** — Crashlytics Gradle plugin does this automatically; don't skip
- **`recordException()` for all caught errors** — non-fatals are signals; don't swallow silently
- **`setCustomKey()` for app state** — screen name, feature flag state, user tier enrich every crash report
- **Analytics abstraction interface** — never call Firebase directly from ViewModel or domain
- **Consistent event naming** — `noun_verb` convention, agreed on by the team, never changed (breaking change)
- **No PII in logs or events** — use user IDs, not names; mask sensitive values
- **Timber in debug, Crashlytics tree in release** — never strip WARN/ERROR from production
- **Velocity alerts** — alert on new/spiking crashes, not just absolute thresholds
- **Release monitoring window** — monitor for 2–4 hours post-release with baseline comparison
- **Custom traces for critical paths** — feed load, checkout, search latency measured explicitly
- **P95/P99 not just averages** — average startup time hides tail latency; use percentiles for user experience
- **BigQuery export for deep analysis** — Firebase Analytics → BigQuery for custom SQL queries, cohort analysis, retention
- **Android Vitals as a lagging indicator** — good for trends, use Crashlytics for real-time

---

## 13. Interview Q&A

**Q1: How do you know your app is healthy in production?**

> Three signals: crash-free rate, ANR rate, and performance metrics. I watch crash-free rate (target > 99.5%), ANR rate (target < 0.47%), and P95 cold start time via Crashlytics and Firebase Performance. I set up velocity alerts — not just threshold alerts — so if a new crash suddenly affects 1000 users I know immediately, not 24 hours later from Play Console. After every release I run a monitoring window comparing the new version's metrics against the previous version's baseline. If crash-free rate drops more than 0.5% within 2 hours of a release, that's a rollback candidate.

---

**Q2: A user reports a crash you can't reproduce locally. How do you debug it?**

> First I look up the crash in Crashlytics using any context the user can give — their device, app version, approximate time. Crashlytics groups similar crashes by stack trace, so I find the crash group and examine: the deobfuscated stack trace, the device model and OS version (often a specific OEM bug), custom keys I set (what screen, what feature flag state, what network type), and breadcrumbs (what the user did before the crash). If it's device-specific, I check if the crash rate correlates with a specific manufacturer. If it's a network condition, I look at the network type key. If the crash is in a third-party library, I check if it's reported elsewhere. Finally, I use Firebase Test Lab to run the crash scenario on the specific device model if available.

---

**Q3: How would you design analytics for a checkout funnel?**

> Define the funnel steps first: `checkout_started` → `address_entered` → `payment_method_selected` → `order_confirmed` → `order_failed`. Each step is an event with consistent properties: `session_id` (to join steps for the same user session), `cart_value`, `item_count`, `payment_method`. The key metric is conversion rate at each step — where do users drop off? I build this in BigQuery using the Firebase Analytics export, querying for users who fired step N but not step N+1. I alert if conversion rate drops > 5% week-over-week on any step — that's a regression worth investigating. I never include payment details or card numbers in any event.

---

**Q4: What's the difference between a crash and an ANR, and how do you handle each?**

> A crash is an unhandled exception or signal that terminates the app process — the user sees "app stopped" dialog. An ANR is when the main thread is blocked for > 5 seconds while processing user input — the user sees "app not responding" dialog and gets the option to kill it. Crashes are caught by Crashlytics automatically with full stack traces. ANRs are harder — the trace shows the main thread at the moment of the block, which is often a blocked lock or synchronous I/O rather than an actual bug. The fix is always the same: nothing blocking should run on the main thread. I monitor both separately with separate alert thresholds. Play Console flags ANR rate > 0.47% as a "bad behavior" — that's the production target to stay below.

---

**Q5: How do you avoid tracking PII in analytics events?**

> Three practices. First, use identifiers not values: log `user_id` not `user_email`; log `payment_method_type` ("credit_card") not the card number. Second, audit every analytics call before release — all `logEvent()`, `setCustomKey()`, and `log()` calls reviewed for PII. Third, use a privacy-by-design principle: don't log what you don't need. If an event parameter doesn't serve an analytical purpose, remove it. For properties that are sensitive (exact location, health data), either aggregate them (city instead of GPS) or don't collect them without explicit user consent. GDPR and CCPA require consent for certain data; analytics SDKs have consent mode APIs to disable collection until consent is granted.

---

**Q6: What metrics would you instrument for a video streaming feature?**

> Four categories: playback start (did the video start? how long did it take from tap to first frame — buffering time P50/P95), quality (video quality level, quality switches per session, buffer rebuffer rate — how often does playback stall after starting), errors (failed to start rate by error type: 404 content not found, DRM error, codec unsupported, network timeout), and engagement (watch completion rate, seek events, pause/resume patterns). On the performance side, I track custom traces for the playback pipeline: manifest fetch → media segment download → DRM license acquisition → decoder initialization. Each step measured separately lets me isolate where slowdowns occur. Alert thresholds: rebuffer rate > 2%, failed to start > 1%, P95 time-to-first-frame > 3 seconds.

---

*Previous: [08 — Threading & Concurrency](./08-threading-concurrency.md)*
*Next: [10 — Security](./10-security.md)*
