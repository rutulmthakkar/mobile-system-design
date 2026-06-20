# 18 — State Management

> **Round type: LLD (primary)**
> **Focus:** Compose state primitives, UDF, state hoisting, side effects, stability, React Native Redux/Zustand/Jotai — everything needed to reason about state in modern mobile apps

---

## 1. Why This Matters in Interviews

State management questions test whether you understand Compose's reactive model deeply — not just the APIs, but *why* things work the way they do. Interviewers probe recomposition behavior, side effect APIs, and React Native library trade-offs to find engineers who can reason about state predictably.

**Common interview angles:**
- "What's the difference between `remember` and `rememberSaveable`?"
- "What is state hoisting and why do we do it?"
- "When would you use `derivedStateOf`?"
- "What's `LaunchedEffect` vs `DisposableEffect` vs `SideEffect`?"
- "How do you pass a callback to a `LaunchedEffect` without restarting it?"
- "What makes a Composable unstable and how do you fix it?"
- "Redux vs Zustand — when would you choose each in React Native?"

---

## 2. State in Jetpack Compose

### 2.1 What State Is

State is any value that changes over time and whose change should update the UI. When Compose detects a state read inside a composable that changed, it **recomposes** that composable — re-runs the function to produce an updated UI.

```kotlin
// mutableStateOf — Compose observable state
var count by remember { mutableStateOf(0) }
// When count changes, every composable that READ count recomposes
// Only those composables — not the whole screen

Text("Count: $count")     // reads count → recomposes on change
Button(onClick = { count++ }) { Text("Increment") }  // doesn't read count → doesn't recompose
```

**How Compose knows what to recompose:** During composition, Compose tracks which state objects each composable reads. This creates a dependency graph. When a state changes, only the composables in that state's dependency graph are invalidated.

### 2.2 `remember` vs `rememberSaveable`

```kotlin
// remember — survives recomposition, NOT configuration changes (rotation, dark mode toggle)
var text by remember { mutableStateOf("") }
// If screen rotates: text resets to ""

// rememberSaveable — survives both recomposition AND configuration changes
// Saved to Bundle (like onSaveInstanceState)
var text by rememberSaveable { mutableStateOf("") }
// If screen rotates: text is restored

// rememberSaveable with custom saver (for non-Bundle types)
var selectedItem by rememberSaveable(stateSaver = MyItemSaver) {
    mutableStateOf(MyItem())
}

// rememberSaveable with key — invalidate saved state when key changes
var page by rememberSaveable(key = userId) { mutableStateOf(0) }
// When userId changes, page resets (new user = start at page 0)
```

**Decision rule:**
- UI interaction state (scroll position, expanded/collapsed, text field) → `rememberSaveable`
- Transient UI state (animation progress, focus, hover) → `remember`
- Business data → ViewModel (`StateFlow`), not `remember` at all

### 2.3 State Hoisting

Move state UP to the lowest common ancestor that needs to read or modify it. Pass state down, events up.

```kotlin
// ❌ Unhoisted — TextField owns its own state
// Cannot be reused, cannot be tested, parent can't read the value
@Composable
fun UnhoistedTextField() {
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}

// ✅ Hoisted — caller owns the state
// Reusable, testable, parent can read and act on value
@Composable
fun HoistedTextField(
    value: String,               // state flows DOWN
    onValueChange: (String) -> Unit,  // events flow UP
    modifier: Modifier = Modifier
) {
    TextField(value = value, onValueChange = onValueChange, modifier = modifier)
}

// Parent controls state
@Composable
fun SearchScreen(viewModel: SearchViewModel = hiltViewModel()) {
    val query by viewModel.searchQuery.collectAsStateWithLifecycle()

    HoistedTextField(
        value = query,
        onValueChange = { viewModel.onQueryChanged(it) }
    )
}
```

**Hoist to the right level:**
- State used by ONE composable → keep it local with `remember`
- State shared by a FEW siblings → hoist to nearest common parent
- State shared across screens → hoist to ViewModel

---

## 3. Unidirectional Data Flow (UDF)

