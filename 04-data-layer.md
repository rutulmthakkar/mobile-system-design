# 04 — Data Layer: DB & Cache

> **Round type: Both (HLD + LLD)**
> **HLD:** How to choose and architect local storage for a large-scale mobile app
> **LLD:** Room schema design, DAOs, migrations, cache eviction, DataStore usage

---

## 1. Why This Matters in Interviews

The data layer is where the offline-first contract is enforced, where bugs are hardest to reproduce (they involve persisted state across sessions), and where performance bottlenecks hide. A weak data layer causes ANRs, stale UI, data corruption on migration, and OOM crashes.

**Common interview angles:**
- "Walk me through your local storage strategy for a messaging app"
- "How do you choose between Room, DataStore, and in-memory cache?"
- "What happens to your local DB when a user upgrades the app?"
- "How do you cache images efficiently?"
- "Design an LRU cache from scratch"

---

## 2. Storage Options — Decision Map

```
What are you storing?
│
├── Small key-value data (settings, tokens, flags)
│     └── Jetpack DataStore (Preferences or Proto)
│           ❌ NOT SharedPreferences (ANR risk, not coroutine-safe)
│
├── Structured relational data (users, posts, messages)
│     └── Room (SQLite ORM)
│
├── Large binary files (images, video, audio)
│     └── File system (internal/external storage)
│           + Memory/disk LRU cache for fast retrieval
│
├── Tiny transient data (current session, scroll position)
│     └── In-memory (ViewModel state, SavedStateHandle)
│
└── Network responses (HTTP caching)
      └── OkHttp Cache (disk-backed HTTP cache)
```

---

## 3. Jetpack DataStore

### 3.1 Why Not SharedPreferences

SharedPreferences has four production-critical problems:
- **Synchronous API** — `getString()`, `commit()` block the calling thread. Called on main thread = ANR
- **Not coroutine/Flow-safe** — no reactive observation; you manually register listeners
- **Not type-safe** — wrong key type throws a runtime exception, not a compile error
- **No transactional writes** — two concurrent writes can corrupt the file

### 3.2 Preferences DataStore (key-value)

```kotlin
// Setup — one instance per process (singleton via delegate)
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

// Keys are typed at compile time
private val DARK_MODE_KEY = booleanPreferencesKey("dark_mode")
private val AUTH_TOKEN_KEY = stringPreferencesKey("auth_token")
private val ONBOARDING_DONE_KEY = booleanPreferencesKey("onboarding_complete")

// Read — returns Flow, always fresh
val isDarkMode: Flow<Boolean> = context.dataStore.data
    .map { prefs -> prefs[DARK_MODE_KEY] ?: false }

// Write — suspend function, safe on any coroutine
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { prefs ->
        prefs[DARK_MODE_KEY] = enabled
    }
}
```

### 3.3 Proto DataStore (typed objects)

Use when you have structured preferences with multiple related fields — avoids managing many individual keys.

```proto
// user_prefs.proto
syntax = "proto3";
message UserPrefs {
  bool dark_mode = 1;
  string language_code = 2;
  int32 font_size_sp = 3;
  bool notifications_enabled = 4;
}
```

```kotlin
val userPrefsFlow: Flow<UserPrefs> = context.userPrefsDataStore.data

suspend fun setLanguage(code: String) {
    context.userPrefsDataStore.updateData { current ->
        current.toBuilder().setLanguageCode(code).build()
    }
}
```

**Proto DataStore vs Preferences DataStore:**

| | Preferences DataStore | Proto DataStore |
|---|---|---|
| Schema | No schema, typed keys | Defined in .proto file |
| Type safety | Key-level | Full object-level |
| Complexity | Low | Medium (protobuf setup) |
| Best for | Simple flags, tokens | Structured multi-field prefs |

### 3.4 DataStore vs Room

| | DataStore | Room |
|---|---|---|
| Data shape | Flat key-value or single object | Relational tables, joins |
| Partial updates | Whole object only | Per-column UPDATE |
| Queries | No query language | Full SQL |
| Relationships | None | FK, One-to-Many, Many-to-Many |
| Best for | User settings, session state | User data, posts, messages |

