# 01 — Architecture Patterns

> **Round type: Both (HLD + LLD)**
> **HLD:** How to structure a large app — layers, module graph, team ownership boundaries
> **LLD:** Pattern implementation, state modeling, effect handling, ViewModel design

---

## 1. Why This Matters in Interviews

**Common interview angles:**
- "Walk me through the architecture of your most complex Android project"
- "How would you structure a large social media app from scratch?"
- "What's the difference between MVVM and MVI?"
- "Why is the domain layer pure Kotlin?"
- "How do feature modules communicate without depending on each other?"
- "When would you use a Use Case vs calling a repository directly?"
- "How do you model loading/error/success states?"
- "What's the right way to handle one-time events like navigation?"

An interviewer asking about architecture is really asking:
- Can you reason about **separation of concerns** under scale?
- Do you understand **testability** and why it matters?
- Can you **justify tradeoffs** — boilerplate vs safety, speed vs structure?
- Do you know what your team's codebase would look like in 2 years?

Always frame your answer around **the problem being solved**, not just pattern names.

---

## 2. The Landscape — Quick Map

```
Presentation Patterns (how UI ↔ state)   Structural Patterns (how layers relate)
─────────────────────────────────────    ─────────────────────────────────────
MVC  →  MVP  →  MVVM  →  MVI            Clean Architecture  +  Modularization
(older)          (most common)  (modern)
```

Presentation patterns and Clean Architecture are **orthogonal** — you can use MVI inside a Clean Architecture, or MVVM without it. They solve different problems.

---

## 3. Presentation Patterns

### 3.1 MVC — Model View Controller

**What it is:** The oldest split. The Controller mediates between Model (data) and View (UI). In Android, Activity/Fragment traditionally acted as both View and Controller — this is why it failed badly.

```
User Input → Controller → Model → View
                 ↑____________________↓  (View can also update Controller directly)
```

**Android reality:** Activities became God classes — network calls, DB queries, click handlers, UI manipulation all in one file. Untestable.

**Verdict:** Dead in modern Android. Understand it only for historical context.

---

### 3.2 MVP — Model View Presenter

**What it is:** Extracts presentation logic into a Presenter. The View becomes a passive interface. The Presenter holds no Android framework references (in theory), making it testable.

```
View (interface) ←→ Presenter ←→ Model/Repository
```

**Key rule:** View only calls Presenter methods. Presenter only calls View interface methods.

