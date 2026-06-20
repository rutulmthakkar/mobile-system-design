# 22 — Debugging, Profiling & Tooling

> **Round type: LLD (practical reference)**
> **Focus:** Android Studio Profiler (CPU/Memory/Network/Energy), Perfetto, Layout Inspector, Database Inspector, LeakCanary, StrictMode, ADB, Flipper, RN debugging tools — what each tool is for and how to use it efficiently

---

## 1. Why This Matters in Interviews

Debugging and profiling questions reveal operational maturity. Any engineer can write code; senior engineers know which tool to reach for when something goes wrong in production. Interviewers often present a scenario: "Your app has an ANR that only happens on specific devices. How do you diagnose it?" The answer is a specific sequence of tools and commands.

**Common interview angles:**
- "Your app uses 800MB of RAM and crashes. How do you debug it?"
- "A user reports jank when scrolling. How do you find the root cause?"
- "How do you detect a memory leak in production?"
- "What's the difference between a debuggable and a profileable build?"
- "How do you trace a slow startup without a debugger attached?"
- "What ADB commands do you use daily?"

---

## 2. Debuggable vs Profileable Builds

This is the first decision when profiling:

| | Debug build | Profileable build | Release build |
|---|---|---|---|
| `debuggable` | ✅ | ❌ | ❌ |
| `profileable` | N/A | ✅ | Optional |
| Performance | 20-40% slower | ~Release speed | Fastest |
| Profiler accuracy | Lower (debug overhead) | **Accurate** | N/A |
| Heap dumps | ✅ | ❌ | ❌ |
| Allocation tracking | ✅ | ❌ | ❌ |
| Method traces | ✅ | ✅ | ❌ |
| CPU sampling | ✅ | ✅ | ❌ |
| Best for | Development, breakpoints | **Performance profiling** | Production |

```xml
<!-- AndroidManifest.xml — profileable build -->
<application>
    <profileable android:shell="true"/>
    <!-- shell="true": allow ADB to profile without Play Store restrictions -->
</application>
```

**Always profile with a profileable build** — debug builds have significant overhead that distorts measurements. The Android Gradle Plugin generates a profileable release variant for you.

---

## 3. Android Studio Profiler

### 3.1 Opening the Profiler

```
Android Studio → View → Tool Windows → Profiler
OR: Run → Profile [app]
OR: Click the Profiler tab at the bottom of the IDE
```

Connect to a running app by selecting the process from the device dropdown. The Profiler shows a timeline with four tracks: CPU, Memory, Network, Energy.

### 3.2 CPU Profiler

**What it shows:** Thread activity, method call stacks, where CPU time is spent.

**Recording modes:**
```
Callstack Sample (sampled):
  → Captures thread call stacks periodically (every 1–10ms)
  → Low overhead — good for long recordings
  → Less precise for very short methods

Java/Kotlin Method Trace (instrumented):
  → Records every method entry/exit
  → High overhead — slows app significantly
  → Best for short specific operations
  → Useful for: finding exactly which method is slow

System Trace (Perfetto-backed):
  → Records system events + your app's custom trace sections
  → Shows thread scheduling, binder calls, lock contention
  → Best for: startup analysis, frame timing, ANR root cause
```

**Workflow for jank diagnosis:**
1. Run app with Profiler open
2. CPU Profiler → Start recording (System Trace)
3. Scroll/interact to reproduce jank
4. Stop recording
5. Look for: long frames in the FrameTimeline, main thread being blocked, long operations on the wrong dispatcher
6. Identify the method in the flame chart taking > 16ms

**Reading a flame chart:**
```
Tall columns = frequent calls (width = total time)
Wide rectangles = methods spending more time (self time)

     ██████████████████████████  ← main culprit (wide = slow)
           ████████              ← called by the one above
                █████            ← and so on down the stack

Bottom of flame chart = entry point (main thread, coroutine launch, etc.)
Top of flame chart = the actual work being done
```