**Rule:** If you need `WHERE`, `JOIN`, or `ORDER BY` → Room. If you need to store a handful of app-wide flags → DataStore.

---

## 4. Room — Deep Dive

### 4.1 Architecture

```
@Database  ← version, list of entities, DAOs
    │
    ├── @Entity       ← maps to a SQLite table
    ├── @Dao          ← SQL queries as Kotlin interface
    └── TypeConverters ← converts custom types to/from SQLite primitives
```

Room generates the implementation at compile time (KSP). SQL errors are caught at build time, not runtime.

### 4.2 Entity Design

```kotlin
@Entity(
    tableName = "messages",
    indices = [
        Index(value = ["conversation_id"]),           // speeds up filtered queries
        Index(value = ["sender_id", "created_at"])    // composite: sort by sender+time
    ],
    foreignKeys = [
        ForeignKey(
            entity = ConversationEntity::class,
            parentColumns = ["id"],
            childColumns = ["conversation_id"],
            onDelete = ForeignKey.CASCADE             // auto-delete messages when conversation deleted
        )
    ]
)
data class MessageEntity(
    @PrimaryKey val id: String,                     // client UUID — no autoGenerate for offline-first
    val conversationId: String,
    val senderId: String,
    val content: String,
    val createdAt: Long,
    val syncStatus: SyncStatus = SyncStatus.PENDING,
    val isDeleted: Boolean = false                  // soft delete
)

// TypeConverter for enums (Room doesn't store enums natively)
class Converters {
    @TypeConverter fun fromSyncStatus(status: SyncStatus): String = status.name
    @TypeConverter fun toSyncStatus(value: String): SyncStatus = SyncStatus.valueOf(value)
}
```

### 4.3 DAO Design

```kotlin
@Dao
interface MessageDao {

    // Flow = reactive — UI auto-updates when data changes
    @Query("SELECT * FROM messages WHERE conversation_id = :convId AND is_deleted = 0 ORDER BY created_at ASC")
    fun observeMessages(convId: String): Flow<List<MessageEntity>>

    // One-shot suspend for background operations
    @Query("SELECT * FROM messages WHERE sync_status = 'PENDING'")
    suspend fun getPending(): List<MessageEntity>

    // Upsert — insert or update on conflict (Room 2.5+)
    @Upsert
    suspend fun upsert(message: MessageEntity)

    @Upsert
    suspend fun upsertAll(messages: List<MessageEntity>)

    // Targeted field update — more efficient than full upsert
    @Query("UPDATE messages SET sync_status = :status WHERE id = :id")
    suspend fun updateSyncStatus(id: String, status: SyncStatus)

    // Soft delete
    @Query("UPDATE messages SET is_deleted = 1, sync_status = 'PENDING' WHERE id = :id")
    suspend fun softDelete(id: String)

    // Transaction — atomic multi-step operation
    @Transaction
    suspend fun replaceConversationMessages(convId: String, messages: List<MessageEntity>) {
        deleteConversationMessages(convId)
        upsertAll(messages)
    }

    @Query("DELETE FROM messages WHERE conversation_id = :convId")
    suspend fun deleteConversationMessages(convId: String)
}
```

### 4.4 Database Setup (Singleton)

```kotlin
@Database(
    entities = [MessageEntity::class, ConversationEntity::class, UserEntity::class],
    version = 3,
    exportSchema = true    // exports schema JSON — commit this to VCS
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun messageDao(): MessageDao
    abstract fun conversationDao(): ConversationDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                .setJournalMode(JournalMode.WRITE_AHEAD_LOGGING)  // concurrent reads + 1 write
                .build()
                .also { INSTANCE = it }
            }
    }
}
```

**In production: inject via Hilt.** Don't call `getInstance()` from ViewModels.

### 4.5 Migrations

A migration is required every time the schema changes. Room throws `IllegalStateException` on version mismatch with no migration.

