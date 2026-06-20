# 34 — Kotlin Multiplatform (KMP)

> **Round type: HLD (primary) + LLD**
> **HLD:** When to adopt KMP, sharing strategies (logic-only vs logic + UI), module boundaries
> **LLD:** expect/actual, shared networking with Ktor, shared DB with SQLDelight, Compose Multiplatform

---

## 1. Why This Matters Now

KMP reached **Stable** in November 2023. Google announced **official Jetpack support** for KMP at I/O 2024. Compose Multiplatform for iOS hit **Stable** in May 2025. Companies using it in production: Netflix, Duolingo, McDonald's, Cash App, Shopify. Interviewers at Google, JetBrains-adjacent companies, and forward-thinking mobile teams are asking about KMP.

**Key insight for interviews:** KMP is not Flutter or React Native. Native UI is preserved — you share only what makes sense (business logic, data, networking) while each platform retains its native UI toolkit.

---

## 2. What KMP Is (and Isn't)

```
KMP IS:
  - Shared Kotlin code compiled to JVM (Android), Native (iOS), JS/Wasm (Web)
  - Shared: business logic, networking, DB, serialization, domain models, use cases
  - Native UI on each platform: Jetpack Compose (Android) + SwiftUI (iOS)
  - Incremental adoption — migrate one module at a time

KMP IS NOT:
  - Flutter (single UI for all platforms rendered by a custom engine)
  - React Native (JS bridge to native components)
  - "Write once, run anywhere" — platform-specific UI still separate

Compose Multiplatform (CMP) = KMP + shared UI:
  - Jetpack Compose UI rendered on Android, iOS, Desktop, Web
  - Stable for iOS since May 2025 (CMP 1.8.0)
  - Still younger ecosystem than React Native/Flutter for shared UI
```

---

## 3. Two Adoption Paths

### Path A: Shared Business Logic Only (Pragmatic — Recommended Starting Point)

```
Android App              iOS App
┌──────────────┐         ┌──────────────┐
│  Jetpack     │         │   SwiftUI    │
│  Compose UI  │         │   / UIKit    │
├──────────────┤         ├──────────────┤
│  Android     │         │   iOS        │
│  ViewModel   │         │   ViewModel  │
├──────────────┤         ├──────────────┤
│  ──────── shared/commonMain ───────── │
│  Domain Models  Use Cases             │
│  Repositories   Networking (Ktor)     │
│  Database       Serialization         │
│  Feature Flags  Analytics             │
└───────────────────────────────────────┘

Code share: ~50-70% of codebase
Risk: Low — native UI unchanged
Benefit: One implementation for all business logic
```

### Path B: Shared UI + Logic (Compose Multiplatform)

```
┌───────────────────────────────────────┐
│  ───── composeApp (shared UI) ─────── │
│  Compose UI (Android + iOS + Desktop) │
│  Navigation, ViewModels, UiState      │
├───────────────────────────────────────┤
│  ─────── shared/commonMain ─────────  │
│  Domain, Data, Networking, DB         │
└───────────────────────────────────────┘

Code share: ~80-95%
Risk: Medium — iOS Compose ecosystem still younger
Benefit: Near-full codebase reuse
Tradeoff: Less platform-native feel on some interactions
```

---

## 4. Project Structure

```
myproject/
├── androidApp/          → Android-specific entry point, Hilt setup
│   └── src/main/kotlin/
│       └── AndroidApp.kt
│
├── iosApp/              → iOS app (Xcode project)
│   └── iosApp.xcodeproj
│
└── shared/              → Kotlin Multiplatform module
    ├── build.gradle.kts
    └── src/
        ├── commonMain/  → All platforms: business logic, domain, data
        │   └── kotlin/
        │       ├── domain/
        │       │   ├── model/User.kt
        │       │   └── usecase/GetUserUseCase.kt
        │       ├── data/
        │       │   ├── network/UserApi.kt     (Ktor)
        │       │   └── db/UserDatabase.kt     (SQLDelight)
        │       └── platform/
        │           └── PlatformContext.kt     (expect)
        │
        ├── androidMain/ → Android-specific implementations (actual)
        │   └── kotlin/
        │       └── platform/
        │           └── PlatformContext.android.kt
        │
        └── iosMain/     → iOS-specific implementations (actual)
            └── kotlin/
                └── platform/
                    └── PlatformContext.ios.kt
```

---

## 5. `expect` / `actual` — Platform-Specific Code

