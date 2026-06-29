# Realm DB — Quick Reference

> **SDK:** Realm Kotlin (GA) — use this for all new projects. Realm Java is being sunset.
> **Round type: LLD + occasional HLD** ("design offline-first storage for a complex object graph")
> ⚠️ **2025 critical note:** Atlas Device Sync (MongoDB) deprecated September 2024, removed by September 2025. If you were using Sync, migrate now. Local-only Realm remains fully supported.

---

## 1. What Realm Actually Is

Realm is **not** SQLite. It's a custom object-oriented database engine that runs directly on device.

```
Room (SQLite wrapper):
  Kotlin object → SQL row → SQLite file → query → SQL row → Kotlin object
  (Multiple copies, ORM translation at every boundary)

Realm (object store):
  Kotlin object → stored directly as object → zero-copy read back
  (One copy — the object you query IS the data, no translation)
```

This zero-copy design is why Realm benchmarks faster than Room for read-heavy workloads. The tradeoff: your model objects must extend `RealmObject` — they're not plain data classes.

---

## 2. Setup

```kotlin
// build.gradle.kts (project)
plugins {
    id("io.realm.kotlin") version "2.3.0" apply false
}

// build.gradle.kts (app/module)
plugins {
    id("io.realm.kotlin")
}

dependencies {
    // Local only (no sync)
    implementation("io.realm.kotlin:library-base:2.3.0")

    // With (now-deprecated) sync — avoid for new projects
    // implementation("io.realm.kotlin:library-sync:2.3.0")
}
```

⚠️ Check Realm's version compatibility matrix before upgrading Kotlin — Realm Kotlin has strict Kotlin version requirements and did NOT support Kotlin 2.1 at time of writing.

---

## 3. Defining Models

```kotlin
// RealmObject — top-level stored entity
class Beer : RealmObject {
    @PrimaryKey
    var id: ObjectId = ObjectId()   // Realm's own ID type (preferred over Int)
    var name: String = ""
    var abv: Double = 0.0
    var tagline: String = ""
    var brewery: Brewery? = null    // direct object relationship (no foreign key needed)
    var tags: RealmList<String> = realmListOf()
}

// EmbeddedRealmObject — owned child, cannot exist without parent
// Deleted when parent is deleted. No separate primary key.
class Brewery : EmbeddedRealmObject {
    var name: String = ""
    var city: String = ""
    var country: String = ""
}
```

### Key model types

| Type | When to use |
|---|---|
| `RealmObject` | Top-level entity. Has its own lifecycle. Can be referenced from multiple parents. |
| `EmbeddedRealmObject` | Value object owned by a parent. Think `Address` inside `User`. Deleted with parent. |
| `RealmList<T>` | List of primitives or RealmObjects. Reactive — updates automatically. |
| `RealmSet<T>` | Unordered unique collection. |
| `RealmDictionary<K, V>` | Key-value map stored in Realm. |
| `@PrimaryKey` | Required for `RealmObject`. Use `ObjectId` (Realm type) or `String`/`Int`. |
| `@Ignore` | Field not persisted to DB. |
| `@Index` | Speeds up queries on that field. Add to fields you filter by frequently. |

### What you CANNOT do with Realm models

- `data class` — Realm objects cannot be data classes (no `copy()`, no structural equality)
- `val` properties — all persisted fields must be `var`
- Pass between threads freely — a managed `RealmObject` is thread-confined (see Threading)
- Default constructor with parameters — Realm requires a no-arg constructor

---

## 4. Opening a Realm

```kotlin
// Configuration
val config = RealmConfiguration.Builder(
    schema = setOf(Beer::class, Brewery::class)
)
    .name("beers.realm")
    .schemaVersion(1)
    .migration(BeerMigration())   // required when schemaVersion increments
    .deleteRealmIfMigrationNeeded() // ONLY in dev — destroys data on schema change
    .build()

// Open (keep open for app lifetime — expensive to open/close)
val realm = Realm.open(config)

// In a ViewModel or Repository — close when scope ends
class BeerRepository(private val realm: Realm) {
    // ...
}
```

**Rule:** Open Realm once per process. Treat it like a database connection. Do NOT open/close per operation.

