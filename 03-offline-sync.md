# 03 — Offline-First & Sync

> **Scope:** When and how to build apps that work without a network connection. Covers the offline-first philosophy, local-as-single-source-of-truth, caching strategies, sync mechanisms, conflict resolution, and the Android toolstack (Room + WorkManager + Flow).

---

## 1. Why This Matters in Interviews

Every real mobile app must answer: *what happens when the network drops?*

Interviewers test whether you think beyond "show a spinner and wait for the network." Senior-level answers talk about data ownership, sync strategies, and conflict resolution — not just caching.

**Common angles:**
- "Design a note-taking app that works offline"
- "A user likes a post while offline. What happens?"
- "How do you sync data when the device reconnects?"
- "What's your conflict resolution strategy?"

---

## 2. The Mental Model Shift

### Traditional (Network-First)
```
User action → API call → update UI on success → show error on failure
```
Network is the **source of truth**. If it's unreachable, the app is broken.

### Offline-First
```
User action → write to local DB → update UI immediately
             → sync to server in background when possible
```
Local DB is the **single source of truth (SSOT)**. Network is just a sync mechanism.

**Key insight:** The UI never waits on the network. It observes the local DB via Flow. When the server sync completes, the DB updates, the Flow emits, the UI re-renders — automatically and silently.

---

## 3. When to Go Offline-First (and When Not To)

### ✅ Go Offline-First When:
- App is used in low-connectivity environments (travel, rural, subway)
- User-generated content must never be lost (notes, drafts, to-dos)
- App has read-heavy patterns (feed, articles, catalog)
- Latency matters — instant read from local DB beats network round trip

### ❌ Don't Go Offline-First When:
- Data must be real-time accurate (stock prices, live auctions, seat booking)
- Actions are inherently online (payments, auth, live calls)
- Data is too sensitive to store locally (some financial data)
- App is simple enough that a loading spinner is fine (one-shot lookup tools)

**Interview tip:** Always ask — "What parts of the app need to be offline-first?" Most apps are hybrid: the feed is offline-first, but checkout is not.

---

## 4. Core Architecture

### 4.1 The Three Layers

```
┌─────────────────────┐
│   UI (Composable)   │  ← Observes StateFlow from ViewModel
└─────────┬───────────┘
          │ collect
┌─────────▼───────────┐
│     ViewModel       │  ← Maps DB stream to UI state
└─────────┬───────────┘
          │ observe Flow
┌─────────▼───────────┐
│     Repository      │  ← ORCHESTRATOR: reads from DB, writes to DB,
│                     │    triggers network sync
└──────┬──────┬───────┘
       │      │
  Room DB   Retrofit API
  (SSOT)    (sync layer)
```

The Repository is the key — it decides: read from DB, write to DB, and separately manage server sync. The UI never touches the network directly.

### 4.2 Room as Single Source of Truth

```kotlin
// Entity with sync metadata
@Entity(tableName = "posts")
data class PostEntity(
    @PrimaryKey val id: String,
    val content: String,
    val authorId: String,
    val createdAt: Long,
    val updatedAt: Long,
    val syncStatus: SyncStatus = SyncStatus.SYNCED,  // SYNCED | PENDING | FAILED
    val localVersion: Int = 0,       // increments on local edit
    val serverVersion: Int? = null   // from server, used for conflict detection
)

enum class SyncStatus { SYNCED, PENDING, FAILED }

// DAO
@Dao
interface PostDao {
    @Query("SELECT * FROM posts ORDER BY createdAt DESC")
    fun observeAll(): Flow<List<PostEntity>>  // returns Flow — live updates

    @Query("SELECT * FROM posts WHERE syncStatus = 'PENDING'")
    suspend fun getPendingSync(): List<PostEntity>

    @Upsert  // INSERT OR REPLACE
    suspend fun upsert(post: PostEntity)

    @Query("UPDATE posts SET syncStatus = :status WHERE id = :id")
    suspend fun updateSyncStatus(id: String, status: SyncStatus)
}
```

**Key: the DAO returns `Flow<List<PostEntity>>`** — not a one-shot result. Room emits a new list every time the table changes. The UI stays current without polling.