```kotlin
// In commonMain — declare the interface
expect class PlatformContext

expect fun getPlatformName(): String

expect class DatabaseDriverFactory {
    fun createDriver(): SqlDriver
}

// In androidMain — Android implementation
actual class PlatformContext(val context: Context)

actual fun getPlatformName() = "Android ${Build.VERSION.RELEASE}"

actual class DatabaseDriverFactory(private val context: Context) {
    actual fun createDriver(): SqlDriver =
        AndroidSqliteDriver(AppDatabase.Schema, context, "app.db")
}

// In iosMain — iOS implementation
actual class PlatformContext

actual fun getPlatformName() = UIDevice.currentDevice.systemName() +
        " " + UIDevice.currentDevice.systemVersion

actual class DatabaseDriverFactory {
    actual fun createDriver(): SqlDriver =
        NativeSqliteDriver(AppDatabase.Schema, "app.db")
}
```

---

## 6. Networking with Ktor (Multiplatform)

```kotlin
// shared/build.gradle.kts
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("io.ktor:ktor-client-core:2.3.12")
            implementation("io.ktor:ktor-client-content-negotiation:2.3.12")
            implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.12")
            implementation("io.ktor:ktor-client-logging:2.3.12")
        }
        androidMain.dependencies {
            implementation("io.ktor:ktor-client-okhttp:2.3.12")  // OkHttp engine for Android
        }
        iosMain.dependencies {
            implementation("io.ktor:ktor-client-darwin:2.3.12")  // URLSession engine for iOS
        }
    }
}

// In commonMain — works on both platforms
class UserApi {
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json { ignoreUnknownKeys = true })
        }
        install(Logging) {
            level = LogLevel.HEADERS
        }
        defaultRequest {
            url("https://api.example.com/")
            header("Authorization", "Bearer ${TokenStorage.getToken()}")
        }
    }

    suspend fun getUser(id: String): User =
        client.get("users/$id").body()

    suspend fun getPosts(page: Int): List<Post> =
        client.get("posts") {
            parameter("page", page)
        }.body()
}
```

---

## 7. Database with SQLDelight (Multiplatform)

```kotlin
// shared/build.gradle.kts
kotlin {
    sourceSets {
        commonMain.dependencies {
            implementation("app.cash.sqldelight:runtime:2.0.2")
            implementation("app.cash.sqldelight:coroutines-extensions:2.0.2")
        }
        androidMain.dependencies {
            implementation("app.cash.sqldelight:android-driver:2.0.2")
        }
        iosMain.dependencies {
            implementation("app.cash.sqldelight:native-driver:2.0.2")
        }
    }
}

// Define schema in shared/src/commonMain/sqldelight/com/example/User.sq
CREATE TABLE User (
    id TEXT PRIMARY KEY NOT NULL,
    name TEXT NOT NULL,
    email TEXT NOT NULL,
    cachedAt INTEGER NOT NULL DEFAULT 0
);

getUser:
SELECT * FROM User WHERE id = :id;

getAllUsers:
SELECT * FROM User;

upsertUser:
INSERT OR REPLACE INTO User(id, name, email, cachedAt)
VALUES (?, ?, ?, ?);

deleteUser:
DELETE FROM User WHERE id = :id;

-- SQLDelight generates type-safe Kotlin code from this .sq file

// Usage in commonMain
class UserLocalDataSource(private val db: AppDatabase) {
    fun observeUser(id: String): Flow<User?> =
        db.userQueries.getUser(id).asFlow().mapToOneOrNull()

    suspend fun upsert(user: User) = db.userQueries.upsertUser(
        id = user.id,
        name = user.name,
        email = user.email,
        cachedAt = Clock.System.now().toEpochMilliseconds()
    )
}
```

---

## 8. Shared Repository (Pure Kotlin, Runs Everywhere)

```kotlin
// commonMain — no Android or iOS imports needed
class UserRepository(
    private val api: UserApi,
    private val db: UserLocalDataSource
) {
    fun observeUser(id: String): Flow<User> = flow {
        // Emit cached
        db.observeUser(id).firstOrNull()?.let { emit(it) }
        // Fetch fresh
        try {
            val fresh = api.getUser(id)
            db.upsert(fresh)
            emit(fresh)
        } catch (e: Exception) {
            // Swallow if we already emitted cached
        }
    }
}

// Shared ViewModel (KMP-friendly, no Android ViewModel dependency needed)
// Use kotlinx.coroutines + StateFlow
class UserViewModel(private val repo: UserRepository) {
    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())
    private val _state = MutableStateFlow<UserState>(UserState.Loading)
    val state: StateFlow<UserState> = _state.asStateFlow()

    fun loadUser(id: String) {
        scope.launch {
            repo.observeUser(id).collect { user ->
                _state.value = UserState.Success(user)
            }
        }
    }
    fun dispose() = scope.cancel()
}

// Android: wrap in androidx.lifecycle.ViewModel if needed
// iOS: call from Swift, observe state as NSObject
```