---

## 5. CRUD Operations

### Write (always inside a transaction)

```kotlin
// Blocking write (use only off main thread)
realm.writeBlocking {
    copyToRealm(Beer().apply {
        name = "Punk IPA"
        abv = 5.6
    })
}

// Suspending write (preferred — Kotlin coroutines)
suspend fun saveBeer(name: String, abv: Double) {
    realm.write {
        copyToRealm(Beer().apply {
            this.name = name
            this.abv = abv
        })
    }
}

// Update inside write block
realm.write {
    val beer = query<Beer>("name == $0", "Punk IPA").first().find()
    beer?.abv = 6.0   // mutate directly — Realm tracks the change
}

// Delete
realm.write {
    val beer = query<Beer>("id == $0", targetId).first().find()
    beer?.let { delete(it) }
}
```

### Read (queries)

```kotlin
// All objects
val allBeers: RealmResults<Beer> = realm.query<Beer>().find()

// Filtered — Realm Query Language (RQL), not SQL
val strongBeers = realm.query<Beer>("abv > $0", 7.0).find()

// Sorted
val sorted = realm.query<Beer>().sort("name", Sort.ASCENDING).find()

// First match
val punkIpa = realm.query<Beer>("name == $0", "Punk IPA").first().find()

// Count
val count = realm.query<Beer>("abv > $0", 5.0).count().find()
```

### Reactive queries (Flow)

```kotlin
// This is Realm's killer feature — live, auto-updating results
val beersFlow: Flow<ResultsChange<Beer>> = realm.query<Beer>()
    .asFlow()

// Collect in ViewModel
viewModelScope.launch {
    realm.query<Beer>()
        .sort("name")
        .asFlow()
        .collect { change ->
            when (change) {
                is InitialResults -> _beers.value = change.list
                is UpdatedResults -> _beers.value = change.list
            }
        }
}
```

This is the core reactive pattern. No manual invalidation, no `notifyDataSetChanged()`. When the DB changes, the Flow emits automatically.

---

## 6. Relationships

```kotlin
// One-to-one (direct reference)
class User : RealmObject {
    var id: ObjectId = ObjectId()
    var profile: UserProfile? = null   // nullable = optional relationship
}

// One-to-many (RealmList)
class Brewery : RealmObject {
    var id: ObjectId = ObjectId()
    var beers: RealmList<Beer> = realmListOf()
}

// Many-to-many via backlinks (inverse relationship)
class Beer : RealmObject {
    var name: String = ""
    // Backlink: all Breweries that contain this Beer in their beers list
    val breweries: RealmResults<Brewery> by backlinks(Brewery::beers)
}
```

**Backlinks are not stored** — they're computed by Realm at query time from the forward relationship. Zero storage cost.

---

## 7. Threading — The Critical Rule

Realm objects are **thread-confined**. A managed `RealmObject` queried on thread A cannot be accessed on thread B.

```kotlin
// ❌ WRONG — crossing thread boundary with managed object
val beer = realm.query<Beer>().first().find()  // queried on thread A
withContext(Dispatchers.IO) {
    print(beer.name)  // CRASH — accessed on thread B
}

// ✅ Option 1: copy to unmanaged object before crossing threads
val beerCopy = realm.query<Beer>().first().find()?.let {
    Beer().apply { name = it.name; abv = it.abv }  // plain Kotlin copy
}
// Now beerCopy is safe to use anywhere

// ✅ Option 2: map to domain model in repository (RECOMMENDED for clean arch)
val beerDomain: BeerDomain = realm.query<Beer>()
    .first().find()?.toDomain() ?: error("not found")

// ✅ Option 3: use freeze() (Realm Kotlin only — makes object immutable + thread-safe)
val frozen = beer.freeze()  // now safe to pass across threads
```

**Clean architecture fix:** Never expose `RealmObject` beyond the data layer. Map to domain models at the repository boundary. This eliminates thread issues entirely.

---

## 8. Clean Architecture Integration

