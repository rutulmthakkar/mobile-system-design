# 21 — Design Patterns & Diagrams

> **Round type: LLD (primary)**
> **Focus:** Creational, structural, and behavioral patterns with class diagrams, sequence diagrams, and concrete Android/Kotlin implementations — optimized for LLD interview rounds

---

## 1. Why This Matters in Interviews

LLD rounds often start with "design a [library/component]" and expect you to draw class diagrams and explain which patterns you're applying. Interviewers test whether you know *when* to apply a pattern, not just what it is. Naming patterns correctly and knowing their trade-offs signals senior-level thinking.

**Common interview angles:**
- "Walk me through the Repository pattern — draw the class diagram"
- "How does the Observer pattern apply to StateFlow?"
- "What pattern would you use to add retry logic to a network call without modifying it?"
- "How would you implement a plugin system? What pattern is that?"
- "Design an image loading library — which patterns would you use?"

---

## 2. How to Read/Write Class Diagrams

```
Box:       Class or Interface
─────────────────────────────
 ClassName
─────────────────────────────
 - privateField: Type
 # protectedField: Type
 + publicField: Type
─────────────────────────────
 + publicMethod(): ReturnType
 - privateMethod(param: Type)

Relationships:
──────────────────────────────────────────────────────
  A ──────▷ B    "A depends on B" (uses/association)
  A ──────▶ B    "A has a B" (unidirectional association)
  A ◀──────▶ B    "A and B know each other" (bidirectional)
  A ━━━━━━▷ B    "A implements B" (realization/implements)
  A ━━━━━━━ B    "A extends B" (inheritance)
  A ◆──────▶ B   "A contains B" (composition: B can't exist without A)
  A ◇──────▶ B   "A has B" (aggregation: B can exist without A)
  A ─ ─ ─ ▷ B   "A creates B" (dependency — creates instances)
  1      0..*   Multiplicity (one A to many B)
```

---

## 3. Creational Patterns

### 3.1 Singleton — One instance globally

**Intent:** Ensure a class has only one instance. Provide a global access point.

**Class diagram:**
```
┌──────────────────────────────┐
│        Database              │
├──────────────────────────────┤
│ - instance: Database         │
├──────────────────────────────┤
│ - Database()                 │
│ + getInstance(): Database    │
│ + query(sql: String): List   │
└──────────────────────────────┘
        ↑ private constructor
```

**Kotlin implementation:**
```kotlin
// Kotlin object — thread-safe singleton by language spec
object Analytics {
    fun track(event: String, props: Map<String, Any> = emptyMap()) {
        FirebaseAnalytics.getInstance(applicationContext).logEvent(event, props.toBundle())
    }
}

// Lazy singleton with initialization parameter
class Database private constructor(context: Context) {
    companion object {
        @Volatile private var instance: Database? = null
        fun getInstance(context: Context): Database =
            instance ?: synchronized(this) {
                instance ?: Database(context.applicationContext).also { instance = it }
            }
    }
}

// With Hilt — preferred over manual singletons
@Provides @Singleton
fun provideDatabase(@ApplicationContext ctx: Context): AppDatabase =
    Room.databaseBuilder(ctx, AppDatabase::class.java, "app.db").build()
```

**When to use:** Shared resources that are expensive to create — database, HTTP client, analytics instance, image loader.
**When NOT to use:** Anywhere you need testability — singletons are notoriously hard to mock/replace. Prefer DI with `@Singleton` scope instead of manual singletons.

---

### 3.2 Factory Method — Delegate object creation to subclasses

**Intent:** Define an interface for creating an object, but let subclasses decide which class to instantiate.

**Class diagram:**
```
┌───────────────────┐         ┌────────────────────┐
│  «interface»      │         │  «interface»        │
│  NotificationFactory│       │  Notification       │
├───────────────────┤         ├────────────────────┤
│+create(): Notif.  │────────▷│+show()              │
└────────▲──────────┘         │+dismiss()           │
         │                    └────────▲────────────┘
    ┌────┴─────────────┐               │
    │                  │        ┌──────┴──────┐
┌───┴────────┐  ┌──────┴──────┐│SnackNotif.  │
│PushFactory │  │InAppFactory ││ToastNotif.  │
├────────────┤  ├─────────────┤└─────────────┘
│+create()   │  │+create()    │
└────────────┘  └─────────────┘
```

