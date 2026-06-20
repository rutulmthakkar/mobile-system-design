# 27 — Common Libraries & Dependency Management

> **Round type: LLD + Practical Reference**
> A living reference of the Android ecosystem's most important libraries — what they do, how they compare, when to pick each, and how to keep your app lean.

---

## 1. Why This Matters in Interviews

Interviewers expect you to know the ecosystem — not just that a library exists, but *why* you'd choose it over alternatives and what tradeoffs come with it. "We used Retrofit" is a weak answer. "We used Retrofit with Kotlin Serialization over Gson because we needed compile-time null-safety and KMP compatibility" is a strong one.

**Common angles:**
- "What JSON library would you use and why?"
- "Hilt or Koin — which do you prefer and when?"
- "How do you keep your APK size down?"
- "How do you test a ViewModel / Composable / repository?"
- "Walk me through your build.gradle structure"

---

## 2. JSON Serialization

### 2.1 The Three Contenders

#### Gson (Google)
```kotlin
// Gradle
implementation("com.google.code.gson:gson:2.10.1")

// Usage
val gson = Gson()
val json = gson.toJson(user)
val user = gson.fromJson(json, User::class.java)

// With Retrofit
.addConverterFactory(GsonConverterFactory.create())
```

**How it works:** Reflection at runtime to discover class structure. No annotation processing, no code gen.

**Pros:** Zero setup, huge community, handles complex polymorphism easily.

**Cons:**
- Uses reflection → slower than codegen alternatives (~20–30%)
- **Not null-safe with Kotlin** — if JSON is missing a non-null field, Gson skips the default value and inserts `null` anyway, bypassing Kotlin's null safety. Runtime NPE instead of parse error.
- Can't be used with Kotlin Multiplatform

**When to use:** Legacy Java codebases, or when you need maximum flexibility with minimal setup and null-safety isn't a concern.

---

#### Moshi (Square)
```kotlin
// Gradle — use codegen, not reflection
implementation("com.squareup.moshi:moshi:1.15.1")
ksp("com.squareup.moshi:moshi-kotlin-codegen:1.15.1")  // KSP for compile-time adapters

// Data class
@JsonClass(generateAdapter = true)  // triggers codegen
data class User(
    val id: String,
    val name: String,
    @Json(name = "avatar_url") val avatarUrl: String,  // field name mapping
    val email: String? = null
)

// Setup
val moshi = Moshi.Builder().build()
val adapter = moshi.adapter(User::class.java)
val user = adapter.fromJson(json)

// With Retrofit
.addConverterFactory(MoshiConverterFactory.create(moshi))
```

**How it works (codegen mode):** KSP generates a `UserJsonAdapter` at compile time. No reflection at runtime — just generated code reading/writing JSON fields directly.

**Pros:**
- Null-safe with Kotlin — missing required field → `JsonDataException` at parse time, not NPE later
- Streaming API via Okio — lower memory, fewer GC spikes
- Faster than Gson (codegen avoids reflection)
- Supports sealed classes and polymorphism with `PolymorphicJsonAdapterFactory`

**Cons:**
- `@JsonClass(generateAdapter = true)` required on every class
- Not KMP-native (though works on JVM targets)
- Slightly more setup than Gson

**When to use:** Kotlin-first Android projects that don't need KMP. Best balance of safety, performance, and maturity.

---

#### Kotlin Serialization (JetBrains) ✅ Recommended for new projects
```kotlin
// Gradle
plugins { id("org.jetbrains.kotlin.plugin.serialization") version "2.0.0" }
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")

// Data class
@Serializable
data class User(
    val id: String,
    val name: String,
    @SerialName("avatar_url") val avatarUrl: String,
    val email: String? = null
)

// Usage
val json = Json.encodeToString(user)
val user = Json.decodeFromString<User>(json)

// Lenient config for production (handles trailing commas, unknown keys)
val json = Json {
    ignoreUnknownKeys = true    // don't crash on new server fields
    isLenient = true
    encodeDefaults = false      // don't serialize default values
}

// With Retrofit
implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
.addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
```

**How it works:** Compiler plugin generates serializers at compile time. Pure Kotlin, no reflection.

**Pros:**
- ✅ Kotlin Multiplatform — works on Android, iOS (via KMP), desktop, web
- ✅ Compile-time null safety
- ✅ Fastest of the three (compiler-generated)
- No annotation processor / KSP needed — compiler plugin handles it
- Supports sealed classes, generics, polymorphism natively

**Cons:**
- `@Serializable` required on every class
- Less mature ecosystem for complex edge cases than Moshi
- Retrofit converter is a third-party library (not official)

**When to use:** All new projects, especially if KMP is a future consideration.

### 2.2 Comparison Summary

| | Gson | Moshi (codegen) | Kotlinx Serialization |
|---|---|---|---|
| Null safety (Kotlin) | ❌ Bypasses | ✅ Enforces | ✅ Enforces |
| Performance | Slowest | Fast | Fastest |
| Setup | None | KSP + annotation | Compiler plugin |
| KMP support | ❌ | ❌ | ✅ |
| Sealed classes | With adapter | With factory | Native |
| Recommended for | Legacy Java | Kotlin Android | New projects |

