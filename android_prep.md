## TABLE OF CONTENTS

1. [Jetpack Compose Internals — Recomposition](#1-jetpack-compose-internals--recomposition)
2. [State: remember vs rememberSaveable vs derivedStateOf](#2-state-remember-vs-remembersaveable-vs-derivedstateof)
3. [State Hoisting — Stateful vs Stateless Composables](#3-state-hoisting--stateful-vs-stateless-composables)
4. [Side Effects — LaunchedEffect, DisposableEffect, SideEffect, produceState](#4-side-effects)
5. [Flow Types — Flow vs StateFlow vs SharedFlow vs Channel](#5-flow-types--flow-vs-stateflow-vs-sharedflow-vs-channel)
6. [collectAsStateWithLifecycle vs collectAsState](#6-collectasstatewithlifecycle-vs-collectasstate)
7. [Sealed Class UiState Pattern](#7-sealed-class-uistate-pattern)
8. [ViewModel — Internals, Scoping, Best Practices](#8-viewmodel--internals-scoping-best-practices)
9. [Repository Pattern — Single Source of Truth](#9-repository-pattern--single-source-of-truth)
10. [Coroutine Exception Handling](#10-coroutine-exception-handling)
11. [Navigation — NavHost, Routes, Back Stack](#11-navigation--navhost-routes-back-stack)
12. [LazyColumn — Performance Internals](#12-lazycolumn--performance-internals)
13. [Modifier — Ordering and Performance](#13-modifier--ordering-and-performance)
14. [Manual Dependency Injection (No Hilt)](#14-manual-dependency-injection-no-hilt)
15. [Testability Patterns](#15-testability-patterns)
16. [Performance Optimization](#16-performance-optimization)
17. [Battery Optimization](#17-battery-optimization)
18. [Memory Management](#18-memory-management)
19. [Network Optimization & Reliability](#19-network-optimization--reliability)
20. [App Reliability & Consistency Patterns](#20-app-reliability--consistency-patterns)
21. [Build-An-App Interview Checklist](#21-build-an-app-interview-checklist)
22. [Patterns & Anti-Patterns — Complete Reference](#22-patterns--anti-patterns--complete-reference)
23. [Architecture Trade-offs — MVVM vs MVI vs MVP vs Clean Architecture](#23-architecture-trade-offs--mvvm-vs-mvi-vs-mvp-vs-clean-architecture)
24. [Android Threading Model — Foreground, Background, Restrictions](#24-android-threading-model--foreground-background-restrictions)
25. [Foreground/Background Restrictions & Android Compliance](#25-foregroundbackground-restrictions--android-compliance)
26. [Android Lifecycle Deep Dive](#26-android-lifecycle-deep-dive)
27. [Security — Crypto, Token Storage, Network Security](#27-security--crypto-token-storage-network-security)
28. [Android Compliance & Best Practices Checklist](#28-android-compliance--best-practices-checklist)
29. [Kotlin Internals — Quick-Fire Interview Questions](#29-kotlin-internals--quick-fire-interview-questions)
30. [RecyclerView vs LazyColumn](#30-recyclerview-vs-lazycolumn)
31. [Compose Testing — Deep Dive](#31-compose-testing--deep-dive)

---

## 1. Jetpack Compose Internals — Recomposition

### How Compose works internally

Compose has 3 phases every frame:
1. **Composition** — runs your `@Composable` functions, builds a slot table (in-memory tree of UI nodes)
2. **Layout** — measures and places each node
3. **Drawing** — renders to canvas

When state changes, Compose does **smart recomposition** — it re-runs only the composable functions that read the changed state, not the whole tree.

### What triggers recomposition?

Only reading a `State<T>` object inside a composable triggers recomposition when that state changes. If a composable doesn't read any state, it never recomposes.

```kotlin
// This recomposes when 'name' changes
@Composable
fun Greeting(name: State<String>) {
    Text(text = name.value) // reading .value = subscribing
}
```

### Stability — the key concept

Compose uses **stability** to decide if it can SKIP recomposing a composable when its parent recomposes.

A composable can be skipped if ALL its parameters are **stable**:
- Primitive types (Int, String, Boolean) → always stable
- Objects annotated `@Stable` or `@Immutable` → stable
- Kotlin data classes with only stable fields → stable (usually)
- Regular classes / lists / mutable objects → **UNSTABLE** → cannot skip

```kotlin
// ✅ Stable — Compose can skip this if 'user' hasn't changed
@Stable
data class User(val name: String, val age: Int)

@Composable
fun UserCard(user: User) { ... }

// ❌ Unstable — List<T> is mutable, Compose treats it as unstable
@Composable
fun UserList(users: List<User>) { ... } // will ALWAYS recompose

// ✅ Fix: use ImmutableList (kotlinx.collections.immutable)
@Composable
fun UserList(users: ImmutableList<User>) { ... }
```

### @Stable vs @Immutable

| Annotation | Meaning | Use when |
|---|---|---|
| `@Immutable` | Fields will NEVER change after construction | True immutable data classes |
| `@Stable` | Changes are always notified to Compose | Mutable but observable objects |

```kotlin
// ✅ DO: annotate your data models
@Immutable
data class CryptoPrice(val symbol: String, val price: Double)

// ❌ DON'T: use raw mutable data class and wonder why everything recomposes
data class CryptoPrice(var symbol: String, var price: Double) // var = unstable
```

### Common recomposition mistakes

```kotlin
// ❌ BAD: lambda created on every recomposition — new object = unstable
@Composable
fun MyButton(onClick: () -> Unit) {
    Button(onClick = { onClick() }) // wrapping creates new lambda every time
}

// ✅ GOOD: pass lambda directly
@Composable
fun MyButton(onClick: () -> Unit) {
    Button(onClick = onClick)
}

// ❌ BAD: reading state too high — recomposes entire parent
@Composable
fun Parent() {
    val counter = remember { mutableStateOf(0) }
    Text("Counter: ${counter.value}") // recomposes Parent on every increment
    HeavyComposable() // also recomposes even though it doesn't use counter
}

// ✅ GOOD: push state reads down — only Text recomposes
@Composable
fun Parent() {
    val counter = remember { mutableStateOf(0) }
    CounterText(counter) // only THIS recomposes
    HeavyComposable() // SKIPPED — doesn't read counter
}

@Composable
fun CounterText(counter: State<Int>) {
    Text("Counter: ${counter.value}")
}
```

---

## 2. State: remember vs rememberSaveable vs derivedStateOf

### remember

- Stores a value in the Compose slot table
- Survives **recomposition**
- Dies on **configuration change** (rotation) and **process death**

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) } // lost on rotation
    Button(onClick = { count++ }) { Text("$count") }
}
```

### rememberSaveable

- Stores value in slot table AND saves to `Bundle` (like `onSaveInstanceState`)
- Survives **recomposition + configuration change**
- Dies on **process death** (unless you use custom Saver)
- Only works with types that Bundle supports: primitives, String, Parcelable

```kotlin
@Composable
fun Counter() {
    var count by rememberSaveable { mutableStateOf(0) } // survives rotation
    Button(onClick = { count++ }) { Text("$count") }
}

// For complex types — write a custom Saver
val UserSaver = Saver<User, Bundle>(
    save = { user -> Bundle().apply { putString("name", user.name) } },
    restore = { bundle -> User(bundle.getString("name")!!) }
)

var user by rememberSaveable(stateSaver = UserSaver) { mutableStateOf(User("Rutul")) }
```

### derivedStateOf

- Creates a state that is **derived** from other states
- Only recomputes when the **result** changes, not every time the input changes
- Critical for performance when input changes frequently but output changes rarely

```kotlin
// ❌ BAD: recomputes + recomposes on EVERY scroll position change
@Composable
fun ScrollList() {
    val listState = rememberLazyListState()
    val showButton = listState.firstVisibleItemIndex > 0 // recalculated every pixel
    if (showButton) ScrollToTopButton()
}

// ✅ GOOD: derivedStateOf — only recomposes when showButton flips true/false
@Composable
fun ScrollList() {
    val listState = rememberLazyListState()
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
    if (showButton) ScrollToTopButton()
}
```

**Rule of thumb:** Use `derivedStateOf` when:
- Input state changes frequently (scroll, text input)
- But output only changes occasionally (threshold crossed, filter result changed)

### Quick comparison

| | Survives recomposition | Survives rotation | Survives process death |
|---|---|---|---|
| `remember` | ✅ | ❌ | ❌ |
| `rememberSaveable` | ✅ | ✅ | ❌ |
| ViewModel | ✅ | ✅ | ❌ |
| DataStore/DB | ✅ | ✅ | ✅ |

---

## 3. State Hoisting — Stateful vs Stateless Composables

### The concept

**State hoisting** = moving state UP to the caller so the composable becomes stateless and reusable.

A **stateless** composable:
- Takes values and callbacks as parameters
- Doesn't own any state
- Is fully testable and reusable

A **stateful** composable:
- Owns state internally with `remember`
- Convenient but hard to test and reuse

### Pattern

```kotlin
// ❌ BAD: stateful — not reusable, not testable
@Composable
fun SearchBar() {
    var query by remember { mutableStateOf("") }
    TextField(value = query, onValueChange = { query = it })
}

// ✅ GOOD: stateless — reusable anywhere, fully testable
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit
) {
    TextField(value = query, onValueChange = onQueryChange)
}

// Caller owns the state:
@Composable
fun SearchScreen(viewModel: SearchViewModel = viewModel()) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    SearchBar(query = query, onQueryChange = viewModel::onQueryChange)
}
```

### When stateful is OK

Small, isolated UI-only state that doesn't affect anything outside the composable:
- Expanded/collapsed animation state
- Tooltip visibility
- Local scroll position within a self-contained widget

```kotlin
// ✅ OK to keep local: pure UI state with no business logic
@Composable
fun ExpandableCard(title: String, content: String) {
    var expanded by remember { mutableStateOf(false) }
    Card(onClick = { expanded = !expanded }) {
        Text(title)
        if (expanded) Text(content)
    }
}
```

### Where state should live — decision tree

```
Is the state shared between multiple composables?
  YES → hoist to common ancestor or ViewModel
  NO → Is it business/data state?
    YES → ViewModel
    NO → Is it needed after config change?
      YES → rememberSaveable or ViewModel
      NO → remember
```

---

## 4. Side Effects

Side effects = operations that escape the composable scope (network calls, navigation, analytics, showing snackbars).

**Golden rule:** Never put side effects directly in composable body — they run on every recomposition.

```kotlin
// ❌ BAD: runs on EVERY recomposition
@Composable
fun MyScreen() {
    analytics.logScreenView("MyScreen") // called multiple times!
    ...
}
```

### LaunchedEffect

- Launches a coroutine tied to the composable's lifecycle
- Cancels and restarts when `key` changes
- Cancels when composable leaves composition

```kotlin
// ✅ DO: one-time side effect on entry
@Composable
fun MyScreen(userId: String, viewModel: MyViewModel) {
    LaunchedEffect(userId) { // restarts if userId changes
        viewModel.loadUser(userId)
    }
}

// ✅ DO: observe one-time events (navigation, snackbar)
@Composable
fun MyScreen(viewModel: MyViewModel, navController: NavController) {
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.Navigate -> navController.navigate(event.route)
                is UiEvent.ShowSnackbar -> snackbarHost.showSnackbar(event.message)
            }
        }
    }
}

// Key rules:
// key = Unit → runs ONCE when composable enters
// key = someValue → reruns whenever someValue changes
// key = true → also runs once (same as Unit)
```

### DisposableEffect

- For effects that need **cleanup** when composable leaves
- Must always end with `onDispose { }`

```kotlin
// ✅ DO: register/unregister listeners
@Composable
fun LocationTracker(onLocation: (Location) -> Unit) {
    val context = LocalContext.current
    DisposableEffect(context) {
        val manager = context.getSystemService(LocationManager::class.java)
        val listener = LocationListener { onLocation(it) }
        manager.requestLocationUpdates(GPS_PROVIDER, 0L, 0f, listener)
        onDispose {
            manager.removeUpdates(listener) // cleanup — critical
        }
    }
}

// ❌ WRONG: missing onDispose — memory leak
DisposableEffect(Unit) {
    setupListener()
    // forgot onDispose!
}
```

### SideEffect

- Runs after EVERY successful recomposition
- For syncing Compose state to non-Compose code

```kotlin
// ✅ USE: when you need to sync state to external non-Compose system
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val user by viewModel.user.collectAsStateWithLifecycle()
    SideEffect {
        // runs after every recomposition
        FirebaseAnalytics.setUserProperty("userId", user?.id)
    }
}
```

### produceState

- Converts non-Compose state sources into Compose State
- Launches a coroutine, produces a value, auto-cancels on exit

```kotlin
// ✅ USE: converting Flow/LiveData/callback to State inline
@Composable
fun NetworkStatusBanner() {
    val isOnline by produceState(initialValue = true) {
        networkMonitor.isOnline.collect { value = it }
    }
    if (!isOnline) OfflineBanner()
}
```

### rememberCoroutineScope

- Gives you a coroutine scope tied to the composable
- Use when you need to launch coroutines from event handlers (button clicks), NOT from composition

```kotlin
// ✅ DO: launch from click handler
@Composable
fun SaveButton(onSave: suspend () -> Unit) {
    val scope = rememberCoroutineScope()
    Button(onClick = {
        scope.launch { onSave() } // safe: launched from event, not composition
    }) {
        Text("Save")
    }
}

// ❌ DON'T: use LaunchedEffect for click-triggered work
// LaunchedEffect is for composition-triggered work, not user events
```

### Side Effect decision tree

```
Need to run when composable ENTERS composition? → LaunchedEffect(Unit)
Need to rerun when a KEY changes? → LaunchedEffect(key)
Need CLEANUP when composable EXITS? → DisposableEffect
Need to sync after EVERY recomposition? → SideEffect
Converting external source to State? → produceState
Need coroutine from USER EVENT (click)? → rememberCoroutineScope
```

---

## 5. Flow Types — Flow vs StateFlow vs SharedFlow vs Channel

### Cold vs Hot

| | Cold | Hot |
|---|---|---|
| Produces values | Only when collected | Always, regardless of collectors |
| Multiple collectors | Each gets own stream | Share same stream |
| Example | `flow { }`, Retrofit | `StateFlow`, `SharedFlow` |

### Flow (cold)

```kotlin
// Cold — nothing happens until collected
val pricesFlow: Flow<List<Price>> = flow {
    while (true) {
        emit(api.getPrices())
        delay(5000)
    }
}
// If no one collects, no API calls happen
```

### StateFlow (hot, current state holder)

- Always has a value (requires initial value)
- New collectors immediately get the **current** value
- Only emits when value actually changes (equates by `==`)
- Perfect for UI state

```kotlin
class MyViewModel : ViewModel() {
    // ✅ Expose as StateFlow for UI state
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    // ✅ Collector always gets current value immediately on subscribe
}

// ❌ DON'T: use MutableStateFlow directly in UI layer
// Always expose as StateFlow (read-only)
```

### SharedFlow (hot, event bus)

- No initial value required
- Can replay N previous emissions to new collectors
- Multiple collectors all receive emissions
- Perfect for one-time events (navigation, snackbar)

```kotlin
class MyViewModel : ViewModel() {
    // ✅ Use SharedFlow for one-time events
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun navigate(route: String) {
        viewModelScope.launch {
            _events.emit(UiEvent.Navigate(route))
        }
    }
}

// replay = 0 (default): new collectors miss past events ✅ for navigation
// replay = 1: new collectors get last event — careful! navigation would re-trigger
```

### Channel (hot, one-to-one queue)

- Each value consumed by exactly ONE collector
- Use for work queues, not UI state

```kotlin
// ✅ USE Channel for: task queues, one-shot work distribution
val workChannel = Channel<WorkItem>(capacity = Channel.BUFFERED)

// ❌ DON'T use Channel for UI state — use StateFlow
// ❌ DON'T use Channel for one-time events — use SharedFlow
```

### Quick decision

```
Need current UI state with initial value? → StateFlow
Need one-time events (navigation, toast)? → SharedFlow(replay=0)
Need work queue (one consumer)? → Channel
Need lazy/on-demand stream? → Flow
```

---

## 6. collectAsStateWithLifecycle vs collectAsState

### The problem with collectAsState

`collectAsState()` collects the flow even when the app is in the background (screen off, app backgrounded). This wastes battery and can cause crashes.

### collectAsStateWithLifecycle (correct choice)

Stops collecting when the lifecycle drops below `Lifecycle.State.STARTED` (app goes to background). Resumes when app comes foreground.

```kotlin
// ❌ BAD: collects in background — wastes battery
val uiState by viewModel.uiState.collectAsState()

// ✅ GOOD: lifecycle-aware — pauses in background
val uiState by viewModel.uiState.collectAsStateWithLifecycle()

// Requires: implementation("androidx.lifecycle:lifecycle-runtime-compose:2.x")
```

**This is a senior-level signal.** Ojaswirajanya will specifically notice which one you use.

---

## 7. Sealed Class UiState Pattern

Never use raw nullable types for UI state. Always model all states explicitly.

### The pattern

```kotlin
// ✅ Model every possible UI state
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String, val throwable: Throwable? = null) : UiState<Nothing>()
}

// In ViewModel:
class CryptoViewModel(private val repository: CryptoRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Crypto>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Crypto>>> = _uiState.asStateFlow()

    init { loadCryptos() }

    fun loadCryptos() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = repository.getCryptos()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error", e)
            }
        }
    }
}

// In Composable:
@Composable
fun CryptoScreen(viewModel: CryptoViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (val state = uiState) {
        is UiState.Loading -> LoadingIndicator()
        is UiState.Success -> CryptoList(state.data)
        is UiState.Error -> ErrorView(state.message, onRetry = viewModel::loadCryptos)
    }
}
```

### ❌ Common mistakes

```kotlin
// ❌ BAD: separate flags — can be in impossible states (isLoading=true AND isError=true)
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Crypto>? = null,
    val error: String? = null
)

// ❌ BAD: nullable data — forces null checks everywhere
var cryptos: List<Crypto>? = null
var isLoading: Boolean = true
```

---

## 8. ViewModel — Internals, Scoping, Best Practices

### How ViewModel survives rotation

ViewModel is stored in `ViewModelStore`, which is retained across configuration changes. The Activity is destroyed and recreated, but `ViewModelStore` is handed off to the new Activity.

ViewModel is destroyed when:
- Activity finishes (user presses back)
- `viewModelStore.clear()` is called

### viewModelScope

- Automatically cancelled when ViewModel is cleared
- Uses `Dispatchers.Main.immediate` by default
- Always launch in `viewModelScope` for ViewModel work

```kotlin
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch { // auto-cancelled when VM cleared
            val data = withContext(Dispatchers.IO) { // switch to IO for network/disk
                repository.getData()
            }
            _uiState.value = UiState.Success(data)
        }
    }

    override fun onCleared() {
        super.onCleared()
        // viewModelScope is already cancelled here automatically
        // Clean up any non-coroutine resources here
    }
}
```

### What belongs in ViewModel vs Composable

| Belongs in ViewModel | Belongs in Composable |
|---|---|
| Business logic | UI logic (expand/collapse) |
| Network calls | Animation state |
| Data transformation | Local scroll position |
| Navigation events | Keyboard visibility |
| Error handling | Focus state |
| UI state (UiState) | |

### ViewModel with parameters — ViewModelFactory

Since no Hilt in interview, use a factory:

```kotlin
class CryptoViewModel(
    private val repository: CryptoRepository
) : ViewModel() {
    companion object {
        fun factory(repository: CryptoRepository) = object : ViewModelProvider.Factory {
            override fun <T : ViewModel> create(modelClass: Class<T>): T {
                @Suppress("UNCHECKED_CAST")
                return CryptoViewModel(repository) as T
            }
        }
    }
}

// In Composable:
val repository = remember { CryptoRepository(api) }
val viewModel: CryptoViewModel = viewModel(factory = CryptoViewModel.factory(repository))
```

---

## 9. Repository Pattern — Single Source of Truth

### The concept

Repository is the single source of truth for data. UI never fetches data directly. ViewModel never knows if data came from network or cache.

```
UI → ViewModel → Repository → [Remote API | Local Cache]
```

### Implementation

```kotlin
// Data sources
class CryptoRemoteDataSource(private val api: CryptoApi) {
    suspend fun getCryptos(): List<CryptoDto> = api.getCryptos()
}

class CryptoLocalDataSource {
    // In-memory cache for interview (use Room in production)
    private var cache: List<Crypto>? = null
    fun getCached(): List<Crypto>? = cache
    fun save(data: List<Crypto>) { cache = data }
}

// Repository
class CryptoRepository(
    private val remote: CryptoRemoteDataSource,
    private val local: CryptoLocalDataSource
) {
    suspend fun getCryptos(): List<Crypto> {
        // Cache-first strategy
        local.getCached()?.let { return it }

        // Fetch from network, cache, return
        return remote.getCryptos()
            .map { it.toDomain() } // DTO → Domain model
            .also { local.save(it) }
    }
}

// ✅ ViewModel only talks to Repository
class CryptoViewModel(private val repository: CryptoRepository) : ViewModel() {
    fun loadCryptos() {
        viewModelScope.launch {
            val cryptos = repository.getCryptos() // doesn't know or care about source
        }
    }
}
```

### ❌ Anti-patterns

```kotlin
// ❌ BAD: ViewModel makes direct API calls
class CryptoViewModel(private val api: CryptoApi) : ViewModel() {
    fun load() { viewModelScope.launch { api.getCryptos() } } // no caching, no abstraction
}

// ❌ BAD: Composable makes network calls
@Composable
fun CryptoScreen() {
    LaunchedEffect(Unit) { api.getCryptos() } // NEVER do this
}
```

---

## 10. Coroutine Exception Handling

### The problem

Uncaught exceptions in coroutines can silently fail or crash the app unpredictably.

### try/catch in viewModelScope

```kotlin
// ✅ GOOD: catch at the right level
viewModelScope.launch {
    try {
        val data = repository.getData() // IOException, HttpException
        _uiState.value = UiState.Success(data)
    } catch (e: IOException) {
        _uiState.value = UiState.Error("Network error. Check connection.")
    } catch (e: HttpException) {
        _uiState.value = UiState.Error("Server error: ${e.code()}")
    } catch (e: Exception) {
        _uiState.value = UiState.Error("Unexpected error")
    }
}
```

### CoroutineExceptionHandler

Global handler for uncaught exceptions — use as a safety net, not primary handling:

```kotlin
val handler = CoroutineExceptionHandler { _, throwable ->
    _uiState.value = UiState.Error(throwable.message ?: "Error")
}

viewModelScope.launch(handler) {
    // handler catches if exception escapes try/catch
    repository.getData()
}
```

### launch vs async exception behavior

```kotlin
// launch — exception propagates immediately to parent scope
viewModelScope.launch {
    throw IOException() // crashes scope immediately
}

// async — exception stored in Deferred, thrown on .await()
val deferred = viewModelScope.async {
    throw IOException() // stored here
}
deferred.await() // exception thrown HERE — must catch here

// ✅ Pattern for parallel calls:
viewModelScope.launch {
    try {
        val (prices, news) = awaitAll(
            async { repository.getPrices() },
            async { repository.getNews() }
        )
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message ?: "Error")
    }
}
```

### SupervisorJob — don't let one failure kill siblings

```kotlin
// Without SupervisorJob: if one child fails, ALL children cancelled
val scope = CoroutineScope(Job() + Dispatchers.Main)
scope.launch { fetchPrices() } // fails → kills scope
scope.launch { fetchNews() }   // ALSO cancelled

// ✅ With SupervisorScope: each child fails independently
viewModelScope.launch { // viewModelScope already uses SupervisorJob
    supervisorScope {
        launch { fetchPrices() } // fails → only this cancelled
        launch { fetchNews() }   // continues normally
    }
}
```

### Dispatcher choice

```kotlin
Dispatchers.Main       // UI updates, Compose state
Dispatchers.Main.immediate // ViewModel work (default in viewModelScope)
Dispatchers.IO         // Network, file I/O, database (thread pool optimized)
Dispatchers.Default    // CPU-intensive: sorting, parsing, algorithms
Dispatchers.Unconfined // Rarely use — no thread guarantee
```

---

## 11. Navigation — NavHost, Routes, Back Stack

### Basic setup (interview-ready)

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(onNavigateToDetail = { id ->
                navController.navigate("detail/$id")
            })
        }
        composable(
            route = "detail/{itemId}",
            arguments = listOf(navArgument("itemId") { type = NavType.StringType })
        ) { backStackEntry ->
            val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
            DetailScreen(itemId = itemId)
        }
    }
}
```

### Typed routes (safer, production pattern)

```kotlin
sealed class Screen(val route: String) {
    object Home : Screen("home")
    data class Detail(val itemId: String) : Screen("detail/$itemId") {
        companion object {
            const val route = "detail/{itemId}"
        }
    }
}

// Usage:
navController.navigate(Screen.Detail("btc-123").route)
```

### Back stack management

```kotlin
// ✅ Navigate and clear back stack (e.g., after login)
navController.navigate("home") {
    popUpTo("login") { inclusive = true }
}

// ✅ Avoid duplicate destinations (e.g., bottom nav)
navController.navigate("home") {
    popUpTo(navController.graph.findStartDestination().id) {
        saveState = true
    }
    launchSingleTop = true
    restoreState = true
}

// ❌ DON'T: navigate in LaunchedEffect with unstable keys — causes multiple navigations
```

### Passing ViewModel to screens (interview pattern)

```kotlin
// ✅ Create ViewModel at NavHost level and pass down, OR
// ✅ Use navigation graph scoped ViewModel

composable("detail/{id}") { entry ->
    val parentEntry = remember(entry) {
        navController.getBackStackEntry("home")
    }
    val sharedViewModel: SharedViewModel = viewModel(parentEntry)
    DetailScreen(viewModel = sharedViewModel)
}
```

---

## 12. LazyColumn — Performance Internals

### How LazyColumn works

LazyColumn only composes items visible on screen + a small buffer. Items not visible are decommissioned (destroyed). When you scroll, new items are composed and old ones discarded.

### The key parameter — item identity

Without `key`, Compose uses **position** as identity. When items reorder, all items recompose. With `key`, Compose tracks items by identity — only truly changed items recompose.

```kotlin
// ❌ BAD: no key — all items recompose on list change
LazyColumn {
    items(cryptoList) { crypto ->
        CryptoItem(crypto)
    }
}

// ✅ GOOD: key = stable unique identifier
LazyColumn {
    items(
        items = cryptoList,
        key = { it.id } // stable, unique → Compose tracks identity
    ) { crypto ->
        CryptoItem(crypto)
    }
}

// ✅ Also use key for other LazyList content types:
LazyColumn {
    item(key = "header") { Header() }
    items(items = list, key = { it.id }) { ItemRow(it) }
    item(key = "footer") { Footer() }
}
```

### contentType — skip recomposition for different item types

```kotlin
// ✅ For mixed-type lists: tell Compose items share same template
LazyColumn {
    items(
        items = feedItems,
        key = { it.id },
        contentType = { item ->
            when (item) {
                is FeedItem.PriceCard -> "price"
                is FeedItem.NewsCard -> "news"
                is FeedItem.AdCard -> "ad"
            }
        }
    ) { item ->
        when (item) {
            is FeedItem.PriceCard -> PriceCardItem(item)
            is FeedItem.NewsCard -> NewsCardItem(item)
            is FeedItem.AdCard -> AdCardItem(item)
        }
    }
}
```

### Avoid heavy work in item composables

```kotlin
// ❌ BAD: parsing in composable body — runs on every recomposition
@Composable
fun PriceItem(price: CryptoPrice) {
    val formatted = String.format("%.4f", price.value) // runs every recompose
    Text(formatted)
}

// ✅ GOOD: derive in ViewModel or use remember
@Composable
fun PriceItem(price: CryptoPrice) {
    val formatted = remember(price.value) { String.format("%.4f", price.value) }
    Text(formatted)
}
```

---

## 13. Modifier — Ordering, Internals, and Performance

### How Modifiers work internally

A `Modifier` is a linked list of `Modifier.Element` objects. When Compose processes a composable, it walks this chain left to right, applying each element in order. Each element wraps the next — like decorators.

```
Modifier.padding(16.dp).clip(RoundedCornerShape(8.dp)).background(Color.Blue).clickable { }

Chain: padding → clip → background → clickable → [Composable content]

Processing order:
1. padding: adds spacing around content
2. clip: clips drawing to rounded shape
3. background: fills clipped area with blue
4. clickable: adds touch handler to clipped+padded area
```

### Why order matters — visual result

```kotlin
// ✅ Padding OUTSIDE background: padding area is transparent
Box(Modifier.padding(16.dp).background(Color.Blue))
// Result: [transparent 16dp gap][blue box]

// ✅ Background OUTSIDE padding: padding area is also blue
Box(Modifier.background(Color.Blue).padding(16.dp))
// Result: [blue box including 16dp padding area]

// ✅ Clickable THEN padding: touch target includes padding
Box(Modifier.clickable { }.padding(16.dp))
// Touch target = content + 16dp padding ← larger touch area ✅

// ❌ Padding THEN clickable: touch target is content only
Box(Modifier.padding(16.dp).clickable { })
// Touch target = content only ← padding not tappable (frustrating UX)

// ✅ RULE for clickable: clickable() before padding for full touch target
Modifier.clickable { }.padding(16.dp)

// ✅ RULE for background + clip: clip BEFORE background
Modifier.clip(RoundedCornerShape(8.dp)).background(Color.Blue)
// Without clip first: background draws as rectangle, THEN clip cuts it ❌
// With clip first: background is constrained to rounded shape ✅
```

### Modifier types — internals

Compose has three internal Modifier interfaces:

```kotlin
// 1. DrawModifier — affects how content is painted (background, border, alpha)
// Runs during Drawing phase — cheapest, doesn't affect layout
Modifier.background(Color.Blue)  // DrawModifier
Modifier.alpha(0.5f)             // DrawModifier
Modifier.border(1.dp, Color.Red) // DrawModifier

// 2. LayoutModifier — affects size and position (padding, size, offset)
// Runs during Layout phase — more expensive, affects everything below
Modifier.padding(16.dp)       // LayoutModifier
Modifier.size(100.dp)         // LayoutModifier
Modifier.fillMaxWidth()       // LayoutModifier
Modifier.offset(x = 8.dp)    // LayoutModifier (prefer graphicsLayer for animations)

// 3. PointerInputModifier — handles touch (clickable, draggable, scrollable)
Modifier.clickable { }
Modifier.draggable(...)
Modifier.scrollable(...)
```

### graphicsLayer — the animation modifier

`graphicsLayer` runs at the **Drawing phase**, not Layout. This means it can animate without triggering a layout pass — much cheaper for animations.

```kotlin
// ❌ SLOW: animating with offset() triggers layout on every frame
val offsetX by animateFloatAsState(if (visible) 0f else -200f)
Box(Modifier.offset { IntOffset(offsetX.toInt(), 0) }) // layout recalculated every frame

// ✅ FAST: graphicsLayer skips layout — only Drawing phase runs
val offsetX by animateFloatAsState(if (visible) 0f else -200f)
Box(Modifier.graphicsLayer { translationX = offsetX }) // no layout recalculation ✅

// graphicsLayer properties:
Modifier.graphicsLayer {
    alpha = 0.8f          // transparency
    scaleX = 1.2f         // horizontal scale
    scaleY = 1.2f         // vertical scale
    translationX = 50f    // horizontal offset (px)
    translationY = 0f     // vertical offset (px)
    rotationZ = 45f       // rotation degrees
    shadowElevation = 8f  // drop shadow
    clip = true           // clip content to bounds
}
```

### Performance: static vs dynamic modifiers

```kotlin
// ❌ BAD: Modifier created in composable body — new object every recomposition
@Composable
fun CryptoItem(crypto: Crypto) {
    Row(modifier = Modifier.fillMaxWidth().padding(16.dp).clickable { }) { ... }
    // New Modifier chain allocated on every recompose ← GC pressure
}

// ✅ GOOD: static parts extracted to top-level val — created once
private val itemModifier = Modifier
    .fillMaxWidth()
    .padding(horizontal = 16.dp, vertical = 8.dp)

@Composable
fun CryptoItem(crypto: Crypto, onClick: () -> Unit) {
    Row(modifier = itemModifier.clickable(onClick = onClick)) { ... }
    // itemModifier reused — only clickable() lambda changes ✅
}

// ✅ BETTER: use Modifier parameter to allow caller to customise
@Composable
fun CryptoItem(
    crypto: Crypto,
    modifier: Modifier = Modifier, // always expose modifier param ✅
    onClick: () -> Unit
) {
    Row(modifier = modifier.clickable(onClick = onClick)) { ... }
}
```

### Modifier best practices summary

```kotlin
// Rule 1: clickable() before padding() for full touch target
Modifier.clickable { }.padding(16.dp) ✅

// Rule 2: clip() before background() for correct shape
Modifier.clip(RoundedCornerShape(8.dp)).background(Color.Blue) ✅

// Rule 3: graphicsLayer for animations, not offset/alpha modifiers
Modifier.graphicsLayer { translationX = animValue } ✅

// Rule 4: always expose modifier param in reusable composables
fun MyComponent(modifier: Modifier = Modifier) ✅

// Rule 5: extract static modifiers to top-level val
private val staticMod = Modifier.fillMaxWidth().padding(16.dp) ✅

// Rule 6: for Compose interop with View system
Modifier.drawWithContent { } // custom drawing that works with Compose pipeline
```

---

## 14. Manual Dependency Injection (No Hilt)

In a 75-min interview, you won't set up Hilt. Here's how to do clean, scoped DI manually that demonstrates architectural understanding.

### Why DI matters

Without DI, classes create their own dependencies — tightly coupled, untestable:

```kotlin
// ❌ No DI — tightly coupled
class CryptoViewModel : ViewModel() {
    private val api = Retrofit.Builder()
        .baseUrl("https://api.coingecko.com/")
        .build()
        .create(CryptoApi::class.java)
    // Cannot test without real network
    // Cannot swap implementation
    // Retrofit created fresh every time ViewModel created
}

// ✅ DI — dependencies injected from outside
class CryptoViewModel(
    private val repository: CryptoRepository // injected ✅
) : ViewModel()
```

### AppContainer — app-scoped dependencies

```kotlin
// Single instance for app lifetime — network, shared resources
class AppContainer {
    // lazy = created only when first accessed
    val okHttpClient by lazy {
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG)
                    HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
            })
            .build()
    }

    val cryptoApi: CryptoApi by lazy {
        Retrofit.Builder()
            .baseUrl("https://api.coingecko.com/api/v3/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(CryptoApi::class.java)
    }

    val localDataSource by lazy { CryptoLocalDataSource() }
    val remoteDataSource by lazy { CryptoRemoteDataSource(cryptoApi) }
    val cryptoRepository: CryptoRepository by lazy {
        CryptoRepositoryImpl(remoteDataSource, localDataSource)
    }
}
```

### ScreenContainer — screen-scoped dependencies

Some dependencies should live as long as a screen, not the whole app:

```kotlin
// Screen-scoped container — created when screen entered, destroyed when left
class CryptoScreenContainer(private val appContainer: AppContainer) {
    val viewModel: CryptoViewModel by lazy {
        CryptoViewModel(appContainer.cryptoRepository)
    }
}
```

### ViewModelFactory — connecting DI to ViewModel

```kotlin
class CryptoViewModel(
    private val repository: CryptoRepository
) : ViewModel() {

    companion object {
        fun factory(repository: CryptoRepository) = object : ViewModelProvider.Factory {
            @Suppress("UNCHECKED_CAST")
            override fun <T : ViewModel> create(modelClass: Class<T>): T =
                CryptoViewModel(repository) as T
        }
    }
}

// Usage in Composable:
@Composable
fun CryptoScreen(appContainer: AppContainer) {
    val viewModel: CryptoViewModel = viewModel(
        factory = CryptoViewModel.factory(appContainer.cryptoRepository)
    )
    // ...
}
```

### Wiring it all together in NavHost

```kotlin
@Composable
fun AppRoot() {
    // AppContainer lives for the lifetime of this composition
    val appContainer = remember { AppContainer() }
    val navController = rememberNavController()

    NavHost(navController, startDestination = "crypto_list") {
        composable("crypto_list") {
            val viewModel: CryptoViewModel = viewModel(
                factory = CryptoViewModel.factory(appContainer.cryptoRepository)
            )
            CryptoListScreen(viewModel = viewModel)
        }
        composable("crypto_detail/{id}") { backStack ->
            val id = backStack.arguments?.getString("id") ?: ""
            val viewModel: DetailViewModel = viewModel(
                factory = DetailViewModel.factory(appContainer.cryptoRepository, id)
            )
            DetailScreen(viewModel = viewModel)
        }
    }
}
```

### Sharing ViewModel between screens

```kotlin
// Use parent back stack entry to scope ViewModel to a navigation graph
composable("detail/{id}") { entry ->
    val parentEntry = remember(entry) {
        navController.getBackStackEntry("crypto_list")
    }
    val sharedViewModel: SharedCryptoViewModel = viewModel(parentEntry)
    DetailScreen(viewModel = sharedViewModel)
}
```

### What to say in interview

*"In production I'd use Hilt — it handles scoping, lifecycle, and code generation. Here I'm wiring dependencies manually to keep the interview focused on architecture. The key principles are the same: single instances for app-level resources like Retrofit, injected interfaces for testability, and ViewModelFactory to connect the DI graph to Compose's ViewModel system."*

---

## 15. Testability Patterns

Ojaswirajanya specifically values testable code. Even if you don't write tests in 75 min, STRUCTURE your code as if you will — and narrate how you'd test it.

### The testing pyramid

```
        /\
       /  \    UI Tests (slow, fragile, few)
      /────\   Compose UI tests, Espresso
     /      \
    /────────\  Integration Tests (medium)
   /          \ Repository + DataSource together
  /────────────\
 /  Unit Tests  \ (fast, reliable, many)
/________________\ ViewModel, Repository, Domain logic
```

### Unit testing ViewModels — runTest + TestCoroutineDispatcher

```kotlin
// ✅ ViewModel takes interfaces — inject fakes
interface CryptoRepository {
    suspend fun getCryptos(): List<Crypto>
    val cryptos: StateFlow<List<Crypto>>
}

class FakeCryptoRepository : CryptoRepository {
    var shouldThrow = false
    var fakeData = listOf<Crypto>()
    private val _cryptos = MutableStateFlow(fakeData)
    override val cryptos: StateFlow<List<Crypto>> = _cryptos.asStateFlow()

    override suspend fun getCryptos(): List<Crypto> {
        if (shouldThrow) throw IOException("Network error")
        return fakeData.also { _cryptos.value = it }
    }
}

// Test class
@OptIn(ExperimentalCoroutinesApi::class)
class CryptoViewModelTest {

    // Replace Main dispatcher with test dispatcher — controls coroutine execution
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private lateinit var repo: FakeCryptoRepository
    private lateinit var viewModel: CryptoViewModel

    @Before
    fun setup() {
        repo = FakeCryptoRepository()
        viewModel = CryptoViewModel(repo)
    }

    @Test
    fun `initial state is loading`() {
        assertEquals(UiState.Loading, viewModel.uiState.value)
    }

    @Test
    fun `load success emits success state`() = runTest {
        repo.fakeData = listOf(Crypto("btc", "Bitcoin", 50000.0))
        viewModel.loadCryptos()
        val state = viewModel.uiState.value
        assertTrue(state is UiState.Success)
        assertEquals(1, (state as UiState.Success).data.size)
    }

    @Test
    fun `network error emits error state`() = runTest {
        repo.shouldThrow = true
        viewModel.loadCryptos()
        assertTrue(viewModel.uiState.value is UiState.Error)
    }
}

// MainDispatcherRule — replaces Dispatchers.Main with test dispatcher
class MainDispatcherRule(
    private val dispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {
    override fun starting(description: Description) {
        Dispatchers.setMain(dispatcher)
    }
    override fun finished(description: Description) {
        Dispatchers.resetMain()
        dispatcher.cleanupTestCoroutines()
    }
}
```

### Testing StateFlow with Turbine

Turbine is a library that makes testing Flows ergonomic:

```kotlin
// Without Turbine — verbose and fragile
@Test
fun testWithoutTurbine() = runTest {
    val emissions = mutableListOf<UiState<List<Crypto>>>()
    val job = launch { viewModel.uiState.collect { emissions.add(it) } }
    viewModel.loadCryptos()
    advanceUntilIdle()
    assertEquals(UiState.Loading, emissions[0])
    assertTrue(emissions[1] is UiState.Success)
    job.cancel()
}

// ✅ With Turbine — clean and readable
@Test
fun `load emits loading then success`() = runTest {
    repo.fakeData = listOf(Crypto("btc", "Bitcoin", 50000.0))

    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())     // first emission
        viewModel.loadCryptos()
        assertTrue(awaitItem() is UiState.Success)     // second emission
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Compose UI testing — ComposeTestRule

```kotlin
@get:Rule
val composeRule = createComposeRule()

@Test
fun `shows loading indicator initially`() {
    composeRule.setContent {
        CryptoScreen(uiState = UiState.Loading, onRetry = {})
    }
    composeRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
}

@Test
fun `shows crypto list on success`() {
    val cryptos = listOf(Crypto("btc", "Bitcoin", 50000.0))
    composeRule.setContent {
        CryptoScreen(uiState = UiState.Success(cryptos), onRetry = {})
    }
    composeRule.onNodeWithText("Bitcoin").assertIsDisplayed()
}

@Test
fun `clicking item fires callback`() {
    var clicked: Crypto? = null
    val cryptos = listOf(Crypto("btc", "Bitcoin", 50000.0))
    composeRule.setContent {
        CryptoList(items = cryptos, onItemClick = { clicked = it })
    }
    composeRule.onNodeWithText("Bitcoin").performClick()
    assertEquals("btc", clicked?.id)
}

// Semantic matchers:
// onNodeWithText("Bitcoin")       — find by visible text
// onNodeWithTag("loading")        — find by testTag
// onNodeWithContentDescription()  — find by accessibility label
// assertIsDisplayed()             — verify visible
// assertIsEnabled() / assertIsNotEnabled()
// performClick() / performScrollTo() / performTextInput("hello")
```

### Adding testTags to composables

```kotlin
// ✅ Add testTag for UI test targeting
@Composable
fun LoadingIndicator() {
    CircularProgressIndicator(
        modifier = Modifier.testTag("loading_indicator")
    )
}

@Composable
fun CryptoItem(crypto: Crypto, onClick: () -> Unit) {
    Row(
        modifier = Modifier
            .testTag("crypto_item_${crypto.id}")
            .clickable(onClick = onClick)
    ) { ... }
}
```

### What to say during the interview

*"I've structured every composable to be stateless — data in, callbacks out — which makes them trivially testable with ComposeTestRule. The ViewModel depends on a repository interface so I can inject a fake in tests without hitting the network. For state transitions I'd use `runTest` with `Turbine` to assert each emission in sequence. If you want I can walk through how I'd test the Loading → Success → Error flow for this ViewModel."*

---

## 16. Performance Optimization

### The render pipeline — frame budget

Android targets 60fps = 16.67ms per frame (90fps = 11.1ms, 120fps = 8.3ms).

Every frame goes through:
```
Input handling → Animation → Measure → Layout → Draw → RenderThread → GPU
```

If any phase takes too long → dropped frame → jank. The CPU must finish all phases within the frame budget.

### Choreographer and VSYNC

`Choreographer` is Android's frame scheduler. It listens for VSYNC signals from the display hardware and schedules frame callbacks at the right time.

```
VSYNC signal (every 16.67ms)
    → Choreographer fires callbacks
    → Compose recomposition runs
    → Layout pass
    → Drawing pass
    → Frame submitted to SurfaceFlinger
    → Displayed on screen
```

If your work misses the VSYNC window → frame is delayed by a full 16.67ms = visible stutter.

### Startup performance

```kotlin
// Cold start timeline:
// 1. Process created (slow — OS overhead)
// 2. Application.onCreate() runs
// 3. MainActivity.onCreate() runs
// 4. First frame drawn

// ✅ Keep Application.onCreate() fast — no blocking work
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // ❌ BAD: initializing heavy SDKs synchronously
        // HeavySDK.initialize(this) // blocks startup

        // ✅ GOOD: defer non-critical init
        lifecycleScope.launch(Dispatchers.IO) {
            HeavySDK.initialize(this@MyApplication)
        }
    }
}

// ✅ lazy in AppContainer — init happens on first access, not at startup
class AppContainer {
    val retrofit by lazy { buildRetrofit() }
    val repository by lazy { CryptoRepositoryImpl(remoteDataSource, localDataSource) }
}

// ✅ SplashScreen API (Android 12+) — keeps splash until data ready
override fun onCreate(savedInstanceState: Bundle?) {
    val splashScreen = installSplashScreen()
    super.onCreate(savedInstanceState)
    splashScreen.setKeepOnScreenCondition {
        !viewModel.isInitialized.value // keep splash until VM is ready
    }
}
```

### Baseline Profiles — 30-40% faster startup

Baseline Profiles precompile critical code paths into machine code at install time, skipping JIT compilation on first run.

```kotlin
// Generate with Macrobenchmark:
@ExperimentalBaselineProfilesApi
class BaselineProfileGenerator {
    @get:Rule val rule = BaselineProfileRule()

    @Test
    fun generateBaselineProfile() = rule.collect(
        packageName = "com.example.cryptotracker"
    ) {
        // Critical user journeys to precompile:
        pressHome()
        startActivityAndWait() // app startup
        device.findObject(By.text("Bitcoin")).click()
        device.waitForIdle()
    }
}

// Result: baseline-prof.txt committed to app/src/main
// Gradle plugin automatically includes it in release APK
```

### Rendering performance — recomposition

```kotlin
// ✅ Push state reads as far DOWN the tree as possible
@Composable
fun PriceList(prices: StateFlow<List<Price>>) {
    // ❌ Reading state here recomposes entire PriceList
    val list by prices.collectAsStateWithLifecycle()
    LazyColumn { items(list) { PriceItem(it) } }
}

// ✅ Each item reads its own state — only that item recomposes
@Composable
fun PriceItem(price: Price) {
    val current by price.currentFlow.collectAsStateWithLifecycle()
    Text(current.formatted) // only THIS recomposes on price change
}

// ✅ Use remember for expensive computations
@Composable
fun CryptoChart(data: List<DataPoint>) {
    val path = remember(data) { // recomputes only when data changes
        buildPath(data) // expensive canvas path calculation
    }
    Canvas { drawPath(path, paint) }
}

// ✅ Avoid creating objects in composable body
@Composable
fun CryptoItem(crypto: Crypto) {
    // ❌ BAD: new TextStyle object every recomposition
    Text(crypto.name, style = TextStyle(fontSize = 16.sp, color = Color.White))

    // ✅ GOOD: use MaterialTheme styles or top-level vals
    Text(crypto.name, style = MaterialTheme.typography.bodyLarge)
}
```

### Overdraw — don't draw the same pixel twice

```kotlin
// ❌ BAD: Window background + Activity background + Card background = 3x overdraw
// In theme: windowBackground = white
// Activity root: background = white (redundant)
// Card: background = white (redundant)

// ✅ FIX: remove redundant backgrounds
// Set windowBackground to transparent if Activity sets its own
// Don't add backgrounds to containers that already have them

// ✅ Debug overdraw: Developer Options → Debug GPU Overdraw
// Blue = 1x (fine), Green = 2x (acceptable), Pink/Red = 3x+ (fix this)
```

### Measuring performance tools

```kotlin
// 1. Layout Inspector (Android Studio)
//    View → Tool Windows → Layout Inspector
//    Shows recomposition counts per composable in real time
//    High numbers = instability

// 2. Systrace / Perfetto
//    adb shell perfetto ...
//    Shows frame timeline, identifies long frames
//    Look for: Choreographer#doFrame taking >16ms

// 3. Android Profiler
//    CPU: identify hot methods, thread activity
//    Memory: heap dumps, allocation tracking
//    Network: request timing, payload sizes

// 4. Macrobenchmark — measure startup and scrolling objectively
@LargeTest
class ScrollBenchmark {
    @get:Rule val rule = MacrobenchmarkRule()

    @Test
    fun scrollList() = rule.measureRepeated(
        packageName = "com.example.app",
        metrics = listOf(FrameTimingMetric()),
        iterations = 5,
        setupBlock = { startActivityAndWait() }
    ) {
        val list = device.findObject(By.res("crypto_list"))
        list.fling(Direction.DOWN)
    }
}
// Reports: P50/P90/P99 frame durations — find your jank percentile
```

### R8 / ProGuard — release build optimisation

```kotlin
// Enabled in release builds (Chapter 27 covers config)
// R8 does:
// 1. Shrinking: removes unused classes, methods, fields
// 2. Obfuscation: renames to short names (a, b, c)
// 3. Optimisation: inlines methods, removes dead code
// Result: smaller APK, harder to reverse-engineer, faster class loading
```

---

## 17. Battery Optimization

### The core principle

The biggest battery drains: **radio (network)**, **GPS**, **CPU wakeups**, **screen**.

### Network — batch and cache

```kotlin
// ❌ BAD: polling every second
LaunchedEffect(Unit) {
    while (true) {
        fetchPrices()
        delay(1000) // 60 network calls/min
    }
}

// ✅ GOOD: reasonable polling interval
LaunchedEffect(Unit) {
    while (true) {
        fetchPrices()
        delay(30_000) // 2 calls/min — 30x less radio wake
    }
}

// ✅ BETTER: stop polling when app is in background
LaunchedEffect(Unit) {
    repeatOnLifecycle(Lifecycle.State.STARTED) { // only polls when foreground
        while (true) {
            fetchPrices()
            delay(30_000)
        }
    }
}
```

### repeatOnLifecycle — the right way to scope background work

```kotlin
// ✅ In Activity/Fragment (not needed in Compose — collectAsStateWithLifecycle handles it)
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { render(it) }
        // automatically pauses in background, resumes in foreground
    }
}
```

### WorkManager for deferrable background work

```kotlin
// ✅ For battery-efficient background tasks:
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true) // won't run on low battery
    .build()

val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(constraints)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "price_sync",
    ExistingPeriodicWorkPolicy.KEEP,
    syncRequest
)
// System batches this with other work — less radio wakeups
```

### WakeLock — avoid unless necessary

```kotlin
// ❌ BAD: manual WakeLock — easy to forget to release = battery drain
val wakeLock = powerManager.newWakeLock(PARTIAL_WAKE_LOCK, "app::tag")
wakeLock.acquire()
// forgot to release!

// ✅ GOOD: use WorkManager which manages WakeLocks automatically
```

### Avoid unnecessary location updates

```kotlin
// ❌ BAD: GPS always on
locationManager.requestLocationUpdates(GPS_PROVIDER, 0, 0f, listener)

// ✅ GOOD: network provider for low-priority, GPS only when needed
locationManager.requestLocationUpdates(
    NETWORK_PROVIDER,
    minTimeMs = 60_000,   // at most once per minute
    minDistanceM = 100f,  // or 100m moved
    listener
)
```

---

## 18. Memory Management & Memory Leaks

### What is a memory leak?

A memory leak occurs when an object that is no longer needed cannot be garbage collected because something else still holds a reference to it. On Android, the GC cannot reclaim memory held by leaked objects — over time this causes:
- Increasing heap usage
- OutOfMemoryError crashes
- ANRs from GC pressure
- App killed by system for excessive memory use

### How Android GC works

Android uses a generational garbage collector. Objects live in:
- **Young generation**: new objects — collected frequently, cheap
- **Old generation**: long-lived objects — collected less often, expensive
- **Large objects heap**: bitmaps and large allocations

A leaked Activity/Fragment prevents the GC from collecting that object AND everything it references — which can be 10s of MBs (views, bitmaps, adapters, etc.).

---

### Leak Type 1 — Context leaks (most common, most damaging)

**The rule:** Never store Activity or Fragment context in anything that outlives it.

```kotlin
// ❌ LEAK: Singleton holding Activity context
object ImageLoader {
    var context: Context? = null // Activity stored in singleton = leaked forever
}

// ❌ LEAK: ViewModel holding Activity
class MyViewModel(private val activity: MainActivity) : ViewModel()
// ViewModel outlives Activity — Activity cannot be GC'd

// ❌ LEAK: static reference to Activity
companion object {
    var instance: MainActivity? = null // static = GC root = never collected
}

// ✅ FIX: Use Application context in singletons and ViewModels
object ImageLoader {
    lateinit var appContext: Context // set once in Application.onCreate()
}

class MyViewModel(application: Application) : AndroidViewModel(application) {
    private val context = application.applicationContext // safe — lives as long as app
}

// ✅ FIX: WeakReference if you genuinely need Activity reference
class MyPresenter(activity: MainActivity) {
    private val activityRef = WeakReference(activity)
    fun doSomething() {
        activityRef.get()?.updateUI() // null-safe — GC can collect Activity
    }
}
```

---

### Leak Type 2 — Listener and callback leaks

**The rule:** Every register must have a matching unregister.

```kotlin
// ❌ LEAK: registering listener in onCreate, never removing
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        locationManager.requestLocationUpdates(provider, 0, 0f, locationListener)
        // Activity destroyed but locationManager still holds reference → leak
    }
}

// ✅ FIX: unregister in matching lifecycle callback
class MainActivity : AppCompatActivity() {
    override fun onStart() {
        super.onStart()
        locationManager.requestLocationUpdates(provider, 0, 0f, locationListener)
    }
    override fun onStop() {
        super.onStop()
        locationManager.removeUpdates(locationListener) // symmetric ✅
    }
}

// ❌ LEAK: anonymous inner class listener — holds implicit reference to outer class
class MyFragment : Fragment() {
    fun setupButton() {
        button.setOnClickListener(object : View.OnClickListener {
            override fun onClick(v: View) {
                // implicit reference to MyFragment — Fragment leaked if button outlives it
                this@MyFragment.doSomething()
            }
        })
    }
}

// ✅ FIX: use lambda (no implicit reference) or WeakReference
button.setOnClickListener { doSomething() } // lambda — no implicit outer reference
```

---

### Leak Type 3 — Coroutine scope leaks

```kotlin
// ❌ LEAK: unscoped CoroutineScope — never cancelled
class DataManager {
    private val scope = CoroutineScope(Dispatchers.IO) // no owner, never cancelled
    fun sync() { scope.launch { api.sync() } } // leak if DataManager is abandoned
}

// ❌ LEAK: GlobalScope — lives forever
GlobalScope.launch { repository.fetch() } // coroutine outlives all components

// ✅ FIX: scope tied to lifecycle owner
// ViewModel work:
class MyViewModel : ViewModel() {
    fun load() { viewModelScope.launch { ... } } // cancelled when VM cleared ✅
}

// Activity/Fragment work:
lifecycleScope.launch { ... } // cancelled when Activity/Fragment destroyed ✅
viewLifecycleOwner.lifecycleScope.launch { ... } // cancelled when view destroyed ✅

// One-off background work with no lifecycle owner:
class DataManager(private val scope: CoroutineScope) { // inject scope from outside
    fun sync() { scope.launch { api.sync() } } // caller controls lifetime ✅
}
```

---

### Leak Type 4 — Handler and Runnable leaks

```kotlin
// ❌ LEAK: Handler with delayed Runnable holds Activity reference
class MainActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())

    override fun onCreate(savedInstanceState: Bundle?) {
        handler.postDelayed({
            updateUI() // implicit reference to MainActivity
        }, 5000)
        // If Activity finishes before 5s — Runnable still runs, Activity leaked
    }
}

// ✅ FIX: remove callbacks in onDestroy
override fun onDestroy() {
    super.onDestroy()
    handler.removeCallbacksAndMessages(null) // cancel all pending ✅
}

// ✅ BETTER FIX: use coroutines instead of Handler
override fun onCreate(savedInstanceState: Bundle?) {
    lifecycleScope.launch {
        delay(5000)
        updateUI() // cancelled automatically if Activity destroyed ✅
    }
}
```

---

### Leak Type 5 — Inner class leaks

```kotlin
// ❌ LEAK: non-static inner class holds implicit reference to outer class
class MainActivity : AppCompatActivity() {
    inner class MyAsyncTask : AsyncTask<Void, Void, String>() {
        override fun doInBackground(vararg params: Void?): String {
            return fetchData() // holds reference to MainActivity implicitly
        }
        override fun onPostExecute(result: String?) {
            textView.text = result // if Activity destroyed mid-task — leak + crash
        }
    }
}

// ✅ FIX: use static inner class + WeakReference OR use coroutines
class MainActivity : AppCompatActivity() {
    fun loadData() {
        lifecycleScope.launch {
            val result = withContext(Dispatchers.IO) { fetchData() }
            textView.text = result // safe — cancelled if Activity destroyed ✅
        }
    }
}
```

---

### Leak Type 6 — Compose-specific leaks

```kotlin
// ❌ LEAK: capturing Activity/Context in a lambda stored in ViewModel
class MyViewModel : ViewModel() {
    var onSuccess: (() -> Unit)? = null // if set to a lambda capturing Activity → leak
}

// In Activity:
viewModel.onSuccess = { this.showToast("Done") } // 'this' = Activity captured in ViewModel ❌
// ViewModel outlives Activity → Activity leaked

// ✅ FIX: use SharedFlow for events — no references stored
class MyViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>()
    val events = _events.asSharedFlow()
    // Composable collects events — no Activity reference in ViewModel ✅
}

// ❌ LEAK: DisposableEffect without onDispose
DisposableEffect(Unit) {
    val listener = SomeManager.addListener { }
    // missing onDispose — listener never removed ❌
}

// ✅ FIX: always onDispose
DisposableEffect(Unit) {
    val listener = SomeManager.addListener { }
    onDispose { SomeManager.removeListener(listener) } // ✅
}

// ❌ LEAK: rememberUpdatedState misuse — capturing stale lambda
@Composable
fun Timer(onTick: () -> Unit) {
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            onTick() // may capture stale lambda if onTick changes ❌
        }
    }
}