### 4.3 Repository Pattern for Offline-First

```kotlin
class PostRepository(
    private val dao: PostDao,
    private val api: PostApi,
    private val syncManager: SyncManager
) {
    // UI always reads from local DB — instant, no network dependency
    fun observePosts(): Flow<List<Post>> =
        dao.observeAll().map { entities -> entities.map(PostEntity::toDomain) }

    // Write: local first, sync in background
    suspend fun createPost(content: String) {
        val localId = UUID.randomUUID().toString()
        val entity = PostEntity(
            id = localId,
            content = content,
            authorId = currentUser.id,
            createdAt = System.currentTimeMillis(),
            updatedAt = System.currentTimeMillis(),
            syncStatus = SyncStatus.PENDING
        )
        dao.upsert(entity)          // immediate local write
        syncManager.scheduleSyncNow() // trigger background sync
    }

    // Called by sync worker — pushes pending to server
    suspend fun syncPendingPosts() {
        val pending = dao.getPendingSync()
        pending.forEach { entity ->
            try {
                val serverPost = api.createPost(entity.toRequest())
                // Update local record with server-assigned ID and version
                dao.upsert(entity.copy(
                    id = serverPost.id,  // replace local UUID with server ID
                    serverVersion = serverPost.version,
                    syncStatus = SyncStatus.SYNCED
                ))
            } catch (e: Exception) {
                dao.updateSyncStatus(entity.id, SyncStatus.FAILED)
            }
        }
    }
}
```

---

## 5. Caching Strategies

These are not exclusive — most apps use a combination.

### 5.1 Cache-First (Offline-First Default)

```
Read: local DB → if empty/stale → fetch from network → save to DB → UI updates via Flow
Write: local DB first → sync to network in background
```
**Best for:** Feeds, articles, product catalogs — anything tolerable to be slightly stale.

### 5.2 Network-First (Online-First)

```
Read: network → success → save to DB → render
            → failure → fallback to DB → render stale data
Write: network → success → save to DB
```
**Best for:** Checkout flow, seat reservation — where staleness is dangerous.

### 5.3 Stale-While-Revalidate

```
Read: return local data immediately (even if stale) → fetch fresh data in background
      → when fresh arrives → update DB → UI auto-refreshes via Flow
```
**Best for:** User profile, settings, account data. User sees something instantly, then gets updated content silently.

```kotlin
fun getProfile(userId: String): Flow<Profile> = flow {
    // Step 1: emit cached immediately
    val cached = dao.getProfile(userId)
    if (cached != null) emit(cached.toDomain())

    // Step 2: fetch fresh in background
    try {
        val fresh = api.getProfile(userId)
        dao.upsert(fresh.toEntity())
        emit(fresh.toDomain())  // emit update
    } catch (e: Exception) {
        if (cached == null) throw e  // nothing to show — propagate error
        // else: silently keep showing cached
    }
}
```

### 5.4 TTL-Based Invalidation (Time-To-Live)

Each cached item has a `cachedAt` timestamp + `maxAgeMs`. The repository checks freshness before deciding to re-fetch.

```kotlin
data class CacheEntry<T>(
    val data: T,
    val cachedAt: Long,
    val maxAgeMs: Long
) {
    val isExpired: Boolean get() = System.currentTimeMillis() - cachedAt > maxAgeMs
    val isStale: Boolean get() = System.currentTimeMillis() - cachedAt > maxAgeMs / 2
}

// Different TTLs for different data types
val USER_PROFILE_TTL = 15.minutes.inWholeMilliseconds
val FEED_TTL = 2.minutes.inWholeMilliseconds
val PRODUCT_PRICE_TTL = 30.seconds.inWholeMilliseconds  // prices change fast
```

### 5.5 Choosing a Strategy

| Data type | Strategy | Why |
|---|---|---|
| Social feed | Cache-first + background refresh | Slight staleness OK, instant load matters |
| User profile | Stale-while-revalidate | Show something instantly, update quietly |
| Messages / chat | Network-first with local fallback | Ordering and freshness matter |
| Product price | Network-first, short TTL | Stale price = wrong order amount |
| Drafted content | Offline-first + pending queue | Must never lose user work |
| Static config / feature flags | Cache + long TTL + refresh on launch | Rarely changes |

