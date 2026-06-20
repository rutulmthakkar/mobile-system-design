# 02 — Networking & API Design

> **Scope:** How mobile apps communicate with backends — protocol choices, request lifecycle, resilience patterns, pagination, versioning, and the OkHttp/Retrofit stack in detail. Includes React Native equivalents.

---

## 1. Why This Matters in Interviews

Networking is where most mobile bugs live in production — flaky connections, race conditions, stale data, token expiry, retry storms. Interviewers test whether you think beyond the happy path.

**Common interview angles:**
- "How would you design the API layer for a feed app?"
- "What happens when a request fails on a bad network?"
- "How do you prevent your app from hammering the server on retry?"
- "How do you keep old app versions working after an API change?"

---

## 2. Protocol Choices

### 2.1 REST

**What it is:** Resources accessed via URLs using standard HTTP verbs (GET, POST, PUT, PATCH, DELETE). Responses are usually JSON.

```
GET  /users/123           → fetch user
POST /users               → create user
PUT  /users/123           → replace user
PATCH /users/123          → partial update
DELETE /users/123         → delete user
```

**Mobile reality:**
- A feed screen might need user profile + posts + ads + stories → 4+ REST calls → 4 round trips
- Each round trip on a mobile network (especially 3G or high-latency) adds perceptible delay
- Solution: BFF (see §5), field filtering (`?fields=id,name,avatar`), or composite endpoints

**Pros:** Simple, universally understood, great tooling, HTTP caching works naturally, easy to debug with curl/Postman.

**Cons:** Over-fetching (response has 30 fields, you need 5), under-fetching (need multiple calls for one screen), versioning pain.

---

### 2.2 GraphQL

**What it is:** Single endpoint (`/graphql`). Client sends a query describing exactly the data shape it needs. Server resolves only those fields.

```graphql
# One request fetches exactly what the feed screen needs
query FeedScreen($userId: ID!) {
  user(id: $userId) {
    name
    avatarUrl
  }
  feed(userId: $userId, limit: 20) {
    id
    text
    imageUrl
    author { name }
    likeCount
  }
}
```

**Mobile benefits:**
- One round trip for complex screens (solves under-fetching)
- No wasted bandwidth (solves over-fetching)
- Schema is the contract — strongly typed, self-documenting
- Schema introspection enables powerful tooling (Apollo DevTools)

**Cons:**
- HTTP caching doesn't work out of the box (all requests are POST to same URL)
- N+1 query problem on server — each `author` field in a list triggers a DB call; requires DataLoader batching
- Query complexity attacks — a malicious client can send a deeply nested query that overloads the server; need query depth/complexity limits
- Harder to debug in production (all errors return HTTP 200)
- File upload is awkward (multipart workarounds needed)
- Overkill for simple CRUD apps

**Android:** Apollo Kotlin is the de facto client
**React Native:** Apollo Client or urql

```kotlin
// Apollo Kotlin
val response = apolloClient.query(FeedScreenQuery(userId = "123")).execute()
val feed = response.data?.feed
```

---

### 2.3 gRPC

**What it is:** Google's RPC framework. Uses HTTP/2 for transport and Protocol Buffers (protobuf) for binary serialization. Client calls feel like local function calls.

```protobuf
// user.proto
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc StreamFeed(FeedRequest) returns (stream FeedItem);  // server streaming
}
message GetUserRequest { string user_id = 1; }
message User { string id = 1; string name = 2; string avatar_url = 3; }
```

**Why faster than REST:**
- Protobuf binary is ~3–10x smaller than JSON
- HTTP/2 multiplexes multiple requests over one TCP connection (no head-of-line blocking)
- HTTP/2 header compression (HPACK)
- Generated stubs eliminate manual serialization