// ✅ FIX: rememberUpdatedState captures latest lambda
@Composable
fun Timer(onTick: () -> Unit) {
    val currentOnTick by rememberUpdatedState(onTick)
    LaunchedEffect(Unit) {
        while (true) {
            delay(1000)
            currentOnTick() // always latest version ✅
        }
    }
}
```

---

### Leak Type 7 — Bitmap and drawable leaks

```kotlin
// ❌ LEAK: Bitmap not recycled (pre-API 26, still relevant for large bitmaps)
val bitmap = BitmapFactory.decodeFile(path)
imageView.setImageBitmap(bitmap)
// bitmap held by imageView → if imageView leaked → bitmap leaked

// ✅ FIX: use Coil/Glide — handles lifecycle, cancellation, recycling
AsyncImage(model = path, contentDescription = null)
// Automatically cancelled when composable leaves ✅
// Memory cache with LRU — auto-evicts under pressure ✅

// ❌ LEAK: Drawable with callback pointing to View
val drawable = AnimatedVectorDrawable()
drawable.registerAnimationCallback(object : Animatable2.AnimationCallback() {
    override fun onAnimationEnd(drawable: Drawable?) {
        imageView.visibility = View.GONE // imageView reference captured ❌
    }
})

// ✅ FIX: unregister in onDetachedFromWindow or use WeakReference
override fun onDetachedFromWindow() {
    super.onDetachedFromWindow()
    drawable.unregisterAnimationCallback(callback) // ✅
}
```

---

### Leak Type 8 — RxJava / Flow subscription leaks

```kotlin
// ❌ LEAK: RxJava subscription not disposed
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        viewModel.data.subscribe { render(it) } // subscription lives forever ❌
    }
}