**Kotlin implementation:**
```kotlin
// Interface
interface Notification {
    fun show(message: String)
    fun dismiss()
}

// Concrete products
class PushNotification(private val context: Context) : Notification {
    override fun show(message: String) { /* show push notification */ }
    override fun dismiss() { /* cancel notification */ }
}

class InAppBanner : Notification {
    override fun show(message: String) { /* show in-app banner */ }
    override fun dismiss() { /* hide banner */ }
}

// Factory
interface NotificationFactory {
    fun create(): Notification
}

class PushNotificationFactory(private val context: Context) : NotificationFactory {
    override fun create() = PushNotification(context)
}
class InAppBannerFactory : NotificationFactory {
    override fun create() = InAppBanner()
}

// Usage — caller doesn't know concrete type
fun showAlert(message: String, factory: NotificationFactory) {
    val notification = factory.create()
    notification.show(message)
}

// In practice — often simplified as a companion object or top-level function
fun createAnalytics(debug: Boolean): Analytics = if (debug) DebugAnalytics() else FirebaseAnalytics()
```

**Android examples:** `LayoutInflater`, `ViewModelProvider.Factory`, `WorkerFactory` (Hilt), `MediaSourceFactory` (ExoPlayer).

---

### 3.3 Builder — Step-by-step object construction

**Intent:** Separate the construction of a complex object from its representation.

**Class diagram:**
```
┌────────────────────────────────┐
│  ImageRequest.Builder          │
├────────────────────────────────┤
│ - data: Any?                   │
│ - size: Size?                  │
│ - transformations: List        │
│ - cachePolicy: CachePolicy     │
├────────────────────────────────┤
│ + data(Any): Builder           │
│ + size(Int, Int): Builder      │
│ + transformations(...): Builder│
│ + build(): ImageRequest        │ ──▶  ImageRequest
└────────────────────────────────┘
```

**Kotlin implementation:**
```kotlin
// Kotlin DSL-style builder (idiomatic)
data class OkHttpRequest(
    val url: String,
    val method: String,
    val headers: Map<String, String>,
    val body: String?,
    val timeoutMs: Long
) {
    class Builder(private val url: String) {
        private var method = "GET"
        private val headers = mutableMapOf<String, String>()
        private var body: String? = null
        private var timeoutMs = 30_000L

        fun method(method: String) = apply { this.method = method }
        fun header(name: String, value: String) = apply { headers[name] = value }
        fun body(body: String) = apply { this.body = body }
        fun timeout(ms: Long) = apply { this.timeoutMs = ms }

        fun build() = OkHttpRequest(url, method, headers.toMap(), body, timeoutMs)
    }
}

// Usage
val request = OkHttpRequest.Builder("https://api.example.com/users")
    .method("POST")
    .header("Authorization", "Bearer $token")
    .body(Json.encodeToString(user))
    .timeout(15_000L)
    .build()

// Kotlin apply — lightweight builder alternative
data class AlertDialogConfig(
    val title: String = "",
    val message: String = "",
    val positiveText: String = "OK",
    val negativeText: String = "Cancel",
    val onPositive: () -> Unit = {}
)
fun dialogConfig(block: AlertDialogConfig.() -> Unit) = AlertDialogConfig().apply(block)
val config = dialogConfig {
    title = "Delete item?"
    message = "This cannot be undone"
    positiveText = "Delete"
    onPositive = { viewModel.deleteItem() }
}
```

**Android examples:** `OkHttpClient.Builder`, `NotificationCompat.Builder`, `AlertDialog.Builder`, `Room.databaseBuilder()`, `Retrofit.Builder`, Media3 `ExoPlayer.Builder`.

---

## 4. Structural Patterns

### 4.1 Repository — Abstract data sources

**Intent:** Encapsulate data access logic, provide a clean domain API regardless of source (network/cache/DB).

**Class diagram:**
```
┌─────────────────────┐
│  «interface»        │
│  UserRepository     │
├─────────────────────┤
│+getUser(id): Flow   │
│+saveUser(User)      │
│+deleteUser(id)      │
└────────────▲────────┘
             │ implements
┌────────────┴────────────────────────────────┐
│  UserRepositoryImpl                          │
├─────────────────────────────────────────────┤
│ - api: UserApi                               │
│ - dao: UserDao                               │
│ - prefs: DataStore                           │
├─────────────────────────────────────────────┤
│ +getUser(id): Flow<User>                     │
│   → emit from dao, refresh from api          │
└──────────┬──────────┬──────────┬────────────┘
           │          │          │
      ┌────┴────┐ ┌───┴────┐ ┌──┴──────┐
      │ UserApi │ │UserDao │ │DataStore│
      └─────────┘ └────────┘ └─────────┘
```

