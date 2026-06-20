# 06 — Performance

> **Round type: Both (HLD + LLD)**
> **HLD:** Designing a performant app architecture — what to measure, where bottlenecks hide, how to instrument
> **LLD:** Startup optimization, frame budget, memory management, profiling tools, Baseline Profiles

---

## 1. Why This Matters in Interviews

Performance is where senior engineers separate themselves. Anyone can make a feature work. Knowing *why* it's slow and how to fix it without guessing shows production maturity.

**Common interview angles:**
- "Your app feels slow on launch — how do you diagnose it?"
- "What's the 16ms frame budget and what happens when you miss it?"
- "How do you detect and fix a memory leak?"
- "What's the difference between cold, warm, and hot start?"
- "How would you reduce startup time by 30%?"
- "What causes ANRs and how do you prevent them?"
- "How do you measure performance in production, not just locally?"

---

## 2. The Three Performance Pillars

```
┌─────────────────────────────────────────────────────────┐
│                  Performance Pillars                     │
├─────────────────┬───────────────────┬───────────────────┤
│   STARTUP       │   RENDERING       │   MEMORY          │
│ Cold/Warm/Hot   │ 16ms frame budget │ Heap, GC, leaks   │
│ TTID / TTFD     │ Jank, overdraw    │ OOM, bitmaps      │
│ App Startup lib │ Compose phases    │ LeakCanary, MAT   │
│ Baseline Profile│ RenderThread      │ Object pooling    │
└─────────────────┴───────────────────┴───────────────────┘
```

All three interact: a startup memory spike causes GC which causes jank.

---

## 3. Startup Performance

### 3.1 Cold vs Warm vs Hot Start

| Type | App process? | Activity? | Speed | Cause |
|---|---|---|---|---|
| **Cold** | Not in memory | Not in memory | Slowest | First launch, force-stop, device boot |
| **Warm** | In memory | Needs recreation | Medium | Back-pressed then relaunched |
| **Hot** | In memory | In memory | Fastest | App switched back from recents |

**Always optimize cold start.** Cold start improvements carry over to warm and hot.

**Android Vitals thresholds (Play Console flags these):**
- Cold start > 5 seconds → excessive
- Warm start > 2 seconds → excessive
- Hot start > 1.5 seconds → excessive

### 3.2 What Happens During Cold Start

```
1. System creates app process (zygote fork)
2. Application.onCreate() runs
3. Main Activity created
4. Layout inflated (View hierarchy built)
5. First frame drawn → TTID (Time to Initial Display)
6. Data loaded, UI fully populated → TTFD (Time to Full Display)
```

**TTID** — time until the first frame is visible (may still show placeholder/skeleton).
**TTFD** — time until the UI shows meaningful data the user can interact with.

Measure both — TTID looks good but TTFD is what users actually experience.

### 3.3 Diagnosing Startup Slowness

```bash
# View TTID and TTFD in logcat
adb logcat | grep "Displayed"
# Output: ActivityManager: Displayed com.example/.MainActivity: +1s200ms

# Force a cold start
adb shell am force-stop com.example
adb shell am start com.example/.MainActivity

# Detailed startup trace with Perfetto
# Or use Android Studio Profiler → App Startup task
```

**Systrace / Perfetto** shows exact timeline: where in `Application.onCreate()` time is lost, how long each library initializer takes, when the first frame is drawn.

### 3.4 Common Startup Killers

**1. Heavy work in `Application.onCreate()`**
```kotlin
// ❌ Bad — blocks app launch
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        Analytics.init(this)         // 50ms
        CrashReporter.init(this)     // 80ms
        SomeHeavySDK.init(this)      // 200ms  ← all blocking launch
        database = Room.build(...)   // 30ms on main thread!
    }
}

// ✅ Good — defer non-critical initialization
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        // Only critical path at launch
        CrashReporter.init(this)     // keep — need it before any crash
        
        // Defer the rest
        ProcessLifecycleOwner.get().lifecycle.addObserver(
            LifecycleEventObserver { _, event ->
                if (event == Lifecycle.Event.ON_START) {
                    initNonCritical()
                }
            }
        )
    }
    
    private fun initNonCritical() {
        Analytics.init(this)
        SomeHeavySDK.init(this)
    }
}
```