```
Presentation Layer
    BeerViewModel
    │  uses domain models (Beer — plain Kotlin data class)
    │
Domain Layer
    data class Beer(val id: String, val name: String, val abv: Double)
    interface BeerRepository { fun getBeers(): Flow<List<Beer>> }
    │
Data Layer
    class Beer : RealmObject { ... }       ← Realm model
    fun Beer.toDomain() = BeerDomain(      ← mapper
        id = id.toHexString(),
        name = name,
        abv = abv
    )
    class BeerRepositoryImpl(val realm: Realm) : BeerRepository {
        override fun getBeers(): Flow<List<Beer>> =
            realm.query<BeerRealmObject>()
                .asFlow()
                .map { it.list.map { beer -> beer.toDomain() } }
    }
```

**Rule:** `RealmObject` classes never leave the data layer. Map to plain Kotlin domain models before returning from any repository method. This solves threading, clean architecture purity, and testability in one move.

---

## 9. Migrations

```kotlin
// Increment schemaVersion whenever you change a model
val config = RealmConfiguration.Builder(schema = setOf(Beer::class))
    .schemaVersion(2)  // was 1
    .migration(object : AutomaticSchemaMigration {
        override fun migrate(migrationContext: AutomaticSchemaMigration.MigrationContext) {
            val oldRealm = migrationContext.oldRealm
            val newRealm = migrationContext.newRealm

            // Add a new field with a default value
            newRealm.schema()["Beer"]
                ?.addProperty("isFavorite", Boolean::class)

            // Copy old data
            oldRealm.query("Beer").find().forEach { old ->
                val new = newRealm.query("Beer", "name == $0", old["name"]).first().find()
                new?.set("isFavorite", false)
            }
        }
    })
    .build()
```

**Key migration helpers:** `addField`, `removeField`, `renameField` for simple schema changes.

**Dev shortcut:** `.deleteRealmIfMigrationNeeded()` — deletes all data on schema change. Never use in production.

---

## 10. Encryption

```kotlin
// AES-256 encryption — built-in, no SQLCipher dependency needed
val key = ByteArray(64)  // must be exactly 64 bytes
SecureRandom().nextBytes(key)
// Store key in Android Keystore — never hardcode

val config = RealmConfiguration.Builder(schema = setOf(Beer::class))
    .encryptionKey(key)
    .build()
```

Realm uses AES-256 for data + HMAC-SHA2 for integrity. Encryption is transparent — read/write API is identical to unencrypted.

---

## 11. Realm vs Room vs SQLDelight

| Dimension | Realm | Room | SQLDelight |
|---|---|---|---|
| **Data model** | Object store (NoSQL) | Relational (SQLite) | Relational (SQLite) |
| **Query language** | Realm Query Language (RQL) | SQL via annotations | Raw SQL (.sq files) |
| **KMP support** | Yes (Kotlin SDK) | No (Android only) | Yes |
| **Paging 3 integration** | Manual (extra work) | Native support | Extra work |
| **APK size impact** | +3–4 MB (own engine) | +~50 KB (SQLite is OS) | Similar to Room |
| **Compile-time safety** | No | Yes (KSP) | Yes (.sq files) |
| **Reactive queries** | Built-in (`.asFlow()`) | Via Flow from DAO | Via Flow |
| **Threading** | Thread-confined objects | Thread-safe | Thread-safe |
| **Migrations** | Manual, utility methods | Migration class | Manual SQL |
| **Encryption** | Built-in AES-256 | Via SQLCipher plugin | Via SQLCipher |
| **Complex joins** | Difficult (object traversal) | Easy (SQL JOINs) | Easy (SQL JOINs) |
| **Real-time sync** | Deprecated (Sept 2025) | No | No |
| **Kotlin 2.1 support** | ⚠️ Check matrix | Yes | Yes |
| **When to use** | Complex object graphs, cross-platform teams, Realm-heavy legacy | Android-only, relational data, Paging 3 | KMP, shared DB layer |
| **When to avoid** | Paging 3 apps, pure Android + SQL teams | KMP, object-graph data | Annotation-allergic teams |

---

## 12. When to Use Realm (Decision Guide)