**Kotlin implementation:**
```kotlin
interface UserRepository {
    fun observeUser(id: String): Flow<User>
    suspend fun refreshUser(id: String)
    suspend fun updateUser(user: User): Result<Unit>
}

class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    private val dao: UserDao,
    private val prefs: DataStore<UserPrefs>
) : UserRepository {

    override fun observeUser(id: String): Flow<User> = flow {
        // Always emit cached first
        dao.getUser(id)?.let { emit(it.toDomain()) }
        // Refresh from network
        refreshUser(id)
    }.flatMapLatest {
        dao.observeUser(id).map { it.toDomain() }  // then observe DB reactively
    }

    override suspend fun refreshUser(id: String) {
        try {
            val user = api.getUser(id)
            dao.upsert(user.toEntity())
        } catch (e: Exception) {
            Timber.e(e, "Failed to refresh user $id")
        }
    }

    override suspend fun updateUser(user: User): Result<Unit> = runCatching {
        api.updateUser(user.toDto())
        dao.upsert(user.toEntity())
    }
}
```

**Why interface:** The ViewModel depends on `UserRepository` interface, not `UserRepositoryImpl`. Tests inject `FakeUserRepository`. The implementation can change (swap Room for SQLDelight) without touching ViewModel code.

---

### 4.2 Adapter — Convert incompatible interfaces

**Intent:** Allow classes with incompatible interfaces to work together by wrapping one with an adapter.

**Class diagram:**
```
Client ──▶ «interface» Target          Adaptee
              + request()        ┌──────────────┐
                   ▲             │ LegacyService │
                   │             ├──────────────┤
           ┌───────┴──────┐      │ +oldMethod() │
           │   Adapter    │──────┤              │
           ├──────────────┤      └──────────────┘
           │ - adaptee    │
           ├──────────────┤
           │ +request()   │
           └──────────────┘
```

**Kotlin implementation:**
```kotlin
// RecyclerView.Adapter is a classic Adapter pattern
// Adapts your data (List<Post>) to RecyclerView's interface

class PostAdapter(
    private var posts: List<Post>
) : RecyclerView.Adapter<PostAdapter.ViewHolder>() {

    class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
        val title: TextView = view.findViewById(R.id.title)
        val body: TextView = view.findViewById(R.id.body)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_post, parent, false)
        return ViewHolder(view)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val post = posts[position]
        holder.title.text = post.title
        holder.body.text = post.body
    }

    override fun getItemCount() = posts.size
}

// Another example: adapting a legacy callback API to a Flow
class LegacyLocationService {
    fun requestLocation(callback: (Lat: Double, Lng: Double) -> Unit) { /* ... */ }
}

// Adapter: legacy callback → coroutine suspend function
suspend fun LegacyLocationService.awaitLocation(): Pair<Double, Double> =
    suspendCancellableCoroutine { cont ->
        requestLocation { lat, lng -> cont.resume(Pair(lat, lng)) }
    }
```

---

### 4.3 Decorator — Add behavior without subclassing

**Intent:** Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing.

**Class diagram:**
```
«interface» Component       Component implements Component
    + operation()        ┌──────────────────────────────┐
        ▲                │  ConcreteDecorator            │
        │                ├──────────────────────────────┤
   ┌────┴──────┐         │ - wrappee: Component          │
   │  Concrete │         ├──────────────────────────────┤
   │  Component│         │ + operation() {               │
   └───────────┘         │     extra behavior            │
        ▲                │     wrappee.operation()       │
        │                │   }                           │
   Decorator             └──────────────────────────────┘
```

**Kotlin implementation:**
```kotlin
// OkHttp Interceptors are the Decorator pattern applied to HTTP requests
// Each interceptor wraps the chain and adds behavior without knowing what comes next

interface Repository {
    suspend fun getProducts(): List<Product>
}

class NetworkRepository(private val api: ProductApi) : Repository {
    override suspend fun getProducts() = api.getProducts()
}

// Decorator: add caching without modifying NetworkRepository
class CachingRepository(
    private val wrapped: Repository,
    private val cache: Cache<List<Product>>
) : Repository {
    override suspend fun getProducts(): List<Product> {
        cache.get("products")?.let { return it }  // cache hit
        val products = wrapped.getProducts()        // delegate to wrapped
        cache.put("products", products)
        return products
    }
}

// Decorator: add logging without modifying other repositories
class LoggingRepository(private val wrapped: Repository) : Repository {
    override suspend fun getProducts(): List<Product> {
        Timber.d("Fetching products...")
        val result = wrapped.getProducts()
        Timber.d("Fetched ${result.size} products")
        return result
    }
}

// Compose decorators
val repository: Repository = LoggingRepository(
    CachingRepository(
        NetworkRepository(api),
        memoryCache
    )
)
```

**Android examples:** `OkHttp Interceptors`, `ContextWrapper`, `InputStreamDecorator`.

---

### 4.4 Facade — Simplified interface to a complex system

**Intent:** Provide a simplified interface to a complex subsystem.

**Class diagram:**
```
              ┌─────────────┐
Client ──────▶│    Facade   │
              ├─────────────┤
              │+playMedia() │──▶ ExoPlayer
              │+pauseMedia()│──▶ AudioFocusManager
              │+stopMedia() │──▶ MediaSession
              └─────────────┘──▶ NotificationManager
```