### 3.3 Memory Profiler

**What it shows:** Heap size, object allocations, GC events.

```
Heap dump button → captures all live objects in memory
Allocation recording → tracks every object created (debug builds only)

Reading a heap dump:
1. Look for: Retained Size vs Shallow Size
   - Shallow Size: memory of the object itself
   - Retained Size: memory of object + everything it keeps alive (more important)
2. Filter by: your package name (com.example.*) to filter out framework objects
3. Sort by: Retained Size descending → top items are the biggest leaks
4. Instance view → References pane → "Path to GC Root"
   → Shows exactly why this object isn't being garbage collected
```

**Common heap patterns:**
```
Normal:      flat line with periodic GC dips (sawtooth pattern)
Memory leak: line trends upward, never comes back down
GC thrash:   frequent rapid up-and-down spikes (too many short-lived allocations)
OOM risk:    line approaching max heap size
```

### 3.4 Network Profiler

Shows HTTP/HTTPS requests in timeline. Click any request to see:
- URL, method, response code
- Request/response headers
- Request/response body (may be compressed)
- Timing breakdown: DNS, Connection, SSL, Request, Response

**Note:** Only shows requests made through `OkHttpClient` instances known to the profiler. Custom sockets or non-OkHttp clients may not appear.

### 3.5 Energy Profiler

Shows estimated battery usage from CPU, network, and location. Look for:
- **Wake lock activity** — is your app holding a wake lock longer than expected?
- **Background CPU spikes** — is a WorkManager job running at unexpected times?
- **GPS usage** — is location being requested more than needed?

---

## 4. Layout Inspector

**Access:** View → Tool Windows → Layout Inspector (while app is running)

**What it shows:** Live hierarchy of composables/views, their properties, recomposition counts.

**For Compose:** The Layout Inspector shows:
```
Component tree (left):
  → All composables in the current composition tree
  → Expandable hierarchy with nesting

Properties pane (right):
  → Parameters, modifiers, state values for selected composable

Recomposition counts (overlay):
  → RED number next to composable = recomposition count
  → HIGH number = stability issue or unnecessary recomposition
  → Click composable to see which state it depends on
```

**Debugging high recomposition counts:**
1. Layout Inspector → enable "Show recomposition counts" (Android Studio Giraffe+)
2. Identify composables with counts much higher than expected
3. Check: is it reading a frequently-changing state (scroll position as Int)?
4. Fix: add `derivedStateOf`, move state reads to `graphicsLayer`, check for unstable types

### 4.1 Network Inspector (DB Profiler)

**Access:** View → Tool Windows → App Inspection → Network Inspector

Shows HTTP traffic in real time with request/response details. More convenient than Network Profiler for inspecting individual requests during development.

---

## 5. Database Inspector

**Access:** View → Tool Windows → App Inspection → Database Inspector

**What it shows:** Live view of Room databases while the app runs.

```
Features:
- Browse all tables, all rows — live, real-time
- Run custom SQL queries against the live database
- Edit cell values directly (for testing)
- Export tables to CSV
- Trigger manual refresh on DAO observers

Workflow for debugging data sync:
1. Open Database Inspector
2. Navigate to the table (e.g., messages)
3. Watch rows appear/update as sync runs in the background
4. Query: SELECT * FROM messages WHERE sync_status = 'PENDING'
   → See exactly which records haven't synced yet
```

---

## 6. Perfetto — System-Level Tracing

Perfetto is the modern replacement for Systrace. It records kernel-level events alongside app-level traces.

### 6.1 Capture a Perfetto Trace

```bash
# Option A: Via ADB (command line)
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace.pftrace \
<<EOF
buffers { size_kb: 65536 }
data_sources {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      atrace_categories: "am" "wm" "gfx" "view" "res" "dalvik"
      atrace_apps: "com.example.myapp"
    }
  }
}
duration_ms: 10000
EOF

# Pull the trace
adb pull /data/misc/perfetto-traces/trace.pftrace ~/Desktop/

# Option B: Via Android Studio (easier)
# Profiler → CPU → Record → System Trace
# Stop recording → exports as .pftrace automatically
```