**Four communication modes:**
| Mode | Description | Mobile use case |
|---|---|---|
| Unary | Single request → single response | Standard API calls |
| Server streaming | Single request → stream of responses | Real-time feed, live scores |
| Client streaming | Stream of requests → single response | Sensor data upload, chunked upload |
| Bidirectional streaming | Full duplex | Chat, live collaboration |

**Android:** `grpc-kotlin` + `grpc-okhttp`
**React Native:** `grpc-web` (limited streaming support in RN)

**Cons:**
- Not human-readable (binary protocol)
- Browser/RN support limited (gRPC-Web is a subset)
- Harder to debug without tooling (need protobuf-aware tools like grpcurl)
- Schema (.proto files) need distribution and versioning
- Not ideal for public APIs (REST is still the standard)

---

### 2.4 Decision Matrix

| Factor | REST | GraphQL | gRPC |
|---|---|---|---|
| Simplicity | ✅ Highest | 🟡 Medium | ❌ Complex |
| Performance | 🟡 Medium | 🟡 Medium | ✅ Highest |
| Mobile bandwidth efficiency | 🟡 Depends on design | ✅ Best (no over-fetch) | ✅ Binary |
| Caching (HTTP) | ✅ Native | ❌ Manual (persisted queries) | ❌ Not applicable |
| Real-time / streaming | ❌ Add WebSocket separately | 🟡 Subscriptions (WebSocket) | ✅ Built in |
| Tooling maturity | ✅ Best | ✅ Good | 🟡 Growing |
| Public API | ✅ Best choice | 🟡 Viable | ❌ Avoid |
| Internal service | 🟡 Works | 🟡 Works | ✅ Best choice |
| Cross-platform (web+mobile) | ✅ | ✅ | 🟡 gRPC-Web |

**Interview answer:** "I default to REST for most mobile features because of tooling, caching, and simplicity. I'd consider GraphQL if the app has complex screens with heterogeneous data and the backend team has GraphQL expertise. I'd only use gRPC for mobile-to-service communication if performance is critical and web support isn't required — for example, an internal data pipeline or real-time streaming where you control both client and server."

---

## 3. The OkHttp + Retrofit Stack (Android)

### 3.1 How They Relate

```
Retrofit (type-safe API interface)
    ↓ delegates to
OkHttp (HTTP client — handles connections, caching, interceptors)
    ↓ over
TCP/HTTP stack
```

Retrofit converts your interface declarations into HTTP calls. OkHttp actually executes them.

### 3.2 OkHttp Internals Worth Knowing

**Connection Pool:** OkHttp reuses TCP connections (default: 5 idle connections, 5-minute keepalive). Creating TCP connections is expensive — SYN, SYN-ACK, ACK + TLS handshake. Pooling amortizes this.

**Interceptor Chain:** Middleware pattern. Each interceptor calls `chain.proceed(request)` to pass to the next. Two types:
- **Application interceptors** — run before OkHttp's internal logic (before caching, before retries). See original request and final response.
- **Network interceptors** — run around actual network calls. Can see redirects and retries. Can short-circuit with cached response.

```
App Interceptors → OkHttp Core (cache, retry, redirect) → Network Interceptors → Server
```

**Common interceptors:**
```kotlin
// Auth interceptor — adds token to every request
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer ${tokenProvider.getToken()}")
            .build()
        return chain.proceed(request)
    }
}

// Logging interceptor (use HttpLoggingInterceptor from OkHttp)
val logging = HttpLoggingInterceptor().apply {
    level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY 
            else HttpLoggingInterceptor.Level.NONE  // never log bodies in prod
}

// GZIP interceptor — OkHttp handles this automatically if server sends Accept-Encoding: gzip
// You only need a custom interceptor if you need to force-compress request bodies
```

### 3.3 Retrofit Setup (Modern Kotlin)