// ✅ FIX: use CompositeDisposable
class MainActivity : AppCompatActivity() {
    private val disposables = CompositeDisposable()
    override fun onCreate(savedInstanceState: Bundle?) {
        disposables.add(viewModel.data.subscribe { render(it) })
    }
    override fun onDestroy() {
        super.onDestroy()
        disposables.clear() // cancel all subscriptions ✅
    }
}

// ✅ BETTER: use Flow + collectAsStateWithLifecycle — no manual disposal needed
val state by viewModel.state.collectAsStateWithLifecycle() // auto-cancelled ✅
```

---

### Detecting and diagnosing leaks

```kotlin
// 1. LeakCanary — add to debug build only
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.x")
    // Automatically detects and reports leaks with stack trace
    // Shows exact reference chain: GC Root → ... → Leaked Object
}

// 2. Android Studio Memory Profiler
// Run app → Profiler → Memory → Record → Trigger leak → Dump heap
// Filter by class, check retained size, find reference chain

// 3. adb shell dumpsys meminfo <package>
// Shows: Java Heap, Native Heap, Code, Stack, Graphics, System
// PSS (Proportional Set Size) = actual memory contribution

// 4. Compose-specific: Layout Inspector
// View → Tool Windows → Layout Inspector
// Check recomposition counts — high counts = instability/potential issue
```

---

### Memory leak prevention checklist

| Scenario | Rule |
|---|---|
| ViewModel | Never pass Activity/Fragment/View as constructor param |
| Singleton | Only Application context, never Activity context |
| Listeners | Every register() needs unregister() in symmetric lifecycle callback |
| Coroutines | Always use viewModelScope / lifecycleScope / viewLifecycleOwner.lifecycleScope |
| Handler/Runnable | removeCallbacksAndMessages(null) in onDestroy, or use coroutines |
| Inner classes | Make static or use coroutines instead of AsyncTask |
| Compose lambdas | Never store Activity reference in ViewModel — use SharedFlow events |
| DisposableEffect | Always implement onDispose { } |
| Bitmaps | Always use Coil/Glide, never load raw bitmaps for UI |
| RxJava | CompositeDisposable.clear() in onDestroy |
| Flow | collectAsStateWithLifecycle() — never raw collectAsState() |

---

## 19. Network Optimization & Reliability

### Retrofit setup (interview-ready)

```kotlin
interface CryptoApi {
    @GET("coins/markets")
    suspend fun getCryptos(
        @Query("vs_currency") currency: String = "usd",
        @Query("order") order: String = "market_cap_desc",
        @Query("per_page") perPage: Int = 20,
        @Query("page") page: Int = 1
    ): List<CryptoDto>
}

