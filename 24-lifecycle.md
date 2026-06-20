# 24 — Lifecycle Management

> **Round type: LLD (primary)**
> **Focus:** Activity/Fragment lifecycle, ViewModel scoping, Compose lifecycle, process death vs config change, SavedStateHandle, ProcessLifecycleOwner, repeatOnLifecycle

---

## 1. Why This Matters in Interviews

Lifecycle is the foundational Android concept. Every memory leak, every crash after rotation, every "my data disappeared when the user pressed home" bug traces back to misunderstanding lifecycle. Interviewers consider this fundamental — getting it wrong at a senior level is a red flag.

**Common interview angles:**
- "What's the difference between a configuration change and process death?"
- "ViewModel survives rotation but not process death — why?"
- "When would you use `SavedStateHandle`?"
- "What's the difference between `lifecycleScope` and `viewLifecycleOwner.lifecycleScope`?"
- "How do you collect a Flow safely in a Fragment?"
- "What is `repeatOnLifecycle` and why was it introduced?"
- "What happens to a ViewModel when you navigate back?"

---

## 2. Activity Lifecycle

```
                  ┌──────────────┐
                  │   Created    │
                  └──────┬───────┘
                  onCreate()
                         │
                  ┌──────▼───────┐
                  │   Started    │
                  └──────┬───────┘
                  onStart()
                         │
                  ┌──────▼───────┐
     ┌──────────▶ │   Resumed    │ ◀──────────┐
     │            └──────┬───────┘            │
     │            (user sees UI)              │
     │            onPause()                   │
     │            ┌──────▼───────┐            │
     │            │   Paused     │            │
     │            └──────┬───────┘            │
     │            onStop()                    │
     │            ┌──────▼───────┐            │
     │            │   Stopped    │            │
     │            └──────┬───────┘            │
     │            onRestart() ────────────────┘
     │            OR
     │            onDestroy()
     │            ┌──────▼───────┐
     └──────────  │  Destroyed   │
                  └──────────────┘
```

**Key state transitions:**
- `onCreate` → `onResume`: Activity becoming visible and interactive
- `onPause`: Another Activity partially obscures (dialog, permission dialog, incoming call)
- `onStop`: Activity fully hidden (user pressed Home, navigated away)
- `onDestroy`: Activity is finishing (user pressed Back, `finish()` called, OR configuration change)

**Distinguish intentional vs configuration destroy:**
```kotlin
override fun onDestroy() {
    super.onDestroy()
    if (isFinishing) {
        // User pressed Back or finish() called — truly done
        cleanUpPermanentResources()
    } else {
        // Configuration change — Activity will be recreated
        // ViewModel already handles data preservation — nothing needed here
    }
}
```

### 2.1 Lifecycle-Aware Operations

| Lifecycle state | RESUMED | STARTED | CREATED |
|---|---|---|---|
| Camera preview | ✅ | ❌ | ❌ |
| Play animation | ✅ | ❌ | ❌ |
| UI updates | ✅ | ✅ (visible) | ❌ |
| Background tasks | ✅ | ✅ | ✅ |
| DB operations | ✅ | ✅ | ✅ |

---

## 3. Fragment Lifecycle — Two Lifecycles!

Fragment has **two separate lifecycles** — this is the most common Fragment lifecycle mistake:

```
Fragment Lifecycle:              Fragment VIEW Lifecycle:
─────────────────────            ──────────────────────────
onAttach()                       (doesn't exist yet)
onCreate()                       (doesn't exist yet)
onCreateView()     ─────────────▶ ON_CREATE (view created)
onViewCreated()    ─────────────▶ ON_START (view visible)
onStart()          ─────────────▶ ON_RESUME (view interactive)
onResume()         ─────────────▶ ON_PAUSE
onPause()          ─────────────▶ ON_STOP
onStop()           ─────────────▶ ON_DESTROY
onDestroyView()    ◀─────────────  (view destroyed)
onDestroy()                      (doesn't exist yet)
onDetach()                       (doesn't exist yet)
```

**Critical difference:** When a Fragment is placed on the back stack:
- The Fragment's VIEW is destroyed (`onDestroyView`)
- BUT the Fragment object itself stays alive (attached to FragmentManager)
- The Fragment's lifecycle stays at `CREATED`
- The view lifecycle goes all the way to `DESTROYED`

This is why you must use `viewLifecycleOwner` for view-related operations:

```kotlin
class ProfileFragment : Fragment() {
    // ✅ viewLifecycleOwner.lifecycleScope — cancelled in onDestroyView
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.profile.collect { profile ->
                binding.name.text = profile.name  // safe — view exists
            }
        }
    }

    // ❌ lifecycleScope — lives as long as Fragment, outlives the view
    // After onDestroyView, this coroutine can still run and try to update
    // a destroyed view → crash or memory leak
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            viewModel.profile.collect { profile ->
                binding.name.text = profile.name  // UNSAFE — binding may be null
            }
        }
    }
}
```

---

## 4. ViewModel Lifecycle

### 4.1 What ViewModel Survives

```
Configuration change (rotation, dark mode, locale change):
  Activity/Fragment DESTROYED and RECREATED
  ViewModel: ✅ SURVIVES (stays in memory)

User presses Back:
  Activity/Fragment DESTROYED (isFinishing = true)
  ViewModel: ❌ CLEARED (onCleared() called)

System-initiated process death (low memory):
  Entire process KILLED
  ViewModel: ❌ GONE (everything in memory is lost)
  SavedStateHandle: ✅ CAN SURVIVE (uses Bundle serialization)

User force-stops the app:
  Everything GONE
```

### 4.2 ViewModel Scoping

```kotlin
// Fragment-scoped — unique to this Fragment, cleared when Fragment is destroyed
class ProfileFragment : Fragment() {
    private val viewModel: ProfileViewModel by viewModels()
}

// Activity-scoped — shared between all Fragments in this Activity
class ProfileFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
}

// NavGraph-scoped — shared between all destinations in the same NavGraph
@Composable
fun ProfileScreen(
    viewModel: ProfileViewModel = hiltViewModel()  // default: NavBackStackEntry scope
)
// OR explicitly:
@Composable
fun ProfileScreen() {
    val navController = rememberNavController()
    val entry = navController.getBackStackEntry("profile_graph")  // NavGraph route
    val viewModel: SharedViewModel = hiltViewModel(entry)
}
```

### 4.3 ViewModel Lifecycle Diagram

```
Activity created ──▶ ViewModel created
     │
Rotate device ──▶ Activity destroyed + recreated
                    ViewModel SURVIVES ✅
     │
Rotate again ──▶ Same pattern
     │
User presses Back ──▶ Activity finished ──▶ ViewModel CLEARED
                                              onCleared() called
```

### 4.4 `onCleared()` — ViewModel Cleanup

```kotlin
class PlayerViewModel : ViewModel() {
    private val player = ExoPlayer.Builder(context).build()
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onCleared() {
        super.onCleared()
        player.release()        // free hardware codec resources
        scope.cancel()          // cancel any running coroutines
        analyticsSession.end()  // end analytics session
    }
}
```

---

## 5. Configuration Change vs Process Death

This is the most important distinction in Android lifecycle:

| Event | ViewModel | rememberSaveable/Bundle | Notes |
|---|---|---|---|
| Screen rotation | ✅ Survives | ✅ Survives | Config change — ViewModel is THE solution |
| Dark mode toggle | ✅ Survives | ✅ Survives | Config change |
| Language change | ✅ Survives | ✅ Survives | Config change |
| User presses Back | ❌ Cleared | ❌ Cleared | Normal finish — intentional |
| User presses Home | ✅ Survives (process stays) | ✅ Survives | Background, may be killed |
| **Process death** | **❌ Lost** | **✅ Survives** | Use SavedStateHandle for critical state |
| Force stop | ❌ Lost | ❌ Lost | Nothing survives |

**How to test process death:**
```bash
# Put app in background (press Home)
adb shell am kill com.example.myapp   # kill process
# Relaunch app from recents — should restore state
# If state is gone → missing SavedStateHandle
```

---

## 6. SavedStateHandle — Surviving Process Death

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,  // injected by Hilt
    private val searchRepo: SearchRepository
) : ViewModel() {

    // Survived process death via SavedStateHandle
    private val _query = savedStateHandle.getStateFlow("search_query", "")
    val query: StateFlow<String> = _query

    // Navigation args are also available via SavedStateHandle
    private val productId: String = checkNotNull(savedStateHandle["productId"])

    fun onQueryChanged(newQuery: String) {
        savedStateHandle["search_query"] = newQuery  // persisted immediately
        viewModelScope.launch {
            searchResults.value = searchRepo.search(newQuery)
        }
    }
}
```

**What can be stored in SavedStateHandle:**
- Primitives (Int, String, Boolean, Float, Long)
- Parcelables (custom classes with `@Parcelize`)
- Serializable classes
- Arrays and Lists of the above

**What CANNOT be stored:**
- Large objects (Bitmaps, full data models)
- Non-serializable objects (Flows, Lambdas, Coroutines)
- More than ~1MB total (Bundle size limit)

**Rule of thumb:** SavedStateHandle is for *user input* and *navigation state* — query strings, form input, selected IDs. Not for loaded data (reload that from network/DB on restart).

```kotlin
// Navigation argument retrieval (Compose Navigation)
@Composable
fun ProductDetailScreen(
    viewModel: ProductDetailViewModel = hiltViewModel()
    // SavedStateHandle automatically receives nav args
)