```kotlin
@Singleton
fun provideOkHttpClient(authInterceptor: AuthInterceptor): OkHttpClient =
    OkHttpClient.Builder()
        .addInterceptor(authInterceptor)         // app interceptor
        .addInterceptor(logging)                  // app interceptor
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()

@Singleton
fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit =
    Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(okHttpClient)
        .addConverterFactory(GsonConverterFactory.create())  // or Moshi, KotlinxSerialization
        .build()

// API interface — Retrofit generates the implementation
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") userId: String): UserDto

    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): UserDto

    @GET("feed")
    suspend fun getFeed(
        @Query("cursor") cursor: String?,
        @Query("limit") limit: Int = 20
    ): FeedResponse
}
```

**Prefer Kotlin serialization over Gson:** Gson uses reflection, Kotlin serialization uses code generation (faster, null-safe).

---

## 4. Resilience Patterns

### 4.1 Retry with Exponential Backoff + Jitter

**Never retry immediately.** A server that's struggling under load will be made worse by every client retrying at the same moment.

**Exponential backoff:** Wait `2^attempt` seconds between retries.
**Jitter:** Add random offset to desynchronize clients (prevents thundering herd).

```kotlin
// Coroutine-based retry in repository layer (preferred over OkHttp interceptor for testability)
suspend fun <T> retryWithBackoff(
    times: Int = 3,
    initialDelay: Long = 1000L,
    maxDelay: Long = 10_000L,
    factor: Double = 2.0,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
        try {
            return block()
        } catch (e: IOException) {  // only retry network errors, not 4xx
            delay(currentDelay + Random.nextLong(0, currentDelay / 2))  // jitter
            currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
        }
    }
    return block() // final attempt, let exception propagate
}

// Usage
val user = retryWithBackoff { api.getUser(userId) }
```

**What to retry vs not:**
| Status | Retry? | Reason |
|---|---|---|
| 408 Request Timeout | ✅ Yes | Transient |
| 429 Too Many Requests | ✅ Yes, with `Retry-After` header | Rate limited |
| 500 Internal Server Error | ✅ Yes (idempotent requests only) | Transient |
| 502/503/504 | ✅ Yes | Upstream/infra issue |
| 400 Bad Request | ❌ No | Client bug, retry won't help |
| 401 Unauthorized | ❌ Refresh token, then retry once | Auth issue |
| 403 Forbidden | ❌ No | Permission issue |
| 404 Not Found | ❌ No | Resource doesn't exist |

**Idempotency matters:** Only retry if the request is idempotent (GET, PUT, DELETE). Retrying a POST order creation without idempotency key = duplicate orders. Always send an `Idempotency-Key` header on non-idempotent retried requests.

### 4.2 Circuit Breaker

Prevents cascading failures. If a service is down, stop hammering it — fail fast locally.

```
States:
CLOSED → requests flow normally, failures counted
   ↓ (failure threshold exceeded — e.g. 5 failures in 10s)
OPEN → requests fail immediately (no network call)
   ↓ (after timeout — e.g. 30s)
HALF-OPEN → allow one test request
   ↓ success → CLOSED     ↓ failure → OPEN
```

**On Android:** Implement manually or use a library. Typical threshold: 5 failures in a 30s window → open circuit for 60s.

**Interview signal:** Mentioning circuit breaker shows you think about production failure modes, not just happy path.

### 4.3 Timeout Strategy

Three timeouts to configure:
- **Connect timeout:** Time to establish TCP connection (default 10s — set to 15–30s for poor networks)
- **Read timeout:** Time to wait for server to start sending response (30s typical)
- **Write timeout:** Time to wait while sending the request body (30s typical)

Set tighter timeouts for user-facing requests (want to show error quickly) and longer for background uploads.

### 4.4 Token Refresh & 401 Handling

**Problem:** Access token expires. Request fails with 401. Need to refresh silently and retry.