### 6.2 Analyze in ui.perfetto.dev

Open `ui.perfetto.dev` in Chrome → drag `.pftrace` file → explore:

```
Key things to look at:
├── FrameTimeline (top)
│   → Green frames = on time
│   → Red frames = deadline missed (jank)
│   → Shows which frame started late and why
│
├── Main thread (UI Thread)
│   → Look for: long Choreographer#doFrame() calls
│   → Long RecyclerView layout passes
│   → Long Binder calls blocking main thread
│
├── RenderThread
│   → Should run parallel to main thread
│   → Long RenderThread = complex draw operations
│
├── Your app's custom sections
│   → Anything you added with trace() {} (see §6.3)
│
└── GC events
    → Frequent GC = too many allocations
    → Long GC = large heap, stop-the-world pauses
```

### 6.3 Adding Custom Trace Sections

```kotlin
// Add named sections visible in Perfetto
import androidx.tracing.trace

fun loadFeed() {
    trace("loadFeed") {           // appears in Perfetto timeline
        val cached = trace("loadFromCache") {
            feedDao.getAllFeed()
        }
        val network = trace("fetchFromNetwork") {
            feedApi.getFeed()
        }
        trace("mergeFeedItems") {
            mergeFeed(cached, network)
        }
    }
}

// Or with TraceCompat for older APIs
fun processImage(bitmap: Bitmap): Bitmap {
    TraceCompat.beginSection("processImage")
    return try {
        applyFilters(bitmap)
    } finally {
        TraceCompat.endSection()
    }
}
```

These custom sections appear as named blocks in the Perfetto timeline, making it easy to see exactly where time is spent.

---

## 7. LeakCanary — Memory Leak Detection

Already covered in Section 06 (Performance). Key points for reference:

```kotlin
// gradle — debug only
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
// No setup needed — auto-installed via ContentProvider

// LeakCanary watches lifecycle of:
// - Activities (after onDestroy)
// - Fragments (after onDestroyView)
// - Fragment Views (after onDestroyView)
// - ViewModels (after onCleared)

// If any survive GC → heap dump → shows reference chain to GC root
```

**Reading LeakCanary output:**
```
┬ LeakingInstance (com.example.MyFragment)
│ Leaking: YES (Fragment.mFragmentManager = null)
│
├─ MyAdapter.fragment
│  Leaking: YES (referenced by leaking object)
│
├─ MyAdapter.clickListener
│  Leaking: YES
│
╰→ MyApplication.singletonRef
   Leaking: NO (held by static)
   ↑ THIS IS THE LEAK ROOT
   Static field holds reference chain to Fragment
```

**Fix:** `MyApplication.singletonRef` should not hold a reference to `MyAdapter` which holds a reference to `MyFragment`. Break the chain — use a WeakReference, clear the reference in `onDestroyView`, or scope the singleton correctly.

---

## 8. StrictMode

```kotlin
// Enable in Application.onCreate() for DEBUG builds only
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectAll()          // catches: disk reads/writes, network, custom slow calls
            .penaltyLog()         // logs violations to Logcat
            .penaltyFlashScreen() // visual flash when violation occurs
            // .penaltyDeath()    // crash on violation (strict)
            .build()
    )
    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedSqlLiteObjects()    // unclosed Cursors/SQLiteDatabase
            .detectLeakedClosableObjects()   // unclosed streams, FileDescriptors
            .detectActivityLeaks()           // Activity instances lingering
            .detectCleartextNetwork()        // HTTP without TLS
            .penaltyLog()
            .build()
    )
}
```

**Run your key user flows with StrictMode enabled.** Any disk I/O or network on the main thread surfaces immediately in Logcat with a full stack trace. Fix all violations before shipping.

