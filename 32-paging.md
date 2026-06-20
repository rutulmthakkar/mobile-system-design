# 32 — Paging 3 & Pagination

> **Round type: LLD (primary) + HLD**
> **Focus:** Paging 3 architecture — PagingSource, RemoteMediator, Compose integration, separators, retry, testing

---

## 1. Why This Matters

Pagination is in every feed, search result, and list at scale. Paging 3 is the standard Android solution and frequently tested in LLD rounds. Interviewers probe: when do you use RemoteMediator vs PagingSource alone? How do you add separators? How do you test pagination?

---

## 2. Paging 3 Architecture

```
                    ┌─────────────────────────────────────────┐
                    │               Pager                     │
                    │  PagingConfig + PagingSource factory    │
                    │  optional: RemoteMediator               │
                    └───────────────┬─────────────────────────┘
                                    │ Flow<PagingData<T>>
                                    ▼
                    ┌───────────────────────────────────────  ┐
                    │            ViewModel                    │
                    │  .cachedIn(viewModelScope)              │
                    └───────────────┬─────────────────────────┘
                                    │ Flow<PagingData<T>>
                                    ▼
                    ┌─────────────────────────────────────────┐
                    │       Compose / LazyColumn              │
                    │  collectAsLazyPagingItems()             │
                    │  items(lazyPagingItems) { }             │
                    └─────────────────────────────────────────┘
```

**Key classes:**
- `PagingSource<Key, Value>` — loads pages from a single source (network OR database)
- `RemoteMediator` — coordinates network + database (network-first, cache-backed)
- `Pager` — creates the `Flow<PagingData<T>>` from config + source
- `PagingConfig` — page size, prefetch distance, initial load size
- `PagingData<T>` — snapshot of paginated data + load states
- `LazyPagingItems<T>` — Compose-friendly wrapper, consumed in `LazyColumn`

---

## 3. PagingSource — Single Source

Use when loading from network only OR database only (no sync between them):

```kotlin
// gradle
implementation("androidx.paging:paging-runtime-ktx:3.3.6")
implementation("androidx.paging:paging-compose:3.3.6")

class PostPagingSource(
    private val api: PostApi,
    private val query: String
) : PagingSource<Int, Post>() {

    override fun getRefreshKey(state: PagingState<Int, Post>): Int? {
        // Which page to reload when data is invalidated
        // anchorPosition = index of most recently accessed item
        return state.anchorPosition?.let { anchor ->
            state.closestPageToPosition(anchor)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchor)?.nextKey?.minus(1)
        }
    }

    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Post> {
        val page = params.key ?: 1  // start from page 1

        return try {
            val response = api.getPosts(
                query = query,
                page = page,
                pageSize = params.loadSize  // Paging 3 controls load size
            )
            LoadResult.Page(
                data = response.posts,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.posts.isEmpty()) null else page + 1
                // nextKey = null signals end of data — no more pages
            )
        } catch (e: IOException) {
            LoadResult.Error(e)   // triggers error state in UI
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }
}
```

### 3.1 Cursor-based pagination

```kotlin
class CursorPagingSource(private val api: PostApi) : PagingSource<String, Post>() {
    override fun getRefreshKey(state: PagingState<String, Post>): String? = null

    override suspend fun load(params: LoadParams<String>): LoadResult<String, Post> {
        val cursor = params.key  // null = first page
        return try {
            val response = api.getPosts(cursor = cursor, limit = params.loadSize)
            LoadResult.Page(
                data = response.posts,
                prevKey = null,                       // no backwards paging
                nextKey = response.nextCursor         // null when no more data
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}
```

---

## 4. RemoteMediator — Network + DB

Use when you want offline support: load from DB, refresh from network, cache to DB.