**Solution — Authenticator (OkHttp):**
```kotlin
class TokenAuthenticator(private val tokenRepo: TokenRepository) : Authenticator {
    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.code != 401) return null
        if (responseCount(response) >= 2) return null  // don't retry twice

        val newToken = runBlocking { tokenRepo.refreshToken() } ?: return null

        return response.request.newBuilder()
            .header("Authorization", "Bearer $newToken")
            .build()
    }

    private fun responseCount(response: Response): Int {
        var count = 1
        var priorResponse = response.priorResponse
        while (priorResponse != null) { count++; priorResponse = priorResponse.priorResponse }
        return count
    }
}
```

`Authenticator` is called by OkHttp automatically on 401. It refreshes the token and returns a new request to retry. If refresh fails, return `null` to propagate the 401.

**Race condition:** Multiple concurrent requests all get 401 simultaneously, all try to refresh. Use `Mutex` to ensure only one refresh happens:
```kotlin
private val refreshMutex = Mutex()
suspend fun refreshToken(): String? = refreshMutex.withLock {
    // check if another coroutine already refreshed while we were waiting
    val cached = tokenStore.getToken()
    if (cached.isNotExpired()) return cached.value
    // actually refresh
    api.refresh(tokenStore.getRefreshToken())
}
```

---

## 5. Backend for Frontend (BFF) Pattern

### 5.1 The Problem

A generic microservices API designed for all clients is bad for mobile:
- Mobile needs fewer fields than desktop (bandwidth, battery)
- One mobile screen may need data from 5 different services (user, posts, ads, reactions, notifications)
- Making 5 calls serially from mobile = 5 round trips over a flaky network

### 5.2 The Solution

A **BFF** is a thin server-side layer owned by the mobile team that:
1. Accepts one mobile-friendly request
2. Fans out to multiple backend services in parallel (server-to-server calls are fast)
3. Aggregates, transforms, and shapes data for the specific screen
4. Returns a single response

```
Mobile App
    │ 1 request: GET /mobile/feed
    ▼
Mobile BFF (owned by mobile team)
    ├──→ UserService.getProfile()    ─┐
    ├──→ PostService.getFeed()        ├ parallel, fast server-to-server
    ├──→ AdService.getAds()           │
    └──→ NotificationService.count() ─┘
    │
    ▼ Aggregated response (exactly what mobile needs)
Mobile App
```

### 5.3 BFF Tradeoffs

| Benefit | Cost |
|---|---|
| Single round trip for complex screens | Another service to maintain and deploy |
| Mobile team controls their API shape | Code duplication risk (mobile BFF vs web BFF) |
| Server-side aggregation (fast) | Increases attack surface |
| Insulates mobile from backend changes | Versioning complexity multiplies |
| Can cache aggregated responses | Data staleness risk if cached too aggressively |

**When NOT to use BFF:**
- Simple app with straightforward data needs
- Small team (no bandwidth to maintain BFF)
- GraphQL already solves the aggregation problem

**Interview tip:** "For a complex consumer app like WhatsApp or Instagram, I'd advocate for a mobile BFF because each screen in a social app requires data from 3–5 services. A BFF lets the mobile team iterate API shape independently without backend coordination. For a simpler internal tool, I'd skip it — REST to microservices is fine."

---

## 6. Pagination

### 6.1 Offset Pagination

```
GET /posts?limit=20&offset=0    → posts 1–20
GET /posts?limit=20&offset=20   → posts 21–40
```

**Problems:**
- If new post is inserted between page 1 and page 2 requests, you see a duplicate or miss a post
- Database must count/skip rows — slow on large datasets (`OFFSET 10000` scans 10,000 rows to discard them)
- Doesn't work well for real-time feeds

**When to use:** Admin dashboards, search results where exact page numbers matter to the user.

### 6.2 Cursor Pagination ✅ Preferred for feeds

```
GET /posts?limit=20                → returns posts + nextCursor: "eyJpZCI6MjB9"
GET /posts?limit=20&cursor=eyJpZCI6MjB9  → next page
```