// Retrofit instance
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.coingecko.com/api/v3/")
    .client(
        OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = HttpLoggingInterceptor.Level.BODY // debug only
            })
            .build()
    )
    .addConverterFactory(GsonConverterFactory.create())
    .build()
```

### Retry logic

```kotlin
// ✅ Simple retry with exponential backoff
suspend fun <T> retryWithBackoff(
    times: Int = 3,
    initialDelay: Long = 1000,
    block: suspend () -> T
): T {
    var delay = initialDelay
    repeat(times - 1) { attempt ->
        try { return block() } catch (e: IOException) {
            delay(delay)
            delay *= 2 // exponential: 1s, 2s, 4s
        }
    }
    return block() // last attempt, let it throw
}

// Usage:
val data = retryWithBackoff { api.getCryptos() }
```

### Offline-first pattern (say this in interview)

```kotlin
// ✅ Expose Flow from cache, fetch from network, update cache
fun getCryptosStream(): Flow<List<Crypto>> = flow {
    // 1. Emit cached data immediately (fast UI)
    localDataSource.getCached()?.let { emit(it) }

    // 2. Fetch fresh from network
    try {
        val fresh = remoteDataSource.getCryptos()
        localDataSource.save(fresh) // update cache
        emit(fresh) // emit updated data
    } catch (e: IOException) {
        if (localDataSource.getCached() == null) {
            throw e // only throw if no cache to fall back on
        }
        // else: silently use cached data
    }
}
```

---

## 20. App Reliability & Consistency Patterns

### Single source of truth

```kotlin
// ✅ One place owns data — everyone reads from it
class CryptoRepository {
    private val _cryptos = MutableStateFlow<List<Crypto>>(emptyList())
    val cryptos: StateFlow<List<Crypto>> = _cryptos.asStateFlow()

    suspend fun refresh() {
        _cryptos.value = api.getCryptos().map { it.toDomain() }
    }
}
// Multiple ViewModels, multiple screens — all read same _cryptos
```

### Idempotent operations

```kotlin
// ✅ Operations that can be safely retried without side effects
// Save with upsert (insert or update) not duplicate insert
fun saveOrUpdate(crypto: Crypto) {
    val existing = cache.find { it.id == crypto.id }
    if (existing != null) {
        cache[cache.indexOf(existing)] = crypto
    } else {
        cache.add(crypto)
    }
}
```

### State consistency — never update UI from multiple sources

```kotlin
// ❌ BAD: UI updated from both ViewModel and direct API call
// Can lead to race conditions and inconsistent state

// ✅ GOOD: all updates flow through ViewModel → Repository → StateFlow → UI
// UI only reads from StateFlow — single source of truth
```

### Loading states — never show stale data silently

```kotlin
// ✅ Always show what state you're in
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T, val isRefreshing: Boolean = false) : UiState<T>()
    data class Error(val message: String, val cachedData: T? = null) : UiState<Nothing>()
}
// Error with cachedData: show stale data WITH error banner — better UX
```

### Handle configuration changes gracefully

```kotlin
// ✅ ViewModel survives rotation — don't re-fetch if data already loaded
class CryptoViewModel(private val repository: CryptoRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Crypto>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Crypto>>> = _uiState.asStateFlow()

    init {
        loadCryptos() // only called once — ViewModel survives rotation
    }

    // ❌ BAD: if LaunchedEffect calls loadCryptos(), it reruns on every recomposition restart
    // ✅ GOOD: call from init — runs once per ViewModel lifetime
}
```

---

## 21. Build-An-App Interview Checklist

### Time management — 75 minutes

```
0:00 – 0:05  Clarify requirements — ask questions before touching keyboard
0:05 – 0:10  Sketch architecture out loud — data flow, layers, screens
0:10 – 0:20  Scaffold: project structure, AppContainer, interfaces
0:20 – 0:45  Core feature — data layer + ViewModel + main screen
0:45 – 1:05  UI polish — all states, navigation, edge cases
1:05 – 1:10  Narrate what you'd add next — tests, error recovery, pagination
```

**Rule: If you're not talking, something is wrong.** Narrate every decision.

### Before writing a single line — ask these questions

- "Should I use a real API or mock data?" (affects networking setup)
- "How many screens?" (affects NavHost setup)
- "What's the primary user flow?" (affects ViewModel structure)
- "Any specific libraries I should/shouldn't use?"
- "Should I prioritise functionality or code quality if I run out of time?"

### Architecture ✅
- [ ] ViewModel created — no business logic in Composables
- [ ] Repository abstracted from ViewModel via interface
- [ ] Remote DataSource separate from Repository
- [ ] Local cache / in-memory DataSource (even simple Map is fine)
- [ ] AppContainer for dependency wiring

### State ✅
- [ ] UiState sealed class: Loading / Success / Error / Empty
- [ ] `MutableStateFlow` private, `StateFlow` public
- [ ] `collectAsStateWithLifecycle()` — NOT `collectAsState()`
- [ ] `remember` vs `rememberSaveable` used correctly
- [ ] `derivedStateOf` for computed/filtered state

### Compose ✅
- [ ] Every composable stateless (data in, callbacks out)
- [ ] State hoisted to ViewModel
- [ ] Side effects isolated in `LaunchedEffect` / `DisposableEffect`
- [ ] `LazyColumn` uses `key = { it.id }`
- [ ] Loading state shown
- [ ] Error state shown with retry button
- [ ] Empty state shown (not just blank screen)

### Coroutines ✅
- [ ] All network calls in `viewModelScope.launch`
- [ ] `withContext(Dispatchers.IO)` for network/disk
- [ ] `try/catch` around every suspend call
- [ ] No `GlobalScope`

### Code quality ✅
- [ ] Clear naming — no `data`, `temp`, `item`, `result` as variable names
- [ ] Single responsibility per class
- [ ] DTOs mapped to domain models in Repository
- [ ] Sealed class for routes (no magic strings)
- [ ] Constants for strings, tags, keys

### Things to say out loud — senior engineer signals

**Architecture decisions:**
- *"I'm using a sealed class for UiState — makes impossible states unrepresentable. isLoading + error both true is a valid boolean combination but an invalid app state."*
- *"I'm keeping this composable stateless — data flows down, events flow up. Makes it trivially testable."*
- *"In production I'd use Hilt for DI. Here I'm wiring manually with an AppContainer to keep the focus on architecture."*

**Performance signals:**
- *"I'm using `collectAsStateWithLifecycle` instead of `collectAsState` — stops collecting when the app is backgrounded, saves battery."*
- *"Adding a `key` to LazyColumn items — helps Compose track identity on list updates instead of recomposing everything."*
- *"I'd extract this static Modifier to a top-level val — avoids allocating a new Modifier object on every recomposition."*

**Testing signals:**
- *"The repository takes an interface so I can inject a fake in tests — no network needed."*
- *"I'd test the Loading → Success and Loading → Error transitions with runTest and Turbine for flow assertions."*

**Things to say when you run out of time:**
- *"Given more time I'd add pagination here — load more on scroll instead of fetching all 100 items upfront."*
- *"I'd add a retry with exponential backoff — right now it fails hard on network error."*
- *"I'd write unit tests for each UiState transition and a Compose UI test for the main user flow."*
- *"For production I'd add Baseline Profiles for faster startup and LeakCanary in debug builds."*

### Red flags to avoid

- Starting to code before clarifying requirements
- Writing UI first without data layer
- Silent catches — `catch (e: Exception) { }` with no error state update
- Single massive composable with 200 lines of inline UI
- Using `collectAsState()` — she will notice
- No empty state — showing nothing when list is empty
- Passing ViewModel directly to child composables instead of state + callbacks

---

## 22. Patterns & Anti-Patterns — Complete Reference

> This chapter is structured as ✅ Pattern (do this) vs ❌ Anti-Pattern (never do this).  
> Every pair includes WHY it matters, not just what to do.

---

### COMPOSE PATTERNS

---

#### P1 — Unidirectional Data Flow (UDF)

**The pattern:** State flows DOWN (ViewModel → Composable). Events flow UP (Composable → ViewModel).

```kotlin
// ✅ PATTERN: UDF — state down, events up
@Composable
fun SearchScreen(viewModel: SearchViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    SearchContent(
        query = uiState.query,           // state flows DOWN
        results = uiState.results,
        onQueryChange = viewModel::onQueryChange,  // events flow UP
        onSearch = viewModel::onSearch,
        onClearClick = viewModel::onClear
    )
}