```kotlin
@OptIn(ExperimentalPagingApi::class)
class PostRemoteMediator(
    private val api: PostApi,
    private val db: AppDatabase,
    private val query: String
) : RemoteMediator<Int, PostEntity>() {

    override suspend fun initialize(): InitializeAction {
        // LAUNCH_INITIAL_REFRESH: always refresh on open (ignores db cache age)
        // SKIP_INITIAL_REFRESH: use DB cache immediately, refresh only when user scrolls to end
        val cacheAge = db.postDao().getNewestPostAge(query)
        return if (cacheAge != null && System.currentTimeMillis() - cacheAge < 1.hours.inWholeMilliseconds) {
            InitializeAction.SKIP_INITIAL_REFRESH  // cache is fresh
        } else {
            InitializeAction.LAUNCH_INITIAL_REFRESH
        }
    }

    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, PostEntity>
    ): MediatorResult {

        val page = when (loadType) {
            LoadType.REFRESH -> {
                // User pulled-to-refresh or initial load
                val remoteKeys = getRemoteKeyClosestToCurrentPosition(state)
                remoteKeys?.nextKey?.minus(1) ?: 1  // restart from page 1 on refresh
            }
            LoadType.PREPEND -> {
                // Scrolled to the top — in most feeds, no prepend needed
                val remoteKeys = getRemoteKeyForFirstItem(state)
                remoteKeys?.prevKey ?: return MediatorResult.Success(endOfPaginationReached = true)
            }
            LoadType.APPEND -> {
                // Scrolled to the bottom — load next page
                val remoteKeys = getRemoteKeyForLastItem(state)
                remoteKeys?.nextKey ?: return MediatorResult.Success(endOfPaginationReached = true)
            }
        }

        return try {
            val response = api.getPosts(query = query, page = page, pageSize = state.config.pageSize)
            val endOfPagination = response.posts.isEmpty()

            db.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    // Clear stale data on refresh
                    db.remoteKeysDao().clearRemoteKeys(query)
                    db.postDao().clearPosts(query)
                }
                // Save remote keys (maps each item to its page position)
                val keys = response.posts.map { post ->
                    RemoteKey(
                        postId = post.id,
                        query = query,
                        prevKey = if (page == 1) null else page - 1,
                        nextKey = if (endOfPagination) null else page + 1
                    )
                }
                db.remoteKeysDao().insertAll(keys)
                db.postDao().insertAll(response.posts.map { it.toEntity(query) })
            }
            MediatorResult.Success(endOfPaginationReached = endOfPagination)
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }

    // Helpers to find page keys for each load type
    private suspend fun getRemoteKeyForLastItem(state: PagingState<Int, PostEntity>) =
        state.pages.lastOrNull { it.data.isNotEmpty() }?.data?.lastOrNull()
            ?.let { db.remoteKeysDao().getRemoteKey(it.id, query) }

    private suspend fun getRemoteKeyForFirstItem(state: PagingState<Int, PostEntity>) =
        state.pages.firstOrNull { it.data.isNotEmpty() }?.data?.firstOrNull()
            ?.let { db.remoteKeysDao().getRemoteKey(it.id, query) }

    private suspend fun getRemoteKeyClosestToCurrentPosition(state: PagingState<Int, PostEntity>) =
        state.anchorPosition?.let { anchor ->
            state.closestItemToPosition(anchor)?.let {
                db.remoteKeysDao().getRemoteKey(it.id, query)
            }
        }
}
```

### 4.1 Remote Keys Entity

```kotlin
@Entity(tableName = "remote_keys")
data class RemoteKey(
    @PrimaryKey val postId: String,
    val query: String,
    val prevKey: Int?,
    val nextKey: Int?
)

@Dao
interface RemoteKeyDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(keys: List<RemoteKey>)

    @Query("SELECT * FROM remote_keys WHERE postId = :postId AND query = :query")
    suspend fun getRemoteKey(postId: String, query: String): RemoteKey?

    @Query("DELETE FROM remote_keys WHERE query = :query")
    suspend fun clearRemoteKeys(query: String)
}
```

---

## 5. Pager + ViewModel

```kotlin
@HiltViewModel
class FeedViewModel @Inject constructor(
    private val postPagingSource: PostPagingSourceFactory,
    private val db: AppDatabase
) : ViewModel() {

    // PagingSource-only (no offline cache)
    val posts: Flow<PagingData<Post>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            prefetchDistance = 5,         // load more when 5 items from bottom
            initialLoadSize = 40,         // load 2x on first fetch (faster apparent load)
            enablePlaceholders = false    // true: shows blank item slots while loading
        ),
        pagingSourceFactory = { postPagingSource.create(query) }
    ).flow.cachedIn(viewModelScope)  // ← CRITICAL: cache survives rotation

    // RemoteMediator (network + Room DB)
    val cachedPosts: Flow<PagingData<Post>> = Pager(
        config = PagingConfig(pageSize = 20),
        remoteMediator = PostRemoteMediator(api, db, query),
        pagingSourceFactory = { db.postDao().getPosts(query) }  // Room provides PagingSource
    ).flow
        .map { pagingData -> pagingData.map { it.toDomain() } }  // entity → domain model
        .cachedIn(viewModelScope)
}
```

---

## 6. Compose Integration

```kotlin
@Composable
fun FeedScreen(viewModel: FeedViewModel = hiltViewModel()) {
    val posts = viewModel.posts.collectAsLazyPagingItems()

    LazyColumn {
        // Items
        items(
            count = posts.itemCount,
            key = posts.itemKey { it.id }
        ) { index ->
            val post = posts[index]
            if (post != null) {
                PostCard(post = post)
            } else {
                // enablePlaceholders = true: show loading card for uncached item
                PostCardPlaceholder()
            }
        }

        // Loading state at bottom (appending)
        when (val appendState = posts.loadState.append) {
            is LoadState.Loading -> item { LoadingSpinner() }
            is LoadState.Error -> item {
                ErrorItem(
                    message = appendState.error.message ?: "Load failed",
                    onRetry = { posts.retry() }
                )
            }
            is LoadState.NotLoading -> { /* no indicator */ }
        }
    }

    // Refresh state (pull-to-refresh or initial load)
    when (val refreshState = posts.loadState.refresh) {
        is LoadState.Loading -> LoadingOverlay()
        is LoadState.Error -> ErrorScreen(
            message = refreshState.error.message ?: "Failed to load",
            onRetry = { posts.refresh() }
        )
        is LoadState.NotLoading -> { /* content shown */ }
    }
}

// Pull to refresh integration
val pullRefreshState = rememberPullRefreshState(
    refreshing = posts.loadState.refresh is LoadState.Loading,
    onRefresh = { posts.refresh() }
)
Box(Modifier.pullRefresh(pullRefreshState)) {
    LazyColumn { /* ... */ }
    PullRefreshIndicator(/* ... */)
}
```

