# 23 — Jetpack Compose Optimization

> **Round type: LLD (primary)**
> **Focus:** Recomposition mechanics, stability system, compiler reports, Strong Skipping Mode, phase-aware state reads, LazyList optimization, measuring and fixing — the deep internals interviewers probe at Google and senior Android roles

---

## 1. Why This Matters in Interviews

Compose optimization is a litmus test for senior Android engineers. The API is easy to learn; understanding *why* things recompose, *what* makes a type stable, and *how* to fix it without cargo-culting `@Stable` everywhere is what interviewers test.

**Common interview angles:**
- "What makes a composable skippable?"
- "Why does `List<T>` cause unnecessary recomposition?"
- "What's the Compose Compiler Report and what does it tell you?"
- "What is Strong Skipping Mode?"
- "When would you use `@NonRestartableComposable`?"
- "How do you make a scroll-driven animation not cause recomposition?"

---

## 2. Compose Rendering Phases (Recap)

Compose renders in three ordered phases. Understanding which phase reads which data is the key to optimization:

```
1. COMPOSITION   ← Composable functions execute, build slot table
                   State reads here → recomposition on change

2. LAYOUT        ← Measure + place each node
                   State reads here → re-layout on change (no recomposition)

3. DRAWING       ← Draw to canvas (hardware accelerated)
                   State reads here → redraw on change (no recomposition, no re-layout)
```

**Optimization insight:** Move state reads to the **latest possible phase**. A state read in the drawing phase costs much less than one in the composition phase — it redraws but doesn't recompose.

```kotlin
// ❌ Reads in composition phase → recomposition on every scroll pixel
@Composable
fun Header(scrollState: ScrollState) {
    val alpha = 1f - (scrollState.value / 300f).coerceIn(0f, 1f)  // read in composition
    Box(modifier = Modifier.alpha(alpha))  // triggers recomposition every frame
}

// ✅ Reads in drawing phase → no recomposition, just redraw
@Composable
fun Header(scrollState: ScrollState) {
    Box(modifier = Modifier.graphicsLayer {
        alpha = 1f - (scrollState.value / 300f).coerceIn(0f, 1f)  // read in drawing phase
    })
}
```

---

## 3. Smart Recomposition — How Compose Decides What to Skip

Compose tracks which composables read which state objects. When a state changes, only the composables that **directly read** that state need to recompose. But there's a catch: Compose can only skip a composable if it can prove the inputs haven't changed. This requires **stability**.

### 3.1 Skippable vs Restartable Composables

**Restartable:** A composable that can be a *scope* for recomposition — Compose can restart execution from this composable without re-running its parent. Most composables are restartable.

**Skippable:** A restartable composable where Compose will *skip* re-execution if all parameters compare equal to their previous values. Only composables with **all stable parameters** are skippable by default.

```
Parent recomposes
        │
        ▼
ChildA (skippable, stable params, params unchanged) → SKIPPED ✅
ChildB (skippable, stable params, params changed)   → RECOMPOSED
ChildC (not skippable, has unstable params)          → ALWAYS RECOMPOSED ❌
```

**`ChildC` recomposes every time its parent recomposes** — even when its inputs haven't actually changed — because Compose can't verify they haven't changed.

---

## 4. Stability — The Core Concept

### 4.1 Stable Types (Compose can skip)

A type is **stable** if Compose can reliably determine whether it has changed between recompositions:

```kotlin
// Primitives — always stable
Int, Boolean, Float, String, Long, Double, Char

// Kotlin data class — stable IF all properties are stable
data class UserProfile(
    val id: String,         // stable ✅
    val name: String,       // stable ✅
    val followerCount: Int  // stable ✅
) // → UserProfile is STABLE ✅

// Kotlin data class — unstable if any property is unstable
data class FeedState(
    val posts: List<Post>,       // List = interface, may be mutable → UNSTABLE ❌
    val isLoading: Boolean       // stable ✅
) // → FeedState is UNSTABLE ❌ (because of List<Post>)

// MutableState — stable (Compose tracks changes internally)
val count = mutableStateOf(0)  // stable ✅

// External class (from library) — may be unstable if Compose can't inspect it
val dateFormatter: DateTimeFormatter  // from java.time — UNSTABLE ❌
```

### 4.2 Why `List<T>` Is Unstable

`List<T>` in Kotlin is an interface. Its implementation could be `ArrayList` (mutable) or `listOf()` (immutable). Compose sees the declared type (`List`), not the runtime implementation. Since `List` is an interface that could be backed by a mutable collection, Compose conservatively treats it as **unstable**.

