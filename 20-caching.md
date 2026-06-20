# 20 — Caching Deep Dive

> **Round type: Both (HLD + LLD)**
> **Focus:** Multi-layer cache architecture, HTTP caching semantics, Coil/Glide image pipeline internals, DiskLruCache, cache invalidation strategies, cache key design, React Native caching

---

## 1. Why This Matters in Interviews

Caching is the single highest-leverage performance optimization in mobile. Every part of the stack has a cache, and understanding which layer handles what — and when to bypass or invalidate each — separates engineers who tune performance from those who just add more spinners.

**Common interview angles:**
- "Walk me through what happens when Coil loads an image"
- "What is an ETag and how does it reduce bandwidth?"
- "How do you invalidate a cache without a version bump?"
- "What's the difference between `max-age` and `must-revalidate`?"
- "How does OkHttp decide whether to use a cached response?"
- "Design a caching strategy for a product catalog"

---

## 2. Three-Layer Cache Architecture

```
Request for data (image, API response, DB query)
         │
         ▼
[L1: Memory Cache]          ← microseconds, volatile, process-scoped
  → LruCache / in-process map
  → Lost on process death
         │ MISS
         ▼
[L2: Disk Cache]            ← milliseconds, persistent, device-scoped
  → DiskLruCache / Room / files
  → Survives process death, app restart
         │ MISS
         ▼
[L3: Network / Origin]      ← hundreds of ms, authoritative
  → HTTP server, CDN
  → Response written back to L2 and L1
```

**Cache hit flow:**
- L1 hit → fastest possible path (microseconds, no I/O)
- L2 hit → fast (disk read, ~5–20ms)
- L3 hit → slow (network, 100–2000ms depending on connection)

**The goal:** maximize L1 and L2 hit rate for high-frequency requests.

---

## 3. HTTP Caching — OkHttp

### 3.1 How HTTP Caching Works

OkHttp implements the full HTTP/1.1 caching spec. The server controls caching behavior via response headers; OkHttp honors them.

```
First request:
  Client → GET /products/123 → Server
  Server → 200 OK
           Cache-Control: max-age=300          (cache for 5 minutes)
           ETag: "abc123"                       (fingerprint of this response)
           Last-Modified: Thu, 01 Jan 2025 10:00:00 GMT
           Content: { product data }
  OkHttp stores response + metadata in disk cache

Second request (within 5 minutes):
  OkHttp checks cache → not expired → serves from cache, NO network call

Third request (after 5 minutes):
  Cache expired → OkHttp sends conditional request:
  GET /products/123
  If-None-Match: "abc123"           (ETag from cached response)
  If-Modified-Since: Thu, 01 Jan 2025 10:00:00 GMT

  Server checks: has the resource changed?
  - If NOT changed → 304 Not Modified (0 bytes body) → use cached response
  - If CHANGED    → 200 OK + new body + new ETag
```

**ETag = "Entity Tag"** — a hash/fingerprint of the response content. `If-None-Match` sends it back; if the server's current ETag matches, nothing changed.

### 3.2 Cache-Control Directives

```
Cache-Control response header options:

max-age=N          Cache for N seconds. After N seconds, revalidate.
s-maxage=N         CDN-specific override for max-age
no-cache           Always revalidate before serving (but can cache)
no-store           Never cache — not even to disk
must-revalidate    When expired, must revalidate (can't serve stale)
immutable          Content will never change — skip revalidation forever
private            Don't cache in shared/CDN caches (user-specific data)
public             OK to cache in CDN and client caches
stale-while-revalidate=N   Serve stale for N seconds while revalidating in background
stale-if-error=N   Serve stale if server returns error (5xx)

Cache-Control request header options (less common, client can set):
no-cache           Force network request even if cache is fresh
only-if-cached     Return cached response or 504 (useful offline)
max-stale=N        Accept responses up to N seconds stale
```

### 3.3 OkHttp Cache Setup