**2. Library content providers**
Many SDKs auto-initialize via `ContentProvider` — each adds 2–10ms. Use Jetpack App Startup to consolidate:

```kotlin
// gradle
implementation("androidx.startup:startup-runtime:1.1.1")

// Replace SDK ContentProviders with initializers
class AnalyticsInitializer : Initializer<Analytics> {
    override fun create(context: Context): Analytics {
        Analytics.init(context)
        return Analytics.getInstance()
    }
    override fun dependencies() = emptyList<Class<out Initializer<*>>>()
}

// AndroidManifest.xml — one ContentProvider for all initializers
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false">
    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />
</provider>
```

**3. Synchronous disk I/O on main thread**
```kotlin
// ❌ Bad — blocks main thread
val prefs = getSharedPreferences("prefs", MODE_PRIVATE)  // disk read
val token = prefs.getString("token", null)               // synchronous

// ✅ Good — use DataStore (async) or move to background
val token = runBlocking(Dispatchers.IO) {  // at least off main thread
    dataStore.data.first()[AUTH_TOKEN_KEY]
}
// Better: collect in coroutine, show splash until ready
```

**4. Inflating too-complex first screen**

Break up the first screen — show a skeleton/placeholder immediately, lazy-load content below the fold.

### 3.5 Baseline Profiles ✅ Biggest single startup win

The Android Runtime (ART) uses JIT compilation — on first run, code is interpreted slowly. Baseline Profiles tell ART which code paths to AOT (ahead-of-time) compile before the first user interaction, improving startup by 20–30%.

```kotlin
// gradle — generate profile
implementation("androidx.profileinstaller:profileinstaller:1.3.1")
// test module
implementation("androidx.benchmark:benchmark-macro-junit4:1.2.3")

// Write a Macrobenchmark test to generate the profile
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()

    @Test
    fun generate() = rule.collect(packageName = "com.example.myapp") {
        pressHome()
        startActivityAndWait()           // cold start
        // Walk through critical user journeys
        device.findObject(By.text("Feed")).click()
        device.waitForIdle()
    }
}
```

The generated `baseline-prof.txt` ships inside your AAB. On install, Play Store pre-compiles the listed code paths. Users get a fast first launch without JIT warmup.

**Startup Profiles** (Android 13+) are a subset — only the startup-critical paths, pre-compiled even more aggressively.

### 3.6 App Startup Library for Lazy Init

```kotlin
// Lazy initialization — init only when first accessed
val analyticsInitializer = lazy {
    Analytics.init(context)
    Analytics.getInstance()
}

// Access triggers initialization
val analytics by analyticsInitializer
analytics.track("screen_view")  // initialized here, not at app start
```

---

## 4. Rendering Performance

### 4.1 The 16ms Frame Budget

The display refreshes at 60fps → **16.67ms per frame**. If your frame takes longer, the next vsync is missed — the previous frame stays on screen an extra 16ms. This is **jank** (visible stutter).

Modern devices run at 90Hz or 120Hz — budgets are 11ms and 8ms respectively. Always profile on the target minimum-spec device.

```
Vsync signal every 16.67ms
        │
        ▼
[UI Thread: measure, layout, draw calls] ← must finish in < 16ms combined
        │
        ▼
[RenderThread: executes draw commands on GPU]
        │
        ▼
Frame displayed on screen
```

**RenderThread** (introduced in Android 5.0) runs draw commands in parallel with the UI thread. Most animations run entirely on RenderThread — why Compose animations are smooth even under main thread load.

### 4.2 Jank Categories

| Type | Threshold | Symptom |
|---|---|---|
| Slow frame | > 16ms | Occasional stutter |
| Frozen frame | > 700ms | App appears hung |
| ANR | > 5s (main thread) | System dialog "App not responding" |

**ANR triggers:**
- No response to input event for > 5 seconds
- BroadcastReceiver not completing in > 10 seconds
- Service not completing in > 20 seconds (> 200s for foreground services)

```kotlin
// ❌ ANR risk — heavy work on main thread
override fun onClick(view: View) {
    val data = File("large_file.json").readText()  // BLOCKS main thread
    processData(data)
}

// ✅ Safe — move to coroutine
override fun onClick(view: View) {
    lifecycleScope.launch {
        val data = withContext(Dispatchers.IO) { File("large_file.json").readText() }
        processData(data)
    }
}
```