// From NavHost:
composable("product/{id}") { backStackEntry ->
    // backStackEntry.arguments?.getString("id") — the old way
    // ViewModel receives it automatically via SavedStateHandle["id"]
    ProductDetailScreen()
}
```

---

## 7. Compose Lifecycle

A composable enters and leaves the composition as it's shown and hidden. The composition lifecycle is separate from Activity/Fragment lifecycle:

```
Composable lifecycle:
  Enter composition → composable is first added to the UI tree
  Recompose → composable runs again due to state change
  Leave composition → composable is removed from UI tree

NOT the same as:
  Activity.onResume / Activity.onPause
  A composable can be in composition while Activity is paused (partially obscured)
```

### 7.1 `LaunchedEffect` and Composition Lifecycle

```kotlin
// LaunchedEffect runs when composable ENTERS composition
// LaunchedEffect coroutine cancels when composable LEAVES composition
// LaunchedEffect restarts when keys change

@Composable
fun ChatScreen(conversationId: String) {
    // Started when ChatScreen first appears
    // Cancelled when ChatScreen is removed from composition (back pressed)
    // Restarted if conversationId changes
    LaunchedEffect(conversationId) {
        webSocketManager.subscribe(conversationId)
    }
    // cleanup → DisposableEffect
}
```

### 7.2 Lifecycle-Aware Collection with `repeatOnLifecycle`

**The problem:** A Flow collected in a composable with `collectAsState()` keeps collecting even when the app is in the background. This wastes resources and may update UI that isn't visible.

```kotlin
// ❌ collectAsState — always collecting, even in background
val state by viewModel.state.collectAsState()

// ✅ collectAsStateWithLifecycle — pauses collection in background
val state by viewModel.state.collectAsStateWithLifecycle()
// Lifecycle.State.STARTED — pauses when Activity/Fragment goes to background
// Resumes when Activity/Fragment comes back to foreground
```

**In non-Compose code (Fragment):**
```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // ❌ Old way — collects even in background
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.state.collect { state -> updateUI(state) }
        }

        // ✅ Modern way — suspends when lifecycle below STARTED
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { state -> updateUI(state) }
            }
        }
        // repeatOnLifecycle: collection pauses in onStop, resumes in onStart
        // Prevents updating UI that's not visible to the user
    }
}
```

**Why `STARTED` not `RESUMED`:**
- `STARTED` = Activity/Fragment is visible (may be partially obscured)
- `RESUMED` = Activity/Fragment is fully interactive (no dialog on top)
- Use `STARTED` for data that should update whenever visible
- Use `RESUMED` only if updates should pause even when partially obscured

---

## 8. ProcessLifecycleOwner — App-Level Foreground/Background

```kotlin
// Detect when the app goes to background/foreground (not just Activity)
class AppLifecycleObserver @Inject constructor(
    private val analyticsTracker: AnalyticsTracker,
    private val syncManager: SyncManager
) : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) {
        // App came to foreground (any Activity became visible)
        analyticsTracker.trackAppForegrounded()
        syncManager.onAppForegrounded()
    }

    override fun onStop(owner: LifecycleOwner) {
        // App went to background (all Activities hidden)
        analyticsTracker.trackAppBackgrounded()
        syncManager.onAppBackgrounded()
    }
}

// Register in Application.onCreate()
class MyApp : Application() {
    @Inject lateinit var appLifecycleObserver: AppLifecycleObserver