The cursor is an opaque token (usually base64-encoded `{id, timestamp}`). The server decodes it to a `WHERE id > 20 ORDER BY id` query — no row scanning.

**Benefits:**
- Stable pagination even if items are inserted/deleted
- Efficient (index seek, not scan)
- Works for infinite scroll

**Tradeoff:** Can't jump to page 7. Only forward (sometimes bidirectional with prev/next cursors).

### 6.3 Keyset Pagination

Similar to cursor but uses stable column values directly:
```
GET /posts?after_id=1234&limit=20
```
Server does `WHERE id > 1234 ORDER BY id LIMIT 20`. Simple and efficient.

### 6.4 Android Implementation with Paging 3

```kotlin
class FeedPagingSource(private val api: FeedApi) : PagingSource<String, Post>() {
    override suspend fun load(params: LoadParams<String>): LoadResult<String, Post> {
        return try {
            val response = api.getFeed(cursor = params.key, limit = params.loadSize)
            LoadResult.Page(
                data = response.posts,
                prevKey = null,             // no backward paging
                nextKey = response.nextCursor  // null means last page
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }

    override fun getRefreshKey(state: PagingState<String, Post>): String? = null
}

// ViewModel
val feed = Pager(PagingConfig(pageSize = 20)) { FeedPagingSource(api) }
    .flow
    .cachedIn(viewModelScope)

// Composable
val lazyItems = feed.collectAsLazyPagingItems()
LazyColumn {
    items(lazyItems) { post -> PostCard(post) }
    lazyItems.apply {
        when {
            loadState.refresh is LoadState.Loading -> item { LoadingSpinner() }
            loadState.append is LoadState.Loading  -> item { LoadingSpinner() }
            loadState.append is LoadState.Error    -> item { RetryButton { retry() } }
        }
    }
}
```

**React Native equivalent:** FlatList with `onEndReached` + manual cursor tracking, or TanStack Query's infinite queries.

---

## 7. API Versioning

### 7.1 Why It's Critical for Mobile

Web apps can be force-updated instantly (deploy new JS). Mobile apps cannot — users may stay on old versions for months or years. Your API must support old clients.

**The rule:** Never break existing clients. Add fields; don't rename or remove them.

### 7.2 Versioning Strategies

**URL versioning** (most common):
```
https://api.example.com/v1/users
https://api.example.com/v2/users
```
✅ Explicit, cacheable, easy to route
❌ URL changes on every major version, proliferation of versions

**Header versioning:**
```
GET /users
Accept: application/vnd.example.v2+json
```
✅ Clean URLs
❌ Not cacheable by default, harder to test in browser

**Query param versioning:**
```
GET /users?version=2
```
✅ Easy to test
❌ Not RESTful, often ignored by caches

**Recommended for mobile:** URL versioning for major breaking changes + backward-compatible field additions within a version.

### 7.3 Breaking vs Non-Breaking Changes

| Change type | Breaking? | Safe to deploy? |
|---|---|---|
| Add optional response field | ❌ Not breaking | ✅ Yes |
| Add optional request field | ❌ Not breaking | ✅ Yes |
| Remove response field | ✅ Breaking | ❌ No |
| Rename field | ✅ Breaking | ❌ No |
| Change field type (int → string) | ✅ Breaking | ❌ No |
| Add required request field | ✅ Breaking | ❌ No |
| Change enum values | ✅ Breaking | ❌ No |

**Mobile client best practice:** Be lenient in parsing — ignore unknown fields (`@JsonIgnoreProperties(ignoreUnknown = true)` in Jackson / ignore unknown keys in Kotlin serialization). This lets the server add new fields without breaking old clients.

---

## 8. Request Batching & Deduplication

### 8.1 Request Batching

Instead of making 10 individual calls, batch into one:
```
GET /users/batch?ids=1,2,3,4,5  → returns array of users
```

Use case: A comment feed shows 50 comments, each with an author. Instead of 50 user calls, one batch call.