---

## 3. Networking

### OkHttp + Retrofit (Standard Stack)
```kotlin
// Gradle
implementation("com.squareup.okhttp3:okhttp:4.12.0")
implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
implementation("com.squareup.retrofit2:retrofit:2.9.0")
```
*Covered in depth in Section 02 — Networking & API Design.*

**Key point:** OkHttp and Retrofit are not interchangeable with alternatives in most teams. They are the de facto standard. Ktor is the alternative for KMP projects.

### Ktor (KMP alternative)
```kotlin
// Gradle (KMP)
implementation("io.ktor:ktor-client-android:2.3.7")
implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")

val client = HttpClient(Android) {
    install(ContentNegotiation) { json() }
    install(HttpTimeout) { requestTimeoutMillis = 30_000 }
}
val user: User = client.get("https://api.example.com/users/123").body()
```

**Use Ktor when:** Sharing networking code across Android + iOS in KMP. Not recommended for Android-only projects — Retrofit's ecosystem is more mature.

---

## 4. Image Loading

### 4.1 The Three Main Options

#### Coil ✅ Recommended for Kotlin/Compose projects
```kotlin
// Gradle
implementation("io.coil-kt.coil3:coil-compose:3.0.4")
implementation("io.coil-kt.coil3:coil-network-okhttp:3.0.4")

// Compose usage
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/avatar.jpg")
        .crossfade(true)
        .placeholder(R.drawable.placeholder)
        .error(R.drawable.error)
        .size(200, 200)        // resize before caching — saves memory
        .build(),
    contentDescription = "Avatar",
    contentScale = ContentScale.Crop,
    modifier = Modifier.clip(CircleShape)
)

// Selective import — only include what you need
implementation("io.coil-kt.coil3:coil-gif:3.0.4")    // GIF support (optional)
implementation("io.coil-kt.coil3:coil-svg:3.0.4")    // SVG support (optional)
implementation("io.coil-kt.coil3:coil-video:3.0.4")  // Video thumbnails (optional)
```

**Pros:** Kotlin-first, Coroutines-native, Compose integration built-in, uses existing OkHttp instance, smallest footprint (~94KB after R8), Coil 3 supports KMP.

**Cons:** Smaller community than Glide, fewer enterprise case studies.

**When to use:** All new Kotlin/Compose projects.

---

#### Glide (Bumptech / Google)
```kotlin
// Gradle
implementation("com.github.bumptech.glide:glide:4.16.0")
ksp("com.github.bumptech.glide:ksp:4.16.0")

// With Compose (via accompanist or official extension)
implementation("com.github.bumptech.glide:compose:1.0.0-beta01")

// Classic View usage
Glide.with(context)
    .load("https://example.com/avatar.jpg")
    .apply(RequestOptions()
        .placeholder(R.drawable.placeholder)
        .error(R.drawable.error)
        .circleCrop()
        .override(200, 200))   // resize
    .into(imageView)

// Selective: only include what you need
implementation("com.github.bumptech.glide:annotations:4.16.0")  // for AppGlideModule
```

**Pros:** Mature, battle-tested, fastest cache loading performance, excellent GIF support, used by Google apps, best for View-based (XML) UIs.

**Cons:** Larger footprint (~222KB), Java-first API feels verbose in Kotlin, Compose integration is still maturing.

**When to use:** View-based (XML) projects, apps with heavy GIF requirements, teams with existing Glide expertise.

---

#### Picasso (Square)
```kotlin
implementation("com.squareup.picasso:picasso:2.8")

Picasso.get()
    .load("https://example.com/avatar.jpg")
    .placeholder(R.drawable.placeholder)
    .resize(200, 200)
    .centerCrop()
    .into(imageView)
```

**Pros:** Smallest library (~36KB after R8), simplest API, reliable for basic use cases.

**Cons:** No GIF support, lower cache performance than Glide, limited Compose support, slower development cadence.

**When to use:** Simple apps with basic image loading needs, extreme size constraints.

### 4.2 Comparison

| | Coil 3 | Glide | Picasso |
|---|---|---|---|
| Size (R8) | ~94KB | ~222KB | ~36KB |
| Kotlin-first | ✅ | ❌ | ❌ |
| Compose support | ✅ Native | 🟡 Beta | ❌ |
| GIF support | ✅ (module) | ✅ Built-in | ❌ |
| Coroutines | ✅ | ❌ | ❌ |
| Cache speed | Fast | Fastest | Slower |
| KMP | ✅ (v3) | ❌ | ❌ |
| Best for | New Kotlin/Compose | View-based + GIF | Simple + tiny |

---

## 5. Dependency Injection