// Composable is dumb — receives data, fires callbacks
@Composable
fun SearchContent(
    query: String,
    results: List<Result>,
    onQueryChange: (String) -> Unit,
    onSearch: () -> Unit,
    onClearClick: () -> Unit
) { ... }
```

```kotlin
// ❌ ANTI-PATTERN: bidirectional — composable modifies ViewModel state directly
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    var query by remember { mutableStateOf("") }
    TextField(
        value = query,
        onValueChange = {
            query = it
            viewModel.query = it // ❌ direct mutation from UI
        }
    )
}
```

**Why it matters:** UDF makes state predictable, testable, and debuggable. You always know where state comes from and who owns it.

---

#### P2 — Hoisted State with Single ViewModel

**The pattern:** One ViewModel per screen. Composables below are stateless.

```kotlin
// ✅ PATTERN: ViewModel owns all screen state
class CryptoListViewModel(private val repo: CryptoRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Crypto>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Crypto>>> = _uiState.asStateFlow()

    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()

    val filteredCryptos: StateFlow<List<Crypto>> = combine(
        _uiState, _searchQuery
    ) { state, query ->
        if (state is UiState.Success) {
            state.data.filter { it.name.contains(query, ignoreCase = true) }
        } else emptyList()
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
}
```

```kotlin
// ❌ ANTI-PATTERN: multiple ViewModels for one screen → state sync nightmare
@Composable
fun CryptoScreen() {
    val listViewModel: ListViewModel = viewModel()
    val searchViewModel: SearchViewModel = viewModel()
    val filterViewModel: FilterViewModel = viewModel()
    // Now you need to sync state between 3 ViewModels — fragile
}
```

---

#### P3 — Composable Decomposition

**The pattern:** Break large composables into small, focused ones. Each does ONE thing.

```kotlin
// ✅ PATTERN: decomposed, each piece reusable
@Composable
fun CryptoScreen(uiState: UiState<List<Crypto>>, onRetry: () -> Unit) {
    when (uiState) {
        is UiState.Loading -> FullScreenLoader()
        is UiState.Success -> CryptoList(uiState.data)
        is UiState.Error -> ErrorScreen(uiState.message, onRetry)
    }
}

@Composable
fun CryptoList(cryptos: List<Crypto>) {
    LazyColumn {
        items(cryptos, key = { it.id }) { CryptoItem(it) }
    }
}

@Composable
fun CryptoItem(crypto: Crypto) {
    Row { /* ... */ }
}
```

```kotlin
// ❌ ANTI-PATTERN: one 300-line composable doing everything
@Composable
fun CryptoScreen(viewModel: CryptoViewModel) {
    val state by viewModel.uiState.collectAsState()
    Box {
        if (state.isLoading) CircularProgressIndicator()
        LazyColumn {
            items(state.cryptos) { crypto ->
                Row {
                    // 50 lines of inline UI
                    Column {
                        // more inline UI
                        if (crypto.change > 0) {
                            // even more inline
                        }
                    }
                }
            }
        }
        if (state.error != null) {
            // error UI inline
        }
    }
}
// Cannot test, cannot reuse, hard to reason about
```

---

#### P4 — Stable Parameters for Skippable Composables

```kotlin
// ✅ PATTERN: immutable data classes — Compose can skip recomposition
@Immutable
data class CryptoDisplayModel(
    val id: String,
    val name: String,
    val price: String,       // pre-formatted
    val changePercent: String,
    val isPositive: Boolean
)

@Composable
fun CryptoItem(crypto: CryptoDisplayModel) { // skippable ✅
    Row {
        Text(crypto.name)
        Text(crypto.price)
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: raw domain object with mutable fields
data class Crypto(
    var id: String,         // var = mutable = unstable
    var name: String,
    var price: Double,      // Double needs formatting in composable
    var change: Double
)

@Composable
fun CryptoItem(crypto: Crypto) { // NOT skippable ❌ — always recomposes
    val formatted = "$${"%.2f".format(crypto.price)}" // formatting in composable ❌
}
```

---

#### P5 — Effect Isolation

```kotlin
// ✅ PATTERN: effects are isolated, keyed correctly
@Composable
fun UserProfileScreen(userId: String, viewModel: ProfileViewModel) {
    // Restarts if userId changes — correct
    LaunchedEffect(userId) {
        viewModel.loadProfile(userId)
    }

    // One-time setup
    DisposableEffect(Unit) {
        val listener = analytics.startSession()
        onDispose { listener.endSession() }
    }
}
```

```kotlin
// ❌ ANTI-PATTERN 1: side effect in composable body
@Composable
fun UserProfileScreen(userId: String) {
    viewModel.loadProfile(userId) // runs every recomposition ❌
    analytics.logView() // called dozens of times ❌
}

// ❌ ANTI-PATTERN 2: wrong key — never restarts when userId changes
LaunchedEffect(Unit) { // Unit never changes — userId change ignored ❌
    viewModel.loadProfile(userId)
}

// ❌ ANTI-PATTERN 3: DisposableEffect without onDispose
DisposableEffect(Unit) {
    locationManager.startUpdates(listener)
    // missing onDispose — listener never removed = memory leak ❌
}
```

---

#### P6 — Correct Loading Image Pattern

```kotlin
// ✅ PATTERN: Coil with proper config
@Composable
fun CryptoLogo(url: String, modifier: Modifier = Modifier) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .crossfade(300)
            .memoryCachePolicy(CachePolicy.ENABLED)
            .diskCachePolicy(CachePolicy.ENABLED)
            .size(48, 48) // only load what you need
            .build(),
        contentDescription = null,
        modifier = modifier,
        placeholder = painterResource(R.drawable.ic_placeholder),
        error = painterResource(R.drawable.ic_error)
    )
}
```

```kotlin
// ❌ ANTI-PATTERN: loading full image, no cache, no placeholder
Image(
    bitmap = BitmapFactory.decodeUrl(url).asImageBitmap(), // blocking main thread ❌
    contentDescription = null
    // no placeholder, no error state, no caching ❌
)
```

---

### ARCHITECTURE PATTERNS

---

#### P7 — Sealed UiState (covered in Ch. 7 — extended here)

```kotlin
// ✅ PATTERN: model ALL possible states, exhaustive when
sealed class UiState<out T> {
    data object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(
        val message: String,
        val isRetryable: Boolean = true
    ) : UiState<Nothing>()
    // Add: RefreshingSuccess for pull-to-refresh
    data class RefreshingSuccess<T>(val data: T) : UiState<T>()
}

// Composable — exhaustive, no else branch needed
when (val s = uiState) {
    is UiState.Loading -> LoadingView()
    is UiState.Success -> ContentView(s.data)
    is UiState.RefreshingSuccess -> ContentView(s.data, isRefreshing = true)
    is UiState.Error -> ErrorView(s.message, showRetry = s.isRetryable)
}
```

```kotlin
// ❌ ANTI-PATTERN: nullable + boolean flags = impossible states
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Item>? = null,
    val error: String? = null
    // Can be: isLoading=true AND error≠null — what does UI show? ❌
    // Can be: data≠null AND error≠null — which wins? ❌
)
```

---

#### P8 — Repository as Single Source of Truth

```kotlin
// ✅ PATTERN: repository exposes Flow, updates propagate automatically
class PriceRepository(
    private val api: CryptoApi,
    private val cache: PriceCache
) {
    private val _prices = MutableStateFlow<List<Price>>(emptyList())
    val prices: StateFlow<List<Price>> = _prices.asStateFlow()

    suspend fun refresh() {
        val fresh = api.getPrices()
        cache.save(fresh)
        _prices.value = fresh // single update point
    }
}

// Any ViewModel collecting prices: always gets latest, always consistent
```

```kotlin
// ❌ ANTI-PATTERN: ViewModel fetches directly, multiple sources of truth
class ViewModel1 : ViewModel() {
    fun load() { viewModelScope.launch { api.getPrices() } } // own copy
}
class ViewModel2 : ViewModel() {
    fun load() { viewModelScope.launch { api.getPrices() } } // different copy
    // Now VM1 and VM2 can show different prices ❌
}
```

---

#### P9 — DTO to Domain Model Mapping

```kotlin
// ✅ PATTERN: map at repository boundary, domain model is clean
// DTO (matches API shape)
data class CryptoDto(
    @SerializedName("id") val id: String,
    @SerializedName("symbol") val symbol: String,
    @SerializedName("current_price") val currentPrice: Double,
    @SerializedName("price_change_percentage_24h") val change24h: Double
)

// Domain model (matches app needs)
data class Crypto(
    val id: String,
    val symbol: String,
    val price: Double,
    val change24h: Double
)

// Mapper — lives in Repository or DataSource
fun CryptoDto.toDomain() = Crypto(
    id = id,
    symbol = symbol.uppercase(),
    price = currentPrice,
    change24h = change24h
)
```

```kotlin
// ❌ ANTI-PATTERN: API model used directly in UI
@Composable
fun CryptoItem(crypto: CryptoDto) { // DTO in UI ❌
    Text(crypto.current_price.toString()) // raw API field name in UI ❌
    // If API changes field name — UI breaks directly
}
```

---

#### P10 — Interface-Driven Dependencies

```kotlin
// ✅ PATTERN: depend on interfaces, inject implementations
interface CryptoRepository {
    suspend fun getCryptos(): List<Crypto>
    val prices: StateFlow<List<Crypto>>
}

class CryptoRepositoryImpl(
    private val api: CryptoApi,
    private val cache: CryptoCache
) : CryptoRepository { ... }

// Test:
class FakeCryptoRepository : CryptoRepository {
    var data: List<Crypto> = emptyList()
    override suspend fun getCryptos() = data
    override val prices = MutableStateFlow(emptyList<Crypto>())
}