```kotlin
// Version 1 → 2: add 'is_deleted' column
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE messages ADD COLUMN is_deleted INTEGER NOT NULL DEFAULT 0")
    }
}

// Version 2 → 3: add index on conversation_id
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("CREATE INDEX IF NOT EXISTS index_messages_conversation_id ON messages(conversation_id)")
    }
}
```

**Renaming a column (SQLite ALTER TABLE can't do this directly):**
```kotlin
val MIGRATION_3_4 = object : Migration(3, 4) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // 1. Create new table with correct schema
        db.execSQL("CREATE TABLE messages_new (id TEXT PRIMARY KEY, body TEXT NOT NULL, ...)")
        // 2. Copy data, mapping old column name to new
        db.execSQL("INSERT INTO messages_new SELECT id, content AS body, ... FROM messages")
        // 3. Drop old table
        db.execSQL("DROP TABLE messages")
        // 4. Rename new table
        db.execSQL("ALTER TABLE messages_new RENAME TO messages")
    }
}
```

**AutoMigration for simple changes (Room 2.4+):**
```kotlin
@Database(version = 5, autoMigrations = [AutoMigration(from = 4, to = 5)])
```

**Always test migrations:**
```kotlin
@Test
fun migrate1To2() {
    val db = migrationTestHelper.createDatabase(TEST_DB, 1)
    db.close()
    migrationTestHelper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)
}
```

### 4.6 Performance Tips

```kotlin
// 1. Batch inserts inside @Transaction — 50-100x faster than individual inserts
@Transaction
suspend fun insertAll(items: List<ItemEntity>) = dao.insertAll(items)

// 2. Use projections — don't SELECT * when you need 3 columns
data class UserSummary(val id: String, val name: String, val avatarUrl: String)
@Query("SELECT id, name, avatar_url FROM users WHERE id = :id")
suspend fun getUserSummary(id: String): UserSummary

// 3. Verify index usage
// Run in DB Inspector or adb shell:
// EXPLAIN QUERY PLAN SELECT * FROM messages WHERE conversation_id = '123'
// Should say: USING INDEX — not FULL TABLE SCAN
```

---

## 5. In-Memory Cache (LRU)

### 5.1 What LRU Is

Fixed-size cache. When full, evicts the item least recently accessed. O(1) get/put using HashMap + doubly linked list.

```
Cache capacity = 3
Access: A → B → C → A → D
After A:  [A]
After B:  [A, B]
After C:  [A, B, C]   ← full
After A:  [B, C, A]   ← A moved to front (recently used)
After D:  [C, A, D]   ← B evicted (least recently used)
```

### 5.2 Android's LruCache

```kotlin
val maxMemory = (Runtime.getRuntime().maxMemory() / 1024).toInt()
val cacheSize = maxMemory / 5  // 20% of available heap

val memoryCache = object : LruCache<String, Bitmap>(cacheSize) {
    override fun sizeOf(key: String, value: Bitmap): Int = value.byteCount / 1024
}
```

### 5.3 LRU Cache — LLD Implementation (Interview)

```kotlin
class LRUCache<K, V>(private val capacity: Int) {
    // LinkedHashMap with accessOrder=true maintains LRU order automatically
    private val map = LinkedHashMap<K, V>(capacity, 0.75f, true)

    @Synchronized
    fun get(key: K): V? = map[key]

    @Synchronized
    fun put(key: K, value: V) {
        if (map.size >= capacity && !map.containsKey(key)) {
            val lruKey = map.keys.iterator().next()  // first entry = LRU
            map.remove(lruKey)
        }
        map[key] = value
    }

    @Synchronized
    fun remove(key: K) { map.remove(key) }
}
// Time complexity: O(1) get, O(1) put, O(1) eviction
// Space: O(capacity)
```

### 5.4 Three-Layer Cache Architecture

```
Request
    │
    ▼
[L1: Memory LruCache]   ← microseconds, volatile, process-scoped
    │ miss
    ▼
[L2: Disk / Room]       ← milliseconds, persistent, survives process death
    │ miss
    ▼
[L3: Network]           ← hundreds of ms, always authoritative
    │ response
    └──→ write to L2 → write to L1
```

### 5.5 Cache Eviction Policies

| Policy | Evicts | Best for |
|---|---|---|
| **LRU** | Oldest accessed | General purpose — temporal locality |
| **LFU** | Least frequently accessed | Popularity-based content (news, products) |
| **FIFO** | Oldest inserted | Sequential data, logs |
| **TTL** | Expired by time | Time-sensitive data (prices, sessions) |
| **Size-based** | Largest item first | Storage-pressure scenarios |

LRU is correct for most mobile use cases — recently accessed items are most likely to be needed again.

### 5.6 OkHttp HTTP Cache

```kotlin
val cache = Cache(File(context.cacheDir, "http_cache"), 10L * 1024 * 1024)
val client = OkHttpClient.Builder().cache(cache).build()
```

OkHttp respects server `Cache-Control` headers:
- `max-age=300` → serve from cache for 5 minutes
- `no-store` → never cache
- `no-cache` → cache but revalidate with ETag before serving

**Force stale cache when offline:**
```kotlin
val offlineRequest = request.newBuilder()
    .cacheControl(CacheControl.Builder().maxStale(7, TimeUnit.DAYS).build())
    .build()
```

---

## 6. HLD vs LLD Quick Reference

| Question type | Focus |
|---|---|
| HLD | Which storage tier for which data type, schema sketch, cache hierarchy |
| LLD | Entity design, DAO methods, migration code, LRU implementation, eviction policy choice |

---

## 7. React Native Equivalent

| Android | React Native |
|---|---|
| Room | WatermelonDB, expo-sqlite, op-sqlite |
| DataStore | MMKV (fastest), AsyncStorage (simple) |
| LruCache | Custom JS Map with size tracking |
| OkHttp Cache | No built-in equivalent |

```typescript
// MMKV — synchronous, C++ backed, 10-100x faster than AsyncStorage
import { MMKV } from 'react-native-mmkv'
const storage = new MMKV()
storage.set('token', value)
const token = storage.getString('token')  // no await needed
```

---

## 8. Common Misunderstandings & Pitfalls

**❌ `fallbackToDestructiveMigration()` in production**
Drops and recreates the entire database on version mismatch — all user data is gone. Only use in dev/debug builds. Always write explicit migrations for production.

**❌ Room queries on the main thread**
Room throws `IllegalStateException` by default. Never use `allowMainThreadQueries()` outside of tests. Always use `suspend` or `Flow`.

**❌ Multiple Room database instances**
Room opens file handles. Multiple instances on the same file = corruption. Use singleton via Hilt `@Singleton` or a `companion object`.

**❌ Forgetting indices on foreign key columns**
Room does NOT auto-index FK columns. `SELECT * FROM messages WHERE conversation_id = X` without an index = full table scan. Always declare `@Index` on FK columns.

**❌ Storing large binary data in Room**
Room stores everything in one SQLite file. Large blobs (images, audio) bloat the file and slow all queries. Store file paths in Room; store actual data on the filesystem or via MediaStore.

**❌ Not exporting the schema**
`exportSchema = false` disables schema history. Export it (`= true`) and commit the generated JSON. It enables `MigrationTestHelper` and creates an audit trail of schema changes.

**❌ `@Insert(onConflict = REPLACE)` instead of `@Upsert`**
`REPLACE` deletes the old row and inserts a new one — this resets autoGenerate IDs, triggers `ON DELETE CASCADE` on child tables, and loses fields you didn't include. `@Upsert` (Room 2.5+) truly updates in place.

**❌ Reusing proto field numbers after removing a field**
Proto fields are identified by number. If you remove field `2` and add a new field as `2`, old clients will deserialize it with the wrong type. Always add new fields with new numbers; never reuse removed numbers.

---

## 9. Best Practices

- **One database, multiple DAOs** — one `@Database` class; one `@Dao` per entity group (not per screen)
- **`Flow` for reactive UI, `suspend` for one-shot background** — don't return `List<T>` from a DAO that the UI observes
- **`@Transaction` for any multi-table write** — atomicity guarantee; if one step fails, all roll back
- **`@Upsert` over `@Insert(REPLACE)`** — true in-place update, preserves child rows
- **Test with in-memory DB** — `Room.inMemoryDatabaseBuilder()` in unit tests; fast and isolated
- **`exportSchema = true` + commit JSON** — schema history, enables migration tests
- **WAL mode for write-heavy or concurrent access** — `.setJournalMode(WRITE_AHEAD_LOGGING)`
- **`EXPLAIN QUERY PLAN` on every frequent query** — catch full table scans before production
- **Periodic cache cleanup via WorkManager** — purge expired rows, orphaned files; don't let local DB grow unbounded
- **Encrypt sensitive DataStore / SharedPreferences** — use `EncryptedSharedPreferences` or Keystore-backed encryption for tokens and PII

---

## 10. Interview Q&A

**Q1: When would you use DataStore vs Room?**

> DataStore is for small, flat, app-wide state — user preferences, session tokens, onboarding flags. It gives you coroutine-safe async reads/writes and a reactive Flow API. Room is for structured relational data that needs querying, filtering, joining, or ordering — posts, messages, users. The deciding question: do I need WHERE, JOIN, or ORDER BY? If yes, Room. If I'm storing "dark mode is on" or "last sync timestamp", DataStore is simpler and correct.

---

**Q2: How do you handle a Room migration that renames a column?**

> SQLite's ALTER TABLE can only add columns on older Android versions. To rename: create a new table with the correct schema, copy data using INSERT INTO new SELECT ... AS newName FROM old, drop the old table, rename the new one. Room 2.4+ handles this automatically with `@RenameColumn` in `AutoMigration` spec. Either way, I write a migration test with `MigrationTestHelper` to verify data survives the migration without loss.

---

**Q3: Design an LRU cache. What's the time complexity?**

> HashMap for O(1) key lookup, doubly linked list for O(1) insertion and removal at any position, and access order tracking. Most recently used is at the head, LRU at the tail. On get: look up in map, move node to head. On put: if at capacity, remove tail node and its map entry, insert new at head. Java's `LinkedHashMap(capacity, loadFactor, accessOrder=true)` implements this natively. Android's `LruCache` wraps it with thread safety and a `sizeOf()` hook. Time complexity: O(1) get, O(1) put, O(1) eviction.

---

**Q4: Why does Room use Flow instead of returning data directly?**

> A direct return is a snapshot — stale the moment the table changes. Flow makes the DAO reactive: Room's invalidation tracker monitors queried tables and emits a new value whenever they change. This means the UI Composable collecting the Flow recomposes automatically when the sync worker writes fresh data — no manual refresh, no polling. It's what makes offline-first seamless: the user never needs to pull-to-refresh because the data layer pushes updates to the UI layer through the reactive stream.

---

**Q5: What's the risk of using `@Insert(onConflict = REPLACE)` for updates?**

> REPLACE doesn't update in place — it deletes the existing row and inserts a new one. This has two dangerous side effects: if the table has an autoGenerate primary key, the new row gets a new ID (breaking any references to the old ID); if other tables have a foreign key pointing to this row with `ON DELETE CASCADE`, their rows are deleted and re-inserted. `@Upsert` (Room 2.5+) fixes this by truly updating in place when the key exists, never triggering cascade deletes. For offline-first apps using UUID primary keys the ID issue is less critical, but the cascade behavior is always a risk.

---

**Q6: How do you secure locally stored auth tokens?**

> SharedPreferences are plaintext XML — readable on rooted devices. I use one of two approaches: `EncryptedSharedPreferences` from the Jetpack Security library, which wraps Android Keystore AES-256-GCM encryption transparently; or store the token directly in the Android Keystore via `KeyStore` API with `UserAuthenticationRequired = true` so it's only accessible after biometric/PIN unlock. For DataStore, I pair it with `EncryptedFile`. The Keystore is hardware-backed on supported devices — keys are generated inside the secure element and can't be extracted even on rooted devices.

---

*Previous: [03 — Offline-First & Sync](./03-offline-sync.md)*
*Next: [05 — Real-Time & Push](./05-realtime-push.md)*