### 5.1 Hilt ✅ Recommended for Android projects
```kotlin
// Gradle (app-level)
plugins {
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}
implementation("com.google.dagger:hilt-android:2.51.1")
ksp("com.google.dagger:hilt-compiler:2.51.1")

// Optional Jetpack integrations
implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
implementation("androidx.hilt:hilt-work:1.2.0")

// Application class
@HiltAndroidApp
class App : Application()

// ViewModel injection
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repo: UserRepository,
    private val analytics: Analytics
) : ViewModel()

// Module — providing dependencies Hilt can't construct
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideOkHttp(): OkHttpClient = OkHttpClient.Builder().build()

    @Provides @Singleton
    fun provideRetrofit(client: OkHttpClient): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(client)
        .build()
}

// Composable — automatic ViewModel injection
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel())
```

**Pros:** Compile-time DI graph verification (misconfiguration = build error, not runtime crash), deep Jetpack integration (ViewModel, WorkManager, Navigation), Google-backed, `@Singleton` / `@ActivityScoped` / `@ViewModelScoped` lifecycle management.

**Cons:** Steep learning curve (Dagger concepts), annotation-heavy, slower build times (annotation processing / KSP), Android-only (no KMP).

**When to use:** Any production Android app with more than one engineer. Compile-time safety is worth the setup.

---

### 5.2 Koin
```kotlin
// Gradle — no annotation processing needed
implementation("io.insert-koin:koin-android:3.5.6")
implementation("io.insert-koin:koin-androidx-compose:3.5.6")

// Module declaration — pure Kotlin DSL
val appModule = module {
    single<OkHttpClient> { OkHttpClient.Builder().build() }
    single<UserApi> { get<Retrofit>().create(UserApi::class.java) }
    single<UserRepository> { UserRepositoryImpl(get()) }
    viewModel { UserViewModel(get()) }
}

// Application class
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin { androidContext(this@App); modules(appModule) }
    }
}

// Composable
@Composable
fun UserScreen(viewModel: UserViewModel = koinViewModel())
```

**Pros:** No annotation processing → faster builds, simple Kotlin DSL, easy to learn, good for KMP (Koin 4+ supports multiplatform), runtime DI resolution.

**Cons:** Runtime errors — misconfigured dependency throws at the first access, not at build time. Slightly slower resolution at runtime vs Hilt's generated code. Smaller community.

**When to use:** Smaller teams, KMP projects, rapid prototyping, or when Hilt's learning curve is a barrier.

### 5.3 Hilt vs Koin

| | Hilt | Koin |
|---|---|---|
| Error detection | Compile-time ✅ | Runtime ❌ |
| Build speed | Slower (KSP) | Faster (no codegen) |
| Learning curve | Steep | Gentle |
| Jetpack integration | Deep (official) | Good (third-party) |
| KMP support | ❌ | ✅ (v4+) |
| Runtime overhead | Minimal (generated) | Slight (reflection-lite) |
| Best for | Production Android | KMP, rapid dev, smaller teams |

---

## 6. Local Database

### Room ✅ Standard choice
*Covered in depth in Section 04 — Data Layer.*

```kotlin
implementation("androidx.room:room-runtime:2.6.1")
implementation("androidx.room:room-ktx:2.6.1")
ksp("androidx.room:room-compiler:2.6.1")
// Optional: Paging 3 integration
implementation("androidx.room:room-paging:2.6.1")
```

### Realm (MongoDB)
```kotlin
implementation("io.realm.kotlin:library-base:1.16.0")

// Schema
class User : RealmObject {
    @PrimaryKey var id: String = ""
    var name: String = ""
}

// Usage
val realm = Realm.open(RealmConfiguration.create(schema = setOf(User::class)))
realm.write { copyToRealm(User().apply { id = "1"; name = "Alice" }) }
val users = realm.query<User>().find()  // returns Flow or list
```

**Pros:** Object-oriented (no SQL), live objects (auto-updating queries like Flow), built-in sync with MongoDB Atlas, good for offline-first with cloud sync.