```kotlin
// ❌ List<Post> → composable not skippable
@Composable
fun PostList(posts: List<Post>) { /* ... */ }
// Compose can't know if posts changed → always recomposes when parent recomposes

// ✅ ImmutableList<Post> → composable skippable
@Composable
fun PostList(posts: ImmutableList<Post>) { /* ... */ }
// Compose knows immutable → can compare by reference → skippable
```

### 4.3 `@Immutable` and `@Stable` Annotations

These annotations are a **contract** with the compiler — you're promising that the type behaves a certain way. Use them only when you're sure.

```kotlin
// @Immutable — promise: properties will NEVER change after construction
@Immutable
data class Post(
    val id: String,
    val content: String,
    val author: Author,
    val createdAt: Long
)
// Now List<Post> parameters in composables are treated as stable ✅

// @Stable — promise: when properties change, Compose WILL be notified
// (appropriate for classes with mutable state observed via mutableStateOf)
@Stable
class ThemeManager {
    var isDarkMode by mutableStateOf(false)  // ← Compose tracks this
    var accentColor by mutableStateOf(Color.Blue)
}

// ⚠️ Warning: @Immutable on a class with var properties is lying to the compiler
// This can cause bugs where Compose doesn't recompose when it should
@Immutable  // ❌ WRONG — properties ARE mutable
data class MutableConfig(var enabled: Boolean)
```

### 4.4 Stable Collections — kotlinx-collections-immutable

```kotlin
// gradle
implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.8")

// Use in data classes and function parameters
data class FeedUiState(
    val posts: ImmutableList<Post> = persistentListOf(),        // ✅ stable
    val selectedCategories: ImmutableSet<String> = persistentSetOf(),  // ✅ stable
    val filters: ImmutableMap<String, String> = persistentMapOf()     // ✅ stable
)

// Convert regular collections
val posts: ImmutableList<Post> = regularList.toImmutableList()
val immutable: ImmutableList<Post> = persistentListOf(post1, post2)

// Add element (creates a new instance)
val updated = posts.toPersistentList().add(newPost)  // immutable original unchanged
```

---

## 5. Compose Compiler Reports

### 5.1 How to Generate

```kotlin
// app/build.gradle.kts
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    metricsDestination = layout.buildDirectory.dir("compose_metrics")
}

// Run to generate
./gradlew :app:assembleRelease
// Output at: app/build/compose_compiler/
```

### 5.2 Reading the Reports

**`composables.txt` — function-level report:**
```
// Example output:
restartable skippable scheme("[androidx.compose.ui.UiComposable]") fun PostCard(
  stable post: Post
  stable modifier: Modifier? = @static Companion
)
// "skippable" ✅ — all parameters stable, will be skipped if inputs unchanged

restartable scheme("[androidx.compose.ui.UiComposable]") fun FeedScreen(
  unstable state: FeedState       // ← "unstable" ❌ — will NEVER be skipped
)
// No "skippable" keyword → always recomposes when parent recomposes

// This means FeedScreen will recompose on EVERY parent recomposition
// even if state.posts didn't change
```

**`classes.txt` — type-level report:**
```
unstable class FeedState {
  stable val isLoading: Boolean
  unstable val posts: List<Post>   // ← this makes the whole class unstable
  <runtime stability> = Unstable
}

stable class Post {
  stable val id: String
  stable val content: String
  stable val author: Author
  <runtime stability> = Stable
}
```

**`metrics.json` — aggregate numbers:**
```json
{
  "skippableComposables": 47,
  "unskippableComposables": 12,     // ← investigate these
  "restartableComposables": 59,
  "totalComposables": 59
}
```

Target: minimize `unskippableComposables` in performance-critical paths.

---

## 6. Strong Skipping Mode ✅ (Compose Compiler 1.5.4+)

Strong skipping is an opt-in mode that changes the skipping rules: **all restartable composables become skippable**, regardless of whether their parameters are stable or not. Unstable parameters are compared with instance equality (`===`) instead of structural equality.

```kotlin
// Enable in app/build.gradle.kts
composeCompiler {
    featureFlags = setOf(ComposeFeatureFlag.StrongSkipping)
}
```

