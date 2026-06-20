# 08 — Threading & Concurrency

> **Round type: LLD (primary) + Both**
> **LLD:** Coroutine internals, dispatcher selection, structured concurrency, race conditions, synchronization
> **HLD:** Threading model decisions for network, DB, and compute-heavy features at scale

---

## 1. Why This Matters in Interviews

Threading is the single most common source of production bugs — ANRs, race conditions, memory leaks, data corruption. Interviewers probe this deeply because it separates engineers who "get async code working" from those who understand *why* it's correct.

**Common interview angles:**
- "What's the difference between `launch` and `async`?"
- "When would you use `Dispatchers.IO` vs `Dispatchers.Default`?"
- "What is structured concurrency and why does it matter?"
- "How do you handle a race condition in coroutines?"
- "What's the difference between `coroutineScope` and `supervisorScope`?"
- "How does `withContext` differ from `launch`?"
- "What causes a deadlock, and how do you avoid one?"
- "Explain `Flow` vs `Channel` — when would you use each?"

---

## 2. Android Threading Model

```
Main Thread (UI Thread)
  → Only thread that can touch Views / Composables
  → Must never be blocked (> 5s = ANR)
  → Orchestrates work: starts tasks, updates UI with results

Background Threads
  → IO Dispatcher thread pool  : disk, network, database
  → Default Dispatcher pool    : CPU computation, parsing
  → Unconfined                 : test/special cases only
  → Custom Executor            : legacy or specific thread needs
```

**The golden rule:** Never block the main thread. Every network call, disk read, heavy computation must run off-main.

---

## 3. Kotlin Coroutines — Fundamentals

### 3.1 What Coroutines Actually Are

Coroutines are **suspendable computations** — they can pause at a suspension point (`suspend fun`) and resume later without blocking the thread. The thread is freed to do other work while the coroutine is suspended.

```
Thread A: ─── launch coroutine ─── [SUSPEND: waiting for IO] ─── [RESUME: process result] ───
                                          ↑
                              Thread A is FREE here — can run other coroutines
```

This is why you can run thousands of coroutines on a handful of threads — unlike threads, suspended coroutines consume no CPU.

### 3.2 `launch` vs `async`

```kotlin
// launch — fire and forget. Returns Job. No result.
val job: Job = viewModelScope.launch {
    repository.sync()  // result discarded
}

// async — returns a Deferred<T>. Caller awaits the result.
val deferred: Deferred<User> = viewModelScope.async {
    repository.getUser(userId)
}
val user = deferred.await()  // suspends until result ready

// In practice: use launch for side effects, async for results
// Prefer sequential suspend funs over async in most cases:
val user = repository.getUser(userId)  // simpler, reads like sync code
```

**When async shines — parallel decomposition:**
```kotlin
// Sequential: totalTime = timeA + timeB
val user = userApi.getUser(id)       // 300ms
val posts = feedApi.getPosts(id)     // 400ms
// Total: 700ms

// Parallel with async: totalTime = max(timeA, timeB)
viewModelScope.launch {
    val userDeferred = async { userApi.getUser(id) }
    val postsDeferred = async { feedApi.getPosts(id) }
    val user = userDeferred.await()    // ┐ both started already
    val posts = postsDeferred.await()  // ┘ total: ~400ms
}
```

### 3.3 Dispatchers

```kotlin
// Main — UI thread. Short work only. Never block.
withContext(Dispatchers.Main) {
    textView.text = result
    _uiState.value = result  // StateFlow update
}

// IO — Network, disk, database. Elastic thread pool (up to 64 threads by default)
withContext(Dispatchers.IO) {
    val data = api.fetchData()      // network
    dao.insert(entity)              // Room DB write
    File("data.json").readText()    // file read
}

// Default — CPU-intensive work. Pool size = number of CPU cores
withContext(Dispatchers.Default) {
    val sorted = list.sortedBy { it.score }  // sorting large list
    val parsed = Json.decodeFromString<List<Item>>(largeJson)  // parsing
    val compressed = compressBitmap(bitmap)  // image processing
}

// Unconfined — don't use in production. Runs in caller's thread until first suspension.
// Only for testing or special edge cases.
```

**IO vs Default — the critical distinction:**

