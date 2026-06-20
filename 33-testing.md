# 33 — Testing Strategies

> **Round type: LLD (primary)**
> **Focus:** Test pyramid for Android, ViewModel testing, Flow testing, Compose UI testing, screenshot testing, fakes vs mocks, test doubles — what senior engineers do vs juniors

---

## 1. Why This Matters

Testing is a seniority signal. Junior engineers write no tests or only UI tests. Senior engineers write a pyramid of fast unit tests, integration tests at the boundaries, and minimal UI tests. Interviewers ask: "How would you test this ViewModel?" — the answer reveals how you think about testability.

---

## 2. The Android Test Pyramid

```
          /─────────\
         /   E2E /   \        ← Slowest, most brittle, least
        /  UI Tests   \          (Espresso, UiAutomator)
       /───────────────\
      / Integration Tests\    ← Medium speed
     /  (Robolectric,     \      (DB, repository layer)
    /  AndroidX Test)      \
   /────────────────────────\
  /       Unit Tests         \  ← Fast, cheap, most of them
 /   (JUnit, MockK, Turbine)   \
/────────────────────────────────\

Target ratio: 70% unit : 20% integration : 10% E2E
```

---

## 3. Dependencies

```kotlin
// Unit tests (runs on JVM — fast)
testImplementation("junit:junit:4.13.2")
testImplementation("io.mockk:mockk:1.13.12")
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
testImplementation("app.cash.turbine:turbine:1.1.0")  // Flow testing
testImplementation("com.google.truth:truth:1.4.2")    // fluent assertions

// Android integration tests (runs on device/emulator — slower)
androidTestImplementation("androidx.test.ext:junit:1.2.1")
androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")
androidTestImplementation("androidx.compose.ui:ui-test-junit4")
androidTestImplementation("androidx.room:room-testing:2.6.1")

// Screenshot testing
testImplementation("app.cash.paparazzi:paparazzi:1.3.4")
```

---

## 4. ViewModel Testing

### 4.1 Fakes over Mocks

```kotlin
// ❌ Mock: too coupled to implementation details
val mockRepo = mockk<UserRepository>()
every { mockRepo.getUser(any()) } returns flowOf(testUser)

// ✅ Fake: explicit behavior, easier to maintain
class FakeUserRepository : UserRepository {
    var user: User? = null
    var shouldThrow = false
    val updateCalls = mutableListOf<User>()

    override fun observeUser(id: String) = flow {
        if (shouldThrow) throw IOException("network error")
        user?.let { emit(it) }
    }
    override suspend fun updateUser(user: User) {
        updateCalls.add(user)
        this.user = user
    }
}
```

**Why fakes:** Fakes are real implementations that you control. Mocks couple tests to implementation details — change a method signature and all mock setups break. Fakes express intent clearly.

### 4.2 TestCoroutineDispatcher + StandardTestDispatcher

```kotlin
class ProfileViewModelTest {

    // Replaces Dispatchers.Main with a test dispatcher
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val fakeRepo = FakeUserRepository()
    private lateinit var viewModel: ProfileViewModel

    @Before
    fun setUp() {
        fakeRepo.user = User(id = "1", name = "Alice", email = "alice@example.com")
        viewModel = ProfileViewModel(fakeRepo)
    }

    @Test
    fun `initial state shows user name`() = runTest {
        val state = viewModel.uiState.value
        assertEquals("Alice", state.name)
    }

    @Test
    fun `save updates name in state`() = runTest {
        viewModel.onNameChanged("Bob")
        viewModel.onSaveTapped()
        advanceUntilIdle()  // runs all pending coroutines

        assertEquals("Bob", viewModel.uiState.value.name)
        assertFalse(viewModel.uiState.value.isSaving)
    }

    @Test
    fun `save shows error on API failure`() = runTest {
        fakeRepo.shouldThrow = true
        viewModel.onSaveTapped()
        advanceUntilIdle()

        assertNotNull(viewModel.uiState.value.error)
    }
}

// MainDispatcherRule — replaces Dispatchers.Main for tests
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

### 4.3 Testing Flows with Turbine

```kotlin
@Test
fun `uiState emits loading then success`() = runTest {
    fakeRepo.delayMs = 500  // simulate network delay

    viewModel.uiState.test {
        val initial = awaitItem()
        assertFalse(initial.isLoading)

        viewModel.loadProfile("user1")

        val loading = awaitItem()
        assertTrue(loading.isLoading)

        val success = awaitItem()
        assertFalse(success.isLoading)
        assertEquals("Alice", success.name)

        cancelAndIgnoreRemainingEvents()
    }
}