### 4.3 Overdraw

Overdraw = a pixel is drawn multiple times in the same frame. Background → card → text = 3x overdraw. Acceptable: 2x (pink). Problem: 3x (red). Critical: 4x+ (dark red).

```
Enable in Developer Options → Debug GPU Overdraw

Fix overdraw:
1. Remove unnecessary backgrounds
   // If parent has background, child View doesn't need one
   android:background="@null"  ← remove redundant backgrounds

2. Use clipRect() to skip drawing outside visible area
3. In Compose: avoid layering multiple backgrounds via Modifiers
```

### 4.4 Compose Rendering Phases

Compose renders in three phases — understanding them prevents unnecessary work:

```
1. COMPOSITION   — What to show (execute composable functions, build slot table)
2. LAYOUT        — Where to place (measure, layout passes)
3. DRAWING       — How to draw (canvas draw calls → GPU)
```

Skipping unnecessary phases = better performance:
- **Avoid recomposition** for pure drawing changes → use `graphicsLayer` (drawing phase only)
- **`derivedStateOf`** for computed values that change less often than their inputs

```kotlin
// ❌ Bad — entire composable recomposes on every scroll pixel
@Composable
fun Header(scrollState: ScrollState) {
    val alpha = scrollState.value / 1000f  // read during composition → always recomposes
    Box(modifier = Modifier.alpha(alpha))
}

// ✅ Good — only drawing phase re-runs, no recomposition
@Composable
fun Header(scrollState: ScrollState) {
    Box(modifier = Modifier.graphicsLayer {
        alpha = scrollState.value / 1000f  // read during drawing phase only
    })
}

// ✅ derivedStateOf — only recomposes when computed value changes
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }  // bool, not int
    }
    // Only recomposes when showButton flips true/false
    // NOT on every scroll position change
    AnimatedVisibility(visible = showButton) { ... }
}
```

*Full Compose recomposition optimization is covered in Section 23.*

### 4.5 RecyclerView / LazyColumn Performance

```kotlin
// Compose LazyColumn
LazyColumn {
    items(
        items = posts,
        key = { post -> post.id }  // ✅ stable key — enables item animations + avoids full recomposition
    ) { post ->
        PostCard(post)
    }
}

// ❌ Missing key — all items recompose when list changes
items(posts) { post -> PostCard(post) }

// For heavy list items — use remember for expensive calculations
@Composable
fun PostCard(post: Post) {
    val formattedDate = remember(post.createdAt) {
        DateFormatter.format(post.createdAt)  // computed once per post, not on every recomposition
    }
}
```

---

## 5. Memory Management

### 5.1 Android Memory Model

```
App heap:
├── Java Heap     ← objects, strings, collections (GC managed)
├── Native Heap   ← JNI/NDK, some Bitmap data (Android 8+)
└── Code & Stack  ← DEX bytecode, stack frames

Low memory:
→ System reclaims background app memory
→ onTrimMemory() called with level (TRIM_MEMORY_RUNNING_LOW, etc.)
→ OOM if app tries to allocate and fails
```

**GC pauses:** When GC runs, all threads pause briefly. Frequent GC = frequent small pauses = jank. Main cause: excessive short-lived object allocation (tight loops creating temporary objects).

### 5.2 Common Memory Leaks in Android

**1. Context leak — holding Activity reference in a long-lived object**
```kotlin
// ❌ Static reference to Activity = Activity never GC'd
companion object {
    var context: Context? = null  // Activity Context stored statically
}

// ✅ Use Application context for singletons
class MyRepository(private val context: Context) {
    // ensure context is applicationContext, not Activity
}

// ✅ Hilt/Koin inject applicationContext
@Provides @Singleton
fun provideRepository(@ApplicationContext ctx: Context) = MyRepository(ctx)
```

**2. Unregistered listeners / callbacks**
```kotlin
// ❌ Listener registered but never removed
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        locationManager.requestUpdates(listener)  // registered
        // forgot to remove in onDestroyView → fragment retained
    }
}

// ✅ Clean up in lifecycle callbacks
override fun onDestroyView() {
    super.onDestroyView()
    locationManager.removeUpdates(listener)
}

// ✅ Or use viewLifecycleOwner to auto-cancel
lifecycleScope.launch {
    locationFlow.collect { /* auto-cancelled when view destroyed */ }
}
```