**Mobile consideration:** Balance batch size vs latency. A 100-item batch is slower to assemble server-side. 20–50 items is typical.

### 8.2 Request Deduplication

If two components on the same screen both request the same resource simultaneously, deduplicate at the repository or ViewModel layer:

```kotlin
// Simple in-flight request cache using Mutex
private val inFlight = ConcurrentHashMap<String, Deferred<User>>()

suspend fun getUser(id: String): User {
    return inFlight.getOrPut(id) {
        viewModelScope.async { api.getUser(id) }
    }.also { deferred ->
        inFlight.remove(id)
    }.await()
}
```

Or use Kotlin's `async`/`Deferred` sharing pattern. React Query and Apollo do this automatically.

---

## 9. Network Layer in React Native

**HTTP client options:**
- `fetch` (built-in) — simple, works for basic calls
- `axios` — popular, interceptors, request/response transform
- `react-query` / **TanStack Query** ✅ — caching, deduplication, background refetch, pagination, offline support built in

```typescript
// TanStack Query — handles caching, retry, background refresh automatically
const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,  // treat data as fresh for 5 minutes
    retry: 3,
    retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 10000) // exponential backoff
})

// Infinite scroll with TanStack Query
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
    queryKey: ['feed'],
    queryFn: ({ pageParam }) => fetchFeed({ cursor: pageParam }),
    getNextPageParam: (lastPage) => lastPage.nextCursor,
})
```

**Interceptors in axios:**
```typescript
// Token refresh interceptor
axios.interceptors.response.use(
    res => res,
    async error => {
        if (error.response?.status === 401 && !error.config._retry) {
            error.config._retry = true
            const newToken = await refreshToken()
            axios.defaults.headers.common['Authorization'] = `Bearer ${newToken}`
            return axios(error.config)
        }
        return Promise.reject(error)
    }
)
```

---

## 10. HLD vs LLD Framing

### HLD Questions
- "Design the networking layer for a social media app"
- "How does your app handle poor connectivity?"

**HLD answer covers:**
- Protocol choice (REST/GraphQL/gRPC) with justification
- BFF vs direct microservice calls
- Authentication flow (OAuth2, JWT)
- CDN for static assets
- WebSocket for real-time

### LLD Questions
- "How do you implement retry in your network layer?"
- "Walk me through what happens when an auth token expires"
- "How do you paginate a feed?"

**LLD answer covers:**
- OkHttp interceptor chain
- Authenticator for 401 handling
- Retry with exponential backoff + jitter
- PagingSource implementation
- Error mapping (network exception → domain error)

---

## 11. Common Mistakes / Gotchas

- **Logging request bodies in production** — tokens, PII, payment data. Use `HttpLoggingInterceptor.Level.NONE` in release builds.
- **No timeout set** — requests can hang indefinitely, draining battery and keeping connections open
- **Retrying 4xx errors** — a 400 Bad Request will never succeed with the same payload; don't retry it
- **Retrying POST without idempotency key** — creates duplicate records
- **All interceptors as application interceptors** — network interceptors see actual network calls including retries, important for logging actual network behavior
- **Blocking main thread with network call** — Room throws an exception; OkHttp doesn't but causes ANR. Always use `suspend` functions or background dispatcher.
- **Not handling `UnknownHostException`** — when device is offline, DNS lookup fails; surface offline state to user, don't just show a generic error
- **GraphQL errors return HTTP 200** — you must check `response.errors` field, not just the HTTP status code
- **Over-aggressive caching** — stale auth tokens in OkHttp cache, or stale prices in a shopping app. Always set `Cache-Control: no-store` for sensitive or time-critical data.

---

## 12. Interview Q&A

**Q1: When would you choose GraphQL over REST for a mobile app?**