---

## 6. Sync Strategies

### 6.1 Full Sync

Pull everything from the server and replace local data.

```
GET /posts → replace all local posts
```

**Pros:** Simple to implement, always consistent.
**Cons:** Expensive for large datasets, wastes bandwidth, can overwrite pending local changes.

**When to use:** First install, after long offline period when delta tracking is impractical.

### 6.2 Delta Sync ✅ Preferred

Only fetch what changed since the last sync.

```
GET /posts?since=1718000000000  → only posts updated after this timestamp
```

Server requires a `updatedAt` index. Client stores `lastSyncTimestamp`.

```kotlin
suspend fun deltaSync() {
    val lastSync = prefs.getLong("lastSyncTimestamp", 0L)
    val delta = api.getPostsSince(since = lastSync)
    dao.upsertAll(delta.posts.map { it.toEntity() })
    prefs.edit { putLong("lastSyncTimestamp", System.currentTimeMillis()) }
}
```

**Gotcha:** Clock skew between server and client. Use server-returned `syncToken` (opaque cursor) instead of client timestamps. The server generates the token and interprets it on the next sync.

### 6.3 Push-Based Sync (FCM/WebSocket)

Server pushes change notifications to the device. Client fetches only the changed item.

```
Server change → FCM data message → app wakes → fetch changed item → update DB → UI updates
```

**Pros:** Near real-time, efficient (only pull what the server says changed).
**Cons:** FCM delivery not guaranteed (doze mode, device offline). Must still do periodic delta sync as fallback.

**Best for:** Chat messages, notifications, collaborative document changes.

### 6.4 Operation Queue (Outbox Pattern)

When offline, user actions are stored as **operations** in a local queue. When online, the worker drains the queue in order.

```kotlin
@Entity(tableName = "sync_queue")
data class SyncOperation(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val entityId: String,
    val operationType: OperationType,  // CREATE | UPDATE | DELETE
    val payload: String,               // JSON of the request body
    val createdAt: Long,
    val retryCount: Int = 0,
    val status: OperationStatus = OperationStatus.PENDING
)
```

**Why an operation queue instead of just marking rows as PENDING?**
- Supports ordering — operations must be replayed in sequence (create then update, not update then create)
- Supports partial failures — operation 3 failing doesn't block operation 4 if they're independent
- Survives process death — the queue is persisted in Room, not in memory

---

## 7. Conflict Resolution

### 7.1 When Conflicts Happen

User edits a document on their phone while offline. Another device (or another user) edits the same document on the server. When the phone syncs, both versions exist — which wins?

### 7.2 Last Write Wins (LWW)

The write with the most recent timestamp wins. Simple and works for most cases.

```kotlin
// Server-side logic
if (incoming.updatedAt > current.updatedAt) {
    // incoming wins
    db.save(incoming)
} else {
    // current wins, reject with 409
    return ConflictResponse(current)
}
```

**Problem:** Clock skew. If the client clock is wrong, the wrong version wins. Mitigate by using server-assigned timestamps on write, not client timestamps.

**Best for:** User profile updates, settings, configuration — one field at a time, single owner.

### 7.3 Server Wins

Server's version always authoritative. Discard local changes on conflict.

**Best for:** Read-heavy data where the client rarely needs to write (product catalog, news feed).

### 7.4 Client Wins

Client's local change always applied. Server is updated to match.

**Best for:** Local-only personalization (sorting preference, theme choice) that doesn't need multi-device consistency.

### 7.5 Merge (Field-Level)

Merge non-conflicting fields. Flag genuinely conflicting fields for user resolution.

```
Server version: { name: "Alice", bio: "Engineer", avatar: "s3://new-avatar.jpg" }
Client version: { name: "Alice", bio: "Mobile engineer", avatar: "s3://old-avatar.jpg" }

Merge result:
- bio: conflict → show user both options
- avatar: client changed it after server, client wins (use LWW per field)
- name: same → no conflict
```

**Best for:** Collaborative documents (Google Docs style), note-taking apps.