**3. Inner class holding outer class reference**
```kotlin
// ❌ Anonymous Runnable holds implicit reference to enclosing Activity
handler.postDelayed({
    textView.text = "Done"  // holds Activity reference for 10 seconds
}, 10_000)

// ✅ Use WeakReference
val weakActivity = WeakReference(this)
handler.postDelayed({
    weakActivity.get()?.let { activity ->
        activity.textView.text = "Done"
    }
}, 10_000)
```

**4. Coroutine scope leak**
```kotlin
// ❌ GlobalScope survives Activity death
GlobalScope.launch { fetchData() }

// ✅ Scope tied to lifecycle — auto-cancelled
lifecycleScope.launch { fetchData() }
viewModelScope.launch { fetchData() }
```

### 5.3 Bitmap Memory

Bitmaps are the #1 source of OOM crashes. A 12MP camera image decoded at full resolution = ~48MB of RAM.

```kotlin
// ❌ Decoding full-size bitmap
val bitmap = BitmapFactory.decodeFile(path)  // could be 48MB

// ✅ Sample down to display size before decoding
fun decodeSampledBitmap(path: String, reqWidth: Int, reqHeight: Int): Bitmap {
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true   // read dimensions without decoding pixels
    }
    BitmapFactory.decodeFile(path, options)
    
    // Calculate subsample
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)
    options.inJustDecodeBounds = false
    return BitmapFactory.decodeFile(path, options)
}

fun calculateInSampleSize(options: BitmapFactory.Options, reqWidth: Int, reqHeight: Int): Int {
    val height = options.outHeight
    val width = options.outWidth
    var inSampleSize = 1
    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}
```

**Use image loading libraries (Coil/Glide) instead of manual bitmap handling.** They handle sampling, caching, and recycling automatically.

### 5.4 Detecting Leaks — LeakCanary

```kotlin
// gradle — debug only
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
// No initialization needed — auto-installs via ContentProvider

// LeakCanary watches:
// - Activities (after onDestroy)
// - Fragments (after onDestroyView)
// - ViewModels (after onCleared)
// - Views (after detach)
// If any survive GC → heap dump → traces reference chain to GC root
```

**Reading a LeakCanary trace:**
```
┬ com.example.MyFragment instance
│    Leaking: YES (Fragment.mFragmentManager is null)
│
├─ com.example.MyAdapter.fragment
│    Leaking: YES (referenced by leaking object)
│
├─ com.example.MyAdapter.listener
│    Leaking: YES
│
╰→ com.example.LongLivedSingleton.callback
     Leaking: YES (GC root: static field)
     ↑ THIS IS THE LEAK — static field holds Fragment
```

### 5.5 onTrimMemory — Respond to Memory Pressure

```kotlin
class MyApp : Application() {
    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        when (level) {
            TRIM_MEMORY_UI_HIDDEN -> {
                // App went to background — release UI caches
                imageCache.evictAll()
            }
            TRIM_MEMORY_RUNNING_CRITICAL,
            TRIM_MEMORY_COMPLETE -> {
                // System is very low on memory — release everything
                imageCache.evictAll()
                dataCache.clear()
            }
        }
    }
}
```

---

## 6. Profiling Tools

### 6.1 Android Studio Profiler

Built into Android Studio. Real-time monitoring of CPU, memory, network, energy.

```
CPU Profiler:
- Method trace: exact time in each method (sample or instrument)
- System trace: Systrace-based, shows thread activity, frame timing
- "App startup" task: dedicated startup analysis

Memory Profiler:
- Heap dump: snapshot of all live objects
- Allocation tracking: what objects are being created and where
- GC events: when GC fires and how long it pauses

Network Profiler:
- Timeline of HTTP requests
- Request/response bodies
- Bandwidth usage

Energy Profiler:
- Battery usage by CPU, network, location
- Wake locks, alarms, jobs
```

### 6.2 Perfetto (System-Level Tracing)

Perfetto is the modern replacement for Systrace. Records detailed system events (CPU scheduling, binder calls, lock contention) alongside app traces.