---

## 9. Compose Multiplatform (Shared UI)

```kotlin
// composeApp/src/commonMain — shared UI
@Composable
fun UserProfileScreen(userId: String) {
    val viewModel = remember { UserViewModel(/* inject repository */) }
    val state by viewModel.state.collectAsState()

    LaunchedEffect(userId) { viewModel.loadUser(userId) }

    when (val s = state) {
        is UserState.Loading -> CircularProgressIndicator()
        is UserState.Success -> UserProfileContent(user = s.user)
        is UserState.Error -> ErrorContent(message = s.message)
    }
}

// This composable runs on BOTH Android and iOS
// Android: renders with Jetpack Compose (Skia on Canvas)
// iOS: renders with Compose UI (Skia on Metal, stable since CMP 1.8.0)

// Platform-specific UI when needed
@Composable
expect fun PlatformSpecificButton(text: String, onClick: () -> Unit)

// androidMain
@Composable
actual fun PlatformSpecificButton(text: String, onClick: () -> Unit) {
    Button(onClick = onClick) { Text(text) }  // Material Design
}

// iosMain
@Composable
actual fun PlatformSpecificButton(text: String, onClick: () -> Unit) {
    // Can use iOS-native appearance or custom Compose styling
    OutlinedButton(onClick = onClick, shape = RoundedCornerShape(10.dp)) { Text(text) }
}
```

---

## 10. KMP-Compatible Jetpack Libraries (as of 2025)

| Library | KMP Support |
|---|---|
| **kotlinx.coroutines** | ✅ Full |
| **kotlinx.serialization** | ✅ Full |
| **Ktor** | ✅ Full |
| **SQLDelight** | ✅ Full |
| **Kotlin DateTime** | ✅ Full |
| **DataStore** (Preferences) | ✅ (Multiplatform DataStore 1.1+) |
| **Room** | ✅ KMP support added (Google I/O 2024) |
| **ViewModel** (lifecycle) | ✅ KMP support added (Google I/O 2024) |
| **Navigation** (Compose) | ✅ Compose Multiplatform navigation |
| **Koin** | ✅ Full |
| **Hilt** | ❌ Android only (use Koin for KMP) |
| **WorkManager** | ❌ Android only |
| **ExoPlayer/Media3** | ❌ Android only |

---

## 11. When to Use KMP

```
✅ Strong case for KMP:
  - New app built for both Android and iOS
  - Significant business logic that is identical on both platforms
  - Small-to-medium team (avoid duplicating logic across two codebases)
  - Company has both Android and iOS engineers (shared Kotlin is a common language)
  - Already using Kotlin on backend (full-stack Kotlin)

❌ Weaker case:
  - App is Android-only (no benefit)
  - App requires heavy platform-specific UI/interaction (custom gestures, ARKit, etc.)
  - Very large established codebases where migration risk is high
  - Team is iOS-only (Swift engineers unfamiliar with Kotlin)
  - App requires WorkManager, ExoPlayer, or other Android-only libraries throughout

🟡 Incremental adoption (best strategy):
  Start with: shared domain models + networking (lowest risk)
  Then add: shared use cases + repositories
  Then evaluate: shared ViewModel + Compose Multiplatform for UI
```

---

## 12. KMP vs React Native vs Flutter