UDF is the architectural pattern where:
- **State flows DOWN** from ViewModel to Composable
- **Events flow UP** from Composable to ViewModel
- The Composable is a pure function of its state — given the same state, it always renders the same UI

```
ViewModel                           Composable
────────────────────────────────────────────────
StateFlow<UiState>  ──── (state) ──▶ renders UI
                    ◀──── (event) ─── user taps button
                                        calls viewModel.onButtonTap()
                    updates state ──▶ Compose recomposes
```

```kotlin
// ViewModel — owns all state
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val userRepo: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProfileUiState())
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    fun onNameChanged(name: String) {
        _uiState.update { it.copy(name = name, isDirty = true) }
    }

    fun onSaveTapped() {
        viewModelScope.launch {
            _uiState.update { it.copy(isSaving = true) }
            try {
                userRepo.updateProfile(_uiState.value.toProfileUpdate())
                _uiState.update { it.copy(isSaving = false, isDirty = false) }
                _effects.send(ProfileEffect.ShowSaveSuccess)
            } catch (e: Exception) {
                _uiState.update { it.copy(isSaving = false) }
                _effects.send(ProfileEffect.ShowError(e.message ?: "Save failed"))
            }
        }
    }
}

data class ProfileUiState(
    val name: String = "",
    val avatarUrl: String = "",
    val isSaving: Boolean = false,
    val isDirty: Boolean = false,
    val error: String? = null
)

sealed class ProfileEffect {
    object ShowSaveSuccess : ProfileEffect()
    data class ShowError(val message: String) : ProfileEffect()
}

// Composable — pure function of state
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    // Handle one-time effects
    val snackbarHostState = remember { SnackbarHostState() }
    LaunchedEffect(Unit) {
        viewModel.effects.collect { effect ->
            when (effect) {
                is ProfileEffect.ShowSaveSuccess ->
                    snackbarHostState.showSnackbar("Profile saved!")
                is ProfileEffect.ShowError ->
                    snackbarHostState.showSnackbar(effect.message)
            }
        }
    }

    ProfileContent(
        name = state.name,
        isSaving = state.isSaving,
        isDirty = state.isDirty,
        onNameChanged = viewModel::onNameChanged,
        onSaveTapped = viewModel::onSaveTapped
    )
}
```

---

## 4. Compose Side Effect APIs

Side effects are operations that affect state outside the composable's scope — launching coroutines, showing snackbars, registering listeners. They must be managed carefully because composables can recompose frequently and unexpectedly.

### 4.1 `LaunchedEffect` — Launch a coroutine in composition

```kotlin
// Runs when the composable enters composition
// Re-runs when KEY changes
// Cancelled when composable leaves composition

@Composable
fun SearchResults(query: String) {
    val results = remember { mutableStateOf<List<Item>>(emptyList()) }

    LaunchedEffect(query) {  // ← KEY = query
        // Cancelled and restarted every time query changes
        results.value = searchRepository.search(query)
    }

    LazyColumn {
        items(results.value) { item -> ItemRow(item) }
    }
}

// LaunchedEffect(Unit) — runs ONCE when composable first appears
LaunchedEffect(Unit) {
    viewModel.loadInitialData()
}

// LaunchedEffect(key1, key2) — runs when EITHER key changes
LaunchedEffect(userId, screenMode) {
    viewModel.loadUserData(userId, screenMode)
}
```

**Common mistake:** Putting a lambda as a key — lambdas are recreated on every recomposition, so the effect would restart on every recomposition. Use stable values as keys.

### 4.2 `rememberUpdatedState` — Stable reference to latest value

```kotlin
// Problem: LaunchedEffect captures the value at launch time (stale closure)
@Composable
fun Timer(onTimeout: () -> Unit) {
    // ❌ BAD: if onTimeout changes, this still calls the OLD callback
    LaunchedEffect(Unit) {
        delay(5000)
        onTimeout()  // might be stale!
    }
}

// ✅ Solution: rememberUpdatedState — always refers to latest value
@Composable
fun Timer(onTimeout: () -> Unit) {
    val currentOnTimeout by rememberUpdatedState(onTimeout)

    LaunchedEffect(Unit) {
        delay(5000)
        currentOnTimeout()  // always calls the CURRENT callback
    }
}
```