// Turbine: test() opens a channel that collects Flow emissions
// awaitItem() waits for the next emission (suspending)
// awaitComplete() waits for flow to complete
// awaitError() waits for an error
// cancelAndIgnoreRemainingEvents() cleans up
```

### 4.4 Testing StateFlow (alternative without Turbine)

```kotlin
@Test
fun `search results update on query change`() = runTest {
    val results = mutableListOf<SearchState>()
    val job = launch { viewModel.searchState.toList(results) }

    viewModel.onQueryChanged("kotlin")
    advanceUntilIdle()

    assertEquals(3, results.size)
    assertTrue(results[0].isLoading)
    assertFalse(results.last().isLoading)
    assertEquals(10, results.last().results.size)

    job.cancel()
}
```

---

## 5. Repository / Use Case Testing

```kotlin
class UserRepositoryTest {
    private val fakeApi = FakeUserApi()
    private val db = Room.inMemoryDatabaseBuilder(
        ApplicationProvider.getApplicationContext(),
        AppDatabase::class.java
    ).allowMainThreadQueries().build()

    private val repository = UserRepositoryImpl(fakeApi, db.userDao())

    @Test
    fun `getUser emits cached then fresh`() = runTest {
        // Seed DB with stale data
        db.userDao().insert(User(id = "1", name = "Old Name"))
        fakeApi.user = User(id = "1", name = "New Name")

        repository.observeUser("1").test {
            // First emission: stale from DB
            assertEquals("Old Name", awaitItem().name)
            // Second emission: fresh from API
            assertEquals("New Name", awaitItem().name)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## 6. Room Database Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest {
    private lateinit var db: AppDatabase
    private lateinit var userDao: UserDao

    @Before
    fun setUp() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).allowMainThreadQueries().build()  // in-memory: no disk I/O
        userDao = db.userDao()
    }

    @After
    fun tearDown() = db.close()

    @Test
    fun insertAndRetrieve() = runTest {
        val user = UserEntity(id = "1", name = "Alice")
        userDao.insert(user)
        val result = userDao.getUser("1")
        assertEquals("Alice", result?.name)
    }