---

## 7. Separators

```kotlin
// Add date headers between items in the paged data
val postsWithSeparators: Flow<PagingData<FeedItem>> = viewModel.posts
    .map { pagingData ->
        pagingData
            .map { post -> FeedItem.PostItem(post) }
            .insertSeparators { before, after ->
                if (before == null) return@insertSeparators null  // top of list
                if (after == null) return@insertSeparators null   // bottom of list

                val beforeDate = (before as FeedItem.PostItem).post.createdAtDay
                val afterDate = (after as FeedItem.PostItem).post.createdAtDay

                if (beforeDate != afterDate) {
                    FeedItem.DateSeparator(afterDate)  // insert date header
                } else null
            }
    }
    .cachedIn(viewModelScope)

// Sealed class for heterogeneous list
sealed class FeedItem {
    data class PostItem(val post: Post) : FeedItem()
    data class DateSeparator(val date: String) : FeedItem()
}
```

---

## 8. Testing PagingSource

```kotlin
class PostPagingSourceTest {

    private val fakeApi = FakePostApi()
    private val pagingSource = PostPagingSource(fakeApi, query = "kotlin")

    @Test
    fun `load returns page with correct data`() = runTest {
        val result = pagingSource.load(
            PagingSource.LoadParams.Refresh(
                key = null,
                loadSize = 20,
                placeholdersEnabled = false
            )
        )
        assertIs<PagingSource.LoadResult.Page<Int, Post>>(result)
        assertEquals(20, result.data.size)
        assertNull(result.prevKey)
        assertEquals(2, result.nextKey)  // next page is 2
    }

    @Test
    fun `load returns error on API failure`() = runTest {
        fakeApi.shouldFail = true
        val result = pagingSource.load(
            PagingSource.LoadParams.Refresh(key = null, loadSize = 20, placeholdersEnabled = false)
        )
        assertIs<PagingSource.LoadResult.Error<Int, Post>>(result)
    }

    @Test
    fun `load returns empty page when API returns nothing`() = runTest {
        fakeApi.posts = emptyList()
        val result = pagingSource.load(
            PagingSource.LoadParams.Append(key = 2, loadSize = 20, placeholdersEnabled = false)
        ) as PagingSource.LoadResult.Page
        assertTrue(result.data.isEmpty())
        assertNull(result.nextKey)  // end of pagination
    }
}
```

---

## 9. PagingSource vs RemoteMediator — Decision Guide

| | PagingSource | RemoteMediator + PagingSource |
|---|---|---|
| Offline support | ❌ | ✅ |
| Source | Single (network OR DB) | Network + DB layered |
| Complexity | Low | High |
| Room required | Optional | ✅ Required |
| Data freshness | Always fresh | May be stale (configurable) |
| Use case | Simple network list | Feed, search with offline |

**Rule:** Use bare `PagingSource` for simple, online-only lists. Use `RemoteMediator` for any list that should work offline or needs caching between sessions.

---

## 10. Common Pitfalls

**❌ Forgetting `.cachedIn(viewModelScope)`**
Without `cachedIn`, rotating the device creates a new `PagingSource` and re-fetches from page 1. Always `cachedIn`.

**❌ Using `collectAsState()` instead of `collectAsLazyPagingItems()`**
`collectAsState()` collects `PagingData` but loses the load state hooks and Compose integration. Always use `collectAsLazyPagingItems()`.

**❌ Not handling all three `loadState` types**
Each has three states (Loading, Error, NotLoading) for three positions (refresh, prepend, append). Only handling `refresh.Loading` and ignoring `append.Error` means silent failures at the bottom of the list.

**❌ `RemoteMediator` without clearing on REFRESH**
If you don't `DELETE` the stale data and remote keys on `LoadType.REFRESH`, you'll append new page 1 results to the old cache — duplicates.

---

## 11. Best Practices

- **`cachedIn(viewModelScope)`** — always, prevents re-fetch on rotation
- **`initialLoadSize = 2 * pageSize`** — loads more on first open for smoother UX
- **`enablePlaceholders = false`** — true only if you have exact item count from API
- **`prefetchDistance = 5`** — trigger next page load 5 items before end
- **Separate Remote Keys table** — required for correct RemoteMediator navigation
- **`db.withTransaction`** in RemoteMediator** — atomic write of data + keys
- **`insertSeparators` for headers** — date separators, section headers stay in sync with data
- **Test PagingSource directly** — fast, no Compose test harness needed

---

*Previous: [31 — WebRTC](./31-webrtc.md)*
*Next: [33 — Testing Strategies](./33-testing.md)*