**Kotlin implementation:**
```kotlin
// MediaPlaybackFacade hides the complexity of ExoPlayer + AudioFocus + MediaSession
class MediaPlaybackFacade @Inject constructor(
    private val player: ExoPlayer,
    private val audioFocusManager: AudioFocusManager,
    private val mediaSession: MediaSession,
    private val notificationManager: NotificationManager
) {
    // Simple API for callers — they don't need to know about any subsystem
    fun play(url: String) {
        if (!audioFocusManager.requestFocus()) return
        player.setMediaItem(MediaItem.fromUri(url))
        player.prepare()
        player.play()
        notificationManager.showNowPlaying()
    }

    fun pause() {
        player.pause()
        audioFocusManager.abandonFocus()
    }

    fun stop() {
        player.stop()
        audioFocusManager.abandonFocus()
        notificationManager.cancelNowPlaying()
    }
}

// ViewModel uses the simple facade — unaware of subsystem complexity
class PlayerViewModel(private val playback: MediaPlaybackFacade) : ViewModel() {
    fun onPlayTapped(url: String) = playback.play(url)
    fun onPauseTapped() = playback.pause()
}
```

---

## 5. Behavioral Patterns

### 5.1 Observer — Subscribe to state changes

**Intent:** Define a one-to-many dependency so all observers are notified automatically when the subject changes.

**Class diagram:**
```
«interface» Observable            «interface» Observer
  + addObserver(Observer)              + onUpdate(data)
  + removeObserver(Observer)
  + notifyObservers()
        ▲                                    ▲
        │                                    │
 ConcreteSubject        ConcreteObserver1  ConcreteObserver2
 (StateFlow, LiveData)  (Composable)       (Analytics)
```

**Kotlin — how it maps to coroutines:**
```kotlin
// StateFlow IS the Observer pattern — subject is MutableStateFlow,
// observers are coroutines collecting it

// The pattern explicitly:
interface Observer<T> {
    fun onUpdate(data: T)
}

class EventBus<T> {
    private val observers = CopyOnWriteArrayList<Observer<T>>()
    private val _flow = MutableSharedFlow<T>(extraBufferCapacity = 64)
    val flow = _flow.asSharedFlow()

    fun subscribe(observer: Observer<T>) = observers.add(observer)
    fun unsubscribe(observer: Observer<T>) = observers.remove(observer)

    fun publish(event: T) {
        observers.forEach { it.onUpdate(event) }
        _flow.tryEmit(event)
    }
}

// In practice: use StateFlow/SharedFlow — they ARE the Observer pattern
// StateFlow = subject with current value
// Coroutine collect{} = observer registration
// emit/update = notifyObservers

class CartViewModel : ViewModel() {
    private val _cartState = MutableStateFlow(CartState())
    val cartState: StateFlow<CartState> = _cartState  // observable subject

    fun addItem(item: Item) {
        _cartState.update { it.copy(items = it.items + item) }  // notifyObservers
    }
}

// Compose composable = observer
val state by viewModel.cartState.collectAsStateWithLifecycle()
```

---

### 5.2 Strategy — Interchangeable algorithms

**Intent:** Define a family of algorithms, encapsulate each one, make them interchangeable.

**Class diagram:**
```
Context ──────▶ «interface» Strategy
 - strategy         + execute(data): Result
 + setStrategy()
 + performAction()       ▲
                   ┌─────┴──────┐
             StrategyA       StrategyB
             (cache-first)   (network-first)
```

**Kotlin implementation:**
```kotlin
// Fetch strategy — interchangeable algorithms for getting data
fun interface FetchStrategy<T> {
    suspend fun fetch(key: String, networkFetch: suspend () -> T): T
}

// Concrete strategies
val cacheFirstStrategy = FetchStrategy<Product> { key, networkFetch ->
    cache.get(key) ?: networkFetch().also { cache.put(key, it) }
}

val networkFirstStrategy = FetchStrategy<Product> { key, networkFetch ->
    try {
        networkFetch().also { cache.put(key, it) }
    } catch (e: Exception) {
        cache.get(key) ?: throw e
    }
}

val staleWhileRevalidateStrategy = FetchStrategy<Product> { key, networkFetch ->
    val cached = cache.get(key)
    // Launch background refresh
    coroutineScope.launch { 
        try { cache.put(key, networkFetch()) } catch (e: Exception) { }
    }
    cached ?: networkFetch()
}

// Context that uses the strategy
class ProductRepository(private var strategy: FetchStrategy<Product>) {
    // Strategy can be swapped at runtime
    fun setStrategy(strategy: FetchStrategy<Product>) { this.strategy = strategy }

    suspend fun getProduct(id: String): Product =
        strategy.fetch(id) { api.getProduct(id) }
}

// Swap strategy based on network condition
val repo = ProductRepository(if (isOnline) networkFirstStrategy else cacheFirstStrategy)
```