```bash
# Capture a Perfetto trace
adb shell perfetto -c - --txt -o /data/misc/perfetto-traces/trace.pftrace << 'EOF'
buffers { size_kb: 65536 }
data_sources { config { name: "linux.ftrace" ftrace_config {
    ftrace_events: "sched/sched_switch"
    atrace_categories: "am" "wm" "gfx" "view" "res"
    atrace_apps: "com.example.myapp"
}}}
duration_ms: 10000
EOF

# Pull and open in ui.perfetto.dev
adb pull /data/misc/perfetto-traces/trace.pftrace
```

**FrameTimeline in Perfetto** — shows each frame's deadline, whether it was on time, late, or missed, and which thread was responsible.

### 6.3 Custom Tracing in Code

```kotlin
// Mark custom sections in Perfetto / Systrace traces
import androidx.tracing.trace

fun loadFeed() {
    trace("loadFeed") {        // appears in Perfetto as a named section
        trace("fetchFromDB") {
            dao.getAll()
        }
        trace("fetchFromNetwork") {
            api.getFeed()
        }
    }
}

// Or with TraceCompat
TraceCompat.beginSection("MyOperation")
try {
    doWork()
} finally {
    TraceCompat.endSection()
}
```

### 6.4 Macrobenchmark (Production-Quality Benchmarks)

```kotlin
// gradle — benchmark module
implementation("androidx.benchmark:benchmark-macro-junit4:1.2.3")

@RunWith(AndroidJUnit4::class)
class StartupBenchmark {
    @get:Rule val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun coldStartup() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(StartupTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.COLD
    ) {
        pressHome()
        startActivityAndWait()
    }

    @Test
    fun scrollFeed() = benchmarkRule.measureRepeated(
        packageName = "com.example.myapp",
        metrics = listOf(FrameTimingMetric()),
        iterations = 5,
        startupMode = StartupMode.WARM
    ) {
        startActivityAndWait()
        val feedList = device.findObject(By.res("feedList"))
        feedList.fling(Direction.DOWN)
    }
}
```

Results include P50, P90, P99 timing — catch regressions before users do.

### 6.5 StrictMode (Dev/Debug Only)

```kotlin
// Enable in Application.onCreate() for debug builds
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()        // catch disk reads on main thread
            .detectDiskWrites()       // catch disk writes on main thread
            .detectNetwork()          // catch network on main thread
            .penaltyLog()             // log violations
            .penaltyFlashScreen()     // flash screen on violation
            .build()
    )
    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedSqlLiteObjects()
            .detectLeakedClosableObjects()
            .detectActivityLeaks()
            .penaltyLog()
            .build()
    )
}
```

Run your critical user flows with StrictMode on. Any I/O on the main thread surfaces immediately.

### 6.6 LeakCanary (Debug)

```kotlin
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
// Auto-detects leaks in Activities, Fragments, ViewModels, Views
// Shows notification with leak trace when detected
// Android Studio Panda+: integrated directly in Profiler
```

---

## 7. HLD vs LLD Framing

### HLD Questions
- "How would you ensure your app feels fast for 100M users?"
- "How do you detect performance regressions before they ship?"

**HLD answer covers:**
1. Baseline Profiles for startup
2. Firebase Performance Monitoring / custom instrumentation for production metrics
3. Macrobenchmark in CI to catch regressions
4. Android Vitals monitoring on Play Console
5. Phased initialization — critical path only on launch

### LLD Questions
- "Your app has an ANR on the main screen. How do you debug it?"
- "Walk me through how you'd fix a memory leak in a Fragment"
- "How do you make a LazyColumn scroll at 60fps?"

**LLD answer covers:**
1. Profiler → CPU → System trace to find main thread bottleneck
2. Move I/O to `Dispatchers.IO`
3. LeakCanary trace to find reference chain
4. `key` in `LazyColumn`, `remember`, `derivedStateOf`, `graphicsLayer`

---

## 8. React Native Performance