```kotlin
@Singleton
fun provideOkHttpClient(
    @ApplicationContext context: Context,
    authInterceptor: AuthInterceptor
): OkHttpClient {
    val cacheSize = 10L * 1024 * 1024  // 10MB HTTP cache
    val cache = Cache(File(context.cacheDir, "http_cache"), cacheSize)

    return OkHttpClient.Builder()
        .cache(cache)
        .addInterceptor(authInterceptor)
        // Network interceptor: modify response headers to control caching
        .addNetworkInterceptor { chain ->
            val response = chain.proceed(chain.request())
            // If server doesn't send Cache-Control, add our own policy
            if (response.header("Cache-Control") == null) {
                response.newBuilder()
                    .header("Cache-Control", "public, max-age=300")  // cache 5 min
                    .build()
            } else {
                response
            }
        }
        .build()
}
```

### 3.4 Force Cache When Offline

```kotlin
// Interceptor: serve stale cache when network is unavailable
class OfflineCacheInterceptor(
    private val connectivityObserver: ConnectivityObserver
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = if (!connectivityObserver.isConnected) {
            chain.request().newBuilder()
                .cacheControl(
                    CacheControl.Builder()
                        .onlyIfCached()          // only serve from cache
                        .maxStale(7, TimeUnit.DAYS)  // accept up to 7-day-old cache
                        .build()
                )
                .build()
        } else {
            chain.request()
        }
        return chain.proceed(request)
    }
}
```

### 3.5 Cache Invalidation via HTTP

```kotlin
// Force-refresh a specific endpoint (bypass cache)
val request = Request.Builder()
    .url("https://api.example.com/user/profile")
    .cacheControl(CacheControl.FORCE_NETWORK)  // bypass cache, fetch fresh
    .build()

// Force-cache a specific request (never hit network)
val request = Request.Builder()
    .url("https://api.example.com/config")
    .cacheControl(CacheControl.FORCE_CACHE)  // return cached or 504
    .build()
```

### 3.6 Inspecting the OkHttp Cache

```kotlin
// Log cache statistics for debugging
val cache = okHttpClient.cache
val requestCount = cache?.requestCount()      // total requests
val networkCount = cache?.networkCount()      // requests that hit network
val hitCount = cache?.hitCount()             // requests served from cache
val hitRate = hitCount?.toFloat()?.div(requestCount?.toFloat() ?: 1f)

Timber.d("Cache hit rate: ${hitRate?.times(100)}%")
// Target: > 80% hit rate for static/semi-static content
```

---

## 4. Image Loading Pipeline — Coil Internals

Coil processes each image request through an **interceptor chain** (inspired by OkHttp). Understanding this chain is how you customize and debug image loading.

### 4.1 The Default Interceptor Chain

```
ImageRequest
     │
     ▼
[MemoryCacheInterceptor]    ← Check L1 (in-process LruCache of Bitmaps)
     │ MISS
     ▼
[EngineInterceptor]         ← Orchestrates decoding + transformations
     │
     ▼
[DiskCacheInterceptor]      ← Check L2 (DiskLruCache of raw image bytes)
     │ MISS
     ▼
[NetworkInterceptor]        ← Fetch from network via OkHttp
     │ response
     ▼
[Write to DiskCache]        ← Store raw bytes
     │
     ▼
[Decode Bitmap]             ← BitmapFactory / ImageDecoder → Bitmap
     │
     ▼
[Apply Transformations]     ← roundedCorners, blur, grayscale
     │
     ▼
[Write to MemoryCache]      ← Store Bitmap in LruCache
     │
     ▼
Display in ImageView / Composable
```

### 4.2 Coil Configuration

```kotlin
// Setup singleton ImageLoader
val imageLoader = ImageLoader.Builder(context)
    .memoryCache {
        MemoryCache.Builder(context)
            .maxSizePercent(0.25)   // 25% of app's available memory
            .build()
    }
    .diskCache {
        DiskCache.Builder()
            .directory(context.cacheDir.resolve("image_cache"))
            .maxSizeBytes(100L * 1024 * 1024)  // 100MB disk cache
            .build()
    }
    .okHttpClient {
        OkHttpClient.Builder()
            .cache(Cache(context.cacheDir.resolve("http_cache"), 20L * 1024 * 1024))
            .build()
    }
    .respectCacheHeaders(true)   // honor server Cache-Control headers
    .crossfade(true)             // crossfade when loading from network
    .build()

// Set as singleton (used by AsyncImage automatically)
SingletonImageLoader.setSafe(context) { imageLoader }
```