| | `Dispatchers.IO` | `Dispatchers.Default` |
|---|---|---|
| Thread pool size | Up to 64 | CPU core count (typically 4–8) |
| Work type | Blocking I/O (waiting) | CPU computation (running) |
| Why sized differently | Many threads can wait simultaneously | CPU-bound: more threads = more context switching = slower |
| Wrong usage | Heavy JSON parsing (wastes IO threads) | Disk reads (starves other IO) |

**Custom limited dispatcher:**
```kotlin
// Limit concurrency for specific work (e.g., max 4 parallel uploads)
val uploadDispatcher = Dispatchers.IO.limitedParallelism(4)
withContext(uploadDispatcher) { upload(file) }
```

### 3.4 `withContext` vs `launch`

```kotlin
// withContext — switches context within the SAME coroutine. Suspends caller. Returns result.
suspend fun getUser(id: String): User = withContext(Dispatchers.IO) {
    api.getUser(id)  // caller waits for this to finish
}

// launch — creates a NEW child coroutine. Does NOT suspend caller. No return value.
fun startSync() {
    viewModelScope.launch {
        repository.sync()  // runs concurrently, caller continues immediately
    }
}

// Rule of thumb:
// suspend fun doing work → withContext (keeps it sequential, easier to reason about)
// fire-and-forget side effect → launch
// need parallel results → async/await
```

---

## 4. Structured Concurrency

### 4.1 What It Means

**Structured concurrency** = coroutines have a parent-child hierarchy. Children cannot outlive their parent. If the parent is cancelled, all children are cancelled. If a child fails with an exception, it propagates to the parent.

```
viewModelScope (parent)
    ├── launch { fetchUser() }     (child 1)
    ├── launch { fetchPosts() }    (child 2)
    └── launch { fetchAds() }      (child 3)

When viewModel is cleared:
    → viewModelScope is cancelled
    → All 3 children cancelled automatically
    → No orphan coroutines, no leaks
```

**Without structured concurrency (GlobalScope):**
```kotlin
// ❌ Orphan coroutine — lives forever, holds ViewModel reference
GlobalScope.launch {
    val data = repository.fetchHeavyData()
    _state.value = data  // ViewModel might be gone by now
}
```

### 4.2 Coroutine Scopes

```kotlin
// viewModelScope — tied to ViewModel lifecycle, cancelled in onCleared()
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = repository.getData()
            _state.value = data
        }
    }
}

// lifecycleScope — tied to Activity/Fragment lifecycle
// Use viewLifecycleOwner.lifecycleScope in Fragments (not just lifecycleScope)
// lifecycleScope lives as long as the Fragment, viewLifecycleOwner.lifecycleScope
// lives as long as the Fragment's VIEW — avoids updating destroyed views
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.state.collect { render(it) }
        }
    }
}

// Custom scope — for repositories, singletons, non-Android classes
class SyncRepository @Inject constructor() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun startSync() = scope.launch { doSync() }

    fun cancel() = scope.cancel()  // call in cleanup
}
```

### 4.3 `coroutineScope` vs `supervisorScope`

```kotlin
// coroutineScope — if ANY child fails, ALL other children are cancelled + exception propagates
suspend fun loadScreenData() = coroutineScope {
    val userDeferred = async { userApi.getUser(id) }   // child 1
    val postsDeferred = async { feedApi.getPosts(id) } // child 2
    // If userApi fails → postsDeferred is also cancelled
    // Exception propagates to caller
    Pair(userDeferred.await(), postsDeferred.await())
}

// supervisorScope — if a child fails, SIBLINGS keep running, exception doesn't propagate up
suspend fun loadOptionalWidgets() = supervisorScope {
    val mainContent = async { loadMainContent() }      // critical
    val recommendations = async { loadRecommendations() } // optional
    val ads = async { loadAds() }                      // optional

    // If recommendations fails → mainContent and ads keep running
    val content = mainContent.await()
    val recs = runCatching { recommendations.await() }.getOrNull()
    val adData = runCatching { ads.await() }.getOrNull()
}
```

**When to use which:**
- `coroutineScope` — when all parallel operations are required; one failing means the whole operation fails (e.g., loading data needed for a screen)
- `supervisorScope` — when some operations are optional; a failure in one shouldn't kill the others (loading optional widgets, enrichments, ads)

### 4.4 Cancellation

Cancellation in coroutines is **cooperative** — a coroutine must check for cancellation at suspension points or explicitly with `isActive`.