**What changes:**
```kotlin
// Without Strong Skipping:
// List<Post> is unstable → FeedScreen is NOT skippable
@Composable
fun FeedScreen(posts: List<Post>) { /* ... */ }  // always recomposes

// With Strong Skipping:
// FeedScreen IS skippable — compares posts by instance equality (===)
// Recomposes only if the List REFERENCE changes, not just content
@Composable
fun FeedScreen(posts: List<Post>) { /* ... */ }  // skipped if same reference
```

**Lambda memoization in Strong Skipping:**
Without Strong Skipping, lambdas are re-allocated every recomposition (new lambda reference = different from old = triggers recomposition of receiving composable). Strong Skipping automatically memoizes lambdas with unstable captures.

```kotlin
// Without Strong Skipping:
// This lambda is re-created on every recomposition → PostCard always recomposes
@Composable
fun FeedList(posts: List<Post>, viewModel: FeedViewModel) {
    LazyColumn {
        items(posts) { post ->
            PostCard(
                post = post,
                onClick = { viewModel.onPostClicked(post.id) }  // new lambda = recomposition
            )
        }
    }
}

// With Strong Skipping: lambda is memoized → PostCard skipped when post unchanged ✅
```

**Caveat:** Strong Skipping uses `===` for unstable params. If you mutate an object in place (instead of creating a new one), Compose won't detect the change. This is why proper immutability still matters.

---

## 7. Stability Configuration File

For classes from external libraries that Compose treats as unstable but you know are safe:

```
// stability_config.conf (in app/ directory)
// Add fully qualified class names — Compose treats them as stable
java.time.LocalDate
java.time.LocalDateTime
com.squareup.moshi.JsonAdapter
```

```kotlin
// app/build.gradle.kts
composeCompiler {
    stabilityConfigurationFiles.add(
        rootProject.layout.projectDirectory.file("stability_config.conf")
    )
}
```

**Common candidates:** `java.time.*` classes, third-party data classes from libraries you don't control.

---

## 8. `derivedStateOf` — Correct Usage

Already covered in Section 18, but key insight for Compose optimization:

```kotlin
// ❌ Reads raw Int on every frame → recomposes every scroll pixel
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val showButton = listState.firstVisibleItemIndex > 0  // Int changes every pixel
    AnimatedVisibility(visible = showButton) { /* FAB */ }
    // AnimatedVisibility recomposes every pixel → animation stutter
}

// ✅ derivedStateOf — recomposes only when boolean flips
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
    AnimatedVisibility(visible = showButton) { /* FAB */ }
    // AnimatedVisibility recomposes only when crossing the threshold
}

// ✅ derivedStateOf for expensive computed values
@Composable
fun FilteredList(items: ImmutableList<Item>, query: String) {
    val filtered by remember(items, query) {
        derivedStateOf {
            items.filter { it.name.contains(query, ignoreCase = true) }
        }
    }
    // Recomputes only when items OR query changes
    // Without derivedStateOf: recomputes on any parent recomposition
    LazyColumn { items(filtered) { ItemRow(it) } }
}
```

---

## 9. LazyList Optimization

### 9.1 Stable Keys

```kotlin
// ❌ No key — Compose uses index as key
// When list order changes: all items recompose
LazyColumn {
    items(posts) { post ->
        PostCard(post)
    }
}

// ✅ Stable key — Compose tracks items by identity
LazyColumn {
    items(
        items = posts,
        key = { post -> post.id }  // stable, unique ID
    ) { post ->
        PostCard(post)
    }
}
// Benefits:
// 1. Item animations work correctly (move animation vs disappear+appear)
// 2. Only changed items recompose (not all items)
// 3. Scroll position preserved when items added at top
```

### 9.2 `contentType` — Reuse ViewHolder-style

```kotlin
// For heterogeneous lists (multiple item types)
LazyColumn {
    items(
        items = feedItems,
        key = { it.id },
        contentType = { item ->
            when (item) {
                is FeedItem.Post -> "post"
                is FeedItem.Ad -> "ad"
                is FeedItem.Story -> "story"
            }
        }  // Compose only reuses composition slots of the same type
    ) { item ->
        when (item) {
            is FeedItem.Post -> PostCard(item)
            is FeedItem.Ad -> AdCard(item)
            is FeedItem.Story -> StoryRow(item)
        }
    }
}
```

### 9.3 Prefetch with `LazyListPrefetchStrategy`

```kotlin
// Compose 1.7+ — prefetch items before they scroll into view
LazyColumn(
    state = rememberLazyListState(),
    // Default prefetch: 1 item ahead
    // Custom: prefetch 3 items ahead for smoother scrolling
    prefetchStrategy = LazyListPrefetchStrategy(
        nestedPrefetchItemCount = 3
    )
)
```