```typescript
// Avoid anonymous functions in render — creates new reference each render
// ❌
<FlatList renderItem={({ item }) => <PostCard post={item} />} />
// ✅
const renderPost = useCallback(({ item }) => <PostCard post={item} />, [])
<FlatList renderItem={renderPost} keyExtractor={(item) => item.id} />

// Use getItemLayout for fixed-height items — enables scroll-to-index and skips measurement
<FlatList
    getItemLayout={(_, index) => ({ length: 80, offset: 80 * index, index })}
/>

// Lazy load heavy screens
const HeavyScreen = React.lazy(() => import('./HeavyScreen'))

// JS thread vs UI thread — use Reanimated 3 for animations that run on UI thread
import Animated, { useSharedValue, withSpring } from 'react-native-reanimated'
// Animations via Reanimated run on UI thread — not blocked by JS work
```

**Hermes:** Enable Hermes (default in new RN projects) — ahead-of-time compiled JS, faster startup, lower memory.

**Flipper:** Debug tool for RN — Network inspector, Layout inspector, Redux DevTools.

---

## 9. Common Misunderstandings & Pitfalls

**❌ Thinking OOM means the bitmap is too large**
OOM is almost always a symptom of a memory leak. A leaked Activity holding a bitmap chain prevents GC. Fix the leak first, then if needed, sample down bitmaps.

**❌ Putting all initialization in `Application.onCreate()`**
Every ms spent here delays app launch for every user on every start. Defer non-critical work (analytics, feature flag fetching, non-essential SDKs) until after the first frame is drawn.

**❌ Using `GlobalScope` for coroutines**
`GlobalScope` lives for the entire process. A coroutine launched there doesn't stop when the Activity or ViewModel is destroyed — it keeps running, may hold references, and causes leaks. Use `lifecycleScope` or `viewModelScope`.

**❌ Not adding `key` to `LazyColumn` items**
Without a stable key, Compose can't match old and new items on list updates — it recomposes all visible items. With `key = { item.id }`, only changed items recompose.

**❌ Reading state in composition instead of drawing phase**
Reading a frequently-changing value (like scroll offset) inside a composable function triggers recomposition on every change. Reading it inside `graphicsLayer { }` or `drawBehind { }` limits the work to the drawing phase only.

**❌ Ignoring `onTrimMemory`**
When the system is low on memory, it calls `onTrimMemory`. Ignoring it means your app contributes to the memory pressure and risks being killed. Release image caches and non-essential data when you receive `TRIM_MEMORY_UI_HIDDEN`.

**❌ Running LeakCanary in release builds**
LeakCanary causes heap dumps — expensive, slow, and exposes memory contents. Always use `debugImplementation` only.

**❌ Not profiling on low-end devices**
Pixel 9 Pro runs everything fast. Your users may be on 2GB RAM devices with slow CPUs. Always profile on or near the minimum supported device spec.

**❌ Measuring startup with the debugger attached**
The debugger adds significant overhead — startup looks 3–5x slower than real users experience. Use `releaseWithDebugging` build or Macrobenchmark for accurate numbers.

---

## 10. Best Practices

- **Profile before optimizing** — never guess what's slow; measure first with Profiler or Perfetto
- **Baseline Profiles for every production app** — 20–30% startup improvement for free
- **StrictMode in debug builds** — catches main-thread I/O before it becomes an ANR
- **LeakCanary in debug builds** — catches leaks before they accumulate in production
- **Macrobenchmark in CI** — catch startup and scroll regressions before they ship
- **Defer non-critical initialization** — App Startup library + lazy init; keep launch path lean
- **`graphicsLayer` for animation state reads** — avoids recomposition for scroll-driven UI
- **`derivedStateOf` for computed state** — reduces recomposition frequency
- **`key` in all `LazyColumn`/`LazyRow`** — enables item diffing and animations
- **`remember(input)` for expensive computations** — recalculate only when input changes
- **Image size matching display size** — let Coil/Glide sample down; never decode full-res into a thumbnail slot
- **`lifecycleScope`/`viewModelScope` always** — never `GlobalScope`
- **`withContext(Dispatchers.IO)` for all I/O** — disk, network, file — off the main thread
- **Monitor Android Vitals** — Play Console shows cold start, ANR, crash rates from real users
- **Release `Executor` and streams in `onDestroyView`** — prevent Fragment leaks via unregistered callbacks

---

## 11. Interview Q&A

**Q1: What's the difference between cold, warm, and hot start, and which should you optimize first?**