**When to use:** Any long-running `LaunchedEffect` (delay, infinite loop, listener) that references a lambda or value that might change. The `LaunchedEffect` key stays stable, but the referenced value updates.

### 4.3 `DisposableEffect` — Setup + Cleanup

```kotlin
// Runs when composable enters composition
// onDispose runs when composable leaves composition OR key changes
// Use for: registering/unregistering listeners, acquiring/releasing resources

@Composable
fun LocationTracker(onLocationChanged: (LatLng) -> Unit) {
    val currentCallback by rememberUpdatedState(onLocationChanged)

    DisposableEffect(Unit) {
        val listener = LocationListener { location ->
            currentCallback(LatLng(location.latitude, location.longitude))
        }
        locationManager.requestUpdates(listener)

        onDispose {
            locationManager.removeUpdates(listener)  // ← cleanup guaranteed
        }
    }
}

// DisposableEffect with key — cleanup + re-setup when key changes
@Composable
fun UserPresenceTracker(userId: String) {
    DisposableEffect(userId) {
        presenceManager.startTracking(userId)
        onDispose {
            presenceManager.stopTracking(userId)  // called when userId changes or composable leaves
        }
    }
}
```

### 4.4 `SideEffect` — Sync Compose state to non-Compose systems

```kotlin
// Runs after EVERY successful recomposition
// Does NOT run if recomposition is cancelled
// Use for: syncing analytics, updating non-Compose objects with Compose state

@Composable
fun ScreenTracker(screenName: String) {
    SideEffect {
        // Runs after every recomposition — update analytics with current screen
        analytics.setCurrentScreen(screenName)
    }
}

// Update a Firebase Crashlytics key on every state change
@Composable
fun CrashlyticsStateTracker(userId: String, screen: String) {
    SideEffect {
        FirebaseCrashlytics.getInstance().setCustomKey("screen", screen)
        FirebaseCrashlytics.getInstance().setUserId(userId)
    }
}
```

### 4.5 `produceState` — Convert non-Compose state to Compose state

```kotlin
// Converts suspend functions, flows, or callbacks into Compose State
@Composable
fun NetworkImage(url: String): State<Bitmap?> = produceState<Bitmap?>(
    initialValue = null,
    key1 = url
) {
    // this is a CoroutineScope — use suspend functions directly
    value = imageLoader.loadBitmap(url)

    awaitDispose {
        // cleanup if needed
    }
}

// Usage
@Composable
fun Avatar(url: String) {
    val bitmap by NetworkImage(url)
    if (bitmap != null) {
        Image(bitmap = bitmap!!.asImageBitmap(), contentDescription = null)
    } else {
        CircularProgressIndicator()
    }
}
```

### 4.6 `snapshotFlow` — Convert Compose state to Flow

```kotlin
// Observes Compose state variables and emits when they change
// Use for: debouncing text input, reacting to scroll position changes

@Composable
fun SearchBar(viewModel: SearchViewModel) {
    var query by remember { mutableStateOf("") }

    // Convert Compose state to Flow for debouncing
    LaunchedEffect(Unit) {
        snapshotFlow { query }
            .debounce(300)
            .distinctUntilChanged()
            .collect { debouncedQuery ->
                viewModel.search(debouncedQuery)
            }
    }

    TextField(value = query, onValueChange = { query = it })
}

// snapshotFlow for scroll position
@Composable
fun ScrollAwareFab(listState: LazyListState) {
    val showFab by remember {
        derivedStateOf { listState.firstVisibleItemIndex == 0 }
    }
    AnimatedVisibility(visible = showFab) { FloatingActionButton(onClick = {}) { } }
}
```

### 4.7 `derivedStateOf` — Computed state that recomposes less often