```kotlin
// Cancellation happens at suspend points automatically
launch {
    repeat(1000) { i ->
        delay(10)        // ← suspension point; if cancelled, throws CancellationException here
        processItem(i)
    }
}

// For CPU-intensive loops without suspend points, check isActive manually
launch(Dispatchers.Default) {
    for (item in largeList) {
        if (!isActive) break  // or throw CancellationException()
        processItem(item)
    }
}

// Clean up on cancellation using try/finally
launch {
    try {
        doWork()
    } finally {
        // Runs even if cancelled
        cleanup()  // close files, release resources
    }
}

// NonCancellable — for cleanup that must complete even if cancelled
launch {
    try {
        doWork()
    } finally {
        withContext(NonCancellable) {
            dao.markAsCancelled(jobId)  // must complete — not cancellable
        }
    }
}
```

**`CancellationException` is special — don't catch it broadly:**
```kotlin
// ❌ Swallows cancellation — coroutine doesn't cancel properly
try {
    delay(1000)
} catch (e: Exception) {  // catches CancellationException too!
    Log.e("TAG", "Error: ${e.message}")
}

// ✅ Only catch what you mean to catch
try {
    delay(1000)
} catch (e: IOException) {
    Log.e("TAG", "Network error: ${e.message}")
}
```

---

## 5. Race Conditions & Thread Safety

### 5.1 What a Race Condition Is

```kotlin
// ❌ Race condition — two coroutines read-modify-write simultaneously
var counter = 0

repeat(1000) {
    launch(Dispatchers.Default) {
        counter++  // NOT atomic: read counter, increment, write back
    }
}
// Expected: 1000. Actual: unpredictable (700? 850? 999?)
// Two coroutines can read same value before either writes back
```

### 5.2 Fix 1 — Atomic Variables (simple values)

```kotlin
// AtomicInteger — lock-free, thread-safe for single value operations
val counter = AtomicInteger(0)

repeat(1000) {
    launch(Dispatchers.Default) {
        counter.incrementAndGet()  // atomic: all 1000 increments are safe
    }
}
// Result: always 1000

// AtomicReference — for any object type
val currentUser = AtomicReference<User?>(null)
currentUser.compareAndSet(null, newUser)  // CAS operation — set only if still null
```

**Use when:** Single value, simple operations (increment, set, compare-and-swap).

### 5.3 Fix 2 — Mutex (complex/multi-step operations)

```kotlin
val mutex = Mutex()
var sharedList = mutableListOf<Item>()

repeat(100) {
    launch(Dispatchers.Default) {
        mutex.withLock {
            // Only one coroutine enters this block at a time
            sharedList.add(item)
            sharedList.sortBy { it.priority }  // multi-step: safe inside lock
        }
    }
}
```

**Mutex suspends** (not blocks) waiting coroutines — the thread is freed to do other work.

**Mutex is NOT reentrant:**
```kotlin
val mutex = Mutex()
mutex.withLock {
    mutex.withLock {  // ❌ DEADLOCK — trying to acquire a lock you already hold
        doWork()
    }
}
```

**Use when:** Multi-step operations that must be atomic, protecting complex shared state.

### 5.4 Fix 3 — Single-Thread Confinement

Confine all access to shared state to a single thread/dispatcher:

```kotlin
// All mutations happen on a single-threaded dispatcher
val singleThread = Dispatchers.IO.limitedParallelism(1)

// Or use a dedicated single-thread context
val stateContext = newSingleThreadContext("StateThread")

suspend fun updateState(update: (State) -> State) = withContext(stateContext) {
    state = update(state)  // only one thread touches state
}
```

**Use when:** State needs complex invariants that are hard to protect with a Mutex.

### 5.5 Fix 4 — StateFlow / MutableStateFlow (UI state)

```kotlin
// StateFlow.update is thread-safe — built-in atomic CAS
private val _state = MutableStateFlow(UiState())

// Thread-safe update
fun increment() {
    _state.update { current -> current.copy(count = current.count + 1) }
}
// .update uses compareAndSet internally — safe from multiple threads
```

**Use when:** Sharing UI state between coroutines in a ViewModel.

### 5.6 Choosing the Right Tool