> Cold start is when the app process doesn't exist in memory — the system creates a new process, instantiates Application, creates the first Activity, and draws the first frame. Warm start is when the process exists but the Activity needs to be recreated. Hot start is when both process and Activity are in memory — the system just brings the app to the foreground. Always optimize cold start first: it's the worst case and improvements carry over. Cold start > 5 seconds triggers Play Console warnings. The main culprits are heavy work in `Application.onCreate()`, too many library content providers initializing synchronously, and synchronous I/O on the main thread.

---

**Q2: What's the 16ms frame budget, and what happens when you miss it?**

> At 60fps, the display refreshes every 16.67ms. If the UI thread plus RenderThread can't complete all measurement, layout, and drawing work within that window, the vsync deadline is missed. The current frame isn't ready, so the previous frame stays on screen for an extra 16ms — this doubled frame time is perceived as a stutter or jank. On 90Hz/120Hz displays the budget shrinks to 11ms/8ms. The fix is to move anything heavy off the main thread, use `graphicsLayer` for animation state reads (runs in drawing phase, not composition), and avoid creating objects in tight render loops to reduce GC pressure.

---

**Q3: A user reports the app is using 800MB of RAM and eventually crashes. How do you diagnose it?**

> First, I'd rule out a legitimate large allocation (e.g., loading many full-res bitmaps) vs a leak. In Android Studio Profiler, I'd capture a heap dump after reproducing the scenario, look at which object types are taking the most memory, and check for retained Activities or Fragments that should have been GC'd. LeakCanary automates this: it watches the lifecycle of Activities/Fragments/ViewModels and dumps the heap if an object survives past its expected lifetime. The trace shows the exact reference chain from the leaked object to a GC root — usually a static field, a long-lived singleton holding a callback, or a background thread holding a context reference. Fix: break the reference chain, use `WeakReference` where appropriate, always clean up listeners in `onDestroyView`.

---

**Q4: What is a Baseline Profile and when would you add one?**

> A Baseline Profile is a list of critical code paths (classes and methods) pre-compiled by ART ahead of time during app install. Without it, ART interprets code on first execution and JIT-compiles it — first launch is slowest. With Baseline Profile, those paths are AOT-compiled at install time, removing JIT warmup on first run. You generate it with a Macrobenchmark test that walks through startup and critical user journeys. The profile ships inside the AAB and is processed by Play Store on install. The result is typically 20–30% faster cold start and smoother first interactions after launch. I'd add a Baseline Profile to any production app — the setup is a one-time cost of ~1 hour and the benefit is immediate for all users.

---

**Q5: How do you prevent ANRs?**

> ANRs happen when the main thread is blocked for > 5 seconds on user input events, or > 10s on broadcast receivers. The root causes are: blocking I/O on the main thread (disk reads, network calls), long-running computation (parsing, encryption, image processing), contended locks (synchronized blocks holding the main thread), and waiting on background threads that are themselves blocked. Prevention: use `suspend` functions with `withContext(Dispatchers.IO)` for all I/O, use `Dispatchers.Default` for CPU-heavy work, avoid `runBlocking` on the main thread, and use StrictMode in dev builds to catch violations early. When an ANR happens in production, the system generates an ANR trace file (`/data/anr/traces.txt`) showing the main thread stack — upload this via Crashlytics or Firebase Performance for analysis.

---

**Q6: How do you make a feed scroll at 60fps?**

> Several layers: first, stable `key` in `LazyColumn` so Compose can diff items instead of recomposing everything. Second, `remember(item.id)` for any expensive computation inside a list item composable — format dates, compute derived values once per item. Third, `graphicsLayer` for any scroll-driven visual effects (parallax, fade) so they run in the drawing phase and don't trigger recomposition. Fourth, `AsyncImage` with Coil targeting the exact size — don't load a 1200px image into a 80x80 avatar slot. Fifth, avoid creating new lambdas or objects inside the composable body on every frame. Finally, profile with Compose layout inspector and Perfetto FrameTimeline to find the actual bottleneck rather than guessing — a hunch about what's slow is usually wrong.

---

*Previous: [05 — Real-Time & Push](./05-realtime-push.md)*
*Next: [07 — Background Work & Battery](./07-background-battery.md)*