### 4.3 Cache Keys

Coil uses the image URL as the default cache key. Problem: same URL with different transformations would collide.

```kotlin
// Custom cache key for the same URL with different transformations
@Composable
fun RoundedThumbnail(url: String, size: Int) {
    AsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(url)
            .size(size, size)
            .memoryCacheKey("${url}_${size}_rounded")     // explicit key
            .diskCacheKey("${url}_${size}_rounded")
            .transformations(CircleCropTransformation())
            .build(),
        contentDescription = null
    )
}

// Listener for debugging cache behavior
val request = ImageRequest.Builder(context)
    .data(url)
    .listener(
        onStart = { Timber.d("Loading: $url") },
        onSuccess = { _, result ->
            val dataSource = result.dataSource  // MEMORY, DISK, or NETWORK
            Timber.d("Loaded from: $dataSource")
        },
        onError = { _, result -> Timber.e(result.throwable, "Error loading: $url") }
    )
    .build()
```

### 4.4 Cache Policy Control

```kotlin
// Force reload — bypass all caches
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(url)
        .memoryCachePolicy(CachePolicy.DISABLED)
        .diskCachePolicy(CachePolicy.DISABLED)
        .networkCachePolicy(CachePolicy.ENABLED)
        .build()
)

// Read-only (never write to cache — for ephemeral content)
.diskCachePolicy(CachePolicy.READ_ONLY)

// Write-only (cache but never serve from cache — for prefetch)
.diskCachePolicy(CachePolicy.WRITE_ONLY)

// Placeholders and error states
AsyncImage(
    model = ImageRequest.Builder(context)
        .data(url)
        .placeholder(R.drawable.placeholder)    // shown while loading
        .error(R.drawable.error_placeholder)    // shown on failure
        .fallback(R.drawable.no_image)          // shown when data is null
        .crossfade(300)                         // ms for crossfade animation
        .build()
)
```

---

## 5. Glide Cache Pipeline

Glide has a more complex cache architecture than Coil — it maintains separate caches for different representations of the same image:

```
Request
   │
   ▼
Active Resources       ← Images currently displayed (WeakReference)
   │ not found
   ▼
Memory Cache (LruCache) ← Recently displayed images (strong reference)
   │ not found
   ▼
Resource Disk Cache    ← Decoded, transformed bitmap written to disk
   │ not found         (cached AFTER decoding + transformations)
   ▼
Data Disk Cache        ← Raw downloaded bytes (before decoding)
   │ not found         (cached BEFORE decoding)
   ▼
Network / Source
```

**Glide vs Coil cache difference:** Coil caches raw bytes on disk (then decodes on next access). Glide can cache the decoded/transformed bitmap on disk — faster to display on next load but uses more disk space and invalidation is more complex.

```kotlin
// Glide cache configuration
@GlideModule
class AppGlideModule : AppGlideModule() {
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        builder
            .setMemoryCache(LruResourceCache(
                MemorySizeCalculator.Builder(context)
                    .setMemoryCacheScreens(2f)  // cache 2 screens worth of images
                    .build()
                    .memoryCacheSize.toLong()
            ))
            .setDiskCache(
                InternalCacheDiskCacheFactory(context, "glide_disk_cache", 250 * 1024 * 1024)  // 250MB
            )
    }
}

// Force skip memory cache (e.g., user avatar that might have changed)
Glide.with(context)
    .load(avatarUrl)
    .skipMemoryCache(true)           // don't use memory cache
    .diskCacheStrategy(DiskCacheStrategy.NONE)  // don't use disk cache
    .into(imageView)

// Cache strategies
DiskCacheStrategy.ALL          // cache raw + decoded (default-ish)
DiskCacheStrategy.DATA         // cache only raw bytes
DiskCacheStrategy.RESOURCE     // cache only decoded bitmap
DiskCacheStrategy.AUTOMATIC    // let Glide decide (smart)
DiskCacheStrategy.NONE         // no disk cache
```