| Scenario | Tool |
|---|---|
| Simple counter/flag | `AtomicInteger` / `AtomicBoolean` |
| Multi-step operation | `Mutex` |
| UI state shared across coroutines | `StateFlow.update` |
| All access from one thread | Single-thread dispatcher / `limitedParallelism(1)` |
| Concurrent read, exclusive write | `ReadWriteMutex` (from kotlinx-atomicfu) |
| Limiting concurrent access | `Semaphore` |

---

## 6. Flow vs Channel

### 6.1 Flow — Cold Stream

A `Flow` is cold — it only produces values when collected. Nothing happens until you call `.collect()`.

```kotlin
// Cold flow — code inside runs fresh for EACH collector
fun fetchItems(): Flow<List<Item>> = flow {
    val items = api.getItems()  // called once per collector
    emit(items)
}

// Two collectors = two API calls
fetchItems().collect { /* collector 1 */ }
fetchItems().collect { /* collector 2 */ }
```

**Operators:**
```kotlin
repository.getItems()
    .filter { it.isActive }
    .map { it.toUiModel() }
    .catch { e -> emit(emptyList()) }       // handle errors in stream
    .flowOn(Dispatchers.IO)                 // upstream runs on IO
    .collect { items -> _state.value = items }
```

### 6.2 SharedFlow — Hot Stream (multi-subscriber broadcast)

```kotlin
// SharedFlow — hot, multicast. Emits to all current collectors.
private val _events = MutableSharedFlow<Event>(
    replay = 0,           // new subscribers get no history
    extraBufferCapacity = 1  // buffer 1 event to avoid suspension on emit
)
val events: SharedFlow<Event> = _events

// Emit from anywhere
viewModelScope.launch { _events.emit(Event.NavigateToHome) }

// Multiple collectors all receive the same emission
events.collect { /* collector 1 */ }
events.collect { /* collector 2 */ }
```

### 6.3 StateFlow — Hot Stream (current state)

```kotlin
// StateFlow — hot, always has a value, emits current value to new subscribers
val state: StateFlow<UiState> = _state.asStateFlow()

// New subscriber immediately gets the current state
state.collect { it }  // receives current value + all future updates
```

### 6.4 Channel — One-Time Delivery

```kotlin
// Channel — FIFO queue. Each value consumed by exactly ONE receiver.
private val _effects = Channel<Effect>(Channel.BUFFERED)
val effects: Flow<Effect> = _effects.receiveAsFlow()

// Emit
viewModelScope.launch { _effects.send(Effect.ShowSnackbar("Saved")) }

// Consume — only one collector receives each effect
effects.collect { effect -> handleEffect(effect) }
```

### 6.5 When to Use What

| | Flow | SharedFlow | StateFlow | Channel |
|---|---|---|---|---|
| Cold/Hot | Cold | Hot | Hot | — (queue) |
| Multiple collectors | Separate streams | Shared | Shared | One consumer |
| Current value | ❌ | ❌ (unless replay) | ✅ Always | ❌ |
| Replay to new sub | ❌ | Configurable | ✅ (last value) | ❌ |
| One-time events | ❌ | ❌ (can replay) | ❌ (replays) | ✅ |
| UI state | ❌ | ❌ | ✅ | ❌ |
| Navigation/Snackbar | ❌ | 🟡 (replay=0) | ❌ | ✅ |
| DB/network stream | ✅ | ❌ | ❌ | ❌ |

---

## 7. Exception Handling

### 7.1 Exception Propagation Rules

```
launch (fire-and-forget):
  → Exception propagates to parent scope
  → Uncaught exception crashes the app (if no CoroutineExceptionHandler)

async (deferred result):
  → Exception stored in Deferred
  → Only thrown when .await() is called
  → Not propagated until awaited
```

```kotlin
// launch — exception propagates immediately
viewModelScope.launch {
    throw IOException("Network error")  // → cancels viewModelScope unless handled
}

// async — exception deferred until await
val deferred = viewModelScope.async {
    throw IOException("Network error")  // stored, not thrown yet
}
deferred.await()  // IOException thrown here
```

### 7.2 CoroutineExceptionHandler

```kotlin
// For launch — catches uncaught exceptions at the scope level
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("Coroutine", "Uncaught exception: $exception")
    _state.update { it.copy(error = exception.message) }
}

viewModelScope.launch(handler) {
    riskyOperation()
}

// Note: CoroutineExceptionHandler does NOT work with async
// Use try/catch around await() instead
```

### 7.3 try/catch in Coroutines