```kotlin
// Without derivedStateOf — recomposes every time firstVisibleItemIndex changes
@Composable
fun BadScrollToTop(listState: LazyListState) {
    val showButton = listState.firstVisibleItemIndex > 0  // reads Int → recomposes on EVERY scroll pixel
    AnimatedVisibility(visible = showButton) { /* ... */ }
}

// With derivedStateOf — recomposes only when boolean FLIPS (0→1 or 1→0)
@Composable
fun GoodScrollToTop(listState: LazyListState) {
    val showButton by remember {
        derivedStateOf { listState.firstVisibleItemIndex > 0 }
    }
    AnimatedVisibility(visible = showButton) { /* ... */ }
}

// Another example: filter large list
@Composable
fun FilteredList(items: List<Item>, filter: String) {
    val filtered by remember(items) {
        derivedStateOf { items.filter { it.name.contains(filter, ignoreCase = true) } }
    }
    // Recomputes only when items OR filter changes, not on every recomposition
    LazyColumn { items(filtered) { ItemRow(it) } }
}
```

**Rule:** Use `derivedStateOf` when state is computed from another state object that changes more frequently than the derived result would.

---

## 5. Compose Stability

### 5.1 What Stability Means

Compose's smart recomposition relies on knowing whether inputs to a composable have changed. If Compose can't be sure (unstable types), it recomposes defensively — even when nothing changed.

**Stable type:** Compose can trust that if equals() returns true, the value hasn't changed. Recomposition is skipped when all inputs are stable and unchanged.

**Unstable type:** Compose can't trust equals() → always recomposes when the parent recomposes.

### 5.2 What Makes a Type Stable

```kotlin
// Stable by default:
// - Primitives: Int, Boolean, Float, String
// - Kotlin data classes where all properties are stable
// - @Stable or @Immutable annotated classes
// - Lambdas (in certain conditions)

// ❌ UNSTABLE — List is mutable; Compose can't trust its contents didn't change
data class FeedState(
    val items: List<Post>,     // MutableList disguised as List → UNSTABLE
    val isLoading: Boolean
)

// ✅ STABLE — ImmutableList (from kotlinx-collections-immutable)
data class FeedState(
    val items: ImmutableList<Post>,  // provably immutable → STABLE
    val isLoading: Boolean
)

// ✅ STABLE — annotate if you know it's stable but Compose can't infer it
@Immutable  // promise: this object will never mutate after creation
data class Post(
    val id: String,
    val content: String,
    val author: Author
)

// @Stable — properties MAY change but equals() is reliable
@Stable
class ThemeManager {
    var isDarkMode by mutableStateOf(false)  // Compose tracks this
}
```

### 5.3 Detecting Instability

```kotlin
// Use Compose Compiler Reports to identify unstable classes
// In app/build.gradle.kts:
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
}
// Run: ./gradlew assembleRelease
// Check: build/compose_compiler/app_release-composables.txt
// Look for "unstable" warnings on your composables

// Or use the Layout Inspector → Recomposition Counts tab
// High recomposition counts on a composable = likely stability issue
```

### 5.4 Kotlinx Immutable Collections

```kotlin
// gradle
implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.8")

// In your data classes
data class FeedUiState(
    val posts: ImmutableList<Post> = persistentListOf(),
    val selectedTags: ImmutableSet<String> = persistentSetOf()
)

// Creating
val state = FeedUiState(
    posts = persistentListOf(post1, post2),
    selectedTags = persistentSetOf("kotlin", "android")
)

// Updating (creates new instance)
val updated = state.copy(
    posts = state.posts.toPersistentList().add(newPost)
)
```

---

## 6. State Holders Beyond ViewModel

### 6.1 Plain State Holder Class

For complex UI state logic that doesn't belong in ViewModel (UI-only concerns):

```kotlin
// State holder — manages UI-only state logic
class DrawerState(
    initialValue: DrawerValue = DrawerValue.Closed
) {
    var currentValue by mutableStateOf(initialValue)
        private set

    val isOpen: Boolean get() = currentValue == DrawerValue.Open
    val isClosed: Boolean get() = currentValue == DrawerValue.Closed

    suspend fun open() { currentValue = DrawerValue.Open }
    suspend fun close() { currentValue = DrawerValue.Closed }
    suspend fun toggle() = if (isOpen) close() else open()
}

// remember it in composition
@Composable
fun rememberDrawerState(initialValue: DrawerValue = DrawerValue.Closed) =
    remember { DrawerState(initialValue) }

// Use in composable
@Composable
fun AppDrawer() {
    val drawerState = rememberDrawerState()
    // drawerState survives recomposition; ViewModel-free for UI-only state
}
```