| | KMP | React Native | Flutter |
|---|---|---|---|
| Language | Kotlin (shared) + native UI | JavaScript/TypeScript | Dart |
| UI | Native (Compose/SwiftUI) or CMP | Native bridge components | Custom engine (Skia) |
| Native feel | ✅ Best | 🟡 Good | 🟡 Good |
| Code sharing | Business logic (50-95%) | Full (business + UI) | Full (business + UI) |
| Android expertise reuse | ✅ Direct (it's Kotlin) | Partial | Minimal |
| iOS ecosystem | 🟡 Growing | ✅ Mature | ✅ Mature |
| Google support | ✅ Official (2024) | 🟡 Via Meta | ✅ First-party |
| Used by | Netflix, Cash App, Shopify | Facebook, Microsoft, Shopify | Google, BMW, eBay |
| Best for | Teams with Kotlin expertise sharing logic | JS-experienced teams | Flutter-first greenfield |

---

## 13. Common Pitfalls

**❌ Trying to share everything immediately**
KMP adoption is most successful incrementally — start with domain models and networking, prove it works, then expand. Trying to share everything on day one creates too much risk and platform-specific friction.

**❌ Using `@HiltViewModel` in shared code**
Hilt is Android-only. For KMP-compatible ViewModels, use `ViewModel` from `androidx.lifecycle:lifecycle-viewmodel` (now KMP-enabled since 2024) or plain Kotlin classes with `CoroutineScope`.

**❌ Ignoring iOS threading model**
Kotlin/Native (iOS) has strict memory management rules — objects can only be accessed from one thread at a time unless using `@SharedImmutable` or coroutines with proper dispatchers. `Dispatchers.Main` on iOS uses the main thread. Always test shared code on iOS, not just Android.

**❌ Using `println` / `Log.d` in shared code**
`android.util.Log` is unavailable in common code. Use a KMP logging library (Napier, Kermit) or `println` with a wrapper.

---

## 14. Best Practices

- **Start with shared domain + networking** — lowest risk, immediate benefit
- **Koin over Hilt** for KMP-compatible dependency injection
- **Ktor for networking** — designed for KMP, great coroutine support
- **SQLDelight for local DB** — generates type-safe Kotlin for all platforms
- **Room** now supports KMP too — choose based on team familiarity
- **kotlinx.datetime** for date/time — avoids `java.time` (Android-only)
- **kotlinx.serialization** for JSON — works in all KMP targets
- **Test commonMain with JUnit** — runs on JVM, fast
- **Incremental migration** — move one module at a time, prove value early
- **Evaluate CMP for UI** — stable for iOS since May 2025, consider for new projects

---

## 15. Interview Q&A

**Q1: What is Kotlin Multiplatform and how does it differ from Flutter or React Native?**

> KMP lets you write Kotlin code that compiles to multiple platforms — JVM for Android, Kotlin/Native for iOS, and JavaScript or WebAssembly for web. The key difference from Flutter and React Native: KMP doesn't impose a cross-platform UI. Native UI is preserved — Jetpack Compose on Android, SwiftUI on iOS. What's shared is business logic: networking, repositories, domain models, use cases, and database access. Flutter and React Native share UI code too, but at the cost of rendering with a non-native engine or native bridge overhead. KMP's approach means full platform fidelity and no performance compromise for UI, while eliminating duplicated business logic. Compose Multiplatform is the opt-in extension that shares UI too, stable on iOS since May 2025.

---

**Q2: When would you adopt KMP and where would you start?**

> I'd adopt KMP when building a new app for both Android and iOS, especially if the team already knows Kotlin. The ROI is immediate — every piece of business logic written once instead of twice. For an existing app, I'd start incrementally with the lowest-risk layer: shared domain models and networking. `kotlinx.serialization` for JSON, Ktor for HTTP — both are designed for KMP and drop in without changing the Android side. Once that's proven, move use cases into `commonMain`. The last step — and most optional — is Compose Multiplatform for shared UI, which makes sense when the Android and iOS UI are nearly identical and you want to eliminate UI duplication too. I'd avoid trying to share WorkManager, ExoPlayer, or any deeply Android-specific library — those stay in `androidMain`.

---

**Q3: What is the `expect`/`actual` mechanism?**

> It's KMP's way of handling platform-specific implementations. In `commonMain`, you declare a type or function with `expect` — this is the interface contract. In each platform source set (`androidMain`, `iosMain`), you provide the `actual` implementation. A classic example: creating a SQLite driver. In `commonMain`, `expect class DatabaseDriverFactory`. In `androidMain`, `actual class DatabaseDriverFactory(context: Context)` that returns `AndroidSqliteDriver`. In `iosMain`, `actual class DatabaseDriverFactory()` that returns `NativeSqliteDriver`. The shared code in `commonMain` uses `DatabaseDriverFactory` without knowing which platform is running. Libraries like SQLDelight use this extensively internally, so often you don't need to write many `expect`/`actual` pairs yourself.

---

*Previous: [33 — Testing Strategies](./33-testing.md)*