```kotlin
// Preferred pattern for ViewModel operations
viewModelScope.launch {
    _state.update { it.copy(isLoading = true) }
    try {
        val result = repository.getData()
        _state.update { it.copy(data = result, isLoading = false) }
    } catch (e: IOException) {
        _state.update { it.copy(error = "Network error", isLoading = false) }
    } catch (e: CancellationException) {
        throw e  // ← ALWAYS rethrow CancellationException
    }
}

// Using Result<T> to make error handling explicit
viewModelScope.launch {
    val result = runCatching { repository.getData() }
    result.onSuccess { data -> _state.update { it.copy(data = data) } }
    result.onFailure { e -> _state.update { it.copy(error = e.message) } }
}
```

### 7.4 `supervisorScope` for Independent Failures

```kotlin
// Without supervisorScope — one failure kills all
viewModelScope.launch {
    coroutineScope {
        launch { loadUser() }    // if this throws...
        launch { loadFeed() }    // ...this gets cancelled too
    }
}

// With supervisorScope — independent failures
viewModelScope.launch {
    supervisorScope {
        launch {
            try { loadUser() } catch (e: Exception) { handleUserError(e) }
        }
        launch {
            try { loadFeed() } catch (e: Exception) { handleFeedError(e) }
        }
    }
}
```

---

## 8. `collectAsStateWithLifecycle` vs `collectAsState`

```kotlin
// collectAsState — always collecting, even when screen is in background
// Wastes resources, may cause updates when UI isn't visible
val state by viewModel.state.collectAsState()

// collectAsStateWithLifecycle ✅ — pauses collection when UI goes to background
// Resumes when UI comes to foreground (respects Lifecycle.State.STARTED)
val state by viewModel.state.collectAsStateWithLifecycle()

// Use collectAsStateWithLifecycle for all UI state in production
// Requires: implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
```

---

## 9. Deadlocks

A deadlock occurs when two operations wait for each other to complete and neither can proceed.

**Classic deadlock with `runBlocking` on main thread:**
```kotlin
// ❌ Deadlock — runBlocking blocks main thread, coroutine needs main thread
fun onCreate() {
    val result = runBlocking {       // blocks main thread
        withContext(Dispatchers.Main) {  // needs main thread — which is blocked!
            api.getData()
        }
    }
}

// ❌ Mutex deadlock — acquiring same lock twice
val mutex = Mutex()
launch {
    mutex.withLock {           // acquires lock
        mutex.withLock {       // tries to acquire again — DEADLOCK (not reentrant)
            doWork()
        }
    }
}

// ✅ Fix — never use runBlocking on the main thread
// ✅ Fix — restructure to not need the same lock twice
// ✅ Fix — use a suspend fun chain instead of nested blocking calls
```

**Detect with:** Android Studio thread debugger, or StrictMode's deadlock detection.

---

## 10. RxJava vs Coroutines (Legacy context)

You may encounter RxJava in existing codebases. Key differences:

| | Coroutines/Flow | RxJava |
|---|---|---|
| Learning curve | Lower | Steeper |
| Cancellation | Structured (automatic) | Manual `CompositeDisposable` |
| Error handling | try/catch, `catch` operator | `onError` callbacks |
| Back pressure | Flow (built-in) | Flowable (explicit) |
| Android integration | Deep (ViewModel, Lifecycle) | Via RxAndroid |
| Performance | Lower overhead | Higher overhead |
| Status | ✅ Recommended | 🟡 Legacy, still supported |

**Migrating RxJava → Coroutines:**
```kotlin
// RxJava Observable → Flow
observable.asFlow()

// RxJava Single → suspend fun
single.await()

// RxJava Completable → suspend fun
completable.await()
```

---

## 11. React Native Threading

React Native uses three threads:
- **JS Thread** — JavaScript execution (your business logic, state management)
- **Main/UI Thread** — native view rendering
- **Shadow Thread** — layout calculation (Yoga engine)

```typescript
// Async operations don't block JS thread
const user = await fetchUser(id)  // JS thread freed while awaiting

// For CPU-heavy JS work — use Web Workers or move to native
// Expensive computations in JS block the JS thread → janky animations

// Native modules run on their own threads
// Bridge calls (JS ↔ native) are async by default
NativeModule.heavyWork().then(result => setData(result))

// With New Architecture (JSI) — synchronous bridge calls possible
// but still offload heavy work to native threads
```

---