---

### 5.3 Command — Encapsulate requests as objects

**Intent:** Encapsulate a request as an object, allowing undo/redo, queuing, and logging.

**Class diagram:**
```
Invoker ──▶ «interface» Command
  - history[]        + execute()
  + execute()        + undo()
  + undo()
                         ▲
                ┌────────┴────────┐
          AddItemCommand   DeleteItemCommand
          - receiver: Cart   - receiver: Cart
          - item: Item        - id: String
          + execute()         + execute()
          + undo()            + undo()
```

**Kotlin implementation:**
```kotlin
interface Command {
    fun execute()
    fun undo()
}

class AddItemCommand(
    private val cart: Cart,
    private val item: CartItem
) : Command {
    override fun execute() = cart.addItem(item)
    override fun undo() = cart.removeItem(item.id)
}

class UpdateQuantityCommand(
    private val cart: Cart,
    private val itemId: String,
    private val newQty: Int
) : Command {
    private var previousQty = 0

    override fun execute() {
        previousQty = cart.getQuantity(itemId)
        cart.setQuantity(itemId, newQty)
    }

    override fun undo() = cart.setQuantity(itemId, previousQty)
}

// Invoker with history
class CartCommandExecutor {
    private val history = ArrayDeque<Command>()

    fun execute(command: Command) {
        command.execute()
        history.addLast(command)
    }

    fun undo() {
        history.removeLastOrNull()?.undo()
    }

    fun canUndo() = history.isNotEmpty()
}

// Usage
val executor = CartCommandExecutor()
executor.execute(AddItemCommand(cart, shirt))
executor.execute(UpdateQuantityCommand(cart, shirt.id, 2))
executor.undo()  // reverts to quantity 1
executor.undo()  // removes shirt from cart
```

**Android use cases:** Text editor undo/redo, drawing app history, form wizard with back-tracking.

---

### 5.4 State — Object changes behavior based on internal state

**Intent:** Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

**Kotlin implementation:**
```kotlin
// Sealed class = State pattern in Kotlin
sealed interface PaymentState {
    object Idle : PaymentState
    object Processing : PaymentState
    data class Success(val transactionId: String) : PaymentState
    data class Error(val message: String, val retryable: Boolean) : PaymentState
}

class PaymentViewModel : ViewModel() {
    private val _state = MutableStateFlow<PaymentState>(PaymentState.Idle)
    val state = _state.asStateFlow()

    fun processPayment(request: PaymentRequest) {
        _state.value = PaymentState.Processing

        viewModelScope.launch {
            _state.value = try {
                val result = paymentApi.charge(request)
                PaymentState.Success(result.transactionId)
            } catch (e: NetworkException) {
                PaymentState.Error("Network error", retryable = true)
            } catch (e: DeclinedException) {
                PaymentState.Error("Card declined", retryable = false)
            }
        }
    }
}

// Composable reacts to state — different UI for each state
@Composable
fun PaymentScreen(viewModel: PaymentViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when (state) {
        is PaymentState.Idle -> PaymentForm(onSubmit = viewModel::processPayment)
        is PaymentState.Processing -> LoadingSpinner()
        is PaymentState.Success -> SuccessScreen((state as PaymentState.Success).transactionId)
        is PaymentState.Error -> ErrorScreen(
            message = (state as PaymentState.Error).message,
            showRetry = (state as PaymentState.Error).retryable,
            onRetry = { /* retry */ }
        )
    }
}
```

---

### 5.5 Chain of Responsibility — Pass request along handler chain

**Intent:** Pass a request along a chain of handlers. Each handler decides to handle it or pass it further.

**Kotlin implementation:**
```kotlin
// OkHttp Interceptors ARE the Chain of Responsibility pattern

// Custom implementation for middleware-style processing
abstract class RequestHandler {
    var next: RequestHandler? = null

    fun setNext(handler: RequestHandler): RequestHandler {
        next = handler
        return handler
    }

    abstract suspend fun handle(request: NetworkRequest): NetworkResponse?
}

class AuthHandler(private val tokenProvider: TokenProvider) : RequestHandler() {
    override suspend fun handle(request: NetworkRequest): NetworkResponse? {
        // Add auth header
        val authedRequest = request.withHeader("Authorization", "Bearer ${tokenProvider.token}")
        return next?.handle(authedRequest)  // pass to next handler
    }
}

class RateLimitHandler(private val rateLimiter: RateLimiter) : RequestHandler() {
    override suspend fun handle(request: NetworkRequest): NetworkResponse? {
        if (!rateLimiter.tryAcquire()) throw RateLimitException()
        return next?.handle(request)
    }
}

class RetryHandler(private val maxRetries: Int) : RequestHandler() {
    override suspend fun handle(request: NetworkRequest): NetworkResponse? {
        repeat(maxRetries) { attempt ->
            try { return next?.handle(request) }
            catch (e: NetworkException) {
                if (attempt == maxRetries - 1) throw e
                delay(1000L * (attempt + 1))
            }
        }
        return null
    }
}

class NetworkHandler(private val client: HttpClient) : RequestHandler() {
    override suspend fun handle(request: NetworkRequest) = client.execute(request)
}

// Wire the chain
val handler = AuthHandler(tokenProvider).apply {
    setNext(RateLimitHandler(rateLimiter))
        .setNext(RetryHandler(maxRetries = 3))
        .setNext(NetworkHandler(client))
}

val response = handler.handle(request)
```