    @Test
    fun `observeUser emits on update`() = runTest {
        userDao.insert(UserEntity("1", "Alice"))
        userDao.observeUser("1").test {
            assertEquals("Alice", awaitItem()?.name)
            userDao.update(UserEntity("1", "Bob"))
            assertEquals("Bob", awaitItem()?.name)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

---

## 7. Compose UI Testing

```kotlin
class FeedScreenTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `posts display correctly`() {
        val fakePosts = listOf(Post("1", "Hello"), Post("2", "World"))
        composeTestRule.setContent {
            FeedScreen(posts = fakePosts, onPostClick = {})
        }

        composeTestRule.onNodeWithText("Hello").assertIsDisplayed()
        composeTestRule.onNodeWithText("World").assertIsDisplayed()
    }

    @Test
    fun `empty state shows message`() {
        composeTestRule.setContent {
            FeedScreen(posts = emptyList(), onPostClick = {})
        }
        composeTestRule.onNodeWithText("No posts yet").assertIsDisplayed()
    }

    @Test
    fun `click on post calls callback`() {
        var clickedId: String? = null
        composeTestRule.setContent {
            PostCard(post = Post("1", "Test"), onClick = { clickedId = it })
        }
        composeTestRule.onNodeWithText("Test").performClick()
        assertEquals("1", clickedId)
    }

    @Test
    fun `loading state shows spinner`() {
        composeTestRule.setContent {
            FeedScreen(isLoading = true, posts = emptyList(), onPostClick = {})
        }
        composeTestRule.onNodeWithTag("loading_spinner").assertIsDisplayed()
    }
}
```

### 7.1 Semantics for Testability

```kotlin
// Tag non-unique nodes for testing
Box(modifier = Modifier.semantics { testTag = "loading_spinner" }) {
    CircularProgressIndicator()
}

// Meaningful content descriptions for assertions
IconButton(
    onClick = { },
    modifier = Modifier.semantics { contentDescription = "Add to favorites" }
) { Icon(...) }

// In test:
composeTestRule.onNodeWithContentDescription("Add to favorites").performClick()
```

---

## 8. Screenshot Testing with Paparazzi

```kotlin
class ProfileCardScreenshotTest {

    @get:Rule
    val paparazzi = Paparazzi(
        deviceConfig = DeviceConfig.PIXEL_6,
        theme = "Theme.MyApp"
    )

    @Test
    fun `profile card default state`() {
        paparazzi.snapshot {
            ProfileCard(
                user = User(name = "Alice", email = "alice@example.com"),
                isPremium = false
            )
        }
    }

    @Test
    fun `profile card premium state`() {
        paparazzi.snapshot {
            ProfileCard(
                user = User(name = "Alice", email = "alice@example.com"),
                isPremium = true
            )
        }
    }
}
// Run: ./gradlew recordPaparazziDebug  → creates golden images
// CI: ./gradlew verifyPaparazziDebug   → fails if screenshots differ
```

Screenshot tests catch visual regressions without running on a device. Paparazzi runs on JVM.

---

## 9. Network Layer Testing

```kotlin
class PostApiTest {

    private val mockWebServer = MockWebServer()
    private lateinit var api: PostApi

    @Before
    fun setUp() {
        mockWebServer.start()
        api = Retrofit.Builder()
            .baseUrl(mockWebServer.url("/"))
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            .build()
            .create(PostApi::class.java)
    }

    @After
    fun tearDown() = mockWebServer.shutdown()

    @Test
    fun `getPosts parses response correctly`() = runTest {
        mockWebServer.enqueue(
            MockResponse()
                .setBody("""{"posts": [{"id": "1", "title": "Hello"}], "nextPage": 2}""")
                .setResponseCode(200)
        )
        val response = api.getPosts(page = 1)
        assertEquals(1, response.posts.size)
        assertEquals("Hello", response.posts[0].title)
        assertEquals(2, response.nextPage)
    }

    @Test
    fun `getPosts throws on 429 Too Many Requests`() = runTest {
        mockWebServer.enqueue(MockResponse().setResponseCode(429))
        assertThrows<HttpException> { api.getPosts(page = 1) }
    }
}
```

---

## 10. Test Doubles Decision Guide

| Type | What it is | Use when |
|---|---|---|
| **Fake** | Working implementation with simplified behavior | Best default — use for repositories, DAOs, APIs |
| **Mock** | Programmatic stub (MockK/Mockito) | OK for simple collaborators with few interactions |
| **Stub** | Returns hardcoded values | Very simple cases |
| **Spy** | Real object that records calls | Rarely needed, verify side effects |
| **Dummy** | Placeholder, never called | Satisfying dependency signatures |

**Rule:** Prefer fakes. Use mocks only when creating a full fake is disproportionately expensive.

---

## 11. What to Test vs What Not to Test

| ✅ Test | ❌ Don't Test |
|---|---|
| ViewModel state transitions | Android framework classes (Activity, Fragment) |
| Repository fetch + cache logic | Hilt injection setup |
| Use case business logic | Auto-generated Room code |
| Flow emission sequences (Turbine) | Third-party library behavior |
| Error/empty/loading states | Trivial getters/setters |
| Navigation events emitted by ViewModel | One-line wrapper methods |
| Paging source load results | Compose UI internals |

---

## 12. Best Practices

- **Fakes over mocks** for repositories — intent clearer, less brittle
- **`runTest` + `advanceUntilIdle()`** for all ViewModel coroutine tests
- **Turbine** for sequential Flow emission assertions
- **In-memory Room** for DAO tests — no disk I/O, test isolation
- **`MockWebServer`** for API tests — test actual JSON parsing, not just mock responses
- **Compose UI tests for interaction** — click flows, navigation triggers, not layout details
- **Screenshot tests for visual regression** — Paparazzi for JVM speed, don't block CI
- **`MainDispatcherRule`** to replace `Dispatchers.Main` in every ViewModel test file
- **One assertion per test** — tests that assert one thing are easier to debug
- **Test the contract, not the implementation** — if you change a private method and tests break, your tests are too coupled

---

*Previous: [32 — Paging 3](./32-paging.md)*
*Next: [34 — Kotlin Multiplatform (KMP)](./34-kmp.md)*