**Cons:** Large binary size (~5MB), object model constraints (must extend RealmObject), migration complexity, proprietary format (can't inspect with SQLite tools), learning curve.

**When to use:** Apps needing built-in cloud sync (MongoDB Atlas Device Sync), or teams strongly preferring object-oriented data access over SQL.

### SQLDelight
```kotlin
implementation("app.cash.sqldelight:android-driver:2.0.1")
// KMP driver
implementation("app.cash.sqldelight:native-driver:2.0.1")
```

SQL-first: you write `.sq` files with SQL queries, SQLDelight generates type-safe Kotlin. Best for KMP projects where you want to share DB logic across platforms.

### Comparison

| | Room | Realm | SQLDelight |
|---|---|---|---|
| Query language | SQL (via annotations) | Kotlin fluent API | SQL (`.sq` files) |
| Type safety | Compile-time | Compile-time | Compile-time |
| KMP | ❌ | ❌ (iOS = separate) | ✅ |
| Cloud sync | ❌ | ✅ (Atlas) | ❌ |
| Size | Small | Large (~5MB) | Small |
| Google-recommended | ✅ | ❌ | ❌ |
| Best for | Android-only apps | Cloud-sync apps | KMP apps |

---

## 7. Media — ExoPlayer vs Media3

### ExoPlayer (legacy)
```kotlin
implementation("com.google.android.exoplayer:exoplayer:2.19.1")
// Selective — don't import the full library if you don't need all formats
implementation("com.google.android.exoplayer:exoplayer-core:2.19.1")
implementation("com.google.android.exoplayer:exoplayer-ui:2.19.1")
implementation("com.google.android.exoplayer:exoplayer-dash:2.19.1")  // only if using DASH
implementation("com.google.android.exoplayer:exoplayer-hls:2.19.1")   // only if using HLS
```

### Media3 ✅ Current standard (ExoPlayer successor)
```kotlin
// Media3 is the replacement for ExoPlayer — use this for all new code
implementation("androidx.media3:media3-exoplayer:1.3.1")
implementation("androidx.media3:media3-ui:1.3.1")
// Selective modules
implementation("androidx.media3:media3-exoplayer-hls:1.3.1")     // HLS streaming
implementation("androidx.media3:media3-exoplayer-dash:1.3.1")    // DASH streaming
implementation("androidx.media3:media3-exoplayer-rtsp:1.3.1")    // RTSP
implementation("androidx.media3:media3-datasource-okhttp:1.3.1") // OkHttp data source
implementation("androidx.media3:media3-session:1.3.1")           // MediaSession integration

// Basic playback
val player = ExoPlayer.Builder(context).build().apply {
    setMediaItem(MediaItem.fromUri("https://example.com/video.m3u8"))
    prepare()
    play()
}

// Compose UI
AndroidView(
    factory = { ctx ->
        PlayerView(ctx).apply { this.player = player }
    },
    modifier = Modifier.fillMaxWidth().aspectRatio(16f / 9f)
)
```

**Key difference from ExoPlayer 2:** Media3 consolidates ExoPlayer, MediaSession, and Transformer into one library under `androidx.media3`. ExoPlayer 2 is now in maintenance mode — new features only in Media3.

**Selective imports are important:** Don't add all codec/format modules. Only import the formats your app actually plays. Each module adds ~200–500KB.

*Covered in depth in Section 15 — Video Playback & DRM.*

---

## 8. Testing Libraries

### 8.1 The Testing Pyramid

```
         /‾‾‾‾‾‾‾‾‾‾‾‾‾\
        /  E2E / UI Tests  \      ← Slow, device needed, few
       /    (Espresso,       \
      /     UI Automator)     \
     /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
    /   Integration Tests      \  ← Medium, Robolectric
   /   (Robolectric, Compose)   \
  /‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\
 /       Unit Tests               \ ← Fast, JVM, most
/  (JUnit, MockK/Mockito, Turbine)  \
‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
```

### 8.2 JUnit 4 vs JUnit 5

```kotlin
// JUnit 4 — still the Android default (Espresso requires it)
testImplementation("junit:junit:4.13.2")

// JUnit 5 — more powerful (parameterized, extensions, better lifecycle)
// Requires additional setup on Android
testImplementation("org.junit.jupiter:junit-jupiter:5.10.0")
```

JUnit 4 is what you'll encounter most in Android due to instrumented test compatibility. JUnit 5 is gaining ground for pure unit tests.

### 8.3 MockK ✅ Recommended for Kotlin

```kotlin
testImplementation("io.mockk:mockk:1.13.10")
// For Android instrumented tests
androidTestImplementation("io.mockk:mockk-android:1.13.10")

// Usage
@Test
fun `getUserById returns user from repository`() = runTest {
    // Arrange
    val repo = mockk<UserRepository>()
    coEvery { repo.getUser("123") } returns Result.success(User("123", "Alice"))

    val viewModel = UserViewModel(repo)

    // Act
    viewModel.loadUser("123")
    advanceUntilIdle()

    // Assert
    assertEquals("Alice", viewModel.uiState.value.user?.name)
    coVerify { repo.getUser("123") }
}

// Mocking objects, companion objects, top-level functions
val spy = spyk(RealAnalytics())
every { spy.track(any()) } just Runs

// Relaxed mock — all methods return default values, no setup needed
val mock = mockk<UserApi>(relaxed = true)
```

**Pros:** Kotlin-native (coroutine support with `coEvery`, `coVerify`), mocks `object`, `companion object`, top-level functions, `final` classes without additional setup, clean readable syntax.

**Cons:** Slightly slower than Mockito for pure Java mocking.

### 8.4 Mockito (with mockito-kotlin)

```kotlin
testImplementation("org.mockito:mockito-core:5.11.0")
testImplementation("org.mockito.kotlin:mockito-kotlin:5.2.1")

// Usage
val repo = mock<UserRepository>()
whenever(repo.getUser("123")).thenReturn(Result.success(User("123", "Alice")))
verify(repo).getUser("123")
```

**Pros:** Mature, massive community, works great for Java classes.

**Cons:** Can't mock `final` classes without the `mockito-inline` extension (Kotlin classes are `final` by default), no native coroutine support (need `mockito-kotlin` wrapper), verbose for Kotlin.

**MockK vs Mockito:**

| | MockK | Mockito |
|---|---|---|
| Kotlin-native | ✅ | ❌ (needs wrapper) |
| Coroutine support | ✅ coEvery/coVerify | ❌ manual wrapping |
| Final classes | ✅ By default | ❌ Needs mockito-inline |
| Mocking objects | ✅ | ❌ |
| Java class mocking | Good | Best |
| Best for | Kotlin projects | Java/mixed projects |

### 8.5 Turbine (Flow testing)

```kotlin
testImplementation("app.cash.turbine:turbine:1.1.0")

// Testing Flow emissions
@Test
fun `uiState emits loading then success`() = runTest {
    val viewModel = UserViewModel(fakeRepo)

    viewModel.uiState.test {
        assertEquals(UiState.Loading, awaitItem())
        viewModel.loadUser("123")
        val success = awaitItem() as UiState.Success
        assertEquals("Alice", success.user.name)
        cancelAndIgnoreRemainingEvents()
    }
}
```

Turbine is essential for testing `Flow` — avoids manual collection and timeout complexity.

### 8.6 Robolectric

```kotlin
testImplementation("org.robolectric:robolectric:4.12.1")

@RunWith(RobolectricTestRunner::class)
@Config(sdk = [34])
class MainActivityTest {
    @Test
    fun `button click updates text`() {
        val activity = Robolectric.buildActivity(MainActivity::class.java)
            .create().start().resume().get()
        activity.findViewById<Button>(R.id.btn).performClick()
        assertEquals("Clicked", activity.findViewById<TextView>(R.id.tv).text)
    }
}
```

**What it does:** Simulates Android framework on JVM — no emulator or device needed. Runs Android components (Activity, Fragment, View) in pure JVM tests.

**When to use:** Testing Android-specific logic that doesn't need a real device — Activity creation, Fragment transactions, View interactions. Faster than instrumented tests, slower than pure unit tests.

**When NOT to use:** Anything needing real hardware (camera, sensors, Bluetooth). For those, use instrumented tests on a device/emulator.

### 8.7 Espresso (UI / Instrumented Tests)

```kotlin
androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
androidTestImplementation("androidx.test:runner:1.5.2")
androidTestImplementation("androidx.test:rules:1.5.0")

@RunWith(AndroidJUnit4::class)
class LoginTest {
    @get:Rule val activityRule = ActivityScenarioRule(LoginActivity::class.java)

    @Test
    fun login_withValidCredentials_navigatesToHome() {
        onView(withId(R.id.emailField)).perform(typeText("user@example.com"))
        onView(withId(R.id.passwordField)).perform(typeText("password123"))
        onView(withId(R.id.loginButton)).perform(click())
        onView(withId(R.id.homeTitle)).check(matches(isDisplayed()))
    }
}
```

**When to use:** End-to-end UI flows that must run on a real device/emulator — login, checkout, navigation flows. Slow but highest confidence.

### 8.8 Compose Testing

```kotlin
androidTestImplementation("androidx.compose.ui:ui-test-junit4")
debugImplementation("androidx.compose.ui:ui-test-manifest")

@RunWith(AndroidJUnit4::class)
class LoginScreenTest {
    @get:Rule val composeTestRule = createComposeRule()

    @Test
    fun loginScreen_displaysErrorOnEmptySubmit() {
        composeTestRule.setContent {
            LoginScreen(viewModel = fakeViewModel)
        }
        composeTestRule.onNodeWithText("Sign In").performClick()
        composeTestRule.onNodeWithText("Email is required").assertIsDisplayed()
    }
}
```

Can also run with Robolectric for faster local execution (no device needed):
```kotlin
@RunWith(RobolectricTestRunner::class)  // runs Compose tests on JVM
class LoginScreenRobolectricTest { ... }
```

### 8.9 Testing Library Summary

| Library | Type | Runs on | Speed | Use for |
|---|---|---|---|---|
| JUnit 4/5 | Test runner | JVM | ⚡⚡⚡ | All unit tests |
| MockK | Mocking | JVM | ⚡⚡⚡ | Kotlin mocking |
| Mockito | Mocking | JVM | ⚡⚡⚡ | Java/mixed mocking |
| Turbine | Flow testing | JVM | ⚡⚡⚡ | StateFlow/Flow assertions |
| Robolectric | Android simulation | JVM | ⚡⚡ | Android component tests |
| Compose UI Test | UI | JVM (Robolectric) or device | ⚡⚡ | Composable tests |
| Espresso | UI automation | Device/emulator | ⚡ | E2E user flows |
| UI Automator | System UI | Device/emulator | ⚡ | Cross-app, system UI |

---

## 9. Gradle — Overview & Dependency Management

### 9.1 Build File Structure (Kotlin DSL — Modern Standard)

```
project/
├── settings.gradle.kts       ← module includes, plugin management
├── build.gradle.kts          ← top-level (plugins, clean task)
├── gradle/
│   ├── libs.versions.toml    ← version catalog (centralized versions)
│   └── wrapper/gradle-wrapper.properties
└── app/
    └── build.gradle.kts      ← app module config
```

```kotlin
// settings.gradle.kts
pluginManagement {
    repositories {
        google(); mavenCentral(); gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositories { google(); mavenCentral() }
}
rootProject.name = "MyApp"
include(":app", ":feature:home", ":core:network")
```

```kotlin
// gradle/libs.versions.toml — Version Catalog (use this for all new projects)
[versions]
kotlin = "2.0.0"
hilt = "2.51.1"
room = "2.6.1"
coil = "3.0.4"

[libraries]
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
coil-compose = { group = "io.coil-kt.coil3", name = "coil-compose", version.ref = "coil" }

[plugins]
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.hilt)
    alias(libs.plugins.kotlin.serialization)
}
dependencies {
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    implementation(libs.coil.compose)
}
```

### 9.2 Dependency Configurations

```kotlin
// implementation — not exposed to consumers of this module (most common)
implementation("com.squareup.okhttp3:okhttp:4.12.0")

// api — exposed to consumers (use sparingly — creates tighter coupling)
api("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0")

// testImplementation — only for unit test source set
testImplementation("io.mockk:mockk:1.13.10")

// androidTestImplementation — only for instrumented test source set
androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")

// debugImplementation — only in debug builds
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")

// releaseImplementation — only in release builds
releaseImplementation("com.example:crash-reporter:1.0.0")

// ksp — annotation processor (KSP, the modern alternative to kapt)
ksp("androidx.room:room-compiler:2.6.1")
```

### 9.3 Excluding Transitive Dependencies

When a library brings in a dependency you don't want (conflict or size):

```kotlin
implementation("com.some.library:library:1.0.0") {
    exclude(group = "com.google.code.gson", module = "gson")     // exclude Gson (you use Moshi)
    exclude(group = "org.jetbrains.kotlin", module = "kotlin-stdlib-jdk7")  // redundant in Kotlin 1.8+
}

// Exclude globally across all dependencies
configurations.all {
    exclude(group = "commons-logging", module = "commons-logging")
}
```

**Warning:** Only exclude transitive deps you're sure aren't needed at runtime. Excluding something that's actually required causes `ClassNotFoundException` at runtime, not at build time.

### 9.4 Selective Library Imports

Many libraries are split into modules — import only what you use:

```kotlin
// Media3 — don't import everything
implementation("androidx.media3:media3-exoplayer:1.3.1")    // core player
implementation("androidx.media3:media3-exoplayer-hls:1.3.1") // only if using HLS
// NOT: implementation("androidx.media3:media3:1.3.1")  — this doesn't exist; always module-specific

// Firebase — modular, import only what you need
implementation(platform("com.google.firebase:firebase-bom:32.7.0"))  // BOM manages versions
implementation("com.google.firebase:firebase-messaging-ktx")    // push only
implementation("com.google.firebase:firebase-analytics-ktx")    // analytics only
// NOT: all firebase modules at once

// Coil — core + optional format modules
implementation("io.coil-kt.coil3:coil-compose:3.0.4")
implementation("io.coil-kt.coil3:coil-gif:3.0.4")    // only if you show GIFs
// Don't add coil-svg, coil-video unless you actually use them

// OkHttp — logging interceptor debug-only
debugImplementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
// NOT in release — leaks request/response bodies including auth tokens
```

---

## 10. App Size Management

### 10.1 R8 (Replaces ProGuard)

R8 is the default shrinker/obfuscator in AGP 3.4+. ProGuard is no longer the default — R8 uses ProGuard-compatible rules.

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true        // code shrinking + obfuscation
            isShrinkResources = true      // resource shrinking
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
        debug {
            isMinifyEnabled = false       // keep debug fast
        }
    }
}
```

R8 does four things:
1. **Code shrinking** — removes unused classes, methods, fields
2. **Resource shrinking** — removes unused drawables, strings, layouts
3. **Obfuscation** — renames classes to `a`, `b`, `c` (reduces DEX size)
4. **Optimization** — inlines methods, removes dead branches

**Common ProGuard/R8 rules:**
```proguard
# Keep data classes used with Gson (if using Gson — Kotlin Serialization doesn't need this)
-keepclassmembers class com.example.data.** { *; }