**Problems:**
- View and Presenter have a 1:1 tight coupling
- Presenter still grows large with complex screens
- Manual lifecycle management (need to detach view on destroy to avoid leaks)
- Not Compose-friendly (Compose doesn't have a "passive view" abstraction)

**When you see it:** Legacy Android codebases (pre-2019), some iOS codebases still use it with UIViewController as the View.

---

### 3.3 MVVM — Model View ViewModel ⭐ Google's recommended pattern

**What it is:** The ViewModel holds UI state and exposes it as observable streams. The View observes and re-renders. The ViewModel survives configuration changes (screen rotation). Data flows down, events flow up.

```
View (Composable/Fragment)
  ↓ user events (onClick, etc.)
ViewModel (survives rotation, holds StateFlow/LiveData)
  ↓ calls
Repository (decides: cache or network)
  ↓ calls
Remote DataSource / Local DataSource
```

**Android (Kotlin + Compose):**
```kotlin
// ViewModel
class UserViewModel(private val repo: UserRepository) : ViewModel() {
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(id: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            val result = repo.getUser(id)
            _uiState.update { it.copy(isLoading = false, user = result) }
        }
    }
}

// Composable
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    // render state
}
```

**React Native equivalent:**
- MVVM maps loosely to: Zustand store (ViewModel) + React component (View)
- `useStore()` hook collects state just like `collectAsStateWithLifecycle()`

**Pros:**
- ViewModel survives config changes — no data reload on rotation
- Testable: ViewModel has no UI dependency
- Google's AAC ViewModel is battle-tested
- Natural fit for Compose's reactive model

**Cons:**
- State can be mutated from multiple places → state management becomes complex on large screens
- Two-way data binding (XML era) created implicit bugs — avoid it
- No strict rule about where state lives → inconsistency across team

**When to use:** Default choice for most Android apps. Works for simple to medium complexity screens.

---

### 3.4 MVI — Model View Intent ⭐ Modern, preferred for complex state

**What it is:** A strict unidirectional data flow pattern. All user actions are modeled as **Intents** (not Android Intents — events/actions). The ViewModel reduces Intent + current State → new State. The View is a pure function of State.

```
View ──(Intent/Action)──→ ViewModel
  ↑                          │
  └──(new State)─────────────┘
         (unidirectional, no cycles)
```

**Three concepts:**
- **State**: Immutable data class representing the full screen state at a point in time. Single source of truth.
- **Intent/Action**: Sealed class of everything a user (or system) can do on that screen.
- **Side Effect**: One-time events that shouldn't be in state — navigation, snackbar, analytics. Usually a `Channel<Effect>` or `SharedFlow`.

**Android (Kotlin + Compose):**
```kotlin
// State
data class LoginState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)

// Intent
sealed class LoginIntent {
    data class EmailChanged(val email: String) : LoginIntent()
    data class PasswordChanged(val pass: String) : LoginIntent()
    object LoginClicked : LoginIntent()
}

// Side effect
sealed class LoginEffect {
    object NavigateToHome : LoginEffect()
    data class ShowError(val message: String) : LoginEffect()
}

// ViewModel
class LoginViewModel : ViewModel() {
    private val _state = MutableStateFlow(LoginState())
    val state = _state.asStateFlow()

    private val _effect = Channel<LoginEffect>()
    val effect = _effect.receiveAsFlow()

    fun handleIntent(intent: LoginIntent) {
        when (intent) {
            is LoginIntent.EmailChanged -> _state.update { it.copy(email = intent.email) }
            is LoginIntent.PasswordChanged -> _state.update { it.copy(password = intent.pass) }
            is LoginIntent.LoginClicked -> login()
        }
    }

    private fun login() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            // ... call repo
            _effect.send(LoginEffect.NavigateToHome)
        }
    }
}
```

**Why Channel for effects, not StateFlow?**  
StateFlow holds the last value — if you navigate away and come back, the navigation fires again. Channel delivers once and is consumed. `SharedFlow(replay=0)` also works.

**React Native equivalent:**
- Redux (reducer = state machine, actions = intents) is essentially MVI
- Redux Toolkit with `createSlice` is the modern version
- Zustand can be used MVI-style with explicit action functions

**Pros:**
- State is always predictable — given the same intents, you always get the same state
- Trivially reproducible bugs — record intent stream, replay it
- Natural fit for Compose (Composable is a function of state)
- Easy to time-travel debug
- Handles complex multi-step screens well (checkout flows, wizards)

**Cons:**
- More boilerplate than MVVM (sealed classes, reducers)
- Overkill for simple screens (a settings toggle doesn't need a sealed intent)
- Learning curve for teams new to reactive patterns
- Effect handling has multiple valid approaches — requires team consensus

**When to use:**
- Complex screens with many interdependent state fields
- Flows requiring deterministic state replay (payment, onboarding)
- Teams already using Redux in RN and moving to Android

---

### 3.5 MVVM vs MVI — Key Differences

| | MVVM | MVI |
|---|---|---|
| State | Multiple `StateFlow`s or one data class | Single immutable state per screen |
| Events | Functions called on ViewModel | Sealed Intent class |
| Side effects | Ad hoc (LiveData events, channels) | Explicit sealed Effect class |
| Predictability | Moderate | High |
| Boilerplate | Low–Medium | Medium–High |
| Best for | Most screens | Complex state, financial/wizard flows |
| Debugging | Standard | Reproducible via intent replay |

**Interview tip:** Don't say "MVI is always better." Say "I default to MVVM and reach for MVI when state complexity justifies it — for example, a multi-step checkout flow where any field change can invalidate other fields."

---

## 4. Clean Architecture

### 4.1 What It Is

Proposed by Robert C. Martin (Uncle Bob). Organizes code into **concentric layers** with one strict rule: **dependencies point inward only**. Inner layers know nothing about outer layers.

```
┌─────────────────────────────────────────┐
│  Frameworks & Drivers (Android, Room,   │
│  Retrofit, Compose, FCM)                │
│  ┌───────────────────────────────────┐  │
│  │  Interface Adapters               │  │
│  │  (ViewModels, Repositories impl,  │  │
│  │   Mappers, DTOs)                  │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  Use Cases / Interactors   │  │  │
│  │  │  (Application logic)        │  │  │
│  │  │  ┌───────────────────────┐  │  │  │
│  │  │  │  Entities             │  │  │  │
│  │  │  │  (Business rules,     │  │  │  │
│  │  │  │   pure Kotlin)        │  │  │  │
│  │  │  └───────────────────────┘  │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 4.2 Three Layers in Practice (Android)

**Domain Layer** — Pure Kotlin. Zero Android imports. Most stable layer.
```kotlin
// Entity
data class User(val id: String, val name: String, val email: String)

// Repository Interface (defined here, implemented in data layer)
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
}

// Use Case
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(id: String): Result<User> = repository.getUser(id)
}
```

**Data Layer** — Implements repository interfaces. Knows about Room, Retrofit, DataStore.
```kotlin
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao,
    private val mapper: UserMapper
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> {
        return try {
            val dto = api.getUser(id)
            Result.success(mapper.toDomain(dto))
        } catch (e: Exception) {
            // fallback to cache
            val cached = dao.getUser(id)
            if (cached != null) Result.success(mapper.toDomain(cached))
            else Result.failure(e)
        }
    }
}
```

**Presentation Layer** — ViewModels, Composables. Knows about domain, not data.
```kotlin
class UserViewModel(private val getUser: GetUserUseCase) : ViewModel() {
    // ViewModel calls use case, never repository directly
}
```

### 4.3 The Mapper Pattern

Each layer has its own data model:
- **Network DTO** (Retrofit response) → lives in data layer
- **DB Entity** (Room entity) → lives in data layer  
- **Domain Model** (pure Kotlin) → lives in domain layer
- **UI Model** (formatted strings, display logic) → lives in presentation layer

Mappers convert between them at layer boundaries.

**Why?** Prevents Android DB schema changes from leaking into UI code. Prevents API response shape from dictating domain logic.

### 4.4 When NOT to use Clean Architecture

Clean Architecture adds indirection. For a small app with 3 screens and 1 engineer, this is wasteful. Consider it when:
- 3+ engineers working in parallel need clear module boundaries
- Domain logic is complex and needs isolated unit testing
- App will grow significantly over time
- You need to swap data sources (e.g., test doubles, feature flags)

---

## 5. Modularization

### 5.1 What It Is

Splitting a monolithic app module into multiple Gradle modules. Each module compiles independently.

### 5.2 Module Types

```
:app                    → thin shell, navigation wiring, DI graph root
:feature:home           → home screen feature (UI + ViewModel)
:feature:profile        → profile feature
:feature:checkout       → checkout feature (can be dynamic)
:core:ui                → shared composables, theme, design system
:core:network           → Retrofit setup, OkHttp, interceptors
:core:data              → Room DB, DataStore, repositories
:core:domain            → Use cases, entities, repository interfaces
:core:testing           → shared test utilities, fakes
```

### 5.3 Dependency Rules for Modules

```
:feature:* → can depend on :core:*
:feature:* → CANNOT depend on other :feature:*
:core:data → can depend on :core:domain
:core:domain → depends on NOTHING (pure Kotlin)
:app → depends on all features (wires navigation)
```

**Why can't features depend on each other?** Circular dependencies. Features communicate through shared :core:domain models or navigation contracts.

### 5.4 Build Time Benefits

In a monolith, changing one file recompiles everything. With modules, Gradle only recompiles changed modules and their dependents. For large teams, this cuts incremental build time significantly.

### 5.5 Dynamic Feature Modules

On top of regular modules, Android supports **dynamic feature modules** — features downloaded on demand after install via Play Feature Delivery.

```xml
<!-- In feature module manifest -->
<dist:module dist:instant="false">
    <dist:delivery>
        <dist:on-demand/>   <!-- downloaded when requested -->
    </dist:delivery>