---

## 9. ADB Commands — Daily Driver

```bash
# === APP LIFECYCLE ===
adb shell am force-stop com.example.myapp   # force kill (for cold start testing)
adb shell am start com.example.myapp/.MainActivity  # launch
adb shell monkey -p com.example.myapp -c android.intent.category.LAUNCHER 1  # launch

# === LOGS ===
adb logcat                                  # all logs
adb logcat -s "MyTag"                       # filter by tag
adb logcat | grep -E "Exception|Error"      # filter errors
adb logcat *:E                              # only error level and above
adb logcat -c && adb logcat                 # clear buffer, then stream

# === PACKAGE / INSTALL ===
adb install -r myapp.apk                   # install (r = replace)
adb install --no-streaming myapp.apk       # for large APKs
adb uninstall com.example.myapp
adb shell pm list packages | grep example  # find package

# === PERFORMANCE ===
adb shell dumpsys gfxinfo com.example.myapp          # frame stats
adb shell dumpsys gfxinfo com.example.myapp reset    # reset frame counters
adb shell dumpsys meminfo com.example.myapp          # memory stats
adb shell dumpsys activity com.example.myapp         # activity stack
adb shell top -d 1 | grep example                   # CPU usage real-time

# === NETWORK ===
adb shell dumpsys connectivity                       # network state
adb shell settings put global airplane_mode_on 1    # enable airplane mode
adb shell am broadcast -a android.intent.action.AIRPLANE_MODE  # apply
adb shell settings put global airplane_mode_on 0    # disable

# === BATTERY / DOZE ===
adb shell dumpsys battery unplug                    # simulate unplugged
adb shell dumpsys deviceidle force-idle deep        # force Doze mode
adb shell dumpsys deviceidle step light             # step through Doze states
adb shell dumpsys battery reset                     # restore real state

# === PERMISSIONS ===
adb shell pm grant com.example.myapp android.permission.ACCESS_FINE_LOCATION
adb shell pm revoke com.example.myapp android.permission.CAMERA
adb shell dumpsys package com.example.myapp | grep permission  # list permissions

# === SCREEN CAPTURE ===
adb exec-out screencap -p > ~/Desktop/screenshot.png
adb shell screenrecord /sdcard/record.mp4  # Ctrl+C to stop
adb pull /sdcard/record.mp4 ~/Desktop/

# === DB / FILES ===
adb shell run-as com.example.myapp ls /data/data/com.example.myapp/databases/
adb shell run-as com.example.myapp cat /data/data/com.example.myapp/shared_prefs/settings.xml
adb pull /sdcard/  # pull SD card contents

# === INPUT SIMULATION ===
adb shell input tap 500 800                         # tap at x=500, y=800
adb shell input swipe 500 1500 500 500 300          # swipe up (300ms)
adb shell input text "Hello World"                  # type text
adb shell input keyevent 4                          # KEYCODE_BACK
adb shell input keyevent 82                         # KEYCODE_MENU

# === SIMULATE NETWORK CONDITIONS ===
# Use emulator: Settings → Extended controls → Cellular (throttle network)
# Physical device: Developer Options → Simulated network speed

# === ANR TRACE ===
adb pull /data/anr/traces.txt                       # ANR stack trace (requires root)
# Or: adb bugreport → unzip → look for ANR traces
```

---

## 10. Flipper — Facebook's Debug Tool

Flipper is a desktop debug tool (Mac/Windows/Linux) that provides a GUI for inspecting running Android (and iOS) apps.