# Keep Retrofit interfaces
-keepclasseswithmembers interface com.example.api.** { *; }

# Keep enum names (for TypeConverters)
-keepclassmembers enum * { public static **[] values(); public static ** valueOf(java.lang.String); }

# Keep Parcelable (if using)
-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}

# Don't warn about missing classes you don't use
-dontwarn okio.**
-dontwarn javax.annotation.**
```

### 10.2 Android App Bundle (AAB) vs APK

```kotlin
// Build AAB (required for Play Store since Aug 2021)
// Android Studio: Build > Generate Signed Bundle/APK > Android App Bundle

// AAB benefits:
// - Play Store delivers only the APKs needed for each device:
//   - Right ABI (arm64-v8a, x86_64)
//   - Right screen density (hdpi, xhdpi, xxhdpi)
//   - Right language (only the languages the device is set to)
// Typical reduction: 15–35% smaller download vs universal APK
```

### 10.3 Dynamic Feature Modules (Download on Demand)

```kotlin
// settings.gradle.kts — declare as dynamic feature
include(":feature:ar-viewer")

// feature/ar-viewer/build.gradle.kts
plugins { id("com.android.dynamic-feature") }
android { ... }
dependencies { implementation(project(":app")) }

// feature/ar-viewer/src/main/AndroidManifest.xml
<dist:module dist:instant="false">
    <dist:delivery>
        <dist:on-demand/>    <!-- not installed at first launch -->
    </dist:delivery>
    <dist:fusing dist:include="true"/>