---

## 7. React Native State Management

### 7.1 Local State — `useState` + `useReducer`

```typescript
// useState — for simple local state
const [count, setCount] = useState(0)
const [user, setUser] = useState<User | null>(null)

// useReducer — for complex local state with multiple sub-values
type State = { isLoading: boolean; data: Post[]; error: string | null }
type Action =
    | { type: 'LOADING' }
    | { type: 'SUCCESS'; payload: Post[] }
    | { type: 'ERROR'; payload: string }

function reducer(state: State, action: Action): State {
    switch (action.type) {
        case 'LOADING': return { ...state, isLoading: true, error: null }
        case 'SUCCESS': return { isLoading: false, data: action.payload, error: null }
        case 'ERROR':   return { isLoading: false, data: [], error: action.payload }
    }
}

const [state, dispatch] = useReducer(reducer, { isLoading: false, data: [], error: null })
```

### 7.2 Context API — Lightweight shared state

```typescript
// For simple cross-component state (theme, auth, locale)
// NOT for frequently-changing state (performance issues)

const ThemeContext = createContext<ThemeContextType>({
    isDark: false,
    toggleTheme: () => {}
})

function ThemeProvider({ children }: { children: ReactNode }) {
    const [isDark, setIsDark] = useState(false)
    return (
        <ThemeContext.Provider value={{ isDark, toggleTheme: () => setIsDark(d => !d) }}>
            {children}
        </ThemeContext.Provider>
    )
}

// Use anywhere in the tree
const { isDark, toggleTheme } = useContext(ThemeContext)
```

**Context pitfall:** When Context value changes, ALL consumers re-render — even those that only use a different slice of the context. Split into multiple contexts for different concerns.

### 7.3 Zustand ✅ Recommended for most RN apps

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

// Simple store
interface FeedStore {
    posts: Post[]
    isLoading: boolean
    fetchPosts: () => Promise<void>
    likePost: (id: string) => void
}

const useFeedStore = create<FeedStore>((set, get) => ({
    posts: [],
    isLoading: false,

    fetchPosts: async () => {
        set({ isLoading: true })
        try {
            const posts = await api.getPosts()
            set({ posts, isLoading: false })
        } catch (e) {
            set({ isLoading: false })
        }
    },

    likePost: (id: string) => {
        set(state => ({
            posts: state.posts.map(p =>
                p.id === id ? { ...p, isLiked: !p.isLiked } : p
            )
        }))
        // Also sync to server (optimistic update)
        api.likePost(id).catch(() => {
            // rollback on failure
            set(state => ({
                posts: state.posts.map(p =>
                    p.id === id ? { ...p, isLiked: !p.isLiked } : p
                )
            }))
        })
    }
}))

// With persistence (MMKV)
const useUserStore = create<UserStore>()(
    persist(
        (set) => ({
            user: null,
            setUser: (user) => set({ user }),
            logout: () => set({ user: null })
        }),
        {
            name: 'user-store',
            storage: createJSONStorage(() => mmkvStorage)
        }
    )
)

// Usage — select only what you need (prevents unnecessary re-renders)
const posts = useFeedStore(state => state.posts)        // re-renders when posts change
const isLoading = useFeedStore(state => state.isLoading)  // re-renders when isLoading changes
const { fetchPosts, likePost } = useFeedStore()          // stable references, no re-render
```

### 7.4 Redux Toolkit — For large teams with complex state

```typescript
import { createSlice, createAsyncThunk, configureStore } from '@reduxjs/toolkit'

// Async thunk
export const fetchPosts = createAsyncThunk('feed/fetchPosts', async (userId: string) => {
    const response = await api.getPosts(userId)
    return response.data
})