### 9.4 Avoid Nested Scrollable Containers

```kotlin
// ❌ Nested scrolling — Compose can't measure inner list efficiently
LazyColumn {
    item {
        LazyRow { /* ... */ }  // Compose doesn't know the inner list's size
    }
}

// ✅ Flatten into a single LazyColumn with different item types
LazyColumn {
    item { Header() }
    items(horizontalItems) { item -> HorizontalItem(item) }  // horizontal row as items
    items(verticalItems) { item -> VerticalItem(item) }
}

// Or: use a fixed-size Row for horizontal content
item {
    Row(modifier = Modifier.horizontalScroll(rememberScrollState())) {
        items.forEach { HorizontalItem(it) }
    }
}
```

### 9.5 `remember` for Expensive Computations in List Items

```kotlin
@Composable
fun PostCard(post: Post) {
    // ❌ Re-formats on every recomposition
    val formattedDate = DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
        .format(post.createdAt)  // expensive

    // ✅ Only recomputes when post.createdAt changes
    val formattedDate = remember(post.createdAt) {
        DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
            .format(post.createdAt)
    }

    // ✅ Stable key means post.id reference survives list updates
    // ✅ remember(post.createdAt) means date only formats when createdAt changes
}
```

---

## 10. `@NonRestartableComposable` — Opt Out of Restartability

Some composables are so lightweight that the overhead of making them restartable (setting up the scope for recomposition to restart from) outweighs the benefit.

```kotlin
// Restartable overhead: Compose must generate code to track state reads
// and set up a restart scope. For very simple composables, this overhead
// is disproportionate.

// @NonRestartableComposable: Compose won't set up a restart scope
// → Can't be a recomposition entry point
// → If inputs change, parent must recompose (not this composable directly)

@NonRestartableComposable
@Composable
fun SimpleText(text: String) {
    Text(text)
}
// Use for: leaf composables with no state reads, pure text/icon wrappers
// Don't use for: composables that read State<T> objects directly
```

**When to use:** After generating Compiler Reports, if you see "restartable" on a composable that never reads state and is called very frequently (e.g., thousands of times in a list), `@NonRestartableComposable` can reduce overhead.

---

## 11. State Reads in the Wrong Place

### 11.1 Modifier.drawBehind vs Background

```kotlin
// For complex drawing that changes frequently
// ❌ Background modifier reads color in composition phase
Box(modifier = Modifier.background(animatedColor))  // recomposes on color change

// ✅ drawBehind reads in drawing phase — no recomposition
Box(modifier = Modifier.drawBehind {
    drawRect(animatedColor)  // read here = drawing phase only
})
```

### 11.2 `Modifier.layout` for Layout Phase Reads

```kotlin
// Read state in layout phase (not composition) for layout-affecting state
@Composable
fun OffsetByScrollBox(scrollState: ScrollState, content: @Composable () -> Unit) {
    Box(modifier = Modifier.layout { measurable, constraints ->
        val placeable = measurable.measure(constraints)
        layout(placeable.width, placeable.height) {
            // Read scroll state here — in layout phase, not composition
            placeable.placeRelative(0, -scrollState.value)
        }
    }) { content() }
}
```

---

## 12. Measuring Recomposition in Practice

### 12.1 Layout Inspector Recomposition Counts

```
Android Studio Flamingo+ → Layout Inspector
While app is running → enable "Show recomposition counts"

Red numbers overlay each composable:
  Small (< 5)   → normal
  Medium (5-20) → investigate
  Large (50+)   → almost certainly a stability issue or missing derivedStateOf

Procedure:
1. Open Layout Inspector
2. Perform the operation you suspect is slow (scroll, animate)
3. Stop and read counts
4. High count on non-animating composable → stability issue
5. High count on parent = high count on all children (even if they're stable)
   → fix the parent's stability first
```

### 12.2 Macrobenchmark for Recomposition

```kotlin
@Test
fun scrollFeedBenchmark() = benchmarkRule.measureRepeated(
    packageName = "com.example.myapp",
    metrics = listOf(FrameTimingMetric()),
    iterations = 5,
    startupMode = StartupMode.WARM
) {
    startActivityAndWait()
    val list = device.findObject(By.res("feedList"))
    repeat(5) { list.fling(Direction.DOWN) }
}
// Compare P95 frame time before/after stability fixes
// P95 > 32ms (2 frames) = visible jank
```

---

## 13. Compose Optimization Decision Tree