### 5.1 Cache Invalidation in Glide

```kotlin
// Invalidate a single image (e.g., user uploaded new avatar)
// Simple: add a cache-busting query param to the URL
val avatarUrl = "https://cdn.example.com/avatars/$userId.jpg?v=${System.currentTimeMillis()}"

// Or use Glide's signature mechanism (doesn't change URL)
val signature = ObjectKey(userProfile.avatarVersion)  // changes when avatar changes
Glide.with(context)
    .load("https://cdn.example.com/avatars/$userId.jpg")
    .signature(signature)
    .into(imageView)
```

---

## 6. DiskLruCache — Internals

Both Coil and Glide use `DiskLruCache` (originally from AOSP / JakeWharton) for disk caching. Understanding it helps debug cache issues.

### 6.1 Structure

```
cache_directory/
├── journal           ← Operations log: DIRTY, CLEAN, REMOVE, READ
├── abc123.0          ← Cache entry "abc123", value index 0 (raw bytes or bitmap)
├── abc123.1          ← Cache entry "abc123", value index 1 (metadata)
├── def456.0
└── def456.1
```

The journal file tracks the state of every cache entry. On next startup, `DiskLruCache` replays the journal to rebuild its in-memory map.

### 6.2 Key Design

Cache keys must be URL-safe strings (alphanumeric + dashes + underscores). Coil and Glide hash the cache key to prevent invalid file names.

```kotlin
// Bad key — spaces and special chars
val badKey = "https://cdn.example.com/images/product image #123.jpg"

// Good key — hash the URL
import java.security.MessageDigest
fun String.toSafeKey(): String {
    val digest = MessageDigest.getInstance("MD5")
    val bytes = digest.digest(toByteArray())
    return bytes.joinToString("") { "%02x".format(it) }
}
// "https://cdn.example.com/product.jpg".toSafeKey() → "a3f2b1..."
```

### 6.3 Cache Sizing

```kotlin
// Adaptive sizing based on available disk space
fun calculateDiskCacheSize(cacheDir: File): Long {
    val available = cacheDir.usableSpace
    return when {
        available > 10L * 1024 * 1024 * 1024 -> 500L * 1024 * 1024  // > 10GB: 500MB cache
        available > 5L * 1024 * 1024 * 1024  -> 250L * 1024 * 1024  // > 5GB: 250MB
        available > 2L * 1024 * 1024 * 1024  -> 100L * 1024 * 1024  // > 2GB: 100MB
        else                                  -> 50L * 1024 * 1024   // < 2GB: 50MB
    }
}
```

---

## 7. Cache Invalidation Strategies

Cache invalidation is famously hard. Here are the main patterns:

### 7.1 TTL (Time-To-Live)

Every cache entry expires after N seconds. Simplest approach.

```kotlin
data class CachedEntry<T>(
    val data: T,
    val cachedAt: Long = System.currentTimeMillis(),
    val ttlMs: Long
) {
    val isExpired: Boolean get() = System.currentTimeMillis() - cachedAt > ttlMs
}

// Different TTLs for different data types
val PRODUCT_TTL = 10 * 60 * 1000L     // 10 minutes
val USER_PROFILE_TTL = 5 * 60 * 1000L // 5 minutes
val CONFIG_TTL = 60 * 60 * 1000L      // 1 hour
val STATIC_ASSET_TTL = Long.MAX_VALUE  // never expires
```

**TTL trade-offs:**
- Short TTL = fresh data, more network requests
- Long TTL = stale data, fewer requests
- Right TTL depends on how often the data changes

### 7.2 ETag / Content Hash

Let the server tell you whether the content has changed:

```kotlin
// Store ETag with cached response
data class CachedResponse<T>(
    val data: T,
    val etag: String?,
    val lastModified: String?
)

// On next request, send the ETag
suspend fun getProduct(id: String): Product {
    val cached = cache.get(id)
    val request = Request.Builder()
        .url("https://api.example.com/products/$id")
        .apply {
            cached?.etag?.let { header("If-None-Match", it) }
            cached?.lastModified?.let { header("If-Modified-Since", it) }
        }
        .build()

    val response = okHttpClient.newCall(request).execute()
    return when (response.code) {
        304 -> cached!!.data  // Not Modified — use cached
        200 -> {
            val newData = parseProduct(response.body!!.string())
            cache.put(id, CachedResponse(
                data = newData,
                etag = response.header("ETag"),
                lastModified = response.header("Last-Modified")
            ))
            newData
        }
        else -> throw IOException("HTTP ${response.code}")
    }
}
```

### 7.3 Event-Based Invalidation

Invalidate specific entries when you know they changed:

```kotlin
// On successful profile update → invalidate user cache
suspend fun updateProfile(update: ProfileUpdate): Result<Unit> {
    return try {
        api.updateProfile(update)
        // Invalidate affected caches
        cache.remove("user_profile_${currentUserId}")
        cache.remove("user_avatar_${currentUserId}")
        // Optionally prefetch fresh data
        prefetchUserProfile(currentUserId)
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

### 7.4 Version-Based Cache Busting

Append a version to cache keys or URLs:

```kotlin
// URL-based versioning
val profileUrl = "https://cdn.example.com/avatars/$userId.jpg?v=${user.avatarVersion}"
// When user uploads new avatar, server increments avatarVersion
// → different URL → cache miss → fresh image loaded

// Key-based versioning in Room
@Entity
data class CachedProduct(
    @PrimaryKey val id: String,
    val data: String,
    val version: Int,    // server-provided version number
    val cachedAt: Long
)
// In repository: if cached.version != server.version → refresh
```

### 7.5 Stale-While-Revalidate (SWR)

Serve stale data immediately, refresh in background — best perceived performance:

```kotlin
fun <T> staleWhileRevalidate(
    cacheKey: String,
    fetch: suspend () -> T,
    cache: Cache<T>
): Flow<T> = flow {
    // Step 1: emit stale immediately if available
    val cached = cache.get(cacheKey)
    if (cached != null) emit(cached)

    // Step 2: fetch fresh in background
    try {
        val fresh = fetch()
        cache.put(cacheKey, fresh)
        if (fresh != cached) emit(fresh)  // only emit if different
    } catch (e: Exception) {
        if (cached == null) throw e  // no cache AND no network = real error
        // else: silently serve stale, log the error
    }
}
```

---

## 8. Repository-Level Cache (Room as Cache)

```kotlin
class ProductRepository @Inject constructor(
    private val api: ProductApi,
    private val dao: ProductDao
) {
    // Room as cache — observe DB, refresh from network when stale
    fun getProduct(id: String): Flow<Product> = flow {
        // Always emit cached value first
        val cached = dao.getProduct(id)
        if (cached != null) emit(cached.toDomain())

        // Check if we should refresh
        val shouldRefresh = cached == null ||
            System.currentTimeMillis() - cached.cachedAt > PRODUCT_TTL

        if (shouldRefresh) {
            try {
                val fresh = api.getProduct(id)
                val entity = fresh.toEntity().copy(cachedAt = System.currentTimeMillis())
                dao.upsert(entity)
                emit(entity.toDomain())
            } catch (e: Exception) {
                if (cached == null) throw e  // nothing to fall back to
                // else: serve stale, don't re-emit
            }
        }
    }

    // Explicitly invalidate and refresh
    suspend fun refreshProduct(id: String) {
        val fresh = api.getProduct(id)
        dao.upsert(fresh.toEntity().copy(cachedAt = System.currentTimeMillis()))
    }
}
```

---

## 9. In-Memory Cache for Computed Values

```kotlin
// Cache expensive computations in ViewModel
@HiltViewModel
class FeedViewModel @Inject constructor(
    private val feedRepo: FeedRepository
) : ViewModel() {

    // Memoize rendered text — formatting is expensive at list scale
    private val renderedTextCache = LruCache<String, AnnotatedString>(500)

    fun getRenderedText(postId: String, rawMarkdown: String): AnnotatedString {
        return renderedTextCache.get(postId)
            ?: renderMarkdown(rawMarkdown)
                .also { renderedTextCache.put(postId, it) }
    }

    // Cache network responses in memory for current session
    private val userProfileCache = ConcurrentHashMap<String, UserProfile>()

    suspend fun getUserProfile(userId: String): UserProfile {
        return userProfileCache.getOrPut(userId) {
            userRepository.getProfile(userId)
        }
    }
}
```

---

## 10. HLD — Cache Architecture for a Product Catalog

```
Product Catalog Caching:

Data characteristics:
  - 50,000 products
  - Updated by merchants (changes are infrequent but must propagate)
  - Read by millions of users simultaneously
  - Prices can change often; images rarely

Cache layers:
  L1: In-process LruCache (ViewModel/Repository)
      → Recent products (last 100 viewed)
      → TTL: session-scoped, no persistence
      → Invalidated by: pull-to-refresh, explicit cache clear

  L2: Room database
      → Products with timestamps
      → TTL: 10 minutes for price fields, 24 hours for images/descriptions
      → Invalidated by: push notification from server (product updated)

  L3: OkHttp HTTP cache
      → Server sends Cache-Control: max-age=600, stale-while-revalidate=60
      → ETags for conditional requests
      → 304 Not Modified saves bandwidth for unchanged products

  Image cache (separate):
      → Coil with 100MB disk cache, 25% memory cache
      → Aggressive caching: images change rarely
      → CDN-side: immutable URLs (product images with hash suffix)
      → Cache-Control: public, immutable, max-age=31536000 (1 year)

Invalidation triggers:
  → Merchant updates product → server publishes FCM data message
  → App receives FCM → removes product from L1 and L2 (by ID)
  → Next access fetches fresh from L3 (HTTP cache expired due to ETag)
  → Image URL changes (new hash suffix) → automatic cache miss for image

Price sensitivity:
  → Prices: short TTL (2 minutes) or fetched separately with no-cache
  → Images/descriptions: long TTL (24 hours or immutable)
```

---

## 11. React Native Caching

```typescript
// TanStack Query — manages caching, staleness, background refresh
import { useQuery } from '@tanstack/react-query'

const { data: products } = useQuery({
    queryKey: ['products', categoryId],
    queryFn: () => fetchProducts(categoryId),
    staleTime: 5 * 60 * 1000,      // fresh for 5 minutes
    gcTime: 30 * 60 * 1000,        // garbage collected after 30 minutes
    refetchOnWindowFocus: true,     // refetch when app comes to foreground
    refetchOnReconnect: true,       // refetch when network reconnects
})

// Manual cache invalidation
const queryClient = useQueryClient()
await queryClient.invalidateQueries({ queryKey: ['products'] })

// Optimistic update with rollback
queryClient.setQueryData(['products', categoryId], (old) =>
    old?.map(p => p.id === updatedId ? updatedProduct : p)
)

// Image caching in RN
// React Native caches images automatically in native layer
// For explicit control:
import FastImage from 'react-native-fast-image'  // wraps SDWebImage/Glide

<FastImage
    source={{
        uri: 'https://cdn.example.com/product.jpg',
        priority: FastImage.priority.high,
        cache: FastImage.cacheControl.immutable  // never refetch
        // cache: FastImage.cacheControl.web     // respect HTTP headers
        // cache: FastImage.cacheControl.cacheOnly // only from cache
    }}
    style={{ width: 200, height: 200 }}
/>