## 12. Common Misunderstandings & Pitfalls

**❌ Using `GlobalScope`**
`GlobalScope` has no parent — coroutines launched there are not cancelled when the screen is gone, can hold references, cause leaks. Always use `viewModelScope`, `lifecycleScope`, or a custom scope with a managed lifecycle.

**❌ `lifecycleScope` vs `viewLifecycleOwner.lifecycleScope` in Fragments**
`lifecycleScope` in a Fragment lives as long as the Fragment — including when the Fragment's view is destroyed and recreated. Collecting a Flow here after `onDestroyView` updates a destroyed view → crash. Always use `viewLifecycleOwner.lifecycleScope` in `onViewCreated`.

**❌ Using `Dispatchers.IO` for CPU-intensive work**
`Dispatchers.IO` has up to 64 threads — if you put sorting, JSON parsing, or image processing there, you're occupying threads that should be for I/O waiting. Use `Dispatchers.Default` (sized to CPU cores) for computation.

**❌ Catching `CancellationException` without rethrowing**
`CancellationException` is how coroutine cancellation propagates. Swallowing it with a broad `catch (e: Exception)` prevents cancellation from working — coroutine keeps running after its scope is cancelled. Always: `catch (e: CancellationException) { throw e }`.

**❌ `runBlocking` on the main thread**
`runBlocking` blocks the calling thread until the coroutine completes. On the main thread, this is instant ANR. `runBlocking` is for tests and `main()` functions only.

**❌ Mutex is not reentrant**
Kotlin's `Mutex` does not support reentrant locking. A coroutine trying to lock a `Mutex` it already holds will deadlock. Design code so the same coroutine never needs to acquire the same lock twice.

**❌ Sharing mutable state between coroutines without synchronization**
`var counter = 0` mutated by multiple coroutines on `Dispatchers.Default` is a race condition. Use `AtomicInteger` for simple values, `Mutex` for complex multi-step updates, or `StateFlow.update` for UI state.

**❌ `async` exception handling**
Exceptions in `async` are not thrown until `.await()` is called. If you never call `.await()`, the exception is silently lost (in coroutineScope) or swallowed (in supervisorScope). Always handle exceptions around `.await()`.

**❌ Not using `flowOn` for upstream operators**
`flowOn` changes the dispatcher for all operators UPSTREAM of it (between it and the flow source). Operators DOWNSTREAM run on the collector's dispatcher. Misplacing `flowOn` puts heavy work on the main thread.

```kotlin
// ✅ Correct — parsing happens on IO, collect on Main
repository.getRawItems()          // IO
    .map { parseHeavy(it) }       // IO (upstream of flowOn)
    .flowOn(Dispatchers.IO)       // everything above runs on IO
    .collect { updateUI(it) }     // Main
```

---

## 13. Best Practices

- **`viewModelScope` for VM work, `viewLifecycleOwner.lifecycleScope` for Fragment UI** — never GlobalScope
- **`withContext(Dispatchers.IO)` for all blocking I/O** — network, disk, database
- **`withContext(Dispatchers.Default)` for CPU work** — sorting, parsing, computation
- **`async`/`await` only for parallel decomposition** — don't use async just to get a result; a `suspend fun` returning a value is simpler
- **`coroutineScope { }` for parallel work that must all succeed** — any failure cancels siblings
- **`supervisorScope { }` for independent parallel work** — one failure doesn't kill others
- **Always rethrow `CancellationException`** — or use `runCatching` which handles this correctly
- **`StateFlow.update` for thread-safe state** — atomic CAS, no Mutex needed for simple state
- **`Mutex` for multi-step atomic operations** — not for single value updates
- **`Channel` for one-time effects** — navigation, snackbar, not StateFlow
- **`collectAsStateWithLifecycle`** — not `collectAsState` — respects lifecycle, saves battery
- **`limitedParallelism(n)` to cap concurrency** — for bounded resource pools (upload slots, DB connections)
- **Test coroutines with `runTest` + `StandardTestDispatcher`** — don't use `runBlocking` in tests; `runTest` handles coroutine scheduling correctly

---

## 14. Interview Q&A

**Q1: What's the difference between `launch` and `async`?**