class CryptoViewModel(private val repository: CryptoRepository) : ViewModel()
// Can test with FakeCryptoRepository without network ✅
```

```kotlin
// ❌ ANTI-PATTERN: concrete dependency — untestable
class CryptoViewModel(
    private val repository: CryptoRepositoryImpl // concrete ❌
) : ViewModel()
// Cannot inject fake in tests without network
```

---

### COROUTINE PATTERNS

---

#### P11 — Structured Concurrency

```kotlin
// ✅ PATTERN: use structured concurrency — all coroutines have a parent scope
class MyViewModel : ViewModel() {
    fun loadDashboard() {
        viewModelScope.launch { // parent scope
            val prices = async { repository.getPrices() }
            val news = async { repository.getNews() }
            val user = async { repository.getUser() }
            // All run in parallel, all cancelled if parent cancelled
            _uiState.value = UiState.Success(
                Dashboard(prices.await(), news.await(), user.await())
            )
        }
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: GlobalScope — escapes structured concurrency
fun loadData() {
    GlobalScope.launch { // never cancelled, leaks ❌
        repository.getData()
    }
}

// ❌ ANTI-PATTERN: unscoped CoroutineScope
val myScope = CoroutineScope(Dispatchers.IO) // who owns this? when is it cancelled? ❌
```

---

#### P12 — Dispatcher Correctness

```kotlin
// ✅ PATTERN: right dispatcher for right work
class MyViewModel : ViewModel() {
    fun processData(rawData: String) {
        viewModelScope.launch {
            val parsed = withContext(Dispatchers.Default) { // CPU work
                parseHeavyJson(rawData)
            }
            val saved = withContext(Dispatchers.IO) { // disk/network
                repository.save(parsed)
            }
            _uiState.value = UiState.Success(saved) // back on Main
        }
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: network on Main thread
viewModelScope.launch(Dispatchers.Main) {
    val data = api.fetchData() // NetworkOnMainThreadException ❌
}

// ❌ ANTI-PATTERN: UI update on IO thread
viewModelScope.launch(Dispatchers.IO) {
    val data = api.fetchData()
    _uiState.value = data // CalledFromWrongThreadException ❌
}
```

---

#### P13 — Cancellation-Safe Operations

```kotlin
// ✅ PATTERN: check cancellation in long-running loops
suspend fun processItems(items: List<Item>) {
    items.forEach { item ->
        ensureActive() // throws CancellationException if cancelled ✅
        processItem(item)
    }
}

// ✅ PATTERN: use withTimeout for time-bounded operations
val result = withTimeout(5000L) {
    api.fetchWithTimeout()
} // throws TimeoutCancellationException after 5s
```

```kotlin
// ❌ ANTI-PATTERN: catching CancellationException (breaks cancellation)
try {
    longRunningWork()
} catch (e: Exception) { // catches CancellationException too ❌
    // coroutine thinks it can continue but parent cancelled it
}

// ✅ FIX: only catch non-cancellation exceptions
try {
    longRunningWork()
} catch (e: CancellationException) {
    throw e // re-throw — let cancellation propagate ✅
} catch (e: IOException) {
    handleError(e)
}
```

---

#### P14 — StateFlow vs LiveData

```kotlin
// ✅ PATTERN: StateFlow in Kotlin-first codebases
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState.Loading)
    val state: StateFlow<UiState> = _state.asStateFlow()
    // Works with Compose natively
    // Kotlin Flow operators (map, filter, combine) work directly
    // No observer lifecycle boilerplate
}
```

```kotlin
// ❌ ANTI-PATTERN: LiveData in Compose — extra conversion layer
class MyViewModel : ViewModel() {
    val state = MutableLiveData<UiState>() // ❌ in Compose-first apps
}

// In Composable:
val state by viewModel.state.observeAsState() // extra conversion ❌
// LiveData is fine in XML-based apps, but in Compose use StateFlow
```

---

### KOTLIN PATTERNS

---

#### P15 — Scope Functions Used Correctly

```kotlin
// let — transform nullable or chain operations
val name = user?.let { "${it.firstName} ${it.lastName}" } ?: "Unknown"

// apply — configure object being constructed
val request = OkHttpRequest.Builder()
    .url(url)
    .apply {
        if (needsAuth) addHeader("Authorization", "Bearer $token")
    }
    .build()

// run — operate on object AND return result
val result = database.run {
    beginTransaction()
    val r = query("SELECT * FROM prices")
    endTransaction()
    r
}

// with — multiple operations on same object, no return
with(binding) {
    titleText.text = title
    subtitleText.text = subtitle
}

// also — side effects without changing receiver
val prices = repository.getPrices()
    .also { analytics.log("Loaded ${it.size} prices") }
```

```kotlin
// ❌ ANTI-PATTERN: wrong scope function for context
val name = user.run { // run returns result — use let for nullable
    "${firstName} ${lastName}" // crashes if user is null
}

// ❌ ANTI-PATTERN: deeply nested scope functions — unreadable
val result = list.let { it.filter { item -> item.isValid }.also { filtered ->
    filtered.forEach { it.also { item -> process(item) } }
} }
```

---

#### P16 — Extension Functions for Clean Code

```kotlin
// ✅ PATTERN: extension functions for reusable transforms
fun Double.toPriceString(): String = "$${"%.2f".format(this)}"
fun Double.toChangeString(): String = "${"%.1f".format(this)}%"
fun String.toSymbolDisplay(): String = this.uppercase()

// Usage in composable — clean, no formatting logic in UI
Text(crypto.price.toPriceString())
Text(crypto.change24h.toChangeString())
```

```kotlin
// ❌ ANTI-PATTERN: formatting logic scattered in composables
@Composable
fun CryptoItem(crypto: Crypto) {
    Text("$${"%.2f".format(crypto.price)}")     // formatting in UI ❌
    Text("${"%.1f".format(crypto.change)}%")     // duplicated everywhere ❌
}
```

---

#### P17 — Sealed Class for Events

```kotlin
// ✅ PATTERN: sealed class for one-time UI events
sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
    data object NavigateBack : UiEvent()
    data class ShowDialog(val title: String, val message: String) : UiEvent()
}

class MyViewModel : ViewModel() {
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun onSave() {
        viewModelScope.launch {
            try {
                repository.save()
                _events.emit(UiEvent.ShowSnackbar("Saved!"))
                _events.emit(UiEvent.NavigateBack)
            } catch (e: Exception) {
                _events.emit(UiEvent.ShowSnackbar("Error: ${e.message}"))
            }
        }
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: String-based events — no type safety, error-prone
private val _event = MutableStateFlow<String?>("")
// "NAVIGATE_HOME", "SHOW_ERROR", "SAVE_SUCCESS" — magic strings ❌
// StateFlow replays last value — navigation fires again on resubscribe ❌
```

---

### NETWORK PATTERNS

---

#### P18 — Result Wrapper for Network Calls

```kotlin
// ✅ PATTERN: wrap network results in sealed Result
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int?, val message: String) : NetworkResult<Nothing>()
    data object NetworkError : NetworkResult<Nothing>() // no internet
}

suspend fun <T> safeApiCall(call: suspend () -> T): NetworkResult<T> {
    return try {
        NetworkResult.Success(call())
    } catch (e: HttpException) {
        NetworkResult.Error(e.code(), e.message())
    } catch (e: IOException) {
        NetworkResult.NetworkError
    }
}

// Repository usage:
suspend fun getCryptos(): NetworkResult<List<Crypto>> =
    safeApiCall { api.getCryptos().map { it.toDomain() } }
```

```kotlin
// ❌ ANTI-PATTERN: raw exceptions thrown from repository to ViewModel to UI
suspend fun getCryptos(): List<Crypto> = api.getCryptos() // throws HttpException ❌
// ViewModel must know about HttpException — leaks network layer details
```

---

#### P19 — OkHttp Interceptors for Cross-Cutting Concerns

```kotlin
// ✅ PATTERN: interceptors for auth, logging, retry — not scattered in each call
val client = OkHttpClient.Builder()
    .addInterceptor(AuthInterceptor(tokenProvider))   // adds auth header
    .addInterceptor(LoggingInterceptor())              // logs in debug
    .addInterceptor(RetryInterceptor(maxRetries = 3)) // auto-retry on 503
    .build()

class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer ${tokenProvider.getToken()}")
            .build()
        return chain.proceed(request)
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: auth header added in every API call
interface CryptoApi {
    @GET("prices")
    suspend fun getPrices(@Header("Authorization") auth: String): List<PriceDto> // ❌
    @GET("news")
    suspend fun getNews(@Header("Authorization") auth: String): List<NewsDto> // ❌
    // Duplicated in every call — forget one and it breaks
}
```

---

### UI/UX PATTERNS

---

#### P20 — Always Handle All States

```kotlin
// ✅ PATTERN: all 4 states always handled
@Composable
fun CryptoScreen(uiState: UiState<List<Crypto>>, onRetry: () -> Unit) {
    when (uiState) {
        is UiState.Loading -> FullScreenLoader()
        is UiState.Success -> if (uiState.data.isEmpty()) {
            EmptyState("No cryptos found") // ✅ empty state
        } else {
            CryptoList(uiState.data)
        }
        is UiState.Error -> ErrorState(
            message = uiState.message,
            onRetry = onRetry // ✅ always give user a way out
        )
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: ignoring states
@Composable
fun CryptoScreen(cryptos: List<Crypto>?) {
    if (cryptos != null) {
        LazyColumn { items(cryptos) { CryptoItem(it) } }
    }
    // no loading state shown ❌
    // no error state shown ❌
    // empty list shows nothing ❌
}
```

---

#### P21 — Pull-to-Refresh Pattern

```kotlin
// ✅ PATTERN: show existing data + refresh indicator
@Composable
fun CryptoScreen(viewModel: CryptoViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    SwipeRefresh(
        state = rememberSwipeRefreshState(isRefreshing),
        onRefresh = viewModel::refresh
    ) {
        when (uiState) {
            is UiState.Success -> CryptoList(uiState.data)
            is UiState.Error -> ErrorState(uiState.message)
            is UiState.Loading -> FullScreenLoader()
        }
    }
}

// ViewModel:
fun refresh() {
    viewModelScope.launch {
        _isRefreshing.value = true
        try { repository.refresh() }
        finally { _isRefreshing.value = false } // always reset ✅
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: replace entire screen with loader on refresh
fun refresh() {
    viewModelScope.launch {
        _uiState.value = UiState.Loading // user loses their scroll position ❌
        _uiState.value = UiState.Success(repository.getCryptos())
    }
}
```

---

#### P22 — Accessibility

```kotlin
// ✅ PATTERN: always provide contentDescription for images, icons
Icon(
    imageVector = Icons.Default.TrendingUp,
    contentDescription = "Price trending up" // ✅
)

AsyncImage(
    model = logoUrl,
    contentDescription = "${crypto.name} logo" // ✅
)

// ✅ Use semantics for custom components
Box(
    modifier = Modifier.semantics {
        contentDescription = "Crypto price card for ${crypto.name}"
        role = Role.Button
    }
)
```

```kotlin
// ❌ ANTI-PATTERN: null contentDescription on informational images
Icon(
    imageVector = Icons.Default.TrendingUp,
    contentDescription = null // ❌ screen reader skips it entirely
)
// Only use null for purely decorative images that add no information
```

---

### PERFORMANCE PATTERNS

---

#### P23 — Pagination Pattern

```kotlin
// ✅ PATTERN: load more on scroll, not all at once
@Composable
fun CryptoList(
    cryptos: List<Crypto>,
    isLoadingMore: Boolean,
    onLoadMore: () -> Unit
) {
    val listState = rememberLazyListState()

    // Detect when user is near bottom
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            val totalItems = listState.layoutInfo.totalItemsCount
            lastVisible >= totalItems - 3 // load when 3 items from end
        }
    }

    LaunchedEffect(shouldLoadMore) {
        if (shouldLoadMore) onLoadMore()
    }

    LazyColumn(state = listState) {
        items(cryptos, key = { it.id }) { CryptoItem(it) }
        if (isLoadingMore) {
            item { LoadingIndicator() }
        }
    }
}
```

```kotlin
// ❌ ANTI-PATTERN: load all data upfront
fun loadCryptos() {
    viewModelScope.launch {
        val ALL_CRYPTOS = api.getCryptos(perPage = 5000) // 5000 items in memory ❌
        _uiState.value = UiState.Success(ALL_CRYPTOS)
    }
}
```

---

#### P24 — Debounce for Search

```kotlin
// ✅ PATTERN: debounce to avoid API call on every keystroke
class SearchViewModel(private val repo: CryptoRepository) : ViewModel() {
    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    val results: StateFlow<List<Crypto>> = _query
        .debounce(300L)           // wait 300ms after user stops typing
        .distinctUntilChanged()    // don't search same query twice
        .flatMapLatest { query ->  // cancel previous search on new query
            if (query.isBlank()) flowOf(emptyList())
            else repo.search(query)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onQueryChange(query: String) { _query.value = query }
}
```

```kotlin
// ❌ ANTI-PATTERN: API call on every character
fun onQueryChange(query: String) {
    viewModelScope.launch {
        val results = api.search(query) // called on EVERY keystroke ❌
        // "bitcoin" = 7 API calls for b, bi, bit, bitc, bitco, bitcoi, bitcoin
    }
}
```

---

#### P25 — stateIn for Shared Flows

```kotlin
// ✅ PATTERN: stateIn converts cold Flow to StateFlow with lifecycle awareness
class CryptoViewModel(private val repo: CryptoRepository) : ViewModel() {
    val prices: StateFlow<List<Crypto>> = repo.getPricesFlow()
        .map { dtos -> dtos.map { it.toDomain() } }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000), // stops 5s after last subscriber
            initialValue = emptyList()
        )
    // Auto-stops collection when no UI is watching (battery-efficient)
    // Auto-restarts when UI resubscribes
}
```

```kotlin
// ❌ ANTI-PATTERN: cold flow collected in composable directly
@Composable
fun CryptoScreen() {
    val prices by repo.getPricesFlow() // cold flow ❌
        .collectAsState(emptyList())
    // New collection started on every recomposition
    // No sharing between collectors
    // No lifecycle awareness
}
```

---

### ANTI-PATTERNS SUMMARY TABLE

| Anti-Pattern | Why it's bad | Fix |
|---|---|---|
| Business logic in Composable | Untestable, reruns on recompose | Move to ViewModel |
| `GlobalScope` | Never cancelled, memory leak | `viewModelScope` / `lifecycleScope` |
| `collectAsState()` | Collects in background, wastes battery | `collectAsStateWithLifecycle()` |
| Nullable UI state (`data: T?`) | Impossible states, null checks everywhere | Sealed UiState class |
| Mutable `var` in data classes | Unstable → always recomposes | `val` only, `@Immutable` |
| Direct API calls in ViewModel | No abstraction, untestable | Repository pattern |
| `LiveData` in Compose | Extra conversion, verbose | `StateFlow` |
| Network on main thread | `NetworkOnMainThreadException` | `withContext(Dispatchers.IO)` |
| No `key` in `LazyColumn` | All items recompose on list change | `items(key = { it.id })` |
| Side effects in composable body | Runs on every recomposition | `LaunchedEffect` / `DisposableEffect` |
| `DisposableEffect` without `onDispose` | Memory/resource leak | Always add `onDispose { }` |
| Catching `CancellationException` | Breaks structured concurrency | Re-throw it |
| Loading full image bitmap manually | OOM on large images | Coil with size sampling |
| Concrete dependencies in ViewModel | Untestable | Depend on interfaces |
| Magic string routes | Typos cause crashes at runtime | Sealed Screen class |
| DTO used in UI layer | Couples UI to API contract | Map DTO → Domain model |
| `var` in ViewModel exposed directly | External mutation, breaks UDF | Private `MutableStateFlow`, public `StateFlow` |
| API call on every search keystroke | Rate limits, performance | `debounce(300)` + `flatMapLatest` |
| Load all data at once | Memory pressure, slow initial load | Pagination |
| Cold Flow in ViewModel without `stateIn` | No sharing, no lifecycle | `.stateIn(viewModelScope, WhileSubscribed, initial)` |
| No error state | User sees blank screen | Always handle Loading/Success/Error/Empty |


---

## 23. Architecture Trade-offs — MVVM vs MVI vs MVP vs Clean Architecture

### Why this matters in the interview
Ojaswirajanya will ask "why did you choose this architecture?" You must defend trade-offs, not just recite definitions.

---

### MVVM (Model-View-ViewModel)

```
View (Composable) ←→ ViewModel ←→ Model (Repository)
```

**How it works:**
- ViewModel exposes StateFlow/LiveData
- View observes and renders
- View sends events to ViewModel via function calls

```kotlin
// MVVM in Compose
class CryptoViewModel(private val repo: CryptoRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Crypto>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Crypto>>> = _uiState.asStateFlow()

    fun onRefresh() { /* update _uiState */ }
    fun onSearch(query: String) { /* update _uiState */ }
}

@Composable
fun CryptoScreen(viewModel: CryptoViewModel = viewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    when (state) {
        is UiState.Loading -> LoadingView()
        is UiState.Success -> ContentView(state.data, onRefresh = viewModel::onRefresh)
        is UiState.Error -> ErrorView(onRetry = viewModel::onRefresh)
    }
}
```

**✅ When to use MVVM:**
- Most Android apps
- Teams familiar with Android Jetpack
- Apps where business logic is straightforward
- When you want minimal boilerplate

**❌ Trade-offs:**
- ViewModel can grow large ("fat ViewModel") if not disciplined
- Multiple ViewModels can get out of sync
- No strict contract for what events look like

---

### MVI (Model-View-Intent)

```
View → Intent → ViewModel/Reducer → State → View (circular)
```

**How it works:**
- View emits **Intents** (user actions — sealed class)
- Reducer/ViewModel processes Intent → new State
- State is a single immutable snapshot of entire screen
- One-directional, no back-and-forth

```kotlin
// MVI
sealed class CryptoIntent {
    object LoadCryptos : CryptoIntent()
    data class SearchCryptos(val query: String) : CryptoIntent()
    data class FavoriteToggled(val id: String) : CryptoIntent()
}

data class CryptoState(
    val isLoading: Boolean = false,
    val cryptos: List<Crypto> = emptyList(),
    val searchQuery: String = "",
    val error: String? = null,
    val favorites: Set<String> = emptySet()
) // single immutable state object — entire screen in one place

class CryptoMviViewModel(private val repo: CryptoRepository) : ViewModel() {
    private val _state = MutableStateFlow(CryptoState())
    val state: StateFlow<CryptoState> = _state.asStateFlow()

    fun processIntent(intent: CryptoIntent) {
        when (intent) {
            is CryptoIntent.LoadCryptos -> loadCryptos()
            is CryptoIntent.SearchCryptos -> search(intent.query)
            is CryptoIntent.FavoriteToggled -> toggleFavorite(intent.id)
        }
    }

    private fun loadCryptos() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            try {
                val data = repo.getCryptos()
                _state.update { it.copy(isLoading = false, cryptos = data) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
}
```

**✅ When to use MVI:**
- Complex screens with many interdependent states
- When you need perfect reproducibility (same intent = same state)
- When debugging/logging state transitions is critical
- Time-travel debugging (undo/redo)
- Large teams needing strict contracts

**❌ Trade-offs:**
- More boilerplate — every action needs an Intent
- State object can grow large
- Overkill for simple screens
- Learning curve for team

---

### MVP (Model-View-Presenter)

```
View ←→ Presenter ←→ Model
```

**Legacy pattern — XML era. Compose prefers MVVM/MVI.**

```kotlin
// MVP — rarely used in new Compose code
interface CryptoView {
    fun showLoading()
    fun showCryptos(cryptos: List<Crypto>)
    fun showError(message: String)
}

class CryptoPresenter(private val repo: CryptoRepository) {
    var view: CryptoView? = null
    fun loadCryptos() {
        view?.showLoading()
        // fetch and update view
    }
    fun onDestroy() { view = null } // manual lifecycle management
}
```

**❌ Why not in Compose:**
- Presenter holds View reference → memory leaks if not careful
- Manual lifecycle management
- Compose's reactive model makes it unnecessary
- No lifecycle awareness built in

---

### Clean Architecture

```
UI Layer → Domain Layer → Data Layer
         (Use Cases)    (Repository + DataSource)
```

**Layers:**
```
┌─────────────────────────────┐
│  UI Layer                   │  Composables, ViewModels
│  (Presentation)             │
├─────────────────────────────┤
│  Domain Layer               │  Use Cases, Domain Models, Interfaces
│  (Business Logic)           │
├─────────────────────────────┤
│  Data Layer                 │  Repository Impl, DataSources, DTOs
│  (Data)                     │  Network, Database, Cache
└─────────────────────────────┘
```

```kotlin
// Domain layer — pure Kotlin, no Android imports
data class Crypto(val id: String, val name: String, val price: Double)

interface CryptoRepository { // interface lives in Domain
    suspend fun getCryptos(): List<Crypto>
}

class GetCryptosUseCase(private val repository: CryptoRepository) {
    suspend operator fun invoke(): List<Crypto> {
        return repository.getCryptos()
            .filter { it.price > 0 } // business rule here, not in ViewModel
            .sortedByDescending { it.price }
    }
}

// ViewModel uses UseCase, not Repository directly
class CryptoViewModel(
    private val getCryptos: GetCryptosUseCase
) : ViewModel() {
    fun load() {
        viewModelScope.launch {
            val cryptos = getCryptos() // clean call
        }
    }
}
```

**✅ When to use Clean Architecture:**
- Large apps with complex business logic
- Multiple teams working in parallel
- When business rules need to be tested independently of Android
- When app might target multiple platforms (KMP)

**❌ Trade-offs:**
- Significant boilerplate (UseCases for everything)
- Over-engineering for small apps
- More files, more indirection
- Slower initial development velocity

---

### Trade-off comparison table

| | MVVM | MVI | MVP | Clean Arch |
|---|---|---|---|---|
| Boilerplate | Low | Medium | Low | High |
| Testability | Good | Excellent | Good | Excellent |
| State predictability | Good | Excellent | Medium | Good |
| Compose fit | ✅ Native | ✅ Great | ❌ Poor | ✅ With MVVM/MVI |
| Best for | Most apps | Complex screens | Legacy XML | Large teams |
| Learning curve | Low | Medium | Low | High |

### What to say in interview

*"For this app I'm using MVVM — it's the right fit for a 75-minute scope. The ViewModel exposes a single StateFlow of sealed UiState, composables are stateless, and the repository is the single source of truth. If this were a larger app with complex interdependent state, I'd layer MVI on top — single immutable state object, intent-based event handling, easier to debug and log state transitions. Clean Architecture UseCases would be the next step if business logic needed to be shared across platforms or tested in isolation from Android."*

---

## 24. Android Threading Model — Foreground, Background, Restrictions

### The main thread (UI thread)

The main thread runs:
- All UI rendering and updates
- Input event handling (touch, click)
- Activity/Fragment lifecycle callbacks
- Composable functions

**Golden rule: Never block the main thread.** >16ms per frame = dropped frames. >5s = ANR (Application Not Responding) dialog.

```kotlin
// ❌ ANTI-PATTERN: blocking main thread
fun loadData() {
    val data = api.fetchData()  // network on main → NetworkOnMainThreadException
    val json = File("data.json").readText() // disk on main → ANR risk
    val result = heavyComputation(data)     // CPU on main → jank
}

// ✅ PATTERN: offload to correct dispatcher
fun loadData() {
    viewModelScope.launch {
        val data = withContext(Dispatchers.IO) { api.fetchData() }
        val result = withContext(Dispatchers.Default) { heavyComputation(data) }
        _uiState.value = UiState.Success(result) // back on Main
    }
}
```

### Thread pool internals

| Dispatcher | Thread pool | Size | Use for |
|---|---|---|---|
| `Dispatchers.Main` | Main thread | 1 | UI updates, state |
| `Dispatchers.IO` | Shared IO pool | up to 64 | Network, disk, DB |
| `Dispatchers.Default` | CPU pool | CPU core count | Sorting, parsing, math |
| `Dispatchers.Unconfined` | Caller thread | — | Rarely — testing only |

```kotlin
// IO and Default share threads intelligently
// IO can expand to 64 — designed for blocking calls that just wait
// Default is bounded to CPU cores — designed for CPU work

// ✅ Correct: IO for Retrofit (even though Retrofit is suspend — good practice to be explicit)
val data = withContext(Dispatchers.IO) { api.getCryptos() }

// ✅ Correct: Default for heavy CPU work
val sorted = withContext(Dispatchers.Default) { list.sortedBy { it.price } }
```

### Looper and Handler (internals Ojaswirajanya will know from Snap)

Every thread can have a `Looper` — a message queue processor.
- Main thread has a `Looper` by default
- Background threads don't unless you create one
- `Handler` posts `Runnable`s or `Message`s to a `Looper`'s queue

```kotlin
// How main thread works internally:
// 1. Looper.loop() runs forever on main thread
// 2. Picks messages from MessageQueue one by one
// 3. Each message = one task (draw frame, handle click, run coroutine resume)
// 4. If one task takes >16ms → next frame dropped

// Modern equivalent: don't use Handler directly
// Use coroutines with Dispatchers.Main instead

// ❌ OLD WAY:
val handler = Handler(Looper.getMainLooper())
handler.post { textView.text = "Updated" }

// ✅ NEW WAY:
viewModelScope.launch { _state.value = "Updated" }
```

---

## 25. Foreground/Background Restrictions & Android Compliance

### Why this matters at TFH
World App does real-time identity verification, wallet sync, notifications. All of these hit Android's background restrictions hard.

---

### Background execution limits (Android 8+)

**Since Android 8.0 (API 26), background apps cannot:**
- Start background services freely
- Run implicit broadcasts
- Access location frequently

```kotlin
// ❌ FAILS on Android 8+:
startService(Intent(this, SyncService::class.java)) // IllegalStateException if app in background

// ✅ FIX: use WorkManager for deferrable background work
val request = OneTimeWorkRequestBuilder<SyncWorker>().build()
WorkManager.getInstance(context).enqueue(request)

// ✅ FIX: use ForegroundService if work must run immediately and continuously
startForegroundService(Intent(this, SyncService::class.java))
// Must call startForeground() within 5 seconds or ANR
```

### Foreground Services

Foreground services show a persistent notification — this is the contract: user can always see something is running.

```kotlin
class WalletSyncService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = buildNotification()

        // MUST call within 5 seconds of service start
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            startForeground(
                NOTIFICATION_ID,
                notification,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC // declare type
            )
        } else {
            startForeground(NOTIFICATION_ID, notification)
        }
        return START_STICKY // restart if killed
    }
}

// AndroidManifest.xml — required
// <service android:name=".WalletSyncService"
//          android:foregroundServiceType="dataSync" />
// <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
// <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" /> // API 34+
```

### Foreground service types (Android 10+, mandatory Android 14+)

| Type | Use case | Example |
|---|---|---|
| `dataSync` | Syncing data | Wallet sync, file upload |
| `location` | GPS tracking | Navigation, delivery tracking |
| `mediaPlayback` | Playing audio/video | Music player |
| `camera` | Camera access | Video recording |
| `microphone` | Mic access | Voice recording |
| `connectedDevice` | Bluetooth/USB | Orb connection |
| `health` | Fitness/health | Heart rate monitoring |
| `remoteMessaging` | Messaging | Chat sync |

```kotlin
// ❌ FAILS on Android 14+ without declared type:
startForeground(id, notification) // crashes with MissingForegroundServiceTypeException

// ✅ MUST declare type in both code AND manifest
startForeground(id, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC)
```

### WorkManager — the right tool for background work

```kotlin
// ✅ For any deferrable background work:
class PriceSyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            val prices = api.getPrices()
            localDb.save(prices)
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() // auto-retry
            else Result.failure()
        }
    }
}