    override fun onCreate() {
        super.onCreate()
        ProcessLifecycleOwner.get().lifecycle.addObserver(appLifecycleObserver)
    }
}
```

**`ProcessLifecycleOwner` vs `Activity.onResume`:**
- `Activity.onResume/onPause` fires when any dialog appears (even in-app)
- `ProcessLifecycleOwner.onStart/onStop` fires only when the entire app enters/leaves foreground
- Use `ProcessLifecycleOwner` for: analytics session tracking, connection management, sync triggers

---

## 9. LifecycleObserver — Decoupled Lifecycle Handling

```kotlin
// Move lifecycle-aware logic OUT of Activity/Fragment
class LocationManager(
    private val fusedLocationClient: FusedLocationProviderClient
) : DefaultLifecycleObserver {

    private var locationCallback: LocationCallback? = null

    override fun onStart(owner: LifecycleOwner) {
        startLocationUpdates()
    }

    override fun onStop(owner: LifecycleOwner) {
        stopLocationUpdates()
    }

    private fun startLocationUpdates() {
        locationCallback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                // handle location
            }
        }
        fusedLocationClient.requestLocationUpdates(request, locationCallback!!, Looper.getMainLooper())
    }

    private fun stopLocationUpdates() {
        locationCallback?.let { fusedLocationClient.removeLocationUpdates(it) }
    }
}

// In Activity/Fragment — just register, no cleanup code needed
class MapActivity : AppCompatActivity() {
    private val locationManager: LocationManager by inject()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(locationManager)  // auto start/stop based on lifecycle
    }
}
```

---

## 10. Handling Configuration Changes Without ViewModel

For edge cases where you need custom configuration change handling:

```kotlin
// Declare in AndroidManifest.xml to handle config changes manually
<activity
    android:name=".VideoActivity"
    android:configChanges="orientation|screenSize|keyboardHidden"/>

// In Activity
override fun onConfigurationChanged(newConfig: Configuration) {
    super.onConfigurationChanged(newConfig)
    // Activity NOT recreated — handle change manually
    // e.g., adjust video player orientation without interrupting playback
    if (newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE) {
        enterFullscreen()
    } else {
        exitFullscreen()
    }
}
```

**Use sparingly** — this bypasses normal lifecycle handling and means you're responsible for all configuration change effects (layout updates, resource reloading).

---

## 11. State Survival Decision Tree

```
What data are you trying to preserve?
│
├── Loaded data (network response, DB query results)?
│     → ViewModel (survives config change)
│     → Reload from DB/network after process death (don't try to save)
│
├── User input (text field, selection)?
│     → rememberSaveable (composable) or SavedStateHandle (ViewModel)
│     → Both survive config change AND process death
│
├── Scroll position?
│     → LazyListState + rememberSaveable (Compose handles this)
│     → RecyclerView.Adapter state preserved by LayoutManager
│
├── Navigation state (which screen)?
│     → NavController state — saved automatically by Navigation
│     → Back stack restored after process death
│
└── Large objects (Bitmaps, full models)?
      → Never save in Bundle (size limit ~1MB)
      → Reload from local DB/network on restart
      → Store source of truth in Room/DataStore
```

---

## 12. Compose + Navigation: ViewModel Scoping

```kotlin
// Default: NavBackStackEntry-scoped — ViewModel cleared when destination popped
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = hiltViewModel())
// When user presses Back from ProfileScreen → viewModel.onCleared() called

// NavGraph-scoped: ViewModel shared across multiple screens in a flow
@Composable
fun CheckoutFlowNavGraph() {
    NavHost(navController, startDestination = "address") {
        navigation(startDestination = "address", route = "checkout") {
            composable("address") { AddressScreen() }
            composable("payment") { PaymentScreen() }
            composable("confirmation") { ConfirmationScreen() }
        }
    }
}

// In AddressScreen/PaymentScreen/ConfirmationScreen:
@Composable
fun AddressScreen() {
    val navController = LocalNavController.current
    val checkoutEntry = navController.getBackStackEntry("checkout")  // NavGraph route
    val checkoutViewModel: CheckoutViewModel = hiltViewModel(checkoutEntry)
    // All three screens share the SAME CheckoutViewModel instance
    // ViewModel only cleared when the entire checkout graph is popped
}
```

---

## 13. React Native Lifecycle

```typescript
// AppState — detect foreground/background transitions
import { AppState, AppStateStatus } from 'react-native'

const AppLifecycleMonitor = () => {
    const appState = useRef(AppState.currentState)

    useEffect(() => {
        const subscription = AppState.addEventListener('change', (nextState: AppStateStatus) => {
            if (appState.current.match(/inactive|background/) && nextState === 'active') {
                // App came to foreground
                syncData()
                reconnectWebSocket()
            } else if (appState.current === 'active' && nextState.match(/inactive|background/)) {
                // App went to background
                saveUnsyncedData()
            }
            appState.current = nextState
        })

        return () => subscription.remove()
    }, [])

    return null
}