// Slice
const feedSlice = createSlice({
    name: 'feed',
    initialState: { posts: [], isLoading: false, error: null } as FeedState,
    reducers: {
        likePost: (state, action: PayloadAction<string>) => {
            const post = state.posts.find(p => p.id === action.payload)
            if (post) post.isLiked = !post.isLiked  // Immer enables direct mutation
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchPosts.pending, (state) => { state.isLoading = true })
            .addCase(fetchPosts.fulfilled, (state, action) => {
                state.isLoading = false
                state.posts = action.payload
            })
            .addCase(fetchPosts.rejected, (state, action) => {
                state.isLoading = false
                state.error = action.error.message ?? null
            })
    }
})

const store = configureStore({ reducer: { feed: feedSlice.reducer } })

// In component
const posts = useSelector((state: RootState) => state.feed.posts)
const dispatch = useDispatch()
dispatch(fetchPosts(userId))
dispatch(feedSlice.actions.likePost(postId))
```

### 7.5 Jotai — Atomic state

```typescript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai'

// Atoms — minimal units of state
const userAtom = atom<User | null>(null)
const postsAtom = atom<Post[]>([])

// Derived atoms (read-only computed state)
const likedPostsAtom = atom((get) => get(postsAtom).filter(p => p.isLiked))

// Async atoms
const fetchPostsAtom = atom(async (get) => {
    const user = get(userAtom)
    if (!user) return []
    return api.getPosts(user.id)
})

// Usage
const [user, setUser] = useAtom(userAtom)            // read + write
const posts = useAtomValue(postsAtom)                 // read only
const setUser = useSetAtom(userAtom)                  // write only (no re-render on read)
const likedPosts = useAtomValue(likedPostsAtom)        // derived

// Only the components using a specific atom re-render when that atom changes
// No Provider needed (unlike Context) — atoms are global by default
```

### 7.6 TanStack Query — Server state management

```typescript
// For data from APIs — handles loading, caching, refetching, pagination
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// Fetch and cache
const { data: posts, isLoading, error } = useQuery({
    queryKey: ['posts', userId],
    queryFn: () => api.getPosts(userId),
    staleTime: 5 * 60 * 1000,   // consider fresh for 5 min
    gcTime: 30 * 60 * 1000,     // keep in cache 30 min after unmount
})