// Prefetch for smoother experience
FastImage.preload([
    { uri: 'https://cdn.example.com/product1.jpg' },
    { uri: 'https://cdn.example.com/product2.jpg' }
])
```

---

## 12. Common Misunderstandings & Pitfalls

**❌ Assuming OkHttp caches by default**
OkHttp only caches responses when: (1) a `Cache` is configured on the client, AND (2) the server sends cacheable `Cache-Control` headers. Without both, nothing is cached. Many APIs don't send cache headers — add your own via a network interceptor.

**❌ Not providing a cache key for transformed images**
If two image requests use the same URL but different transformations (one circular, one square), they'll collide in Coil's cache. The circular one overwrites the square. Always set `memoryCacheKey` and `diskCacheKey` when applying transformations.

**❌ Using memory cache for large images without size limiting**
Setting `MemoryCache.maxSizePercent(0.5)` (50% of heap) for images will cause frequent GC and OOM crashes. Use 20–25% max. Images that don't fit the target view should be downsampled before caching.

**❌ Caching auth-sensitive responses in shared cache**
OkHttp's disk cache is on device storage. Responses containing user-specific auth data (profile, payment info) should be `Cache-Control: private, no-store` — or at minimum `private` — to prevent them from being served to other users if the device is shared or inspected.

**❌ Not handling 304 Not Modified correctly in custom implementations**
If you implement your own HTTP layer and send `If-None-Match`, you must handle the 304 response by returning the previously cached body — a 304 has an empty body. Libraries like OkHttp handle this automatically; custom implementations often forget.

**❌ Cache poisoning via user-controlled input in cache keys**
If you build a cache key from user-provided data (like a search query), ensure it's sanitized. A malicious user could craft a search query that produces a key matching a cached response for a different user's sensitive data.

**❌ Invalidating cache on every mutation**
Full cache invalidation after every write defeats the purpose of caching. Prefer targeted invalidation: only invalidate the specific entities that changed. "User updated their profile" → invalidate `user_${id}`, not the entire user cache.

---

## 13. Best Practices

- **Three cache layers always** — memory (LruCache), disk (DiskLruCache/Room), network (OkHttp)
- **Size memory cache at 20–25% of heap** — balance performance vs OOM risk
- **Different TTLs for different data types** — prices: 2 min; profiles: 15 min; static content: 24h+
- **Immutable URLs for static assets** — `image.jpg?v=abc123` — allows 1-year `max-age`
- **ETags for conditional requests** — 304 response = header overhead only, no body bandwidth
- **Stale-while-revalidate** — emit cached immediately, refresh in background, re-emit if changed
- **Event-based invalidation for known mutations** — invalidate precisely, not broadly
- **Coil `memoryCachePolicy` and `diskCachePolicy`** — tune per request type (skip memory for lists, skip disk for ephemeral)
- **Log cache hit rates in debug builds** — target > 80% for semi-static content
- **`no-store` for sensitive data** — auth tokens, payment info, health data
- **Cache size monitoring** — alert when disk cache grows beyond budget
- **Periodic cleanup job** — WorkManager periodic task to purge expired Room rows

---

## 14. Interview Q&A

**Q1: Walk me through what happens when Coil loads an image for the first time vs the second time.**

> First load: Coil checks the memory cache (LruCache of decoded Bitmaps) — miss. Checks the disk cache (DiskLruCache of raw bytes) — miss. Fetches via OkHttp, which checks its own HTTP cache — miss, so goes to the network. Receives raw bytes, writes them to the disk cache. Passes bytes to the decoder (ImageDecoder API 28+ or BitmapFactory for older) which produces a Bitmap. Applies any transformations (crop, rounded corners). Writes the decoded Bitmap to the memory cache. Displays it. Second load (same session): Coil checks memory cache — hit. Returns the Bitmap immediately without touching disk or network. Microseconds vs hundreds of milliseconds. Second load (different session): Memory cache is gone (process restart). Disk cache has the raw bytes. Coil decodes them again (no network call), caches in memory, displays. The disk cache is the key to fast subsequent session loads.

---

**Q2: What is an ETag and why does it save bandwidth?**

> An ETag is a server-generated fingerprint of a response — typically a hash of the content. The server sends it as a header: `ETag: "abc123"`. The client stores it with the cached response. When the cache expires, instead of re-downloading the full response, the client sends a conditional request: `If-None-Match: "abc123"`. The server checks if its current version has the same ETag. If nothing changed, it responds with `304 Not Modified` — a response with no body, just headers, typically ~200 bytes. If content changed, it returns `200 OK` with the new full body and a new ETag. The bandwidth saving is dramatic for large responses: a 200KB JSON response becomes a ~200-byte 304 when the content hasn't changed. This matters most for content that changes infrequently but is checked often — product catalogs, user profiles, config.

---

**Q3: How do you invalidate a cached image without changing the image URL?**

> Glide's `signature()` mechanism. You pass an `ObjectKey` containing a version identifier that changes when the image changes. Glide incorporates this signature into the cache key — so even though the URL is the same, a different signature produces a different cache key, causing a cache miss. Example: store `avatarVersion` on the user object. When the user uploads a new avatar, the server increments this version. The app uses `Glide.with(context).load(url).signature(ObjectKey(user.avatarVersion)).into(view)`. Old version = old cache key. New version = new cache key = cache miss = fresh download. For Coil, set an explicit `memoryCacheKey("${url}_${version}")` and `diskCacheKey` similarly.

---

**Q4: What are the trade-offs between TTL-based and event-based cache invalidation?**

> TTL-based is simple: every entry expires after N seconds and is refetched. It requires no server coordination — the client decides freshness. The downside is staleness within the TTL window and wasted bandwidth from periodic refetches even when nothing changed. Event-based invalidation is precise: the server sends a signal (FCM push, WebSocket message) when a specific item changes, and the client invalidates only that entry. This gives instant consistency and minimizes unnecessary requests. The downside is implementation complexity: you need a push infrastructure, and you must ensure delivery — if the invalidation message is missed, the cache stays stale indefinitely. In practice, production apps use both: event-based invalidation for known mutations (user changed profile, admin updated product) plus a TTL backstop (24 hours) to catch any missed events.

---

**Q5: How do you design a cache for data that changes frequently (prices) vs rarely (product images)?**

> Separate strategies for each. Prices: short TTL (2–5 minutes), no disk caching (always fetch fresh or use `stale-while-revalidate` with 60-second staleness window), serve from in-memory cache only. The server sends `Cache-Control: max-age=120, stale-while-revalidate=60` so the client can show the last price instantly while checking for an update in the background. Product images: use immutable URLs (`image.jpg?hash=abc123` where the hash is the content hash). The server sends `Cache-Control: public, immutable, max-age=31536000` (1 year). The client caches aggressively — the URL only changes when the image content changes, so there's never a staleness issue. When the merchant uploads a new photo, the hash changes, the URL changes, and a cache miss naturally occurs. This asymmetric strategy — tight TTL for dynamic data, immutable URLs for static — is how Instagram, Amazon, and most large e-commerce platforms handle their catalog.

---

**Q6: How would you implement stale-while-revalidate for a social feed?**

> When the user opens the feed, emit the cached feed from Room immediately — the user sees content in under 100ms. Simultaneously, launch a background coroutine to fetch fresh data from the API. When the fresh data arrives: if it's different from cached (check by comparing IDs or last-updated timestamp), upsert into Room. The UI observes Room via Flow, so it automatically re-renders with the fresh content — no explicit "reload" required. The user sees a smooth experience: stale content immediately, updated content appears within 1–2 seconds. The key implementation detail: only emit the fresh data if it's actually different — comparing the first item's timestamp or a list hash prevents an unnecessary recomposition when the feed hasn't changed. Also handle the edge case where the user has scrolled deep into the feed when the update arrives — either preserve scroll position or show a "New posts" banner rather than jumping the user back to the top.

---

*Previous: [19 — Animation](./19-animation.md)*
*Next: [21 — Design Patterns & Diagrams](./21-design-patterns.md)*
