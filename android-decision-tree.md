# Android Architecture & Library Decision Tree
> Staff-level reference. Read top-down per decision. Pick the simplest option that solves your problem — add complexity only when the problem demands it.

---

## Table of Contents
1. [Architecture Pattern Decision Tree](#1-architecture-pattern-decision-tree)
2. [Layer Strategy: When to Add Each Layer](#2-layer-strategy-when-to-add-each-layer)
3. [Modularization Decision Tree](#3-modularization-decision-tree)
4. [Networking Library Decision Tree](#4-networking-library-decision-tree)
5. [Local Storage Decision Tree](#5-local-storage-decision-tree)
6. [Dependency Injection Decision Tree](#6-dependency-injection-decision-tree)
7. [Image Loading Decision Tree](#7-image-loading-decision-tree)
8. [Async/Concurrency Decision Tree](#8-asyncconcurrency-decision-tree)
9. [Serialization Decision Tree](#9-serialization-decision-tree)
10. [Pagination Decision Tree](#10-pagination-decision-tree)
11. [Navigation Decision Tree](#11-navigation-decision-tree)
12. [Testing Decision Tree](#12-testing-decision-tree)
13. [App Type Reference Matrix](#13-app-type-reference-matrix)

---

## 1. Architecture Pattern Decision Tree

### The Core Question: What Problem Are You Solving?

```
Is this a production app that will grow over time?
│
├─ NO (prototype, hackathon, demo, solo weekend project)
│   └─ → MVVM with ViewModel + StateFlow only
│       No use cases. No repositories. Direct ViewModel → data source.
│       ADD COMPLEXITY LATER if/when it ships.
│
└─ YES → How complex is the UI state?
    │
    ├─ Simple (forms, settings, CRUD screens, info screens)
    │   └─ → MVVM
    │       Multiple StateFlows per screen are fine at this scale.
    │       Google recommends this as the baseline.
    │
    └─ Complex (checkout flows, chat, video player, multi-step wizards,
        real-time feeds, screens where many actions modify shared state)
        └─ → MVI on top of MVVM foundation
            Single UiState sealed class. Sealed Intent/Event class.
            Immutable state transitions only.
```

### MVVM vs MVI: The Detailed Decision

| Signal | Use MVVM | Use MVI |
|---|---|---|
| UI state variables | Few, independent | Many, interdependent |
| State combinations | Limited | Can produce impossible states |
| Team familiarity | New to reactive | Experienced, wants predictability |
| Debugging needs | Standard | Needs state replay / logging |
| Compose adoption | Mixed XML+Compose | Full Compose |
| App type | Dashboard, settings, profile | Checkout, chat, media player |

**The hybrid rule (staff-level answer):** MVVM for 80% of screens. MVI for the 20% where state complexity justifies the boilerplate. They coexist cleanly — just be consistent per screen.

### MVC — When to Use / Avoid

```
MVC
├─ USE: Legacy codebases maintaining existing patterns. Never greenfield.
├─ USE: Quick Activity-level scripts or internal tools with 1–2 screens.
└─ AVOID: Any production app with growth expectations.
   WHY: Activity becomes God class (View + Controller in one). Untestable.
```

### MVP — When to Use / Avoid

```
MVP
├─ USE: Legacy apps pre-ViewModel that are too large to rewrite.
├─ USE: Teams coming from iOS MVP who need a familiar mental model.
└─ AVOID: Any new Android project.
   WHY: Manual View-Presenter lifecycle binding → memory leaks.
        ViewModel + StateFlow solves this better with zero boilerplate.
```

### App Type → Architecture Map

| App Type | Primary Pattern | Why |
|---|---|---|
| **Social media feed** (Twitter/Instagram clone) | MVI + Clean | Complex feed state, interactions, optimistic updates |
| **E-commerce app** | MVI for cart/checkout, MVVM for browse | Cart state has many interdependencies |
| **News/content reader** | MVVM | Mostly display + pagination, low state complexity |
| **Banking/fintech** | MVI + Clean + strict modularization | Compliance, testability, auditability |
| **Chat/messaging** | MVI | Real-time state, connection status, typing indicators |
| **Media player** | MVI | Player state machine is complex (play/pause/buffering/error) |
| **Food delivery** | MVI for order flow, MVVM for browse | Order placement is a stateful wizard |
| **Fitness tracker** | MVVM + Coroutines + Flow | Sensor data, mostly display, low UI complexity |
| **Beer app / catalog** | MVVM | Pagination + detail = standard MVVM territory |
| **Settings/admin app** | MVVM | CRUD, forms — no complex state interactions |
| **Internal enterprise tool** | MVVM | Speed of delivery > architecture purity |
| **SDK/library** | No pattern — expose clean API | You don't own the app layer |

---

## 2. Layer Strategy: When to Add Each Layer

Clean Architecture ≠ always three layers. Add layers when you have a concrete reason.

### When to add a Domain Layer (Use Cases)

```
Do you have business logic that is shared across multiple ViewModels?
│
├─ YES → Domain layer with Use Cases
│
└─ NO → Does your repository logic need to be isolated from both
        the data source and the ViewModel for testing?
    │
    ├─ YES → Domain layer
    │
    └─ NO → Skip domain layer. Repository talks directly to ViewModel.
             Adding use cases for CRUD-passthrough is pure boilerplate.
```

**When Use Cases are worth it:**
- Combining data from two repositories (e.g. merge local + remote)
- Business validation (e.g. "can this user purchase?")
- Complex transformation before display
- Shared across 2+ ViewModels

**When Use Cases are waste:**
- `class GetUserUseCase { fun invoke() = repo.getUser() }` — this is a wrapper with zero value
- Single-ViewModel screens
- Simple CRUD apps

### When to add a Data Layer abstraction

Always. The repository pattern is always worth it for anything beyond a single-screen app. It decouples your ViewModel from knowing about Room vs Retrofit vs DataStore.

### Repository Pattern Rule

```
ViewModel → Repository Interface (domain layer)
                   ↑ implemented by
            RepositoryImpl (data layer) → [Room DAO | Retrofit API | DataStore]
```

The interface lives in domain. The impl lives in data. ViewModel never imports Room or Retrofit directly.

---

## 3. Modularization Decision Tree

```
How many engineers work on this codebase?
│
├─ 1–2 engineers, app < 30 screens
│   └─ → Single module (:app)
│       Multi-module adds Gradle complexity with no parallelism benefit.
│       Structure folders well: feature/home, feature/profile, core/network
│
├─ 3–5 engineers OR app > 30 screens OR 6+ month timeline
│   └─ → Feature modules + Core modules
│       :app → thin shell
│       :feature:home, :feature:profile, :feature:checkout
│       :core:network, :core:data, :core:ui, :core:domain
│
└─ 5+ engineers OR multiple teams OR large-scale app
    └─ → Full modularization + Dynamic Feature Modules
        Features that can be downloaded on demand → DFM
        Strict dependency graph: feature cannot depend on feature
        Features communicate via :core:domain contracts or navigation args
```

### Module Dependency Rules (never violate)

```
:feature:* → can import :core:*
:feature:* → CANNOT import other :feature:* (circular dep risk)
:core:domain → imports NOTHING (pure Kotlin only)
:core:data → imports :core:domain (implements interfaces)
:core:ui → imports :core:domain (for display models)
:app → imports all features (wires navigation graph)
```

### When to use Dynamic Feature Modules

- Feature is large (>2MB) and used by <30% of users (e.g., AR camera, admin tools)
- App needs to stay under install size budget (e.g., Lite variants)
- Feature is optional (onboarding flows, premium features)

**Avoid DFMs if:** team is small, feature is core to daily use, or you haven't shipped v1 yet.

---

## 4. Networking Library Decision Tree

```
Is your app Android-only now AND for the foreseeable future?
│
├─ YES → Retrofit + OkHttp
│   Industry standard. 73% of Android devs use it.
│   Massive ecosystem, battle-tested, annotation-based, easy to mock.
│
└─ NO (KMP planned, iOS/Desktop/Web sharing networking code)
    └─ → Ktor Client
        Kotlin-first. Multiplatform. Coroutines native.
        More boilerplate for complex APIs but portable.
```

### Retrofit vs Ktor: Full Comparison

| Dimension | Retrofit | Ktor Client |
|---|---|---|
| **Maturity** | Since 2013. Massive ecosystem. | Newer. JetBrains-backed. Growing. |
| **Setup** | Annotation-based, minimal code | DSL-based, more verbose for complex APIs |
| **Coroutines** | Supported via adapter | Native — built-in from day 1 |
| **Multiplatform** | Android-only | Android, iOS, JS, Desktop, Server |
| **Error handling** | OkHttp interceptors, well-documented | Custom plugins, more manual |
| **Interceptors** | Rich OkHttp interceptor ecosystem | Plugin system (powerful but younger) |
| **Mock in tests** | MockWebServer (Square) — excellent | MockEngine — good but less mature |
| **Bundle size** | ~3MB with OkHttp + converter | ~2MB |
| **Team familiarity** | Universally known | Requires learning |
| **When to use** | Any Android-only app | KMP shared module, multiplatform strategy |
| **When to avoid** | KMP codebase | Complex APIs with many interceptors |

### JSON Serialization (pairs with networking)

```
Retrofit/OkHttp → What serializer?
│
├─ kotlinx.serialization (RECOMMENDED for new projects)
│   Kotlin-first. Compile-time. KMP-compatible. No reflection.
│   Works with Ktor natively.
│
├─ Moshi
│   Kotlin-friendly. Codegen via KSP. Better than Gson for Kotlin.
│   No KMP support.
│
└─ Gson (AVOID for new projects)
    Java-based. Reflection-heavy. Doesn't understand Kotlin nullability.
    data class Foo(val x: String = "default") → Gson ignores defaults.
    Use only in legacy codebases.
```

---

## 5. Local Storage Decision Tree

```
What kind of data are you storing?
│
├─ Key-value pairs (user prefs, feature flags, session tokens, small config)
│   │
│   ├─ Simple, non-sensitive → DataStore (Preferences)
│   │   Kotlin-first, Flow-based, replaces SharedPreferences.
│   │   Atomic, handles concurrent writes.
│   │
│   ├─ Sensitive (auth tokens, PII) → DataStore + Tink (AES encryption)
│   │   NOTE: EncryptedSharedPreferences deprecated April 2025.
│   │   Replace with DataStore + Google Tink library.
│   │
│   └─ AVOID: SharedPreferences (no coroutines, blocking I/O, no migration)
│             EncryptedSharedPreferences (deprecated April 2025)
│
├─ Structured relational data (entities with relationships, queries, pagination)
│   │
│   ├─ Android-only app → Room
│   │   Jetpack-supported. KSP-based. Flow integration. Paging 3 support.
│   │   Type-safe compile-time query verification.
│   │   Works with RemoteMediator for offline-first patterns.
│   │
│   ├─ KMP (Android + iOS sharing DB layer) → SQLDelight
│   │   Pure Kotlin. SQL written in .sq files. Generates Kotlin classes.
│   │   Compile-time verified queries. No Room on iOS.
│   │   Tradeoff: you write raw SQL (no annotation magic).
│   │
│   └─ High-performance object graph / real-time sync → Realm
│       Object database (not relational). Fast for complex object graphs.
│       Tradeoff: different query model, heavier SDK, vendor dependency.
│       Use when: you need real-time sync (Realm Sync), mobile game state,
│       or object graph queries that are painful in SQL.
│
└─ Files / binary data (images, audio, documents)
    └─ Internal storage (getFilesDir) or MediaStore for media
       Pair with WorkManager for background writes
```

### Room vs SQLDelight vs Realm

| Dimension | Room | SQLDelight | Realm |
|---|---|---|---|
| **Paradigm** | SQL + annotation ORM | SQL-first, generates Kotlin | Object database |
| **KMP** | No (Android only) | Yes (Android + iOS + Desktop) | Yes (Realm SDK) |
| **Paging 3** | Native support | Manual integration | Not supported |
| **Compile-time checks** | Yes (KSP) | Yes (.sq files) | No |
| **Learning curve** | Low (Jetpack familiar) | Medium (raw SQL) | Medium (object model) |
| **Migrations** | Room Migration API | Manual SQL migrations | Automatic (Realm Sync) |
| **Real-time sync** | No | No | Yes (Realm Sync — paid) |
| **When to use** | Any Android-only offline-first app | KMP shared data layer | Real-time sync, complex object graphs |
| **When to avoid** | KMP app | Paging 3 (extra work) | Simple CRUD apps (overkill) |

### DataStore vs SharedPreferences

| | DataStore | SharedPreferences |
|---|---|---|
| Threading | Flow-based, safe for coroutines | Blocking, requires main thread care |
| Consistency | Atomic | Not atomic (can corrupt on crash) |
| Error handling | Exception propagation via Flow | Silent failures |
| Migration | Built-in SharedPreferences → DataStore | N/A |
| Type safety | Proto DataStore = fully typed | Preferences DS = stringly-typed |
| **Verdict** | Always prefer | Avoid in new code |

---

## 6. Dependency Injection Decision Tree

```
Does your project need compile-time safety and deep Jetpack integration?
│
├─ YES → Hilt (recommended for 99% of Android apps)
│   Built on Dagger. Compile-time verified.
│   Auto-integrates with ViewModel, WorkManager, Compose, Navigation.
│   30% less boilerplate than raw Dagger.
│   Slower build times due to annotation processing (use KSP, not KAPT).
│
├─ Build times are critical OR KMP is a goal → Koin
│   Runtime DI (not compile-time). No code generation.
│   DSL-based setup. Faster builds.
│   Tradeoff: DI errors found at RUNTIME not compile time.
│   First-class KMP support.
│   Good for: fast-moving startups, KMP projects, teams new to DI.
│
└─ Legacy / enterprise with complex custom DI → Dagger 2
    Maximum flexibility. Maximum boilerplate.
    Only if existing codebase uses it. Never greenfield.
```

### Hilt vs Koin: The Real Tradeoff

| Dimension | Hilt | Koin |
|---|---|---|
| **Safety** | Compile-time — crash-free DI guaranteed | Runtime — wrong dep = crash on launch |
| **Build time** | Slower (annotation processor runs) | Faster (no codegen) |
| **KMP** | Android-only | Yes |
| **Learning curve** | Medium (annotations, scopes) | Low (Kotlin DSL) |
| **ViewModel injection** | `@HiltViewModel` — one line | `viewModel { MyViewModel(get()) }` |
| **Jetpack integration** | Deep (SavedState, WorkManager, Nav) | Good (Compose extension) |
| **Team size** | Better for large teams (errors caught early) | Fine for small teams |
| **When to use** | Any Android-only app, FAANG-style team | KMP projects, small teams, fast iteration |
| **When to avoid** | KMP projects | Apps where DI correctness is safety-critical |

---

## 7. Image Loading Decision Tree

```
Is your app Compose-first?
│
├─ YES → Coil 3 (RECOMMENDED)
│   Kotlin-first. Coroutines-native. Compose-ready.
│   AsyncImage Composable. Fast. Actively maintained.
│   Coil 3 supports KMP.
│
├─ Mixed XML + Compose OR large media/GIF/video thumbnail focus → Glide
│   Google-backed. Mature. Best memory management.
│   20% better memory usage than Picasso.
│   GIF and video thumbnail support out of the box.
│   Glide Compose (landscapist-glide or accompanist-glide) for Compose.
│
└─ AVOID Picasso (new projects)
    Square's library. Lightest of the three.
    No built-in GIF support. Less maintained.
    Only if existing codebase uses it.
```

### Coil vs Glide: The Detail

| Dimension | Coil | Glide |
|---|---|---|
| **Language** | Kotlin-first | Java/Kotlin |
| **Coroutines** | Native | Adapters required |
| **Compose** | `AsyncImage {}` — native | Via third-party wrappers |
| **GIF support** | Plugin required | Built-in |
| **Video thumbnails** | Plugin | Built-in |
| **KMP** | Yes (Coil 3) | No |
| **Memory management** | Good | Best-in-class |
| **When to use** | Compose-first, KMP, new projects | Media-heavy apps, GIF, video thumbnails |
| **When to avoid** | Heavy GIF-only apps | KMP projects |

---

## 8. Async/Concurrency Decision Tree

```
New project in 2025?
│
└─ → Coroutines + Flow. Full stop.
    No decision needed. Kotlin Coroutines is the standard.
    RxJava is a legacy choice.
```

### RxJava — Only if:
- Existing codebase already uses it (migration cost > benefit)
- Third-party SDK forces it (rare in 2025)
- Team has deep RxJava expertise and no plans to grow

**Never use RxJava in a new project.** Coroutines handle every use case with less cognitive overhead.

### Flow Variant Decision

```
What are you modeling?
│
├─ Single value that changes over time (UI state, user, settings)
│   └─ StateFlow (always has a current value, replays last to new collectors)
│
├─ Events / one-time actions (navigation, snackbar, toast)
│   └─ SharedFlow with replay=0 (or Channel as backing)
│       NOT StateFlow — events must not replay on rotation.
│
├─ Sequence of values from a data source (DB, network stream)
│   └─ Flow (cold) — emits on collection, cancels on scope cancellation
│
└─ Paged data
    └─ Flow<PagingData<T>> via Paging 3
```

### Coroutine Scope Decision

```
Where does this work run?
│
├─ Tied to ViewModel lifecycle → viewModelScope
├─ Tied to App lifetime (repository, background sync) → custom CoroutineScope
│   with SupervisorJob() + Dispatchers.IO
├─ Tied to Composable lifecycle → rememberCoroutineScope()
├─ Fire-and-forget from a Service → lifecycleScope (LifecycleService)
└─ WorkManager tasks → CoroutineWorker (built-in scope)
```

---

## 9. Serialization Decision Tree

```
New project?
└─ kotlinx.serialization (RECOMMENDED)
   Compile-time. Reflection-free. KMP. Works with Ktor natively.
   Works with Retrofit via kotlinx-serialization-converter.

Existing Retrofit project, moderate complexity?
└─ Moshi with KSP codegen
   Kotlin-aware. Understands default params and null safety.
   Faster than Gson. No KMP.

Legacy / existing Gson-heavy codebase?
└─ Keep Gson — migration cost > benefit for working code.
   But document: Gson does NOT respect Kotlin default values.
   data class Foo(val x: String = "default") → Gson sets x = null if missing.
   Wrap nullable fields defensively.
```

### Comparison

| | kotlinx.serialization | Moshi | Gson |
|---|---|---|---|
| Kotlin null safety | Yes | Yes | No |
| Default params | Yes | Yes | No |
| KMP | Yes | No | No |
| Codegen | KSP | KSP | Reflection |
| Speed | Fast | Fast | Slow (reflection) |
| Retrofit adapter | 3rd party converter | Built-in | Built-in |

---

## 10. Pagination Decision Tree

```
Do you need to load large lists incrementally?
│
├─ NO (< 100 items, rarely changes) → Load all at once. No pagination needed.
│
└─ YES → How complex is your data source?
    │
    ├─ Network-only, no offline support needed
    │   └─ → Paging 3 with PagingSource
    │       PagingSource hits Retrofit directly.
    │       No Room involved.
    │       Use when: beer app browse, search results, public API lists.
    │
    └─ Offline-first, data must be cached locally
        └─ → Paging 3 with RemoteMediator + Room
            Room is SSOT. RemoteMediator fills Room from network.
            Use when: news feed, social feed, e-commerce catalog.
            Requires: remote keys table for page tracking.
```

### Paging 3 vs Manual Pagination

| Dimension | Paging 3 | Manual (page variable + loadMore) |
|---|---|---|
| LoadState | Built-in (Loading/Error/NotLoading) | DIY |
| Compose integration | `collectAsLazyPagingItems()` | DIY |
| Retry logic | `retry()` built-in | DIY |
| Room integration | Native (PagingSource from @Query) | DIY |
| Refresh on data change | Automatic (Room invalidates) | DIY |
| Complexity | High initial setup | Low initial, high maintenance |
| **When to choose** | Any production paginated list | Prototypes, < 3 pages total |

**When to avoid Paging 3:** fewer than 3 pages, data never changes after load, content is fully static (e.g., FAQ list). Manual loading is cleaner then.

---

## 11. Navigation Decision Tree

```
Is this a new project with Compose?
│
├─ YES → Navigation Compose (Nav 2.8+ with type-safe routes)
│   No string routes. Kotlin @Serializable data classes as destinations.
│   Single-Activity. Nested graphs supported.
│
├─ Mixed XML + Compose → Navigation Component (NavController)
│   SafeArgs plugin for type-safe fragment arguments.
│   Fragments as destinations, Compose can be embedded.
│
└─ Multiple independent feature modules with complex back stacks
    └─ → Consider Decompose or custom NavigationManager
        Decompose: lifecycle-aware component trees. Cross-platform.
        Use when Nav Compose's single back stack model is insufficient
        (e.g., bottom nav with independent back stacks per tab).
```

### Navigation Back Stack: Bottom Nav Pattern

```
Does each bottom nav tab need its own independent back stack?
│
├─ YES → Use rememberSaveable NavHostController per tab
│   OR use Decompose for true independent stacks
│
└─ NO (simple apps) → Single NavHost with saveState/restoreState
    navController.navigate(route) {
        popUpTo(navController.graph.findStartDestination().id) { saveState = true }
        launchSingleTop = true
        restoreState = true
    }
```

---

## 12. Testing Decision Tree

### The Pyramid Rule

```
70% unit tests (fast, JVM, no device)
20% integration tests (Room in-memory, MockWebServer, Robolectric)
10% UI/E2E tests (device/emulator, slow, expensive)
```

Do NOT invert this pyramid. UI tests are slow, flaky, and expensive to maintain.

### Unit Testing

```
What do you need to test?
│
├─ Pure Kotlin logic (Use Cases, Repositories, Mappers, domain logic)
│   └─ JUnit4 + MockK
│       JUnit4 is universal. MockK is Kotlin-native mocking.
│       WHY MockK over Mockito? Understands Kotlin: mocks object classes,
│       extension functions, coroutines, companion objects natively.
│
├─ Coroutines + Flow
│   └─ JUnit4 + MockK + kotlinx-coroutines-test + Turbine
│       runTest {} for suspending functions.
│       Turbine for Flow: turbine.test { assertItem() }.
│       StandardTestDispatcher vs UnconfinedTestDispatcher:
│       - Standard: you control when coroutines run (advanceUntilIdle)
│       - Unconfined: runs eagerly (simpler for most tests)
│
└─ ViewModel
    └─ JUnit4 + MockK + kotlinx-coroutines-test
        Use TestDispatcher to control timing.
        Inject dispatcher via constructor for testability.
        Never test ViewModels with real Dispatchers.IO — it's slow and flaky.
```

### Integration Testing

```
What are you integrating?
│
├─ Database (Room queries, DAOs, migrations)
│   └─ JUnit4 + Room in-memory database + AndroidJUnit4
│       Room.inMemoryDatabaseBuilder() — no disk, fast, isolated.
│       Run as instrumented test (androidTest sourceset).
│       Test actual SQL, not mocked DAOs.
│
├─ Network layer (Retrofit, OkHttp)
│   └─ JUnit4 + MockWebServer (OkHttp/Square)
│       Spin up a local HTTP server in tests.
│       Queue mock responses. Assert request shape.
│       Runs on JVM — no device needed.
│
└─ Full slice (ViewModel + Repository + Room, no network)
    └─ Hilt testing (HiltAndroidTest) + in-memory Room
        Replace real modules with test modules via @TestInstallIn.
```

### UI Testing

```
Is your UI built with Compose?
│
├─ YES → Compose Testing API
│   composeTestRule.setContent { MyScreen() }
│   onNodeWithTag("btn").performClick()
│   onNodeWithText("Hello").assertIsDisplayed()
│   Runs on JVM via Robolectric (fast) or on device (slow but accurate).
│
└─ NO (Views/XML) → Espresso
    onView(withId(R.id.button)).perform(click())
    Runs on device or emulator. Slower than Compose testing on JVM.
    Use Robolectric to run Espresso tests locally without a device.
```

### Screenshot / Visual Regression Testing

```
Do you need to catch visual regressions (UI looks changed unexpectedly)?
│
├─ Compose-only, want JVM speed → Paparazzi (Square)
│   Renders via LayoutLib (same engine as Android Studio Previews).
│   No device needed. Fast. Good for design system components.
│   Tradeoff: renders like a Preview, not exactly like a real device.
│
├─ Compose-only, want Robolectric accuracy → Roborazzi
│   Robolectric under the hood. More realistic than Paparazzi.
│   Better for interaction testing (click → screenshot).
│   Tradeoff: slower than Paparazzi.
│
├─ Google's official tool (alpha, newest) → Compose Preview Screenshot Testing
│   Snapshots your @Preview functions directly.
│   Simplest setup but least mature (alpha as of Jan 2025).
│
└─ Need real device screenshots (system UI, edge-to-edge, notifications)
    └─ Dropshots (instrumented, on device)
```

### E2E / Acceptance Testing

```
Need to test full user journeys across screens?
└─ UI Automator (system-level, can interact with other apps)
   OR Espresso for app-internal flows
   OR Appium (cross-platform, QA team tool)

WARNING: Keep E2E tests small. ~5% of all tests.
Run only in CI on merge to main, not on every PR.
They are slow, flaky, and expensive to maintain.
```

### Mocking Framework Decision

```
New project / Kotlin codebase?
└─ MockK (RECOMMENDED)
   Kotlin-native: mocks objects, companion objects, extension functions.
   Coroutine-aware: coEvery {} for suspending functions.
   Spyk for partial mocks.

Java legacy codebase / team coming from Java?
└─ Mockito (+ mockito-kotlin for coroutine support)
   Mature. Huge ecosystem. But not Kotlin-native.
   Cannot mock Kotlin objects, final classes without extra setup.

NEVER: PowerMock
Static method mocking is a code smell. Refactor to be injectable instead.
```

### Fake vs Mock Decision

```
Can you create a Fake (real implementation with in-memory behavior)?
│
├─ YES → Prefer Fakes over Mocks
│   Fakes are real implementations. They're more reliable,
│   test what users actually experience, and are less brittle.
│   Example: FakeUserRepository(mutableListOf()) vs mockk<UserRepository>()
│
└─ NO (third-party class, Android SDK class, too complex to fake)
    └─ Use MockK
```

**Rule:** Fakes for repositories and data sources. Mocks for external dependencies you don't own.

### Testing: When to Avoid

| Library / Approach | When to Avoid |
|---|---|
| Espresso on every PR | Too slow. Gate only major flows in CI |
| Mockito in new Kotlin projects | MockK is strictly better |
| PowerMock | Always — refactor the code instead |
| Mocking DAOs | Test real Room with in-memory DB instead |
| UI tests for business logic | Unit test the ViewModel/UseCase instead |
| Robolectric for pure logic | Overkill — pure JUnit4 is enough |
| Screenshot tests in a pre-v1 app | UI changes constantly; too much maintenance |
| E2E tests on every commit | Too flaky. Run nightly or on merge only |

---

## 13. App Type Reference Matrix

> Quick lookup: given an app type, what's the full stack?

### Social Media App (Instagram/Twitter clone)
```
Architecture: MVI + Clean Architecture + Feature Modules
Network:      Retrofit + OkHttp + kotlinx.serialization
Local DB:     Room + RemoteMediator (offline-first feed)
DI:           Hilt
Images:       Coil (AsyncImage in Compose feed)
Pagination:   Paging 3 + RemoteMediator + Room
Async:        Coroutines + StateFlow + SharedFlow (events)
Nav:          Navigation Compose (type-safe routes)
Testing:      JUnit4 + MockK + Turbine + Compose Testing + Paparazzi
```

### E-Commerce App
```
Architecture: MVI for cart/checkout, MVVM for browse, Clean Architecture
Network:      Retrofit + OkHttp
Local DB:     Room (cart persistence) + DataStore (session/prefs)
DI:           Hilt
Images:       Coil
Pagination:   Paging 3 for product catalog
Async:        Coroutines + StateFlow
Nav:          Navigation Compose
Testing:      JUnit4 + MockK + Turbine + Room in-memory + MockWebServer
```

### Banking / Fintech App
```
Architecture: MVI + Clean Architecture + strict modularization
              Feature modules with clear security boundaries
Network:      Retrofit + OkHttp (custom CertificatePinner, no cache)
Local DB:     DataStore + Tink (encrypted). Room for transaction history.
DI:           Hilt (compile-time safety critical)
Images:       Coil (minimal use)
Async:        Coroutines + Flow
Nav:          Navigation Compose (single back stack, clear flows)
Security:     TLS pinning, BiometricPrompt, ProGuard/R8 full obfuscation
Testing:      JUnit4 + MockK + in-memory Room + Hilt testing
              UI Automator for biometric flow E2E
```

### News / Content Reader
```
Architecture: MVVM + Clean Architecture (domain layer worth it for article parsing)
Network:      Retrofit
Local DB:     Room + Paging 3 RemoteMediator (offline reading)
DI:           Hilt
Images:       Coil
Pagination:   Paging 3
Async:        Coroutines + StateFlow
Testing:      JUnit4 + MockK + Turbine + Paparazzi (article card UI)
```

### Chat / Messaging App
```
Architecture: MVI (connection state machine is complex)
Network:      Retrofit (REST for history) + WebSocket (OkHttp) for real-time
Local DB:     Room (message cache, offline support)
DI:           Hilt
Images:       Coil (media messages)
Async:        Coroutines + Flow (WebSocket as Flow<Message>)
Nav:          Navigation Compose
Testing:      JUnit4 + MockK + Turbine (WebSocket Flow testing)
```

### Fitness Tracker
```
Architecture: MVVM + Repository (no domain layer needed — simple CRUD + sensors)
Local DB:     Room (workout history) + DataStore (user prefs)
DI:           Hilt
Network:      Retrofit (optional sync to backend)
Async:        Coroutines + StateFlow + SensorManager callbacks → Flow
Background:   WorkManager (daily summary, sync)
Testing:      JUnit4 + MockK + Room in-memory
```

### KMP App (Android + iOS)
```
Architecture: MVVM or MVI in shared module
Network:      Ktor Client (shared networking)
Local DB:     SQLDelight (shared DB layer)
DI:           Koin (KMP support)
Images:       Coil 3 (KMP support in Compose Multiplatform)
Serialization: kotlinx.serialization
Testing:      kotlin.test (common), MockK-Android + Mockative (shared)
              Turbine for Flow testing
```

### Beer App / Simple Catalog (your app)
```
Architecture: MVVM + Repository (domain layer optional unless growing)
Network:      Retrofit + OkHttp
Local DB:     Room + Paging 3 (RemoteMediator for offline-first)
DI:           Hilt
Images:       Coil
Serialization: Moshi or kotlinx.serialization
Async:        Coroutines + StateFlow
Nav:          Navigation Compose
Testing:      JUnit4 + MockK + Turbine + Room in-memory + MockWebServer
```

### Internal Enterprise Tool
```
Architecture: MVVM (ship fast, iterate)
Network:      Retrofit
Local DB:     Room or DataStore (minimal local state)
DI:           Hilt
Images:       Coil
Testing:      JUnit4 + MockK (unit tests only — UI rarely tested in internal tools)
```

---

## Quick Reference: Staff-Level Decision Rules

1. **Architecture:** Start with MVVM. Reach for MVI only when state complexity demands it. Add Clean Architecture layers only when you have a concrete reason (shared business logic, swappable data sources, multiple teams).

2. **Libraries:** Choose the option your team understands over the theoretically superior option. A misunderstood Hilt is worse than a well-understood Koin.

3. **Modularization:** Don't modularize before you ship. Modularize when build times hurt or team parallelism is blocked.

4. **DI:** Hilt for Android-only. Koin for KMP or fast-moving teams. Never Dagger from scratch.

5. **Networking:** Retrofit unless KMP is a real near-term goal. Then Ktor.

6. **DB:** Room unless KMP. Then SQLDelight.

7. **Images:** Coil for new Compose projects. Glide for media-heavy or GIF-heavy apps.

8. **Serialization:** kotlinx.serialization for new projects. Moshi as a solid second. Avoid Gson.

9. **Testing:** Mock what you don't own. Fake what you do. Never mock Room — use in-memory.

10. **The override rule:** Any of the above can be wrong for your specific situation. Know why you're deviating, document it, and be ready to explain the tradeoff.