### 7.6 CRDTs (Conflict-free Replicated Data Types)

Mathematical data structures designed so all concurrent modifications can be merged deterministically without conflicts. No "winner" needed — all operations commute.

**Examples:**
- G-Counter: distributed counter that only increments (like count)
- LWW-Register: per-field LWW with vector clocks
- OR-Set: sets where adds and removes from different devices merge correctly

**Used by:** Figma (collaborative design), Notion, linear.app for real-time collaboration.

**On mobile:** Rarely needed unless building collaborative editing. Too complex for most apps. Worth mentioning in interviews as "the right answer for Google Docs-style collaboration."

### 7.7 Optimistic Concurrency Control (OCC)

Client sends a `version` number with every update. Server rejects if its version doesn't match.

```
GET /note/123  → { content: "Hello", version: 5 }
... user edits offline ...
PUT /note/123  → { content: "Hello world", version: 5 }  ← client sends version it last saw

Server response:
- If server version = 5: accept, save as version 6
- If server version = 6: reject with 409 Conflict + current server content
```

Client receives 409, shows conflict UI to user.

```kotlin
// HTTP header approach using ETag
GET /note/123
Response: ETag: "version-5"

PUT /note/123
If-Match: "version-5"   ← include the ETag
→ 200 OK (saved) or 412 Precondition Failed (conflict)
```

**Best for:** Forms, documents, any content edited by one user at a time across devices.

---

## 8. Background Sync with WorkManager

### 8.1 Why WorkManager

WorkManager is the recommended solution for background work on Android. It:
- Survives app restart and device reboot
- Respects system constraints (battery, network type, storage)
- Handles retries automatically with backoff
- Works across API levels
- Supports chaining and periodic work

```kotlin
// Sync worker
class SyncWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            val repo = getRepository()  // inject via Hilt WorkerFactory
            repo.syncPendingPosts()
            repo.deltaSync()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()  // WorkManager retries with backoff
            else Result.failure()
        }
    }
}

// Schedule: sync when connected, with exponential backoff on failure
val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
    .build()

WorkManager.getInstance(context).enqueueUniqueWork(
    "sync",
    ExistingWorkPolicy.KEEP,  // don't start another sync if one is already queued
    syncRequest
)

// Periodic sync (every 15 minutes minimum, system decides exact time)
val periodicSync = PeriodicWorkRequestBuilder<SyncWorker>(15, TimeUnit.MINUTES)
    .setConstraints(Constraints.Builder().setRequiredNetworkType(NetworkType.CONNECTED).build())
    .build()
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "periodic-sync",
    ExistingPeriodicWorkPolicy.KEEP,
    periodicSync
)
```

### 8.2 Connectivity Observation

Trigger sync the moment the device reconnects:

```kotlin
// Modern API (API 24+)
class ConnectivityObserver(private val context: Context) {
    val isConnected: Flow<Boolean> = callbackFlow {
        val manager = context.getSystemService(ConnectivityManager::class.java)
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
        }
        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
        manager.registerNetworkCallback(request, callback)
        awaitClose { manager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
}

// In ViewModel or sync coordinator
connectivityObserver.isConnected
    .filter { it }  // only when connected
    .onEach { syncManager.scheduleSyncNow() }
    .launchIn(viewModelScope)
```

---

## 9. Optimistic UI Updates

Show the result of an action immediately — before the server confirms — for a snappy feel.

```kotlin
// Liking a post while possibly offline
suspend fun toggleLike(postId: String) {
    val post = dao.getPost(postId)
    val optimisticPost = post.copy(
        isLiked = !post.isLiked,
        likeCount = if (!post.isLiked) post.likeCount + 1 else post.likeCount - 1,
        syncStatus = SyncStatus.PENDING
    )
    dao.upsert(optimisticPost)  // UI updates immediately via Flow

    try {
        val serverPost = api.toggleLike(postId)
        dao.upsert(serverPost.toEntity().copy(syncStatus = SyncStatus.SYNCED))
    } catch (e: Exception) {
        // Rollback on failure
        dao.upsert(post.copy(syncStatus = SyncStatus.FAILED))
        // Surface error to UI via a separate error channel
    }
}
```