</dist:module>

// Trigger download at runtime
val splitInstallManager = SplitInstallManagerFactory.create(context)
val request = SplitInstallRequest.newBuilder()
    .addModule("ar-viewer")
    .build()
splitInstallManager.startInstall(request)
    .addOnSuccessListener { /* module downloaded */ }
    .addOnFailureListener { /* handle error */ }
```

**Size impact:** AR libraries can be 10–20MB. Moving them to on-demand means users who never use AR never download them.

### 10.4 Analyze APK Size

```
Android Studio → Build → Analyze APK
- See breakdown by: classes.dex, res/, lib/ (native .so), assets/
- Identify: which library contributes most to DEX size
- Check: unused resources that R8 missed
```

**Tools:**
- **`./gradlew :app:bundleRelease` + `bundletool`** — test AAB locally
- **`./gradlew :app:dependencies`** — see full dependency tree
- **Android Studio APK Analyzer** — visual breakdown of APK contents
- **Baseline Profiles** — not size reduction but improves startup performance

### 10.5 Quick Wins for Size Reduction

```kotlin
// 1. Use WebP instead of PNG/JPG (30-50% smaller)
// Convert in Android Studio: right-click image → Convert to WebP

// 2. Use vector drawables for icons (no density variants needed)
// src/main/res/drawable/ic_home.xml  ← one file, scales to any size