---

## 6. Mobile-Specific Patterns

### 6.1 ViewHolder — Cached view references in lists

```kotlin
// RecyclerView.ViewHolder — reuse Views, cache findViewById calls
class PostViewHolder(private val binding: ItemPostBinding) :
    RecyclerView.ViewHolder(binding.root) {

    fun bind(post: Post) {
        binding.title.text = post.title
        binding.body.text = post.body
        binding.likeCount.text = post.likes.toString()
        binding.root.setOnClickListener { onPostClicked(post) }
    }
}

// In Compose — ViewHolder equivalent is a Composable function
// Compose handles view reuse automatically in LazyColumn
@Composable
fun PostItem(post: Post, onLike: (String) -> Unit) {
    Card { /* render post */ }
}
```

### 6.2 Use Case / Interactor

```kotlin
// Single responsibility: one class per application action
// Lives in domain layer — pure Kotlin, no Android imports

class GetFeedUseCase @Inject constructor(
    private val feedRepo: FeedRepository,
    private val userRepo: UserRepository
) {
    // operator fun invoke — allows calling as a function: getFeed(userId)
    operator fun invoke(userId: String): Flow<List<FeedItem>> =
        combine(
            feedRepo.observeFeed(userId),
            userRepo.observeFollowing(userId)
        ) { feed, following ->
            feed.filter { post -> post.authorId in following.map { it.id } }
                .sortedByDescending { it.createdAt }
        }
}
```

---

## 7. Sequence Diagrams

Sequence diagrams show interactions over time between objects.

### 7.1 Login Flow Sequence

```
User     LoginScreen     LoginViewModel    AuthRepository     AuthApi
 │            │                │                  │               │
 │ tap Login  │                │                  │               │
 │──────────▶ │                │                  │               │
 │            │ onLoginTapped()│                  │               │
 │            │───────────────▶│                  │               │
 │            │                │ login(email,pwd) │               │
 │            │                │─────────────────▶│               │
 │            │                │                  │ POST /auth    │
 │            │                │                  │──────────────▶│
 │            │                │                  │ 200 {token}   │
 │            │                │                  │◀──────────────│
 │            │                │                  │ saveToken()   │
 │            │                │                  │ (DataStore)   │
 │            │                │  Result.success  │               │
 │            │                │◀─────────────────│               │
 │            │ emit state=    │                  │               │
 │            │ LoggedIn       │                  │               │
 │            │◀───────────────│                  │               │
 │ navigate   │                │                  │               │
 │ to Home    │                │                  │               │
 │◀───────────│                │                  │               │
```

### 7.2 Cache-First Repository Sequence

```
ViewModel         Repository          Room DAO          Network API
    │                  │                  │                  │
    │ getProduct(id)   │                  │                  │
    │─────────────────▶│                  │                  │
    │                  │ getProduct(id)   │                  │
    │                  │─────────────────▶│                  │
    │                  │ CachedProduct?   │                  │
    │                  │◀─────────────────│                  │
    │ emit cached      │                  │                  │
    │◀─────────────────│                  │                  │
    │                  │                  │ GET /products/id │
    │                  │                  │─────────────────▶│
    │                  │                  │ 200 Product      │
    │                  │                  │◀─────────────────│
    │                  │ upsert(product)  │                  │
    │                  │─────────────────▶│                  │
    │                  │ Room emits update│                  │
    │                  │◀─────────────────│                  │
    │ emit fresh       │                  │                  │
    │◀─────────────────│                  │                  │
```

### 7.3 BLE Connect + Read Sequence