// Mutation with optimistic update
const queryClient = useQueryClient()
const likeMutation = useMutation({
    mutationFn: (postId: string) => api.likePost(postId),
    onMutate: async (postId) => {
        await queryClient.cancelQueries({ queryKey: ['posts', userId] })
        const previous = queryClient.getQueryData(['posts', userId])
        queryClient.setQueryData(['posts', userId], (old: Post[]) =>
            old.map(p => p.id === postId ? { ...p, isLiked: !p.isLiked } : p)
        )
        return { previous }
    },
    onError: (_, __, context) => {
        queryClient.setQueryData(['posts', userId], context?.previous)
    },
    onSettled: () => queryClient.invalidateQueries({ queryKey: ['posts', userId] })
})
```

### 7.7 RN State Management Comparison

| | useState/useReducer | Context | Zustand | Redux Toolkit | Jotai | TanStack Query |
|---|---|---|---|---|---|---|
| Learning curve | None | Low | Low | Medium | Low | Low |
| Boilerplate | None | Low | Minimal | Medium | Minimal | Low |
| DevTools | None | Limited | ✅ | ✅ Great | ✅ | ✅ |
| Perf (fine-grained) | Per-component | ❌ Context re-renders all | ✅ Selectors | ✅ Selectors | ✅ Atom-level | ✅ |
| Server state | Manual | Manual | Manual | Manual | Via async atoms | ✅ Built-in |
| Persistence | Manual | Manual | Via middleware | Via redux-persist | Via atomWithStorage | ❌ |
| Best for | Local UI state | Simple shared config | Most apps | Large teams, complex | Atomic/graph state | API data |

**Decision guide:**
- **Local UI state** → `useState` / `useReducer`
- **Simple config (theme, auth)** → Context
- **App state, most RN projects** → Zustand (minimal boilerplate, great DX)
- **Large team, need strict patterns** → Redux Toolkit (enforced conventions, excellent DevTools)
- **Complex interdependencies** → Jotai (atomic, fine-grained reactivity)
- **Server/API data** → TanStack Query (combine with any of the above for local state)

---

## 8. Common Misunderstandings & Pitfalls

**❌ Using `remember` for data that should survive rotation**
`remember` only survives recomposition. Use `rememberSaveable` for any user-visible state (form inputs, selected items, scroll position) that should survive configuration changes.

**❌ `LaunchedEffect(Unit)` when a key should trigger restart**
`LaunchedEffect(Unit)` runs exactly once. If the effect depends on a changing value (user ID, search query), use that value as the key so it restarts when the value changes. Using `Unit` with a changing dependency inside is a stale closure bug.

**❌ Lambdas directly as `LaunchedEffect` keys**
Lambdas are recreated on every recomposition — using `onClick` as a key means the effect restarts on every recomposition. Use stable values (IDs, enums, primitives) as keys. For the lambda itself, use `rememberUpdatedState`.

**❌ `derivedStateOf` without `remember`**
`derivedStateOf { }` without `remember { }` creates a new derived state object on every recomposition — defeating its purpose. Always: `val x by remember { derivedStateOf { ... } }`.

**❌ Putting mutable collections in `UiState`**
`List<T>` in Kotlin is actually often backed by `ArrayList` (mutable). Compose can't determine if it changed, so it treats it as unstable and recomposes unnecessarily. Use `ImmutableList<T>` from kotlinx-collections-immutable or make the list a distinct type.

**❌ Reading state in the wrong Compose phase**
Reading `lazyListState.firstVisibleItemIndex` in a composable body triggers recomposition on every scroll event. Read it inside `derivedStateOf`, `snapshotFlow`, or inside lambda callbacks to defer the read to the correct phase.

**❌ Zustand: selecting the entire store**
`const state = useStore()` — the component re-renders whenever ANY piece of store state changes. Always select minimal slices: `const posts = useStore(s => s.posts)`.

**❌ Context for frequently changing state**
Every Context value change re-renders ALL consumers. Using Context for a counter that updates every second means every component in the tree re-renders every second. Use Zustand or Jotai for frequently changing global state.

**❌ `SideEffect` for coroutine work**
`SideEffect` is synchronous — it cannot call `suspend` functions. Use `LaunchedEffect` for async work. Use `SideEffect` only for syncing Compose state to external imperative systems.

---

## 9. Best Practices

**Compose:**
- **`rememberSaveable` for user-visible UI state** — form inputs, selections, scroll position
- **State hoisting** — move state to the lowest common ancestor that needs it
- **Single `UiState` data class per screen** — one StateFlow, not five separate ones
- **`collectAsStateWithLifecycle`** — not `collectAsState` — respects lifecycle, saves battery
- **`derivedStateOf` + `remember`** — always both, never one without the other
- **`ImmutableList` for collections in UiState** — prevents unnecessary recomposition
- **`snapshotFlow` for debouncing Compose state** — bridges Compose → coroutines
- **`rememberUpdatedState` for lambdas in long-lived effects** — prevents stale closure
- **`DisposableEffect` for any listener registration** — `onDispose` guarantees cleanup
- **Compose compiler reports** — run to detect unstable classes before they cause perf issues

**React Native:**
- **Zustand for most apps** — minimal boilerplate, good performance, great DX
- **Select minimal slices from Zustand** — `useStore(s => s.specificField)` not `useStore()`
- **TanStack Query for server state** — don't reinvent caching and refetching
- **Avoid Context for frequently-changing state** — too broad re-render scope
- **Redux Toolkit only for large teams** — its conventions are valuable at scale, overkill for smaller apps
- **`useMemo`/`useCallback`** — memoize expensive computations and stable callbacks

---

## 10. Interview Q&A

**Q1: What's the difference between `remember` and `rememberSaveable`?**

> `remember` stores a value in the composition — it survives recomposition (when Compose re-runs the function due to state changes) but is lost on configuration changes like screen rotation or system dark mode toggle. `rememberSaveable` additionally saves the value to a Bundle (like `onSaveInstanceState`) so it survives configuration changes. I use `rememberSaveable` for any user-visible state that would be annoying to lose — a partially typed form field, a selected item, expanded/collapsed sections. I use `remember` for transient UI state that naturally resets — animation progress, focus state, ephemeral computed values. For anything data-related that should truly persist, I use the ViewModel and neither of these.

---

**Q2: When would you use `derivedStateOf`?**

> `derivedStateOf` is for computing state that changes less frequently than its inputs. The canonical example: `listState.firstVisibleItemIndex` changes as an integer on every scroll pixel. If I have a button that shows when the list has scrolled past the first item, computing `isScrolled = firstVisibleItemIndex > 0` without `derivedStateOf` recomposes the button composable on every pixel of scroll. With `remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }`, Compose only recomposes the button when the boolean flips — from false to true or vice versa. That's potentially hundreds fewer recompositions per second during a scroll gesture. The key signal for when to use it: the derived value changes less frequently than the source state.

---

**Q3: What's the difference between `LaunchedEffect`, `DisposableEffect`, and `SideEffect`?**

> `LaunchedEffect` launches a coroutine when the composable enters composition. It cancels and relaunches when the key changes. Use it for async operations: data fetching, observing flows, delays. `DisposableEffect` is for side effects that need cleanup — registering a listener, acquiring a resource. The `onDispose` block runs when the composable leaves composition or the key changes, guaranteeing the resource is released. `SideEffect` runs synchronously after every successful recomposition. No coroutine, no cleanup. Use it to sync Compose state to imperative external systems — updating an analytics key, syncing state to a non-Compose SDK. The key distinction: `LaunchedEffect` for async, `DisposableEffect` for acquire/release pairs, `SideEffect` for synchronous sync.

---

**Q4: A callback passed to `LaunchedEffect` is stale — what's happening and how do you fix it?**

> This is the stale closure problem. `LaunchedEffect(Unit)` captures the value of `onTimeout` at the time it launches — that snapshot is frozen inside the coroutine. If `onTimeout` is updated by the parent (e.g., parent recomposes with a different lambda), the `LaunchedEffect` is still holding the old one. The fix is `rememberUpdatedState`: `val currentOnTimeout by rememberUpdatedState(onTimeout)`. This creates a Compose state object whose value is always updated to the latest `onTimeout` on every recomposition. Inside the `LaunchedEffect`, `currentOnTimeout()` reads through this state object to get the current value — no stale closure.

---

**Q5: What makes a Composable unstable and how do you fix it?**

> A composable is unstable when Compose can't determine whether its parameters have changed without re-running the function — it recomposes defensively even when nothing changed. Common causes: parameters that are `List<T>` (Kotlin's List is not guaranteed immutable — Compose can't trust `equals()`), classes with mutable `var` properties, lambdas created inline in the call site. The fix depends on the cause: replace `List<T>` with `ImmutableList<T>` from `kotlinx-collections-immutable`; annotate stable data classes with `@Immutable` or `@Stable` to promise Compose the type is well-behaved; use `remember { }` to stabilize lambdas at the call site. I use the Compose Compiler Reports to identify unstable types — add `reportsDestination` to the Compose compiler config and look for "unstable" annotations in the generated report.

---

**Q6: Zustand vs Redux Toolkit — when would you choose each in a React Native app?**

> Zustand for most apps — it's minimal boilerplate, easy to reason about, performs well with selective subscriptions, and doesn't require wrapping the app in providers or boilerplate action/reducer files. A complete store is 20 lines. Redux Toolkit for large teams or complex requirements: its strict conventions (slices, reducers, selectors) enforce consistency when 10+ engineers are touching the same codebase. Redux DevTools are excellent for debugging time-travel and action history. Redux also has the most mature ecosystem (redux-persist, redux-saga, redux-observable) for complex async flows. The signal to migrate from Zustand to Redux Toolkit is when team coordination and predictability of state transitions outweigh the cost of its boilerplate. For server state specifically (API data), TanStack Query is the answer regardless of which you choose for local state.

---

*Previous: [17 — Audio Input Stream Manipulation](./17-audio-input.md)*
*Next: [19 — Animation](./19-animation.md)*