```
Is your composable slow?
│
├── High recomposition count in Layout Inspector?
│     │
│     ├── Composable reads a frequently-changing value (scroll offset, animation)?
│     │     → Wrap in derivedStateOf OR move to graphicsLayer
│     │
│     └── Composable not reading frequently-changing state?
│           → Run Compiler Report → look for "unstable" parameters
│           → Fix: @Immutable, ImmutableList, stability config
│           → Enable Strong Skipping Mode
│
├── Jank during scrolling despite stable types?
│     → Check: key in LazyColumn items
│     → Check: contentType for heterogeneous lists
│     → Check: expensive computation in list item → add remember(key)
│     → Check: nested scrollable containers
│
└── Slow startup?
      → Baseline Profile generation
      → Defer heavy init from App.onCreate() (covered in Section 06)
```

---

## 14. Common Misunderstandings & Pitfalls

**❌ `@Immutable` on a class with mutable properties**
`@Immutable` is a promise to the compiler that properties will never change. If you lie (use `var` properties or internally mutable collections), Compose won't recompose when you expect it to — silent bugs that are very hard to track down. Only annotate truly immutable classes.

**❌ `derivedStateOf` without `remember`**
`remember { derivedStateOf { } }` is the correct form. Without `remember`, a new `DerivedState` object is created on every recomposition — no performance benefit. This is one of the most common Compose optimization mistakes.

**❌ Assuming all Kotlin data classes are stable**
A `data class` with a `List<T>` property is NOT stable — Compose sees `List` as potentially mutable. Only data classes where ALL properties are stable (primitives, Strings, other @Immutable types, MutableState) are treated as stable.

**❌ Using `===` mentally when Compose uses `equals()`**
Stable parameters use `equals()` for comparison — if you create new objects but with the same content, `equals()` returns true and the composable is correctly skipped. Don't wrap stable data classes in additional wrappers trying to force reference equality.

**❌ Passing lambdas as parameters expecting them to cause recompositions**
Without Strong Skipping, a lambda passed to a child composable is a new object on every parent recomposition — it triggers recomposition of the child. Either enable Strong Skipping (memoizes lambdas), or use `remember { { } }` to stabilize lambdas manually.

**❌ `key` in `LazyColumn` using object equality on unstable types**
`items(posts, key = { post -> post })` — using the whole `Post` object as a key when `Post` is unstable means Compose creates a new key on every recomposition. Use the stable ID: `key = { post -> post.id }`.

**❌ Reading scroll state in composable body for animations**
`val offset = scrollState.value` in a composable function → every pixel of scroll triggers recomposition. Use `graphicsLayer { translationY = scrollState.value * 0.5f }` to move the read to the drawing phase.

---

## 15. Best Practices

- **Run Compiler Reports before optimization** — measure first, fix what the report identifies as unstable, not what you guess
- **Enable Strong Skipping Mode** — `ComposeFeatureFlag.StrongSkipping` in Compose Compiler 1.5.4+, most apps benefit immediately
- **`ImmutableList` for collections in UiState** — the single highest-impact stability fix for most feed-based apps
- **`@Immutable` only on truly immutable types** — lying to the compiler causes silent recomposition bugs
- **`derivedStateOf` always wrapped in `remember`** — `val x by remember { derivedStateOf { ... } }`
- **Stable `key` in every `LazyColumn`/`LazyRow`** — use unique ID, not the object itself
- **`contentType` for heterogeneous lists** — reduces composition slot churn
- **`graphicsLayer` for scroll/animation state reads** — moves from composition phase to drawing phase
- **Layout Inspector recomposition counts** — first diagnostic step; high counts = stability problem
- **Stability config file for external library types** — `java.time.*`, third-party stable classes
- **`remember(key)` for expensive in-list computations** — date formatting, text layout, image transformations
- **Baseline Profile for Compose startup** — generates AOT compilation hints, 20-30% faster first screen

---

## 16. Interview Q&A

**Q1: What makes a composable skippable and why does it matter?**

> A composable is skippable if the Compose compiler can prove that all its parameters are stable — meaning Compose can reliably determine whether each parameter has changed since the last composition. When Compose needs to recompose a parent, it evaluates each child: if the child is skippable and all its parameters compare equal (via `equals()` for stable types), Compose skips re-executing that child entirely. This matters for performance because in a feed with 50 items, without skipping, one parent recomposition would re-execute all 50 item composables. With stable parameters and stable keys, only the items whose data actually changed are re-executed — potentially 1 out of 50. The Compiler Report reveals which composables are and aren't skippable, which is the starting point for any Compose performance investigation.