```
App              BluetoothLeScanner    BluetoothDevice      BluetoothGatt
 │                      │                    │                    │
 │ startScan()          │                    │                    │
 │─────────────────────▶│                    │                    │
 │                      │ onScanResult(dev)  │                    │
 │◀─────────────────────│                    │                    │
 │ stopScan()           │                    │                    │
 │─────────────────────▶│                    │                    │
 │ connectGatt(device)  │                    │                    │
 │──────────────────────────────────────────▶│                    │
 │                                           │ create BluetoothGatt│
 │                                           │────────────────────▶│
 │                                 onConnectionStateChange(CONNECTED)│
 │◀────────────────────────────────────────────────────────────────│
 │ discoverServices()                                               │
 │─────────────────────────────────────────────────────────────────▶│
 │                                         onServicesDiscovered()   │
 │◀─────────────────────────────────────────────────────────────────│
 │ readCharacteristic(uuid)                                         │
 │─────────────────────────────────────────────────────────────────▶│
 │                                      onCharacteristicRead(value) │
 │◀─────────────────────────────────────────────────────────────────│
 │ processValue(value)                                              │
```

---

## 8. Pattern Recognition Quick Reference

| You need to... | Pattern | Android example |
|---|---|---|
| One instance globally | Singleton | `@Singleton` Retrofit, Room |
| Create without knowing concrete type | Factory | `ViewModelProvider.Factory` |
| Build complex object step by step | Builder | `OkHttpClient.Builder` |
| Simplify a complex subsystem | Facade | `MediaPlaybackFacade` |
| Add behavior without subclassing | Decorator | `OkHttp Interceptors` |
| Convert incompatible interfaces | Adapter | `RecyclerView.Adapter` |
| Abstract data sources | Repository | `UserRepositoryImpl` |
| Notify many of state changes | Observer | `StateFlow`, `LiveData` |
| Interchangeable algorithms | Strategy | Cache-first vs network-first |
| Encapsulate requests | Command | Undo/redo in editors |
| Behavior changes with state | State | `PaymentState` sealed class |
| Pass request through handlers | Chain of Responsibility | `OkHttp Interceptor chain` |
| Single action encapsulation | Use Case | `GetFeedUseCase` |

---

## 9. Common Misunderstandings & Pitfalls