**UX considerations:**
- Show a subtle indicator for unsynced items ("Saving..." or a cloud icon)
- On rollback, show a non-intrusive snackbar: "Couldn't save. Tap to retry."
- Don't rollback silently — users get confused when UI state reverts unexpectedly

---

## 10. Schema Design for Offline

### Room entity fields to always include:
```kotlin
@Entity
data class NoteEntity(
    @PrimaryKey val id: String,          // client-generated UUID (don't depend on server ID)
    val content: String,
    val createdAt: Long,
    val updatedAt: Long,                 // for delta sync and LWW
    val serverVersion: Int? = null,      // for OCC conflict detection
    val syncStatus: SyncStatus,          // SYNCED | PENDING | FAILED
    val isDeleted: Boolean = false,      // soft delete — don't hard delete before sync
    val isPinned: Boolean = false        // survive cache cleanup
)
```

**Soft deletes:** Mark as `isDeleted = true` instead of removing the row. The sync worker propagates the delete to the server, then removes the row locally. Hard-deleting before sync means the server never hears about it.

**Client-generated IDs:** Use UUID on the client, not auto-increment from the server. This lets you write to the local DB immediately without waiting for the server to assign an ID.

---

## 11. React Native Equivalent

**Local DB options:**
- **WatermelonDB** ✅ — designed for offline-first, lazy loading, high performance, works with SQLite
- **MMKV** — fast key-value store (replaces AsyncStorage for small data)
- **AsyncStorage** — simple key-value, not suitable for relational data
- **SQLite via expo-sqlite** — raw SQLite access

**Sync / state:**
- **TanStack Query** — `staleTime`, `gcTime`, background refetch, offline mode via `networkMode: 'offlineFirst'`
- **Redux Persist** — persists Redux store to AsyncStorage between sessions

```typescript
// TanStack Query offline-first
const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    staleTime: 5 * 60 * 1000,          // serve cached for 5 min without refetch
    gcTime: 24 * 60 * 60 * 1000,       // keep in memory cache for 24h
    networkMode: 'offlineFirst',         // return cached even when offline
})

// Optimistic update with TanStack Query
const mutation = useMutation({
    mutationFn: toggleLike,
    onMutate: async (postId) => {
        await queryClient.cancelQueries({ queryKey: ['posts'] })
        const previous = queryClient.getQueryData(['posts'])
        queryClient.setQueryData(['posts'], (old) =>
            old.map(p => p.id === postId ? { ...p, isLiked: !p.isLiked } : p)
        )
        return { previous }
    },
    onError: (err, postId, context) => {
        queryClient.setQueryData(['posts'], context.previous)  // rollback
    },
    onSettled: () => queryClient.invalidateQueries({ queryKey: ['posts'] })
})
```

---

## 12. HLD vs LLD Framing

### HLD Questions
- "Design a note-taking app like Notion that works offline"
- "How does a messaging app handle offline message sending?"

**HLD answer covers:**
1. Local DB as SSOT (Room / WatermelonDB)
2. Repository orchestrating local writes + background sync
3. Sync strategy (delta sync with server-issued sync token)
4. Conflict resolution strategy (OCC for single-user, LWW or merge for multi-user)
5. WorkManager for reliable background sync
6. Push (FCM) + pull (periodic) hybrid sync

### LLD Questions
- "Show me the schema for an offline-capable task list"
- "Walk through what happens when a user creates a task offline"
- "How do you resolve a conflict when two devices edit the same note?"

**LLD answer covers:**
1. Entity with `syncStatus`, `updatedAt`, `serverVersion`, `isDeleted`
2. Client UUID generation (no server dependency)
3. Operation queue draining in order
4. OCC with `If-Match` / version field
5. Rollback on sync failure + user notification

---

## 13. Common Mistakes / Gotchas