// 3. Restrict ABI splits (if not using Play AAB)
android {
    defaultConfig {
        ndk { abiFilters.addAll(listOf("arm64-v8a", "x86_64")) }
        // drops armeabi-v7a, x86 — covers ~99% of active devices
    }
}

// 4. Keep only needed languages
android {
    defaultConfig {
        resourceConfigurations.addAll(listOf("en", "es", "fr"))
        // only include string resources for these locales
    }
}

// 5. Strip unused resources explicitly
android { buildTypes { release { isShrinkResources = true } } }
```

---

## 11. Common Misunderstandings & Pitfalls

**❌ Using Gson with Kotlin data classes**
Gson bypasses Kotlin's null checks — a missing JSON field for a non-null property gives you `null` at runtime instead of a parse error. Switch to Moshi (codegen) or Kotlin Serialization.

**❌ `kapt` for annotation processing**
`kapt` (Kotlin annotation processing) is deprecated. Use `ksp` (Kotlin Symbol Processing) — it's significantly faster and the ecosystem has migrated. Replace all `kapt(...)` with `ksp(...)`.

**❌ Adding `implementation` instead of `debugImplementation` for logging/debug tools**
LeakCanary, OkHttp logging interceptor, and Chucker should be `debugImplementation` or `releaseImplementation("no-op")`. Shipping debug tools in release leaks sensitive data and adds size.

**❌ Not using Version Catalog**
Hardcoding versions in each module's build file leads to version drift across modules. Use `gradle/libs.versions.toml` — single source of truth for all dependency versions.

**❌ `api` instead of `implementation` for module dependencies**
`api` exposes the dependency to consumers of your module, creating accidental coupling. Use `implementation` unless you explicitly need consumers to access the dependency's API.

**❌ Not keeping R8 mappings**
R8 obfuscates class/method names. Without the mapping file, a production crash stacktrace is unreadable. Always save the `mapping.txt` from each release build. Firebase Crashlytics can do this automatically.

**❌ Mockito mocking final classes without `mockito-inline`**
Kotlin classes are `final` by default. Mockito can't mock them without the `mockito-inline` extension or the `@MockitoSettings(strictness = LENIENT)` workaround. MockK handles this without extra setup.

**❌ Writing Espresso tests for everything**
Espresso tests are slow (need emulator), flaky (timing-sensitive), and expensive to maintain. Write Espresso only for critical user journeys. Unit test the logic; Robolectric test the Android components; Espresso test the full flow.

---

## 12. Best Practices

- **Kotlin Serialization for new projects, Moshi for existing Kotlin** — both are null-safe; avoid Gson in new Kotlin code
- **`ksp` over `kapt`** — KSP is 2x faster; migrate all annotation processors
- **Version Catalog (`libs.versions.toml`)** — single source of dependency truth across all modules
- **`debugImplementation` for debug-only tools** — LeakCanary, logging interceptors, Chucker
- **AAB for Play Store releases** — required since Aug 2021; gives 15–35% smaller downloads
- **Dynamic feature modules for large optional features** — AR, video editor, scanner (features > 2MB)
- **Coil for Compose, Glide for View-based** — don't fight the grain
- **MockK for Kotlin unit tests, Turbine for Flow** — these two together cover 90% of unit testing needs
- **Robolectric for Android component tests** — faster than emulator, catches real Android bugs
- **R8 enabled in release always** — `isMinifyEnabled = true` + `isShrinkResources = true`
- **Save R8 mapping.txt per release** — essential for deobfuscating production crash stacktraces
- **Selective imports for large libraries** — Media3, Firebase, Coil all support modular imports; only add what you use
- **Exclude transitive deps carefully** — only when you're certain they're not needed at runtime

---

## 13. Interview Q&A

**Q1: Why would you choose Kotlin Serialization over Gson?**

> Gson uses reflection at runtime and bypasses Kotlin's null safety — a missing required field in JSON creates a null value in a non-null Kotlin property, which causes an NPE later rather than a parse error immediately. Kotlin Serialization is a compiler plugin that generates serializers at compile time — no reflection, enforces null safety at parse time, and supports Kotlin Multiplatform. For any new project I'd default to Kotlin Serialization. I'd keep Gson only in existing Java codebases or when dealing with very complex dynamic polymorphism that's easier to handle with Gson's flexible adapters.

---

**Q2: Hilt or Koin — which do you prefer and when?**

> Hilt for production Android apps — compile-time graph verification means a misconfigured dependency fails the build, not the first time a user triggers that code path at 2am. The setup cost is worth it for any app with multiple engineers. Koin for KMP projects (Koin 4+ supports multiplatform, Hilt doesn't), rapid prototyping where build speed matters, or smaller teams where Hilt's learning curve is a real friction point. The key tradeoff is Hilt catches errors at build time; Koin catches them at runtime when the dependency is first resolved.

---

**Q3: How do you reduce APK/AAB size?**

> Several layers: First, enable R8 with `isMinifyEnabled = true` and `isShrinkResources = true` — this removes unused code and resources. Second, use AAB over APK — Play Store delivers only the right ABI and density split for each device, saving 15–35%. Third, move large optional features (AR, video editor) to dynamic feature modules downloaded on demand. Fourth, import only the library modules you actually use — Media3 and Firebase are both modular. Fifth, use vector drawables for icons and WebP for photos. Finally, constrain ABI to `arm64-v8a` and `x86_64` which covers ~99% of devices and drops the 32-bit builds.

---

**Q4: When would you use Espresso vs Robolectric vs a unit test?**

> The test pyramid guides this. Unit tests — ViewModel logic, repository decisions, use case rules — run on JVM with MockK, are fast (milliseconds), and should be the majority. Robolectric sits in the middle — when I need to test something that involves the Android framework (Activity lifecycle, Fragment transactions, View states) but I don't want the overhead of an emulator. Espresso is for real end-to-end flows that must be verified on an actual device — login, checkout, critical navigation paths. Espresso tests are slow and flaky; I keep them to the minimum set of critical journeys, not comprehensive coverage.

---

**Q5: What's the difference between `implementation` and `api` in Gradle?**

> `implementation` adds a dependency that's available to this module but not exposed to modules that depend on it. `api` exposes the dependency to consumers — they can use the library's types in their own code without declaring the dependency themselves. The practical rule: use `implementation` for almost everything. Use `api` only when your module's public API surface includes types from the dependency — for example, if your `:core:network` module's `Repository` interface returns a `Response<T>` type from OkHttp, consumers need OkHttp types, so you'd `api` that. Overusing `api` creates tight coupling and slows incremental builds because a change in the transitive dependency forces recompilation of all downstream modules.

---

**Q6: How do you handle R8 obfuscation when debugging production crashes?**

> R8 obfuscates class and method names in release builds — a stacktrace from production shows `a.b.c()` instead of `UserRepository.fetchUser()`. The fix is to save the `mapping.txt` file generated by every release build. For crash tools like Firebase Crashlytics, I enable automatic mapping upload via the Firebase Gradle plugin — it uploads the mapping file with each release and deobfuscates stacktraces automatically in the Crashlytics dashboard. For manual deobfuscation, the `retrace` tool (from AGP) converts an obfuscated stacktrace back using the mapping file: `retrace mapping.txt stacktrace.txt`.

---

*Previous: [05 — Real-Time & Push](./05-realtime-push.md)*
*Next: [06 — Performance](./06-performance.md)*