// Schedule with constraints
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true)
    .build()

val syncWork = PeriodicWorkRequestBuilder<PriceSyncWorker>(
    repeatInterval = 15, TimeUnit.MINUTES // minimum interval
)
    .setConstraints(constraints)
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 1, TimeUnit.MINUTES)
    .build()

WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork("price_sync", ExistingPeriodicWorkPolicy.KEEP, syncWork)
```

### Doze mode and App Standby

Android aggressively restricts background apps to save battery.

**Doze mode** (device stationary + screen off + unplugged):
- Network access blocked
- WakeLocks ignored
- JobScheduler/WorkManager deferred
- Alarm intervals expanded
- Only **high-priority FCM** breaks through

**App Standby** (app not recently used):
- Network access restricted
- Background work deferred

```kotlin
// ✅ Check if app is in battery optimization exemption
val pm = getSystemService(PowerManager::class.java)
val isIgnoring = pm.isIgnoringBatteryOptimizations(packageName)

// ✅ Use FCM high-priority push to wake app from Doze
// (server sends data message with priority: "high")
// FCM delivery = guaranteed even in Doze — only reliable wake mechanism

// ❌ DON'T rely on:
// alarmManager.setExact() — deferred in Doze
// setRepeating() — not exact in any mode
// Handler.postDelayed() — killed when process dies
```

### Notifications — channels and permissions

```kotlin
// Android 8+ requires notification channels
fun createNotificationChannels(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val channel = NotificationChannel(
            "price_alerts",
            "Price Alerts",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "Alerts when crypto prices change significantly"
            enableVibration(true)
        }
        val manager = context.getSystemService(NotificationManager::class.java)
        manager.createNotificationChannel(channel)
    }
}

// Android 13+ requires runtime notification permission
// <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

@Composable
fun RequestNotificationPermission() {
    val permissionState = rememberPermissionState(
        Manifest.permission.POST_NOTIFICATIONS
    )
    LaunchedEffect(Unit) {
        if (!permissionState.status.isGranted) {
            permissionState.launchPermissionRequest()
        }
    }
}

// ✅ ALWAYS check permission before showing notification (Android 13+)
if (ActivityCompat.checkSelfPermission(
        context, Manifest.permission.POST_NOTIFICATIONS
    ) == PackageManager.PERMISSION_GRANTED) {
    notificationManager.notify(id, notification)
}
```

### Exact alarms (Android 12+)

```kotlin
// Android 12+ restricts exact alarms — must declare permission
// <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"/>
// OR
// <uses-permission android:name="android.permission.USE_EXACT_ALARM"/> // API 33+ auto-granted for specific use cases

val alarmManager = getSystemService(AlarmManager::class.java)

// ✅ Check permission before scheduling exact alarm
if (alarmManager.canScheduleExactAlarms()) {
    alarmManager.setExactAndAllowWhileIdle(
        AlarmManager.RTC_WAKEUP,
        triggerTime,
        pendingIntent
    )
} else {
    // Request permission or use inexact alarm
    alarmManager.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerTime, pendingIntent)
}
```

---

## 26. Android Lifecycle Deep Dive

### Activity lifecycle

```
onCreate() → onStart() → onResume() ← App running, visible, interactive
                                  ↓
                              onPause() ← Partially obscured, another Activity on top
                                  ↓
                              onStop() ← Fully hidden (home pressed, another app)
                                  ↓
                             onDestroy() ← Finishing OR system killing for memory
```

**What happens at each stage:**

| Callback | Trigger | Do here | Don't do here |
|---|---|---|---|
| `onCreate` | First create | Initialize ViewModel, set content | Heavy work |
| `onStart` | Becomes visible | Register lightweight listeners | Network calls |
| `onResume` | Interactive | Resume animations, camera | Heavy init |
| `onPause` | Losing focus | Save draft, pause animations | Long operations (blocks transition) |
| `onStop` | Fully hidden | Save persistent data, release camera | Anything that delays it |
| `onDestroy` | Finishing | Cleanup, cancel non-scoped work | Assume it's always called (it's not) |

```kotlin
// ❌ COMMON MISTAKE: saving data in onDestroy
override fun onDestroy() {
    super.onDestroy()
    saveUserData() // might never be called if system kills process ❌
}

// ✅ CORRECT: save in onPause or onStop
override fun onStop() {
    super.onStop()
    saveUserData() // always called before process can be killed ✅
}
```

### What survives what

```
Configuration change (rotation):
  Activity → DESTROYED + RECREATED
  ViewModel → SURVIVES ✅
  remember { } → LOST ❌
  rememberSaveable { } → SURVIVES ✅
  Bundle (savedInstanceState) → SURVIVES ✅

Process death (system kills app for memory):
  Everything in memory → LOST ❌
  ViewModel → LOST ❌
  rememberSaveable → SURVIVES ✅ (saved to Bundle before kill)
  DataStore/SharedPrefs/DB → SURVIVES ✅ (persisted to disk)
```

### Compose lifecycle vs Activity lifecycle

Composables have their own lifecycle: **Enter Composition → Recompose (many times) → Leave Composition**

```kotlin
// Composable lifecycle hooks:
@Composable
fun MyScreen() {
    DisposableEffect(Unit) {
        // Called when composable ENTERS composition
        setupSomething()
        onDispose {
            // Called when composable LEAVES composition
            cleanupSomething()
        }
    }
}
// This is NOT tied to Activity lifecycle
// A composable can enter/leave composition many times while Activity is alive
// e.g., navigation pushes new screen → old screen's composables leave composition
```

### Lifecycle-aware collection — the right pattern

```kotlin
// ❌ collectAsState — keeps collecting even in background
val state by viewModel.state.collectAsState()

// ✅ collectAsStateWithLifecycle — stops at STARTED (app backgrounded)
val state by viewModel.state.collectAsStateWithLifecycle()

// ✅ For non-Compose code: repeatOnLifecycle
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { render(it) }
        // Automatically cancels at STOPPED, restarts at STARTED
    }
}

// ❌ WRONG: launchWhenStarted (deprecated — doesn't cancel, just suspends)
lifecycleScope.launchWhenStarted {
    viewModel.state.collect { render(it) } // coroutine stays alive in background ❌
}
```

### Fragment lifecycle quirk — double onCreate

Fragment has TWO lifecycles:
- **Fragment lifecycle**: `onCreate` → `onDestroy`
- **View lifecycle** (`viewLifecycleOwner`): `onCreateView` → `onDestroyView`

```kotlin
// ❌ WRONG: observe in Fragment lifecycle — can receive updates when view is null
override fun onCreate() {
    viewModel.state.observe(this) { render(it) } // 'this' = fragment lifecycle ❌
}

// ✅ CORRECT: observe in view lifecycle — tied to when view exists
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    viewModel.state.observe(viewLifecycleOwner) { render(it) } ✅
}
```

---

## 27. Security — Crypto, Token Storage, Network Security

### Why this matters at TFH
World App handles private keys, biometric data, real money. Security is not optional.

---

### Never store secrets in code

```kotlin
// ❌ NEVER: hardcoded API keys, secrets, private keys
const val API_KEY = "sk-abc123secret" // visible in APK decompile ❌
const val PRIVATE_KEY = "0x4c0883a69102937d..." // catastrophic ❌

// ✅ API keys: use BuildConfig (still in APK but harder to find)
// Better: server-side proxy — client never sees the key
val apiKey = BuildConfig.API_KEY // set in local.properties, not committed to git

// ✅ For truly sensitive secrets: never ship in APK
// Route through your own backend — client gets short-lived tokens
```

### Secure token storage

```kotlin
// ❌ NEVER: plain SharedPreferences for tokens
sharedPrefs.edit().putString("auth_token", token).apply() // unencrypted on disk ❌

// ✅ EncryptedSharedPreferences (Jetpack Security)
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val encryptedPrefs = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)
encryptedPrefs.edit().putString("auth_token", token).apply() // AES-256 encrypted ✅

// ✅ Android Keystore for cryptographic keys (hardware-backed on supported devices)
val keyGenerator = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_EC, "AndroidKeyStore"
)
keyGenerator.initialize(
    KeyGenParameterSpec.Builder("world_id_key",
        KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY)
        .setDigests(KeyProperties.DIGEST_SHA256)
        .setUserAuthenticationRequired(true) // requires biometric/PIN ✅
        .build()
)
```

### Certificate pinning

Prevents man-in-the-middle attacks by verifying the server's certificate matches expected value.

```kotlin
// ✅ OkHttp certificate pinning
val client = OkHttpClient.Builder()
    .certificatePinner(
        CertificatePinner.Builder()
            .add(
                "api.worldcoin.org",
                "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=" // cert hash
            )
            .build()
    )
    .build()

// ❌ Never disable SSL verification (common mistake in debugging)
val client = OkHttpClient.Builder()
    .hostnameVerifier { _, _ -> true } // disables verification ❌ NEVER ship this
    .build()
```

### Biometric authentication

```kotlin
// ✅ BiometricPrompt — supports fingerprint, face, device credential
val biometricPrompt = BiometricPrompt(
    activity,
    executor,
    object : BiometricPrompt.AuthenticationCallback() {
        override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
            // User authenticated — proceed with sensitive operation
            val signature = result.cryptoObject?.signature
            signTransaction(signature)
        }
        override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
            showError(errString.toString())
        }
        override fun onAuthenticationFailed() {
            // Finger not recognized — don't lock out, let user try again
        }
    }
)

val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("Confirm Transaction")
    .setSubtitle("Use biometric to authorize")
    .setNegativeButtonText("Cancel")
    .setAllowedAuthenticators(
        BiometricManager.Authenticators.BIOMETRIC_STRONG // require strong biometric
    )
    .build()