```
Do you need Paging 3 integration?
├─ YES → Room. Realm + Paging 3 requires significant custom work.

Is this a KMP project AND you need a shared DB layer?
├─ YES → SQLDelight (simpler KMP story than Realm Kotlin)
│         OR Realm Kotlin (if team already knows Realm)

Is your data model highly relational (4+ table JOINs)?
├─ YES → Room. Complex joins in Realm require object traversal — painful.

Do you have complex nested object graphs with NO heavy SQL needed?
├─ YES → Realm is a great fit. Object traversal is cleaner than JOINs.

Do you need built-in encryption without a third-party plugin?
├─ YES → Realm (AES-256 built-in) or Room + SQLCipher (manual setup)

Are you on an Android-only app with a team that knows SQL?
└─ Room. Realm's learning curve + threading gotchas aren't worth it.

Did Atlas Device Sync drive your Realm choice?
└─ ⚠️ Sync is removed September 2025. Evaluate ObjectBox Sync or
       custom sync strategy before committing to Realm.
```

---

## 13. Common Pitfalls

| Mistake | Why it fails | Fix |
|---|---|---|
| Treating `RealmObject` as a data class | No `copy()`, no structural equality | Map to domain model in repository |
| Accessing managed object across threads | Realm objects are thread-confined | `freeze()` or map to plain Kotlin before crossing threads |
| Opening/closing Realm per operation | Realm open is expensive (~100ms) | Open once, keep alive for app lifetime |
| Using `deleteRealmIfMigrationNeeded()` in production | Silently deletes all user data on app update | Write real migrations, gate behind `schemaVersion` |
| Forgetting `@PrimaryKey` | Realm can create duplicate objects silently | Always define a primary key on `RealmObject` |
| Storing `RealmObject` in ViewModel state | Leaks Realm instance, threading issues | Map to domain model first |
| Not closing Realm in tests | File locks cause flaky tests | `realm.close()` in `@After` |
| Using `val` for Realm fields | Compilation error or ignored by Realm | All persisted fields must be `var` |

---

## 14. Interview Q&A

**Q1: What's the fundamental difference between Realm and Room?**
> Room is a layer over SQLite — you work with SQL under the hood, data is stored in tables, and Kotlin objects are an ORM abstraction. Realm has its own custom storage engine. Data is stored directly as objects — there's no SQL, no table, and no ORM translation. This zero-copy design is why Realm reads faster, but it means your model objects must extend `RealmObject` rather than being plain data classes.

**Q2: How does Realm handle reactivity?**
> Realm's queries return "live" results — `RealmResults` objects that automatically reflect any DB change. Wrapped with `.asFlow()`, you get a `Flow` that emits whenever the query result changes. You don't call invalidate or notify manually. When a write happens anywhere in the app that affects your query, the Flow emits the new list. This is Realm's biggest advantage over Room for reactive UIs, though Room's Flow integration achieves the same result through SQLite triggers.

**Q3: What's the threading rule with Realm?**
> Realm objects are thread-confined. A managed `RealmObject` instance is tied to the thread (or coroutine dispatcher) it was queried on. Accessing it from another thread throws an exception. The clean architecture solution is to never expose `RealmObject` outside the data layer — map to plain Kotlin domain models at the repository boundary, and those models are thread-safe.

**Q4: Atlas Device Sync is deprecated — what does that mean for Realm today?**
> MongoDB deprecated Atlas Device Sync in September 2024 and it's removed by September 2025. This was Realm's headline feature — automatic multi-device sync via MongoDB Atlas. Without it, Realm is now a local-only object database. The core local database functionality is still supported and open source. Teams that relied on sync need to build a custom sync layer or migrate to alternatives like ObjectBox Sync.

**Q5: When would you choose Realm over Room in a greenfield Android app?**
> Honestly, for most Android-only apps today, Room is the default. Realm makes sense when: your data model is a complex object graph with many nested relationships that would be painful to model as SQL JOINs, or when you're sharing the data layer with an iOS team that also uses Realm, or when you need built-in AES-256 encryption without adding SQLCipher. I'd avoid Realm if I need Paging 3 integration, if the team is SQL-fluent, or if Kotlin 2.x compatibility is a concern — Realm's Kotlin version support has lagged behind.