- **Using server-assigned auto-increment IDs as primary key** — can't insert to local DB until server responds; breaks offline creation
- **Hard deleting records before sync** — server never gets the delete; item reappears on next full sync
- **Not handling clock skew** — client timestamp is wrong; use server-issued sync tokens instead of client timestamps for delta sync
- **Syncing on every keystroke** — debounce writes; don't sync while user is still typing
- **Not persisting the sync queue** — if the app is killed mid-sync, in-memory queue is lost; always write operations to Room
- **Blocking the UI thread with Room queries** — use `suspend` functions or `Flow`; Room will throw `IllegalStateException` if you query on main thread
- **No TTL on cache** — stale data forever; always include `cachedAt` and define expiry per data type
- **Overwriting pending local changes with a server pull** — always check `syncStatus` before overwriting; don't replace `PENDING` rows with server data unless resolving a conflict

---

## 14. Interview Q&A

**Q1: What's the difference between caching and offline-first?**

> Caching is saving a copy of server data to serve faster on the next request. Offline-first is an architectural commitment: the local database is the source of truth, and the UI always reads from it — even when online. With caching, the network path is primary and cache is a performance optimization. With offline-first, the local DB path is primary and the network is a sync channel. The practical difference: in a cached app, going offline breaks the write path. In an offline-first app, writes work offline and sync later.

---

**Q2: A user creates a task while offline. Walk me through exactly what happens.**

> The user taps "Create". The ViewModel calls the repository. The repository generates a client UUID, creates a `TaskEntity` with `syncStatus = PENDING`, writes it to Room. Room's Flow emits the new list; the Composable recomposes; the task appears in the UI instantly — no spinner. In the background, I either trigger an immediate WorkManager one-time job (if connectivity is available) or wait for the periodic sync. The sync worker queries for all PENDING tasks, calls `POST /tasks` with an `Idempotency-Key` header (the client UUID), and on success updates the entity with the server-assigned ID and sets `syncStatus = SYNCED`. If the server call fails, the status becomes `FAILED` and the UI shows a subtle retry indicator.

---

**Q3: What's your conflict resolution strategy for a note-taking app?**

> It depends on the use case. For single-user multi-device (most note apps), I use optimistic concurrency control: the client sends the `version` number it last saw with every update. The server rejects with 409 if its version is newer. On 409, I fetch the current server version and show a diff UI letting the user choose which version to keep. For true multi-user collaboration (like Google Docs), LWW per field for non-overlapping edits, and CRDTs for concurrent edits to the same field — but that's a significant investment and I'd only go there if the product requires real-time co-editing.

---

**Q4: How does delta sync work, and what's the problem with using client timestamps?**

> Delta sync fetches only records changed since the last sync — `GET /notes?since=<timestamp>`. It's far more efficient than full sync for large datasets. The problem with client timestamps: the client clock can be wrong (user changed timezone, clock drift, time zone bugs). If the client clock is 5 minutes behind, every sync misses the last 5 minutes of changes. The fix: use a server-issued opaque sync token. After each sync, the server returns a token representing "you're up to date as of this point." The client stores it and sends it back on the next sync. The server interprets the token — the client never needs to know what's inside it.

---

**Q5: When would you not go offline-first?**

> Any time the action's correctness depends on real-time server state. Seat booking — I can't show you a seat as "selected" locally if the server might have given it to someone else in the last 100ms. Payment — I can't optimistically tell you a payment succeeded before the payment processor confirms. Live inventory — if I cache product stock, I might let you add an out-of-stock item to cart. In these cases I use network-first with a local fallback for reads only, and I make writes blocking: show a spinner, wait for the server, then show the result. The latency is worth the correctness guarantee.

---

**Q6: What's the outbox pattern and why use it over just marking rows as PENDING?**

> The outbox pattern stores user operations as discrete events in a separate queue table, not just as flags on data rows. The advantage: it preserves operation ordering and intent. If a user creates a note, edits it, then deletes it — all while offline — the outbox records three operations in order: CREATE, UPDATE, DELETE. When syncing, I replay them in sequence. If I only used a PENDING flag on the note row, by the time I sync the row would show "deleted" — I wouldn't know about the CREATE and UPDATE. The outbox also handles the case where the same entity is edited multiple times offline: each edit is a separate operation that can be applied atomically.

---

*Previous: [02 — Networking & API Design](./02-networking-api.md)*  
*Next: [04 — Data Layer: DB & Cache](./04-data-layer.md)*