biometricPrompt.authenticate(promptInfo, BiometricPrompt.CryptoObject(signature))
```

### ProGuard / R8

```kotlin
// build.gradle — enable minification + obfuscation for release
android {
    buildTypes {
        release {
            isMinifyEnabled = true      // removes unused code
            isShrinkResources = true    // removes unused resources
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}

// proguard-rules.pro — keep data classes used with Gson/Moshi
-keep class com.example.data.dto.** { *; }
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
```

### Network security config (Android 7+)

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Only allow HTTPS in production — no cleartext -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>

    <!-- Allow cleartext for local debug only -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system"/>
            <certificates src="user"/> <!-- trust user-added certs in debug -->
        </trust-anchors>
    </debug-overrides>
</network-security-config>

<!-- AndroidManifest.xml -->
<!-- android:networkSecurityConfig="@xml/network_security_config" -->
```

### Root/tamper detection (relevant for crypto wallets)

```kotlin
// Basic root detection — not foolproof but raises the bar
fun isDeviceRooted(): Boolean {
    val paths = arrayOf(
        "/system/app/Superuser.apk",
        "/sbin/su",
        "/system/bin/su",
        "/system/xbin/su"
    )
    return paths.any { File(it).exists() } ||
        Build.TAGS?.contains("test-keys") == true
}

// ✅ Show warning or restrict sensitive features on rooted devices
if (isDeviceRooted()) {
    showRootWarningDialog()
    // TFH/World App should restrict private key operations on rooted devices
}
```

### Sensitive data in memory

```kotlin
// ✅ Clear sensitive data when done
var privateKey: ByteArray? = null
try {
    privateKey = loadKey()
    signTransaction(privateKey)
} finally {
    privateKey?.fill(0) // overwrite with zeros ✅
    privateKey = null
}

// ❌ DON'T store private keys in String — Strings are immutable, can't be zeroed
val key = "0xprivatekeyhere" // stays in memory until GC ❌
```

---

## 28. Android Compliance & Best Practices Checklist

### Target API compliance

```kotlin
// Always target latest stable API in build.gradle
android {
    compileSdk = 35
    defaultConfig {
        minSdk = 24      // covers 98%+ of devices
        targetSdk = 35   // required for Play Store compliance
    }
}
// Google Play requires targetSdk ≥ 34 for new apps (2024)
// Targeting lower = restricted distribution
```

### Runtime permissions (Android 6+)

```kotlin
// ✅ ALWAYS check and request at runtime — never assume granted
@Composable
fun CameraFeature() {
    val cameraPermission = rememberPermissionState(Manifest.permission.CAMERA)

    when {
        cameraPermission.status.isGranted -> {
            CameraPreview()
        }
        cameraPermission.status.shouldShowRationale -> {
            // User denied once — explain why you need it
            PermissionRationale(
                message = "Camera needed to scan QR codes for World ID verification",
                onRequest = { cameraPermission.launchPermissionRequest() }
            )
        }
        else -> {
            // First request OR permanently denied
            LaunchedEffect(Unit) { cameraPermission.launchPermissionRequest() }
        }
    }
}
```

### Battery optimization compliance

```kotlin
// Request exemption only when genuinely needed (e.g., crypto wallet sync)
val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
    data = Uri.parse("package:$packageName")
}
startActivity(intent)

// AndroidManifest — declare if you need this
// <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"/>
// Note: Play Store may reject apps that abuse this — only use for critical background sync
```

### Privacy compliance

```kotlin
// Android 10+: Scoped storage — no more raw file system access
// ❌ FAILS on Android 10+:
File("/sdcard/Download/wallet_backup.json").writeText(data)

// ✅ Use MediaStore or app-specific directories:
val file = File(context.getExternalFilesDir(null), "wallet_backup.json")

// Android 12+: Approximate location (new option)
// <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
// Users can choose coarse vs precise — respect their choice

// Android 13+: Photo picker (no need for READ_MEDIA_IMAGES permission)
val pickMedia = registerForActivityResult(PickVisualMedia()) { uri ->
    uri?.let { processImage(it) }
}
pickMedia.launch(PickVisualMediaRequest(PickVisualMedia.ImageOnly))
```

### App startup compliance

```kotlin
// ✅ Use App Startup library for initializing libraries efficiently
class CryptoTrackerInitializer : Initializer<CryptoTracker> {
    override fun create(context: Context): CryptoTracker {
        return CryptoTracker.init(context)
    }
    override fun dependencies() = emptyList<Class<out Initializer<*>>>()
}
// Avoids multiple ContentProviders at startup — faster cold start

// ✅ Splash Screen API (Android 12+)
override fun onCreate(savedInstanceState: Bundle?) {
    val splashScreen = installSplashScreen()
    super.onCreate(savedInstanceState)
    splashScreen.setKeepOnScreenCondition { !viewModel.isReady.value }
}
```

### Play Store compliance summary

| Requirement | Policy |
|---|---|
| Target SDK | ≥ 34 for new apps, ≥ 33 for updates |
| Permissions | Only declare what you use |
| Background location | Requires explicit Play approval |
| Exact alarms | Requires justification |
| Accessibility service | Restricted use only |
| Overlay permission | Restricted use |
| MANAGE_EXTERNAL_STORAGE | Requires approval |
| Data safety form | Must be accurate and complete |


---

## 29. Kotlin Internals — Quick-Fire Interview Questions

Ojaswirajanya may ask these as warm-up questions before the build. Know the answers cold.

---

### val vs var

```kotlin
val name = "Rutul"  // read-only reference — cannot reassign
var count = 0       // mutable reference — can reassign

// val does NOT mean the object is immutable
val list = mutableListOf(1, 2, 3)
list.add(4) // ✅ allowed — list contents are mutable, reference is not

// Use val by default — forces you to think before making something mutable
```

### data class internals

```kotlin
data class Crypto(val id: String, val price: Double)

// Compiler auto-generates:
// 1. equals() — structural equality based on declared properties
// 2. hashCode() — consistent with equals()
// 3. toString() — "Crypto(id=btc, price=50000.0)"
// 4. copy() — shallow copy with optional overrides
// 5. componentN() — destructuring: val (id, price) = crypto

// copy() example:
val updated = crypto.copy(price = 51000.0) // new object, id unchanged

// Gotcha: only PRIMARY constructor properties included in equals/hashCode
data class Crypto(val id: String) {
    var price: Double = 0.0 // NOT included in equals/hashCode ❌
}

// Gotcha: data class should not be open (subclassing breaks equals contract)
// ❌ open data class Base(...) — avoid
```

### object vs companion object

```kotlin
// object: singleton — one instance for app lifetime
object AppConfig {
    const val BASE_URL = "https://api.coingecko.com/"
    fun getHeaders() = mapOf("Accept" to "application/json")
}
AppConfig.BASE_URL // access directly

// companion object: class-level members (like Java static)
class CryptoViewModel(private val repo: CryptoRepository) : ViewModel() {
    companion object {
        fun factory(repo: CryptoRepository) = object : ViewModelProvider.Factory { ... }
        const val TAG = "CryptoViewModel"
    }
}
CryptoViewModel.factory(repo) // called on class, not instance

// Key difference:
// object = singleton instance (one object in memory)
// companion object = belongs to class, not instance (like static)
// Both: initialized lazily on first access, thread-safe by JVM class loading
```

### sealed class vs enum

```kotlin
// enum: fixed set of constants, same type, no data
enum class Status { LOADING, SUCCESS, ERROR }

// sealed class: fixed set of TYPES, can carry different data
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>() // carries data ✅
    data class Error(val message: String, val code: Int) : UiState<Nothing>() // carries data ✅
}

// Use sealed class when:
// - Subclasses need to carry different data
// - You want exhaustive when expressions
// Use enum when:
// - Simple fixed set of constants with no extra data
```

### inline functions and reified

```kotlin
// inline: compiler copies function body at call site — no lambda object allocation
// Use for HOF (higher-order functions) to avoid overhead

inline fun measure(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block() // inlined — no lambda object created
    return System.currentTimeMillis() - start
}

// reified: access type parameter at runtime (normally erased by JVM)
// Only works with inline functions
inline fun <reified T> parseJson(json: String): T {
    return Gson().fromJson(json, T::class.java) // T is real class at runtime ✅
}

val crypto = parseJson<Crypto>(jsonString) // no Class<T> parameter needed ✅

// ❌ Without reified — T erased at runtime:
fun <T> parseJson(json: String, clazz: Class<T>): T { ... } // must pass Class explicitly
```

### by lazy vs lateinit

```kotlin
// by lazy: for val — initialized on first access, thread-safe by default
val retrofit by lazy { buildRetrofit() }
// Type: any
// Thread-safe: yes (synchronized by default)
// Can be null: no
// Initialized: on first access

// lateinit: for var — initialize later, skip null check
lateinit var adapter: CryptoAdapter
// Type: non-null reference types only (no primitives)
// Thread-safe: no
// Can be null: no (but throws UninitializedPropertyAccessException if accessed before init)
// Initialized: manually before first use

// lateinit check:
if (::adapter.isInitialized) { adapter.notifyDataSetChanged() }

// When to use:
// by lazy → DI objects, singletons, expensive one-time init
// lateinit → Dependency injection fields, test setup (@Before)
```

### Extension functions vs member functions

```kotlin
// Extension function: adds function to existing class without modifying it
fun Double.toPriceString() = "$${"%.2f".format(this)}"
50000.0.toPriceString() // "$50000.00"

// Member function: defined inside the class
class Crypto(val price: Double) {
    fun toPriceString() = "$${"%.2f".format(price)}"
}

// Key differences:
// Extension: cannot access private members
// Extension: resolved statically (no polymorphism) — type at compile time wins
// Member: full access to private state
// Member: dynamically dispatched (virtual by default)

// Kotlin stdlib is full of extensions:
// listOf(), filter(), map(), let, apply, run, also, with
```

### Scope functions — when to use which

```kotlin
// let: transform or null-safe block, returns lambda result
val len = "hello"?.let { it.length } // 5, or null if string is null

// apply: configure object, returns receiver (the object itself)
val paint = Paint().apply { color = Color.RED; strokeWidth = 2f }

// run: operate on object AND return result
val result = database.run { beginTransaction(); query("..."); endTransaction(); lastResult }

// with: multiple calls on same object, returns lambda result (not receiver)
with(binding) { title.text = "Hello"; subtitle.text = "World" }

// also: side effect without changing receiver, returns receiver
val list = mutableListOf(1,2,3).also { log("Created list: $it") }
```

### Coroutine vs Thread

```kotlin
// Thread:
// - OS-level — expensive to create (~1MB stack)
// - Blocking — sleeps while waiting
// - Thousands of threads → OOM

// Coroutine:
// - Lightweight — thousands on one thread
// - Suspending — gives up thread while waiting, resumes when ready
// - Structured — tied to a scope, automatically cancelled

// 1 Thread = 1000s of coroutines
// delay(1000) suspends coroutine, Thread.sleep(1000) blocks OS thread
```

---

## 30. RecyclerView vs LazyColumn

Ojaswirajanya came from Snap — RecyclerView era. She may ask you to compare directly.

---

### RecyclerView (View system)

```kotlin
// Setup — significant boilerplate
class CryptoAdapter(
    private val items: List<Crypto>,
    private val onClick: (Crypto) -> Unit
) : RecyclerView.Adapter<CryptoAdapter.ViewHolder>() {

    class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
        val name: TextView = view.findViewById(R.id.name)
        val price: TextView = view.findViewById(R.id.price)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_crypto, parent, false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val crypto = items[position]
        holder.name.text = crypto.name
        holder.price.text = crypto.price.toPriceString()
        holder.itemView.setOnClickListener { onClick(crypto) }
    }

    override fun getItemCount() = items.size
}

// In Fragment:
recyclerView.adapter = CryptoAdapter(cryptos) { crypto -> navigate(crypto) }
recyclerView.layoutManager = LinearLayoutManager(context)
```

### LazyColumn (Compose)

```kotlin
// Same result — dramatically less code
@Composable
fun CryptoList(
    cryptos: List<Crypto>,
    onItemClick: (Crypto) -> Unit
) {
    LazyColumn {
        items(
            items = cryptos,
            key = { it.id }
        ) { crypto ->
            CryptoItem(crypto = crypto, onClick = { onItemClick(crypto) })
        }
    }
}
```

### Direct comparison

| | RecyclerView | LazyColumn |
|---|---|---|
| Boilerplate | High (Adapter, ViewHolder, XML layout) | Minimal |
| Item recycling | Explicit ViewHolder pattern | Automatic — Compose handles |
| Partial updates | DiffUtil (must implement) | Automatic with `key` param |
| Performance | Excellent — mature, optimised | Good — improving with each release |
| Custom layout | LayoutManager (complex) | `LazyVerticalGrid`, `LazyVerticalStaggeredGrid` |
| Animation | `ItemAnimator` (complex setup) | `animateItemPlacement()` (one modifier) |
| Headers/Footers | Complex — multiple view types | `item(key = "header") { Header() }` |
| Scrolling state | `LinearLayoutManager.findFirstVisibleItemPosition()` | `rememberLazyListState()` |
| Nested scroll | `NestedScrollView` (complex) | Built-in — works automatically |
| Testing | Espresso matchers | `ComposeTestRule` semantic matchers |

### DiffUtil vs key parameter

```kotlin
// RecyclerView: must implement DiffUtil for efficient updates
class CryptoDiffCallback : DiffUtil.ItemCallback<Crypto>() {
    override fun areItemsTheSame(old: Crypto, new: Crypto) = old.id == new.id
    override fun areContentsTheSame(old: Crypto, new: Crypto) = old == new
}
// Then use ListAdapter instead of Adapter — handles diffing automatically

// LazyColumn: key parameter does the same thing, automatically
LazyColumn {
    items(cryptos, key = { it.id }) { CryptoItem(it) }
    // Compose tracks by id — only changed items recompose
    // Insertions/deletions animate automatically with animateItemPlacement()
}
```

### When RecyclerView is still better

```kotlin
// 1. Interoperability with existing View-based codebase
// 2. Very complex custom layouts (custom LayoutManager)
// 3. Performance-critical lists with 10,000+ items (RecyclerView is more mature)
// 4. When using libraries only built for RecyclerView (some drag-and-drop libs)
```

### What to say in interview

*"LazyColumn is the Compose equivalent of RecyclerView. It gives you the same composing-on-demand behaviour — only items visible on screen are composed — but with a fraction of the boilerplate. You get DiffUtil-equivalent behaviour for free with the `key` parameter, and animations with `animateItemPlacement()`. RecyclerView is still more battle-tested for extremely large datasets and has more mature ecosystem tooling, but for a new Compose app LazyColumn is the right default."*

---

## 31. Compose Testing — Deep Dive

---

### Test types and setup

```kotlin
// 1. Unit tests — pure Kotlin, no Android framework
// Run on JVM, fast
// Test: ViewModel, Repository, Use Cases, pure functions
// Dependencies: JUnit4, kotlinx-coroutines-test, Turbine, MockK

// 2. Instrumented tests — run on device/emulator
// Test: Compose UI, navigation, database
// Dependencies: compose-ui-test, espresso-core

// Compose UI test setup:
@get:Rule
val composeRule = createComposeRule()          // no Activity — just Compose
// OR
val composeRule = createAndroidComposeRule<MainActivity>() // with real Activity
```

### Semantic tree — how Compose tests find UI

Compose builds a **semantic tree** parallel to the UI tree. Tests navigate this tree, not the UI directly.

```kotlin
// UI: Box → Text("Bitcoin") → Text("$50,000")
// Semantic tree: Node(text="Bitcoin") → Node(text="$50,000")

// Finding nodes:
composeRule.onNodeWithText("Bitcoin")            // by text content
composeRule.onNodeWithTag("crypto_item_btc")     // by testTag
composeRule.onNodeWithContentDescription("logo") // by accessibility label
composeRule.onAllNodesWithText("$")              // all matching nodes

// Adding semantic info to composables:
Text(
    text = crypto.name,
    modifier = Modifier.semantics {
        contentDescription = "Crypto name: ${crypto.name}"
        testTag = "crypto_name_${crypto.id}"
    }
)

// Simpler testTag:
Text(
    text = crypto.name,
    modifier = Modifier.testTag("crypto_name_${crypto.id}")
)
```

### Common assertions and actions

```kotlin
// Assertions:
composeRule.onNodeWithText("Bitcoin").assertIsDisplayed()
composeRule.onNodeWithText("Bitcoin").assertIsNotDisplayed()
composeRule.onNodeWithTag("loading").assertExists()
composeRule.onNodeWithTag("loading").assertDoesNotExist()
composeRule.onNodeWithText("Submit").assertIsEnabled()
composeRule.onNodeWithText("Submit").assertIsNotEnabled()
composeRule.onNodeWithText("error message").assertIsDisplayed()

// Actions:
composeRule.onNodeWithText("Bitcoin").performClick()
composeRule.onNodeWithTag("search_field").performTextInput("eth")
composeRule.onNodeWithTag("search_field").performTextClearance()
composeRule.onNodeWithTag("crypto_list").performScrollToIndex(20)
composeRule.onNodeWithText("Swipe me").performTouchInput { swipeLeft() }
```

### Testing navigation

```kotlin
@Test
fun `clicking item navigates to detail`() {
    val navController = TestNavHostController(LocalContext.current)
    composeRule.setContent {
        navController.setGraph(R.navigation.nav_graph)
        AppNavigation(navController = navController)
    }

    composeRule.onNodeWithText("Bitcoin").performClick()

    assertEquals(
        "detail/btc",
        navController.currentBackStackEntry?.destination?.route
    )
}
```

### Testing with ViewModel — fake ViewModel

```kotlin
// ✅ Test with real ViewModel + fake repository
@Test
fun `shows error when network fails`() = runTest {
    val repo = FakeCryptoRepository().apply { shouldThrow = true }
    val viewModel = CryptoViewModel(repo)

    composeRule.setContent {
        CryptoScreen(viewModel = viewModel)
    }

    composeRule.onNodeWithTag("error_view").assertIsDisplayed()
    composeRule.onNodeWithText("Retry").assertIsDisplayed()
}
```

### Waiting for async operations

```kotlin
// Compose test automatically advances clock for animations
// But for async data loading, use waitUntil:

@Test
fun `shows data after loading`() {
    val repo = FakeCryptoRepository()
    val viewModel = CryptoViewModel(repo)

    composeRule.setContent { CryptoScreen(viewModel = viewModel) }

    // Wait for loading to complete (with timeout)
    composeRule.waitUntil(timeoutMillis = 5000) {
        composeRule.onAllNodesWithTag("crypto_item").fetchSemanticsNodes().isNotEmpty()
    }

    composeRule.onNodeWithText("Bitcoin").assertIsDisplayed()
}
```

### MockK — mocking in Kotlin

```kotlin
// MockK is idiomatic Kotlin mocking (preferred over Mockito in Kotlin)
val mockRepo = mockk<CryptoRepository>()

// Stub behaviour:
coEvery { mockRepo.getCryptos() } returns listOf(Crypto("btc", "Bitcoin", 50000.0))
coEvery { mockRepo.getCryptos() } throws IOException("Network error")

// Verify calls:
coVerify { mockRepo.getCryptos() }
coVerify(exactly = 2) { mockRepo.getCryptos() }
coVerify(exactly = 0) { mockRepo.save(any()) }

// Relaxed mock — returns defaults for unstubbed calls:
val mockRepo = mockk<CryptoRepository>(relaxed = true)
```


### Complete Kotlin Flow Operators Reference Guide

| Operator | Real-World Use Case | When to Use | When to Avoid |
| :--- | :--- | :--- | :--- |
| **`map`** | Transforming raw data models into UI display models. | Transforming *every* item into something else synchronously. | When the transformation requires an async network/DB call. |
| **`filter`** | Dropping search terms shorter than 2 characters. | Skipping items that do not match a boolean condition. | When you want to alter the item's underlying data structure. |
| **`debounce`** | Waiting for a user to finish typing a search query. | Pausing emissions until a brief quiet period has passed. | When every single event matters (like button clicks). |
| **`distinctUntilChanged`** | Preventing UI redraws if the user re-clicks the same filter. | Skipping duplicate consecutive emissions. | When identical sequential events still require action. |
| **`combine`** | Merging user filter selections with database results. | Merging 2+ independent flows; triggers on *any* update. | When you need to fetch new async data based on an input. |
| **`flatMapLatest`** | Canceling an old API search request when user types more text. | Switching to a *new async flow* and canceling the old one. | When your internal transformation is fast and synchronous. |
| **`flatMapMerge`** | Fetching data or downloading files concurrently from a list. | Transforming items into async flows that run *at the same time*. | When order matters, or when old tasks should be canceled. |
| **`zip`** | Pairing a User Profile and a User Wallet together once. | Pairing items from 2 flows strictly 1-to-1 (waits for both). | When either flow updates frequently and independently. |
| **`onEach`** | Logging updates to Logcat or analytics tracking. | Performing side-effects *without* changing the data payload. | Transforming data or running heavy async tasks. |
| **`onStart`** | Showing a loading spinner before network data drops. | Executing an action or emitting a value before collection begins. | Performing heavy, non-UI background tasks. |
| **`catch`** | Catching a database crash and showing an error UI. | Handling exceptions thrown by upstream operators safely. | Trying to catch errors that occur *outside* the flow stream. |

---

### Complete Production-Ready Flow Pipeline

This architecture brings all 11 operators together into a production-ready reactive pipeline inside an Android `ViewModel`.

```kotlin
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.flow.*
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope

class CryptoViewModel(
    private val repository: CryptoRepository
) : ViewModel() {

    // Input streams driven by user actions
    val searchQuery = MutableStateFlow("")
    val filterFavoritesOnly = MutableStateFlow(false)

    @OptIn(ExperimentalCoroutinesApi::class)
    val uiState: StateFlow<CryptoUiState> = searchQuery
        // 1. DEBOUNCE: Wait 300ms for user to pause typing before forwarding query
        .debounce(300L)
        
        // 2. DISTINCTUNTILCHANGED: Ignore if user types 'A', deletes it, types 'A' within 300ms
        .distinctUntilChanged()
        
        // 3. ONSTART: Instantly emit a safe signal to show loading indicators during setup
        .onStart { emit("") }
        
        // 4. FILTER: Stop processing short queries to protect network bandwidth
        .filter { query -> query.length >= 2 || query.isEmpty() }
        
        // 5. FLATMAPLATEST: Call the DB/API. Kills previous search job if user types a new letter
        .flatMapLatest { query -> 
            if (query.isEmpty()) flowOf(emptyList()) else repository.searchCryptos(query)
        }
        
        // 6. COMBINE: Blend search results with favorite filters; updates if either change
        .combine(filterFavoritesOnly) { cryptoList, favoritesOnly ->
            if (favoritesOnly) cryptoList.filter { it.isFavorite } else cryptoList
        }
        
        // 7. MAP: Wrap finalized domain models inside your UI layer State wrapper
        .map { finalFilteredList -> 
            CryptoUiState.Success(data = finalFilteredList) 
        }
        
        // 8. ONEACH: Execute side-effects like background analytics or debugging logs
        .onEach { state -> 
            println("Pipeline updated UI state successfully: \$state") 
        }
        
        // 9. CATCH: Final safety net catching upstream errors (DB errors, HTTP issues, etc.)
        .catch { exception -> 
            emit(CryptoUiState.Error(exception.message ?: "An unexpected error occurred")) 
        }
        
        // 10. STATEIN: Cast cold processing stream into a UI-readable Hot StateFlow
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000), // 5s grace period for screen rotations
            initialValue = CryptoUiState.Loading
        )

    // 11. ZIP / FLATMAPMERGE BONUS EXAMPLE: Fetching independent metadata concurrently
    fun fetchDetailsAndPriceConcurrently(cryptoId: String) {
        val detailsFlow = repository.getDetails(cryptoId)
        val priceFlow = repository.getRealTimePrice(cryptoId)

        detailsFlow.zip(priceFlow) { details, price ->
            "Price of \({details.name} is now \$\){price.amount}"
        }.onEach { result ->
            println(result)
        }.launchIn(viewModelScope)
    }
}

// Support definitions for context
sealed interface CryptoUiState {
    object Loading : CryptoUiState
    data class Success(val data: List<Crypto>) : CryptoUiState
    data class Error(val message: String) : CryptoUiState
}
data class Crypto(val id: String, val name: String, val isFavorite: Boolean)
data class Price(val amount: Double)
interface CryptoRepository { 
    fun searchCryptos(query: String): Flow<List<Crypto>> 
    fun getDetails(id: String): Flow<Crypto>
    fun getRealTimePrice(id: String): Flow<Price>
}
```