// useEffect cleanup — runs on unmount (similar to onDestroy)
useEffect(() => {
    const subscription = eventEmitter.addListener('event', handler)
    return () => subscription.remove()  // cleanup on unmount
}, [])

// Persisting state across process death in RN:
// MMKV or AsyncStorage for simple values
// WatermelonDB for relational data
// Redux Persist for store state
```

---

## 14. Common Misunderstandings & Pitfalls

**❌ `lifecycleScope` instead of `viewLifecycleOwner.lifecycleScope` in Fragments**
`lifecycleScope` is tied to the Fragment's lifecycle (from `onAttach` to `onDetach`). When the Fragment goes to the back stack, its view is destroyed but the Fragment itself stays alive. A coroutine in `lifecycleScope` keeps running and can access `binding` (which is null after `onDestroyView`) → crash or memory leak. Always use `viewLifecycleOwner.lifecycleScope` for view-related coroutines.

**❌ Storing large objects in `SavedStateHandle`**
`SavedStateHandle` uses Android's Bundle mechanism with a ~1MB limit. Storing full lists of items, Bitmaps, or large models causes `TransactionTooLargeException` (often a silent crash). Store only IDs or small primitives — reload data from DB/network.

**❌ Assuming `ViewModel` survives process death**
ViewModel data is in-memory. Process death destroys all memory. Users who put the app in background for a long time may find their ViewModel data gone. Use `SavedStateHandle` for critical user-visible state (search query, form input) that should survive.

**❌ Collecting Flows in `onResume`/`onPause`**
Collecting in `onResume` and cancelling in `onPause` means collection pauses whenever a dialog or permission prompt appears (both cause `onPause`). The correct granularity is `onStart`/`onStop` (visible) via `repeatOnLifecycle(Lifecycle.State.STARTED)`.

**❌ Calling `finish()` in `onCreate()` without `return`**
Calling `finish()` triggers `onDestroy()` eventually, but `onCreate()` continues executing. Forget to `return` and you'll set up UI, start coroutines, and register observers on an Activity that's finishing. Always `return` after `finish()`.

**❌ `activityViewModels()` when Fragment-scoped ViewModel is appropriate**
`activityViewModels()` creates a ViewModel scoped to the Activity — it's never cleared while the Activity lives. Using it for data that's only relevant to one Fragment means the ViewModel holds data it no longer needs. Use Fragment-scoped `viewModels()` for Fragment-private data, `activityViewModels()` only for genuinely shared cross-Fragment state.

**❌ Not implementing `onCleared()` for resource-holding ViewModels**
ViewModels that create `ExoPlayer`, `AudioRecord`, `BluetoothGatt`, or custom coroutine scopes must release them in `onCleared()`. Forgetting causes: media codec leaks, audio focus never released, Bluetooth connections held open, coroutines running forever.

---

## 15. Best Practices

- **ViewModel for transient data** — anything that survives rotation but can be reloaded on process death (most UI data, network responses)
- **`SavedStateHandle` for user-visible state** — search queries, form input, filter selections, pagination position
- **`viewLifecycleOwner.lifecycleScope`** — always, never `lifecycleScope` for view operations in Fragments
- **`collectAsStateWithLifecycle`** in Compose — not `collectAsState` — respects lifecycle, saves battery
- **`repeatOnLifecycle(STARTED)`** in non-Compose — the correct granularity for UI updates
- **`ProcessLifecycleOwner`** for app-level foreground/background detection (not per-Activity)
- **`LifecycleObserver`** to move lifecycle code out of Activity/Fragment into dedicated classes
- **NavGraph-scoped ViewModel** for multi-screen flows (checkout, onboarding) — shared state without Activity scope
- **`onCleared()` for all resource cleanup** — ExoPlayer, AudioRecord, custom scopes, Bluetooth
- **Test process death** — `adb shell am kill` while app is backgrounded, then relaunch from recents
- **Never hold Activity reference in ViewModel** — ViewModel outlives Activity, causes memory leak; use `ApplicationContext` if context is needed

---

## 16. Interview Q&A

**Q1: What's the difference between a configuration change and process death?**

> A configuration change is when the Android system destroys and recreates the Activity due to a system configuration change — screen rotation, dark mode toggle, locale change, keyboard appearance. The app process keeps running; ViewModel survives in memory; only the Activity and Fragments are recreated. Process death is when the OS kills the entire app process due to memory pressure. The process is gone — ViewModel data, all in-memory state, everything is lost. When the user relaunches the app, it starts fresh. The critical distinction for data preservation: ViewModel handles configuration changes automatically, but process death requires `SavedStateHandle` (uses Bundle serialization, survived via OS) or reloading from persistent storage (Room, DataStore).

---

**Q2: Why does `viewLifecycleOwner.lifecycleScope` matter in Fragments?**

> A Fragment has two separate lifecycles. The Fragment object itself lives from `onAttach` to `onDetach` — which includes when the Fragment is on the back stack. The Fragment's *view* lives from `onCreateView` to `onDestroyView`. When a Fragment is placed on the back stack, `onDestroyView` is called — the view is gone — but the Fragment object stays alive. If you launch a coroutine in `lifecycleScope` that accesses `binding`, it keeps running after the view is destroyed and crashes when it tries to access a null view. `viewLifecycleOwner.lifecycleScope` is tied to the view lifecycle — it's cancelled in `onDestroyView`, exactly when the view ceases to exist. This is the safe scope for any coroutine that touches the view.

---

**Q3: When would you use `SavedStateHandle` vs just `ViewModel`?**

> ViewModel alone is sufficient for most data — anything that can be reloaded from the network or database on restart (a list of posts, user profile, recommendations). ViewModel survives configuration changes, and reloading after process death is the right design for loaded data. `SavedStateHandle` is for state that the user explicitly created and expects to persist — a typed search query, form input mid-way through, selected filter options, scroll position, navigation arguments. If a user types a search query, backgrounds the app, the OS kills the process, and they return to find an empty search field — that's a bug. With `SavedStateHandle["search_query"] = query` in `onQueryChanged`, the value is serialized to a Bundle and survives process death. Rule: if the user would be annoyed to lose it, use `SavedStateHandle`.

---

**Q4: What is `repeatOnLifecycle` and why was it introduced?**

> Before `repeatOnLifecycle`, the standard pattern for collecting Flows in Fragments was `viewLifecycleOwner.lifecycleScope.launch { flow.collect { } }`. This keeps collecting even when the Fragment is not visible — in the background, the Flow still delivers updates, the UI still processes them (even though nothing is visible), and battery is wasted. `repeatOnLifecycle(State.STARTED)` creates a coroutine that automatically suspends its block when the lifecycle drops below `STARTED` (app backgrounds) and resumes when it returns to `STARTED`. This means Flow collection pauses when the user is not looking at the screen and resumes when they return — behavior analogous to how `LiveData` worked with automatic lifecycle awareness. It was introduced because `lifecycleScope.launch { }` doesn't pause collection, leading to unnecessary resource usage. In Compose, `collectAsStateWithLifecycle()` provides the same behavior.

---

**Q5: How would you share a ViewModel between two Fragments?**

> Three approaches depending on the scope needed. First, Activity-scoped (`activityViewModels()`): both Fragments in the same Activity get the same ViewModel instance. The ViewModel lives as long as the Activity. Use this when the data needs to outlive both Fragments. Second, NavGraph-scoped: using `navGraphViewModels(R.id.my_nav_graph)` or `hiltViewModel(backStackEntry)` in Compose, the ViewModel is scoped to the NavGraph and cleared when the graph is popped. Best for multi-step flows (checkout, onboarding) where Fragments need shared state within the flow but the state should be cleared when the flow ends. Third, NavBackStackEntry of a parent destination: a parent Fragment or composable owns the ViewModel, and child destinations retrieve it via the parent's BackStackEntry. I default to the narrowest scope that satisfies the use case — NavGraph scope over Activity scope when possible.

---

**Q6: What cleanup should you do in `ViewModel.onCleared()`?**

> `onCleared()` is called when the ViewModel is definitively going away — the user pressed Back, the navigation destination was popped, or the Activity finished. Any resources the ViewModel acquired must be released here. Specifically: `ExoPlayer.release()` to free hardware codec and audio focus; custom `CoroutineScope.cancel()` for scopes created inside the ViewModel (not `viewModelScope` which is cancelled automatically by AndroidX); `AudioRecord.stop()` and `release()` for recording sessions; `BluetoothGatt.close()` for BLE connections; analytics session or logging context finalization. The most common oversight is a ViewModel that creates a custom `CoroutineScope` for dependency isolation — forgetting to cancel it means coroutines keep running after the ViewModel is supposed to be gone, holding references and leaking memory.

---

*Previous: [23 — Jetpack Compose Optimization](./23-compose-optimization.md)*
*Next: [25 — Notifications](./25-notifications.md)*