```kotlin
// gradle (debug only)
debugImplementation("com.facebook.flipper:flipper:0.258.0")
debugImplementation("com.facebook.soloader:soloader:0.10.5")
debugImplementation("com.facebook.flipper:flipper-network-plugin:0.258.0")
debugImplementation("com.facebook.flipper:flipper-leakcanary2-plugin:0.258.0")

// Application.onCreate()
if (BuildConfig.DEBUG) {
    SoLoader.init(this, false)
    val client = AndroidFlipperClient.getInstance(this)
    client.addPlugin(InspectorFlipperPlugin(this, DescriptorMapping.withDefaults()))
    client.addPlugin(NetworkFlipperPlugin())
    client.addPlugin(DatabasesFlipperPlugin(this))  // Browse Room DB
    client.addPlugin(SharedPreferencesFlipperPlugin(this, listOf("settings")))
    client.start()
}

// OkHttp integration
val networkPlugin = NetworkFlipperPlugin()
val okHttpClient = OkHttpClient.Builder()
    .addNetworkInterceptor(FlipperOkhttpInterceptor(networkPlugin))
    .build()
```

**Flipper features:**
- **Layout Inspector** — view/edit layout hierarchy (similar to Android Studio's, but sometimes faster)
- **Network** — inspect all HTTP requests/responses in real time (like Charles Proxy, but simpler)
- **Databases** — browse and query SQLite/Room databases live
- **Shared Preferences** — view and edit SharedPreferences live
- **LeakCanary** — see leak reports in Flipper UI instead of notification
- **Logs** — stream Logcat with filtering

**Flipper vs Charles Proxy:**
- Flipper: simpler setup, built-in to app, great for internal debug
- Charles Proxy: works with any app (no SDK needed), better for testing API endpoints, HTTPS decryption

---

## 11. Common Debugging Workflows

### 11.1 ANR Diagnosis Workflow

```
Symptom: App Not Responding dialog on a specific screen/action

Step 1: Reproduce with StrictMode enabled
  → If StrictMode catches it: fix the disk/network I/O on main thread → done

Step 2: If not StrictMode: capture with System Trace (Perfetto)
  → Record during the ANR-causing interaction
  → Look at main thread: what's it doing for 5+ seconds?
  → Common causes:
    a) Binder call blocking (IPC to another process)
    b) Synchronized block contention (lock held by background thread)
    c) Database query on main thread
    d) Complex layout inflation

Step 3: Check ANR trace file
  adb bugreport → unzip → look for "ANR in com.example"
  → Main thread stack at time of ANR shows exactly where it was blocked

Step 4: Fix
  → Move blocking work to Dispatchers.IO
  → Replace synchronized with Mutex (non-blocking)
  → Cache inflated views
```

### 11.2 Memory Leak Workflow

```
Symptom: App memory grows over time, eventually crashes with OOM

Step 1: Enable LeakCanary → reproduce the scenario
  → Navigate to a screen → press back → repeat 10 times
  → LeakCanary will notify if the screen's Activity/Fragment leaks

Step 2: Read the LeakCanary trace
  → Find the GC root (bottom of the chain — usually static field or thread)
  → Trace upward to find which object holds the reference

Step 3: Heap dump in Memory Profiler (if LeakCanary doesn't catch it)
  → Memory Profiler → take heap dump
  → Filter by your package name
  → Sort by retained size
  → Select suspicious object → References pane → Path to GC Root

Step 4: Common fixes
  → Clear listener in onDestroyView/onDestroy
  → Use WeakReference for Context in singletons
  → Don't hold Activity context in long-lived objects
  → Stop coroutines in their parent scope (viewModelScope, not GlobalScope)
```

### 11.3 Startup Slowness Workflow

```
Symptom: Cold start takes > 3 seconds

Step 1: Measure TTID and TTFD
  adb logcat | grep "Displayed"
  → "Displayed com.example/.MainActivity: +2s400ms"

Step 2: App Startup task in Profiler
  → Profiler → CPU → App Startup → Start recording
  → Launch app → stop recording
  → See flame chart of Application.onCreate() + Activity.onCreate()

Step 3: Identify the slow initializations
  → Look for: library init (Analytics, CrashReporter, heavy SDKs)
  → Add custom trace sections around suspected calls

Step 4: Defer non-critical work
  → Init crash reporter early (needed before any crash)
  → Defer analytics, feature flags, sync to post-first-frame
  → Use App Startup library to consolidate ContentProvider initializers

Step 5: Baseline Profile
  → Generate Macrobenchmark startup test
  → generateBaselineProfile task → commit baseline-prof.txt
  → Typical result: 20-30% faster cold start
```

### 11.4 Jank / 60fps Workflow

```
Symptom: Scrolling feels choppy

Step 1: Enable GPU rendering profile
  Developer Options → Profile GPU rendering → On screen as bars
  → Green line = 16ms target
  → Red bars = frames over budget

Step 2: System Trace in Profiler
  → Record during scroll → look at FrameTimeline
  → Click a "red" (janky) frame → see which thread caused the miss

Step 3: Layout Inspector → Recomposition counts (for Compose)
  → High count on a list item composable = read unstable state

Step 4: Common fixes
  → Remove heavy work from onDraw / composable body
  → Add stable keys to LazyColumn items
  → Use graphicsLayer for scroll-driven animations
  → Cache computed values with remember(key)
  → Move state reads to drawing phase (graphicsLayer)
```

---

## 12. React Native Debugging Tools

### 12.1 Metro Bundler & Dev Menu

```bash
# Start Metro bundler
npx react-native start

# Open dev menu on device
adb shell input keyevent 82     # or shake the device

# Dev menu options:
# - Enable Fast Refresh (auto-reload on file save)
# - Show Inspector (tap elements to inspect)
# - Perf Monitor (FPS, memory, RAM)
# - Network Inspector (via Flipper)
```

### 12.2 Flipper for React Native

```bash
# Install Flipper desktop app
# Open Flipper → connect → see:
# - React DevTools (component tree, state, props)
# - Network (all fetch/XHR requests)
# - Databases (AsyncStorage, SQLite)
# - Logs
# - Hermes Debugger
```

### 12.3 Hermes Debugger (Chrome DevTools)

```bash
# 1. Enable Hermes in build.gradle:
# hermesEnabled = true

# 2. Open Chrome → chrome://inspect
# 3. "React Native" device appears → Inspect
# 4. Full Chrome DevTools: breakpoints, call stack, memory, console

# OR: In Dev Menu → "Open Debugger"
# Opens a full-featured debugger for Hermes-compiled JS
```

### 12.4 React Native Performance Monitor

```typescript
// Enable FPS monitor programmatically
import { PerformanceObserver } from 'react-native'

// Measure render time
const renderStart = Date.now()
// ... render ...
const renderTime = Date.now() - renderStart

// react-native-performance (library)
import performance from 'react-native-performance'
performance.mark('screenStart')
// ... do work ...
performance.measure('screenLoad', 'screenStart')
const measures = performance.getEntriesByName('screenLoad')
```

### 12.5 Why Did You Render (WDYR)

```typescript
// Find unnecessary re-renders
// gradle: only in development
import './wdyr'  // src/wdyr.ts

// wdyr.ts
if (__DEV__) {
    const whyDidYouRender = require('@welldone-software/why-did-you-render')
    whyDidYouRender(React, {
        trackAllPureComponents: true,
        logOnDifferentValues: true  // logs when same-value props cause re-render
    })
}
```

---

## 13. Macrobenchmark — Automated Performance Testing

```kotlin
// gradle — separate benchmark module
@get:Rule
val benchmarkRule = MacrobenchmarkRule()

@Test
fun startupBenchmark() = benchmarkRule.measureRepeated(
    packageName = "com.example.myapp",
    metrics = listOf(
        StartupTimingMetric(),        // TTID, TTFD
        FrameTimingMetric(),          // jank, frame duration
        TraceSectionMetric("loadFeed") // custom trace sections
    ),
    iterations = 5,
    startupMode = StartupMode.COLD
) {
    pressHome()
    startActivityAndWait()
    // Wait for content to load
    device.wait(Until.hasObject(By.res("feedList")), 5000)
}

// Run: ./gradlew :benchmark:connectedReleaseAndroidTest
// Output: P50, P90, P99 for each metric
// Catches regressions: if P95 startup > 3s → CI fails
```

---

## 14. Useful Lint Rules

```kotlin
// Custom lint checks in build.gradle
android {
    lintOptions {
        // Fail build on these
        error("HardcodedText")           // UI strings not in strings.xml
        error("LogNotTimber")            // using Log.d instead of Timber
        error("UnusedResources")         // dead XML resources
        error("UnsafeDynamicallyLoadedCode")
        // Warn on these
        warning("ObsoleteSdkInt")        // API check for SDK version no longer needed
    }
}

// Run lint manually
./gradlew :app:lint
// Output: app/build/reports/lint-results-debug.html
```

---

## 15. Common Misunderstandings & Pitfalls

**❌ Profiling with a debug build**
Debug builds have 20–40% overhead — method tracing hooks, debug info in every class, ProGuard disabled. Startup times in debug can be 3x slower than release. Always profile with a profileable release build for accurate numbers.

**❌ Taking heap dumps with the app under load**
Heap dumps freeze the app while capturing. Take them when the app is idle to see the true resident set. Heap dumps taken during heavy work capture all the temporary objects in flight, making it hard to find the actual leaks.

**❌ Trusting ADB logcat for performance timing**
`System.currentTimeMillis()` logged via Logcat has inconsistent buffering delays. Use `System.nanoTime()` for timing and custom trace sections in Perfetto for accurate frame-level timing.

**❌ Not separating crash reporting from Flipper**
LeakCanary in production build (not just debug) is a serious mistake — it triggers heap dumps in production, significantly slowing real users. Always `debugImplementation` only.

**❌ Ignoring "Background Restricted" in battery optimization**
Some OEMs (Xiaomi, Huawei) aggressively kill apps in the background. What works perfectly in developer mode (background restrictions off) may break on user devices (background restrictions on). Test with Developer Options → Background Process Limit set to "No background processes".

---

## 16. Best Practices

- **Profileable build always** for performance measurements — never debug build
- **StrictMode in debug builds** — run all key user flows before each release
- **LeakCanary debugImplementation** — run weekly on key flows, not just on symptom
- **Custom trace sections (`trace { }`)** — add to all non-trivial suspend functions for Perfetto visibility
- **Macrobenchmark in CI** — catch startup and scroll regressions before users do
- **`adb shell dumpsys gfxinfo`** — quick jank check without Profiler overhead
- **Database Inspector for sync debugging** — real-time table view beats print statements
- **Flipper for network debugging** — faster than Charles for most use cases
- **`adb logcat -c`** before reproducing bugs — clean log makes stack traces easier to find
- **`adb bugreport` for ANR evidence** — provides the full thread dump at ANR time
- **Layout Inspector recomposition counts** — first check for Compose performance issues

---

## 17. Interview Q&A

**Q1: Your app has an ANR that only happens on specific devices. How do you diagnose it?**

> First, I'd enable StrictMode in a debug build and run the same flow — if it catches disk or network I/O on the main thread, that's likely the cause. If StrictMode doesn't catch it, I'd use Perfetto System Trace: record during the ANR-triggering interaction and look at the main thread timeline for what's blocking it for 5+ seconds. Common culprits are binder IPC calls, lock contention from a synchronized block held by a background thread, or database queries that accidentally ended up on the main thread. For device-specific ANRs, I'd pull the ANR trace file from `adb bugreport` — it contains the full main thread stack at the ANR moment, showing exactly which call was blocked. If it's a specific OEM, I'd check if they have custom background restrictions killing threads unexpectedly.

---

**Q2: What's the difference between a debuggable and a profileable build?**

> A debuggable build has full debugging support: breakpoints, heap dumps, allocation tracking, method tracing. But it's 20–40% slower than a release build because of the debug instrumentation overhead — startup times can be 3x longer. This makes profiling measurements inaccurate. A profileable build is based on the release build variant with ProGuard/R8 running, same performance characteristics as what users get, but with the profiling interface exposed for ADB to connect to. It supports CPU sampling and system traces but not heap dumps or allocation tracking. For performance profiling — startup time, scroll FPS, memory usage — always use a profileable build. The Android Gradle Plugin generates a profileable variant automatically. Never report startup numbers measured on a debug build.

---

**Q3: How do you find what's causing high recomposition counts in a Composable?**

> Android Studio's Layout Inspector shows recomposition counts as red numbers overlaid on the component tree (enable in the toolbar). High counts on a leaf composable usually mean it's reading state that changes more frequently than necessary. I'd look at what state the composable reads: if it's reading a scroll offset as an Int value, it recomposes on every pixel of scroll. The fix is `derivedStateOf` to compute a less-frequent boolean: `val showButton by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }` — now it only recomposes when the boolean flips, not on every scroll position. Another common cause is passing an unstable type (a plain `List<T>` instead of `ImmutableList<T>`) as a parameter — Compose can't verify it hasn't changed, so it recomposes defensively. I use the Compose Compiler Reports to detect unstable types systematically.

---

**Q4: How do you track down a memory leak when LeakCanary hasn't triggered?**

> LeakCanary watches specific objects (Activities, Fragments, ViewModels, Views). If the leak is elsewhere — a singleton holding a large Bitmap, a background thread keeping a reference to a context, a cache growing unbounded — LeakCanary won't catch it. For these cases, I use the Memory Profiler heap dump. Trigger the suspected leak scenario several times, then take a heap dump. Filter by my package name to ignore framework objects. Sort by retained size descending — the top entries are consuming the most memory including their reference graph. For the top entry, click it, go to the References pane, and follow "Path to GC Root" — this shows the exact reference chain from the leaked object to a GC root (usually a static field or live thread). The first static field in the chain is the leak point. A flat-then-growing memory graph in the Memory Profiler timeline is the smoking gun that a leak exists even before taking a heap dump.

---

**Q5: What ADB commands do you use most in daily development?**

> Daily: `adb logcat -s MyTag` to filter logs to my tag, `adb shell am force-stop com.example.myapp` before measuring cold start (ensures a true cold start), and `adb shell dumpsys gfxinfo com.example.myapp reset` followed by the user interaction then `dumpsys gfxinfo` again to check frame stats. For debugging background work: `adb shell dumpsys deviceidle force-idle deep` to simulate Doze, `adb shell dumpsys battery unplug` to simulate battery drain. For permissions: `adb shell pm grant/revoke` to quickly test permission-denied paths. For capturing issues: `adb bugreport` for the full system state when something goes wrong that I can't reproduce consistently — it includes ANR traces, logs, battery stats, and process state. `adb shell input keyevent 82` to open the dev menu on an Android device when testing React Native.

---

**Q6: How do you add performance instrumentation to your app code?**

> Two levels. First, custom trace sections with `trace("sectionName") { }` from `androidx.tracing`. These appear in Perfetto and Android Studio System Trace, making it easy to see how much time your code takes relative to system events like frame boundaries. I add these to all non-trivial suspend functions, repository methods, and rendering operations. Second, Macrobenchmark tests that run in CI. The test records startup time, frame timing, and custom trace section durations across multiple iterations, reporting P50, P90, and P99. The benchmark output goes to a JSON file that CI compares against a baseline — if P95 cold startup increases by more than 200ms, the build fails. This means performance regressions are caught before they merge, not after users complain. For production monitoring, Firebase Performance custom traces track the same operations with real user data and geographic distribution.

---

*Previous: [21 — Design Patterns & Diagrams](./21-design-patterns.md)*
*Next: [23 — Jetpack Compose Optimization](./23-compose-optimization.md)*