> I'd choose GraphQL when two conditions are true: first, the app has screens with heterogeneous data requirements — for example, a home feed that needs user profile, posts, stories, ads, and unread counts all at once. Second, the backend team is equipped to support it with proper tooling (persisted queries, query complexity limits, DataLoader for N+1 prevention). REST can solve the same problem with BFF or composite endpoints, but GraphQL bakes it into the protocol. I'd avoid GraphQL for simple CRUD screens or if the team lacks experience — the operational complexity isn't worth it for a settings page.

---

**Q2: How do you handle a scenario where 50 concurrent requests all get a 401 and all try to refresh the token simultaneously?**

> This is the token refresh race condition. The fix is a Mutex around the refresh logic. When the first coroutine acquires the lock and refreshes, subsequent coroutines waiting on the lock see the already-refreshed token when they acquire it, so they skip the refresh and use the new token directly. In OkHttp, I implement this in the `Authenticator` using `runBlocking { mutex.withLock { ... } }`. The key check: before refreshing, verify the token actually expired — it might have been refreshed by another coroutine while this one was waiting.

---

**Q3: What's the difference between exponential backoff and jitter, and why do you need both?**

> Exponential backoff increases the wait time between retries multiplicatively (1s, 2s, 4s, 8s...) to avoid overwhelming a server that's already struggling. But if 10,000 clients all backed off and then all retry at the same time (e.g., after a 4s delay), you get a synchronized retry storm — the same load spike that caused the outage in the first place. Jitter adds a random offset to each client's wait time, spreading retries over a time window instead of synchronizing them. Together they protect both the server and the individual client.

---

**Q4: How do you design pagination for an infinite-scrolling feed that has new items arriving in real time?**

> Offset pagination breaks here — if 5 new posts are added to the top while the user is on page 2, page 3 will include those posts again as duplicates. Cursor pagination solves this: the cursor encodes the position in the dataset (e.g., the ID or timestamp of the last seen item), not a page number. New items at the top don't affect the cursor position. On Android, Paging 3's `PagingSource` supports this natively with a cursor `Key` type. For new items arriving while the user is reading, I handle that separately via a real-time channel (WebSocket/FCM) that inserts new items at the top with a "New posts" banner rather than disrupting the scroll position.

---

**Q5: What happens when a user on an old app version hits your updated API?**

> This is the mobile versioning challenge. My approach: never remove or rename fields — only add optional fields within a version. For breaking changes, add a new version endpoint and run both in parallel with a sunset period. The app must be written defensively: use `@JsonIgnoreProperties(ignoreUnknown = true)` / ignore unknown keys in Kotlin serialization so new server fields don't crash old clients. Use feature flags on the server to gradually roll out new behavior. For minimum supported app version enforcement, return a `426 Upgrade Required` with a `X-Min-App-Version` header when an app is too old. The client shows a "please update" dialog.

---

**Q6: How do you design the network layer for a screen that needs data from 3 different endpoints?**

> Three strategies. (1) BFF: create a server-side endpoint that aggregates the three calls — one round trip from mobile, fast parallel calls server-side. Best when you own the backend. (2) Parallel calls on mobile: use `async`/`await` in coroutines to fire all three simultaneously, then `awaitAll()` — faster than serial but still 3 round trips. (3) GraphQL: model all three as one query. I'd go with BFF for a production app because it minimizes mobile round trips, server-side calls are much faster (datacenter latency vs mobile network latency), and it gives the mobile team control over the shape of data without backend coordination.

```kotlin
// Parallel calls example
viewModelScope.launch {
    val userDeferred = async { userApi.getUser(userId) }
    val feedDeferred = async { feedApi.getFeed(userId) }
    val notifDeferred = async { notifApi.getCount(userId) }

    val (user, feed, notifCount) = awaitAll(userDeferred, feedDeferred, notifDeferred)
    // combine into screen state
}
```

---

*Previous: [01 — Architecture Patterns](./01-architecture-patterns.md)*  
*Next: [03 — Offline-First & Sync](./03-offline-sync.md)*