**❌ Overusing Singleton**
The Singleton pattern is a form of global state — it makes testing hard (can't inject a mock), creates hidden dependencies, and causes subtle initialization ordering bugs. Prefer DI-managed singletons (`@Singleton` in Hilt) over manual singletons. Never use Singleton for objects that need different behavior in tests.

**❌ Confusing Adapter with Facade**
Adapter changes an interface to match what a client expects — it doesn't simplify, it converts. Facade simplifies a complex system behind a clean API — it doesn't convert. `RecyclerView.Adapter` adapts your data to RecyclerView's interface. `MediaPlaybackFacade` simplifies ExoPlayer + AudioFocus + MediaSession into `play()`, `pause()`, `stop()`.

**❌ Using the same pattern for every problem**
The Strategy pattern is powerful but adds indirection. A simple `if` statement for two code paths is usually better than a `FetchStrategy` interface with two implementations. Apply patterns when complexity justifies the abstraction, not to demonstrate knowledge.

**❌ Making Builder methods return `Unit` instead of `this`**
Builders need to return `this` (or `apply { }` in Kotlin) for chaining. A method that returns `Unit` breaks the fluent API. In Kotlin, `fun header(name: String, value: String) = apply { headers[name] = value }` is the idiomatic way.

**❌ Not defining which object "owns" the chain in CoR**
In Chain of Responsibility, it must be clear which handler is the "fallback" (handles everything) vs which are "filters" (handle specific cases). Without clarity, requests can fall through all handlers without being handled.

---

## 10. Best Practices

- **Match pattern to problem** — choose patterns that solve a real problem, not to show design knowledge
- **Prefer Kotlin idioms** — `object` for Singleton, `sealed class` for State, `fun interface` for Strategy, `apply` for Builder
- **Favor composition over inheritance** — Decorator and Strategy over deep class hierarchies
- **Repository always behind an interface** — enables fakes in tests, swappable implementations
- **Use Case for reusable domain logic** — if the same operation appears in two ViewModels, extract to Use Case
- **Observer pattern via Flow** — don't build custom observable — StateFlow/SharedFlow already implement the pattern correctly with lifecycle awareness
- **Name patterns explicitly in code review** — "I'm using the Strategy pattern here so we can swap caching algorithms" communicates intent
- **Sequence diagrams for async flows** — draw the timeline for auth, payment, BLE connection — async ordering bugs are best caught on paper before code

---

## 11. Interview Q&A

**Q1: What is the Repository pattern and why does it use an interface?**

> The Repository pattern abstracts data access behind a clean domain API. The ViewModel doesn't know whether data comes from a network call, a database, a cache, or a combination — it just calls `getProduct(id)` on a `ProductRepository` interface. The implementation (`ProductRepositoryImpl`) orchestrates the actual sources. The interface is critical for two reasons: first, the ViewModel has no compile-time dependency on Room, Retrofit, or any storage technology — those are implementation details. Second, tests inject a `FakeProductRepository` that returns controlled test data without hitting a database or network. This is the dependency inversion principle in practice: high-level modules (ViewModel) depend on abstractions (interface), not on low-level details (Room, Retrofit).

---

**Q2: How does the Strategy pattern apply to caching?**

> The Strategy pattern encapsulates interchangeable algorithms behind a common interface. For caching, the algorithms are data fetch strategies: cache-first (return cached immediately, refresh in background), network-first (try network, fall back to cache on failure), or stale-while-revalidate (return stale immediately, refresh asynchronously). Each is a separate `FetchStrategy` implementation with the same `fetch(key, networkFetch)` signature. The `ProductRepository` holds a reference to the current strategy and delegates fetching to it. At runtime, you can swap strategy based on context — when the device is offline, switch to cache-first; when freshness is critical (prices), use network-first. This is closed for modification (Repository code doesn't change) and open for extension (new strategies can be added).

---

**Q3: What's the difference between the Decorator and the Adapter patterns?**

> Adapter converts one interface into another — its job is compatibility. A legacy audio API that uses callbacks gets adapted to a coroutine suspend function. The caller uses the Adapter because it can't use the Adaptee directly. Decorator adds behavior to an existing object while keeping the same interface — its job is enhancement. A `LoggingRepository` wraps a `NetworkRepository`, both implement `Repository`. The caller doesn't know it's talking to a decorator. OkHttp Interceptors are Decorators: each interceptor wraps the chain (same `intercept(chain): Response` interface) and adds logging, auth headers, or retry logic without the caller knowing. RecyclerView.Adapter is an Adapter: it converts your `List<Post>` domain model into RecyclerView's expected interface of `ViewHolder` and `getItemCount()`.

---

**Q4: When would you use the Command pattern in a mobile app?**

> Whenever you need undo/redo or deferred/queueable operations. A text editor where every character insertion, deletion, and formatting change is a `Command` — `undo()` reverses exactly the last operation. A drawing app where every stroke is a `Command` allows unlimited undo. A shopping cart where `AddItemCommand` and `RemoveItemCommand` can be replayed or reversed. An offline-first app where user operations are `Command` objects queued in a database — the sync queue processes them in order when connectivity returns, and each operation's `undo()` allows rollback if the server rejects it. The key signal: you need operations to be reversible, replayable, or serializable. If your operations are fire-and-forget with no undo requirement, a simple method call is sufficient — don't add Command complexity without a clear use case.

---

**Q5: Draw the class diagram for a simplified image loading library.**

> ```
> «interface» ImageLoader          «interface» Fetcher<T>
>   + enqueue(request: ImageRequest)    + fetch(data: T): Source
>   + execute(request: ImageRequest)
>         ▲                                    ▲
>         │                          ┌─────────┴──────────┐
>  RealImageLoader              HttpFetcher          FileFetcher
>   - memoryCache: MemCache       - okHttpClient         - file: File
>   - diskCache: DiskCache
>   - fetcherFactory: Factory
>   - decoderFactory: Factory     «interface» Decoder<T>
>                                      + decode(source: Source): Bitmap
>  «interface» ImageRequest                  ▲
>   + data: Any                    BitmapDecoder  SvgDecoder
>   + size: Size
>   + transformations: List     «interface» Transformation
>   + memoryCacheKey: String         + transform(bitmap: Bitmap): Bitmap
>                                          ▲
>                               RoundedCornersTransformation
>                               CircleCropTransformation
>                               BlurTransformation
> ```
> Patterns: Factory (creating Fetchers and Decoders for unknown data types), Decorator (transformation chain), Chain of Responsibility (interceptor chain inside `RealImageLoader`), Builder (`ImageRequest.Builder`), Strategy (different fetch strategies per data type).

---

**Q6: How does the Observer pattern relate to StateFlow?**

> StateFlow is an implementation of the Observer pattern where `MutableStateFlow` is the subject (it holds state and notifies observers) and each `collect { }` call is an observer subscription. `_state.update { }` is `notifyObservers()`. The Compose `collectAsStateWithLifecycle()` is an observer that automatically subscribes when the composable enters composition and unsubscribes when it leaves — exactly the `addObserver`/`removeObserver` lifecycle management the pattern requires, handled automatically. StateFlow adds important properties the classic Observer pattern doesn't have: it always has a current value (new observers immediately receive the current state), it's lifecycle-aware (won't notify dead observers), and it's thread-safe. SharedFlow is a variant where multiple observers all receive the same emission — useful for events (navigation, snackbar) rather than state.

---

*Previous: [20 — Caching Deep Dive](./20-caching.md)*
*Next: [22 — Debugging, Profiling & Tooling](./22-debugging.md)*