</dist:module>
```

**Delivery types:**
- `install-time` — bundled at install, always available
- `on-demand` — user triggers download (e.g., AR feature in a shopping app)
- `conditional` — auto-installed if device meets conditions (API level, country, RAM)

**Tradeoffs:**
| | Regular Module | Dynamic Feature Module |
|---|---|---|
| Available at launch | ✅ Always | ❌ Must be downloaded first |
| Reduces install size | ❌ No | ✅ Yes |
| Complexity | Low | High (split APK, SplitInstallManager) |
| Navigation | Standard | Reflection-based or DFM nav |
| Use case | Core features | Large rarely-used features (AR, video editor) |

**In dynamic features, the app module doesn't know about the feature module** (reversed dependency). Communication happens via reflection or shared API interfaces in a `:feature:xyz:api` module.

### 5.6 React Native Equivalent

RN doesn't have Gradle module system. Equivalent concepts:
- **Feature folders** — `/src/features/checkout/` with co-located components, hooks, store slices
- **Metro bundler code splitting** — not natively supported but achievable with lazy imports
- **Hermes + lazy loading** — import heavy screens lazily to improve startup
- **Monorepo with workspaces** — `yarn workspaces` to share code across RN + web

---

## 6. HLD vs LLD Framing

### HLD (High-Level Design) Questions
- "How would you structure a large social media app?"
- "How do you share code across features without coupling?"
- "How do you handle navigation in a multi-module app?"

**Your HLD answer should cover:**
1. Layer breakdown (presentation / domain / data)
2. Module graph (which modules exist, what depends on what)
3. Navigation strategy (how screens are wired)
4. DI strategy (Hilt in :app, scoped to feature)

### LLD (Low-Level Design) Questions
- "Design the ViewModel for a paginated feed screen"
- "How does your Repository decide between cache and network?"
- "How do you model loading/error/success states?"

**Your LLD answer should cover:**
1. State data class design
2. Data flow through layers (coroutine + StateFlow)
3. Error handling strategy (Result<T>, sealed class)
4. Testing approach (fake repository, ViewModel unit test)

---

## 7. Common Mistakes / Gotchas

- **ViewModel calling Android Context** — violates testability; use ApplicationContext via Hilt if unavoidable, or move logic to repository
- **Putting business logic in Composables** — Composables should only render state; logic belongs in ViewModel
- **Sharing state via global singletons** — use scoped ViewModels (shared between screens in same nav graph) instead
- **Domain layer importing Android** — if you see `import android.*` in your domain module, it's wrong
- **Mapper overkill for small apps** — 3 identical data classes for one object is over-engineering if the app has 2 screens
- **MVVM two-way binding in Compose** — Compose is already reactive; data binding XML is not needed and shouldn't be imported
- **Using LiveData in new code** — prefer StateFlow; LiveData is lifecycle-aware but StateFlow works better in Compose and doesn't require Activity/Fragment context
- **Effect handling via StateFlow** — navigation or snackbar in StateFlow causes double-trigger on recomposition; use Channel or SharedFlow(replay=0)

---

## 8. Android vs React Native Comparison

| Concept | Android (Kotlin) | React Native |
|---|---|---|
| Architecture pattern | MVVM / MVI | Container-Component / Redux / Zustand |
| ViewModel equivalent | `androidx.lifecycle.ViewModel` | Zustand store / Redux slice |
| State observable | `StateFlow` / `LiveData` | `useSelector` / `useStore` / `useState` |
| Side effects | `Channel<Effect>` | Redux middleware / `useEffect` |
| DI | Hilt / Koin | React Context / custom providers |
| Layers | domain / data / presentation modules | features / services / store folders |
| Navigation | Jetpack Navigation | React Navigation v6/v7 |
| Build modularity | Gradle modules | Feature folders / yarn workspaces |

---

## 9. Best Practices

- **Default to MVVM, escalate to MVI** — don't start with MVI for every screen; add it when state complexity justifies the boilerplate
- **Single Activity architecture** — one Activity, all screens as Composables managed by Jetpack Navigation; avoids lifecycle and back-stack complexity of multi-Activity
- **ViewModel never holds View references** — no Activity, Fragment, or Composable reference in ViewModel; pass callbacks, not contexts
- **State flows downward, events flow upward** — Composable receives state and emits events/intents; it never mutates state directly
- **One UiState data class per screen** — combine related fields into a single state class; avoids partial state inconsistency when two StateFlows update independently
- **Side effects via Channel, not StateFlow** — navigation and snackbar are one-time events; StateFlow replays its last value on new collectors which re-triggers them
- **Domain layer = zero Android imports** — enforce with a lint rule or module-level Gradle dependency constraint; it erodes gradually if left unchecked
- **Use Cases only when they add value** — pure passthrough Use Cases are dead weight; they earn their place when combining repos, applying business rules, or reused across ViewModels
- **Feature modules cannot depend on other feature modules** — all cross-feature sharing goes through `:core:*` modules or navigation contracts
- **Test ViewModels with fake repositories** — inject `FakeUserRepository` that returns test doubles; ViewModel tests run in milliseconds with no Android framework dependency

---

## 10. Interview Q&A

**Q1: What's the difference between MVVM and MVI?**

> MVVM uses observable state (StateFlow/LiveData) that the View binds to, and the View calls ViewModel functions directly. State can be updated from multiple places. MVI enforces a single state object and all changes flow through an explicit Intent → reduce cycle. MVI is more predictable and debuggable but adds boilerplate. I use MVVM as default and MVI when screen state has many interdependent fields or I need reproducible state for debugging — think a checkout flow or a media player controller.

---

**Q2: Why is the domain layer pure Kotlin with no Android imports?**

> The domain layer contains your business logic — the most valuable and stable part of the app. If it imports Android classes, you can't run unit tests without an emulator or Robolectric. Keeping it pure Kotlin means you can test use cases with plain JUnit in milliseconds. It also means the same domain layer can theoretically power an Android app, a backend service, or a KMP shared module without changes.

---

**Q3: When would you NOT use Clean Architecture?**

> When it adds more overhead than value. A 2-person team building an MVP with 4 screens doesn't need 4 Gradle modules and mappers. The overhead of maintaining layer boundaries, mapper classes, and use cases slows iteration speed. I'd use a pragmatic MVVM structure — ViewModel + Repository + data classes — and refactor toward Clean Architecture if the app grows or if multiple engineers need clear ownership boundaries.

---

**Q4: How do feature modules communicate with each other if they can't depend on each other?**

> Three approaches: (1) **Navigation contracts** — features expose a route string and the :app module wires them; (2) **API module pattern** — a `:feature:xyz:api` module exposes only interfaces and data models, which other modules can depend on without taking the full implementation; (3) **Shared domain models** — both features depend on the same `:core:domain` entities, which is usually enough. You avoid importing a feature's ViewModel or Composable from another feature.

---

**Q5: How do you handle navigation in a multi-module app?**

> Jetpack Navigation with a single NavHost in the :app module. Each feature module exposes a `NavGraphBuilder` extension function that registers its routes. The app module calls `addFeatureGraph()` which the feature registers without the app knowing the feature's internals. The feature gets access to a shared `NavController` via CompositionLocal. For deep links, each feature registers URI patterns in its own nav graph and the manifest. For dynamic feature modules, you use `DynamicNavHostFragment` or the Navigation Dynamic Features artifact.

---

**Q6: What is a Use Case and when is it not needed?**

> A Use Case (or Interactor) encapsulates a single application operation — "get user profile", "submit order", "toggle favorite". It sits in the domain layer and orchestrates repositories. The ViewModel calls use cases, not repositories directly. When is it not needed? When the use case is a pure passthrough — `class GetUserUseCase { fun invoke(id) = repo.getUser(id) }` adds zero value and is just boilerplate. In that case, calling the repository from the ViewModel is fine. Use cases earn their place when they: combine multiple repositories, apply business rules (e.g., validate, transform), or need to be reused across multiple ViewModels.

---

**Q7: How is state modeled for loading / error / success?**

> Sealed class approach:
> ```kotlin
> sealed class UiState<out T> {
>     object Loading : UiState<Nothing>()
>     data class Success<T>(val data: T) : UiState<T>()
>     data class Error(val message: String) : UiState<Nothing>()
> }
> ```
> Alternative: a flat data class with nullable fields:
> ```kotlin
> data class FeedState(
>     val items: List<Item> = emptyList(),
>     val isLoading: Boolean = false,
>     val error: String? = null
> )
> ```
> The flat data class wins in MVI because it allows partial updates (e.g., show items AND a loading spinner for pagination). The sealed class wins when states are mutually exclusive (initial load — you're either loading, showing data, or showing error, never both).

---

**Q8: What's the difference between `SharedFlow` and `Channel` for side effects?**

> Both deliver one-time events. `Channel` is a coroutine primitive — FIFO queue, suspends sender if buffer is full, consumed exactly once. `SharedFlow(replay=0)` is a hot stream — delivers to all current collectors, drops if no one is collecting. For UI effects (navigation, snackbar), `Channel` is safer because it guarantees delivery even if the UI isn't actively collecting when the event is emitted. If you use `SharedFlow` and the View is in the background, the effect is lost. The convention in Android is: state → `StateFlow`, effects → `Channel` received as `Flow` via `receiveAsFlow()`.

**Q9: How do you test a ViewModel?**

> I inject a fake implementation of the repository interface into the ViewModel constructor. The fake returns controlled test data synchronously via Flow or suspending functions. Using `kotlinx-coroutines-test` with `StandardTestDispatcher` and `runTest`, I can advance time, test loading states, and verify state transitions. Because the ViewModel has no Android dependency (no Context, no Activity), it runs as a pure JUnit test — fast and reliable. I test: initial state is correct, success state is emitted after data loads, error state is emitted when the repo throws, side effects are emitted to the channel.

---

**Q10: When would you choose Hilt over Koin for dependency injection?**

> Hilt is compile-time verified — misconfigured bindings fail at build time, not at runtime. It's Google's recommended solution and integrates tightly with Jetpack (ViewModel injection, WorkManager, Navigation). Koin is a runtime DI framework — simpler to set up, less boilerplate, but errors surface at runtime (app crash on first access). I default to Hilt for any app with more than one engineer or that will be in production long-term — the compile-time safety is worth the setup cost. Koin is reasonable for prototypes or small apps where simplicity matters more than strictness.

---

*Next: [02 — Networking & API Design](./02-networking-api.md)*