> Both create a new coroutine. `launch` returns a `Job` — it's fire-and-forget, no result. `async` returns a `Deferred<T>` — you call `.await()` to get the result when it's ready. In practice, I use `launch` for side effects and event handlers, and `async` specifically for parallel decomposition — starting multiple independent operations simultaneously and collecting their results. For sequential operations that return a value, a plain `suspend fun` is cleaner than `async` — no unnecessary `Deferred` wrapping.

---

**Q2: What is structured concurrency and why does it matter?**

> Structured concurrency means coroutines have a parent-child hierarchy — a child coroutine cannot outlive its parent scope. When the parent scope is cancelled, all children are cancelled. When a child fails, the exception propagates to the parent. This matters because it makes coroutine lifecycles predictable and prevents leaks. If you use `viewModelScope`, all coroutines are automatically cancelled when the ViewModel is cleared — you don't manually track and cancel them. Without structured concurrency (using `GlobalScope`), coroutines become orphans — they run until process death and can hold references to destroyed objects.

---

**Q3: When would you use `coroutineScope` vs `supervisorScope`?**

> `coroutineScope` has fail-fast semantics — if any child coroutine fails, all siblings are cancelled and the exception propagates out. Use this when all parallel operations are required for correctness — loading three data sources that all need to succeed for the screen to make sense. `supervisorScope` isolates failures — one child failing doesn't affect siblings. Use this when some operations are optional — loading a main content feed plus recommendations and ads. If ads fail, you still want to show the main content. The tradeoff: with `supervisorScope` you must handle each child's exceptions individually since they don't propagate automatically.

---

**Q4: How do you handle a race condition in Kotlin coroutines?**

> The fix depends on the complexity. For a simple counter or flag, `AtomicInteger` or `AtomicBoolean` provides lock-free, thread-safe operations — fast and simple. For multi-step operations where you read-modify-write multiple values atomically, use a `Mutex` — it suspends (not blocks) waiting coroutines, and `mutex.withLock { }` ensures only one coroutine enters the critical section at a time. For UI state in a ViewModel, `MutableStateFlow.update { }` is thread-safe via CAS semantics — no Mutex needed. The biggest pitfall is using a Mutex for simple single-value updates when an atomic is sufficient, and conversely using atomics for multi-step operations where only a Mutex guarantees atomicity.

---

**Q5: What's the difference between `StateFlow`, `SharedFlow`, and `Channel`?**

> `StateFlow` always holds a current value and emits it immediately to new collectors — it's for UI state that should always have a value. `SharedFlow` is configurable — no default value, configurable replay buffer, broadcasts to all current collectors. `Channel` is a FIFO queue — each emission is delivered to exactly one collector. In practice: UI state → `StateFlow`; broadcast events to multiple listeners → `SharedFlow`; one-time side effects like navigation or snackbar → `Channel` (via `receiveAsFlow()`). The reason I don't use `StateFlow` for navigation: it replays the last value to new collectors, so navigating away and coming back would trigger the navigation again.

---

**Q6: What does `flowOn` do and where does it apply?**

> `flowOn` changes the `CoroutineDispatcher` for all operators **upstream** of it — between `flowOn` and the flow source. Everything downstream of `flowOn` (including `collect`) runs on the collector's dispatcher, which is typically `Main` in a ViewModel. This is a common source of bugs: putting `map { heavyWork(it) }` below `flowOn(Dispatchers.IO)` means `heavyWork` runs on Main, not IO. The rule: put all heavy operators before `flowOn(Dispatchers.IO)`, and keep the `collect { updateUI() }` call on Main. Room DAOs that return `Flow` already handle their own dispatcher internally, so you don't need `flowOn` for DB flows.

---

**Q7: How do you prevent a coroutine from leaking when a Fragment view is destroyed?**

> Use `viewLifecycleOwner.lifecycleScope` instead of `lifecycleScope` in Fragments. `lifecycleScope` is tied to the Fragment's lifecycle — which persists across view creation and destruction when the Fragment is on the back stack. If you launch a coroutine in `lifecycleScope` and collect a Flow, it keeps running after `onDestroyView` and will try to update views that no longer exist. `viewLifecycleOwner.lifecycleScope` is tied to the Fragment's view lifecycle — it's cancelled in `onDestroyView`. Combined with `repeatOnLifecycle(Lifecycle.State.STARTED)`, collection also pauses when the Fragment goes to the background and resumes when it returns.

---

*Previous: [07 — Background Work & Battery](./07-background-battery.md)*
*Next: [09 — Observability](./09-observability.md)*