---

**Q2: Why does `List<T>` cause unnecessary recomposition in Compose?**

> `List<T>` in Kotlin is an interface. The concrete implementation could be `ArrayList` (mutable) or `listOf()` result (read-only view of an array). The Compose compiler sees the declared type — `List` — which is an interface and could be backed by a mutable collection. Since Compose can't guarantee the contents didn't change between recompositions (someone could have mutated the underlying list), it conservatively marks any composable with a `List<T>` parameter as not skippable. The fix is `ImmutableList<T>` from `kotlinx-collections-immutable` — it's a concrete type that Compose knows is immutable, making composables that receive it skippable. Alternatively, Strong Skipping Mode handles this by comparing list references with `===` rather than structural equality, which works correctly as long as you create new list objects rather than mutating in place.

---

**Q3: What is Strong Skipping Mode and when would you enable it?**

> Strong Skipping Mode (Compose Compiler 1.5.4+) changes the default skipping rule: instead of only allowing composables with all-stable parameters to be skippable, it makes all restartable composables skippable regardless of parameter stability. For unstable parameters, it uses instance equality (`===`) instead of structural equality — a composable is skipped if all parameters are the same reference as before. It also automatically memoizes lambdas, preventing unnecessary recompositions caused by newly-allocated lambda objects on every parent recomposition. I'd enable it for most apps — the performance gains are usually significant, especially for list-heavy UIs. The key requirement: you must not mutate objects in place; always create new instances when data changes. If your code already follows this convention (Kotlin data classes, UiState updates via `copy()`), Strong Skipping just works.

---

**Q4: Walk me through diagnosing a Compose composable that's recomposing too often.**

> Step one: Android Studio Layout Inspector with recomposition counts enabled. Interact with the screen and look for high counts on composables that shouldn't be recomposing (a list item that shows 50 recompositions while just scrolling, when the data hasn't changed). Step two: run the Compose Compiler Report — `./gradlew :app:assembleRelease` with `reportsDestination` configured. Check `composables.txt` for the suspect composable — if it lacks the `skippable` keyword, it's always recomposing. Check `classes.txt` to find which parameters are marked `unstable`. Step three: fix the root cause. If the parameter is a `List<T>`, switch to `ImmutableList<T>`. If it's a third-party type, add it to the stability config. If it's a data class with a mutable property, either make it immutable or annotate carefully with `@Immutable`. Step four: verify with another Compiler Report that the composable is now `skippable`, and confirm in Layout Inspector that recomposition counts normalized.

---

**Q5: How do you make a scroll-driven header animation perform at 60fps without causing recomposition?**

> The key is moving the state read to the drawing phase via `graphicsLayer`. If I write `val alpha = 1f - scrollOffset / 300f` in the composable body and use `Modifier.alpha(alpha)`, the composable recomposes on every scroll pixel — potentially 60+ times per second, all on the main thread. Instead, I pass `scrollState` into `graphicsLayer { }`: `Modifier.graphicsLayer { alpha = 1f - scrollState.value / 300f.coerceIn(0f, 1f) }`. The lambda inside `graphicsLayer` executes during the drawing phase on the RenderThread, never during composition. No recomposition occurs, no state invalidation cascades, no layout pass. The animation runs entirely in hardware at display refresh rate. The same technique applies to parallax (`translationY`), scale effects, and rotation — anything driven by continuous input like scroll or gesture.

---

**Q6: What is `@NonRestartableComposable` and when should you use it?**

> By default, Compose makes most composables "restartable" — it sets up bookkeeping so that recomposition can start from this composable as an entry point, reading its state dependencies. This has a small overhead. `@NonRestartableComposable` opts out: Compose won't set up a restart scope. If this composable's inputs change, the parent recomposes and re-executes this composable as part of that recomposition — there's no dedicated restart entry. Use it for leaf composables that are very simple (no state reads, no complex logic), are called very frequently (thousands of times in a list), and where the restart overhead is measurable. The Compose Compiler Report marks functions as "restartable" — if you see a very simple composable that's restartable and appears in a hot path, `@NonRestartableComposable` is worth testing. Don't apply it to composables that directly read `State<T>` objects — they need to be restartable to respond to state changes independently.

---

*Previous: [22 — Debugging, Profiling & Tooling](./22-debugging.md)*
*Next: [24 — Lifecycle Management](./24-lifecycle.md)*
