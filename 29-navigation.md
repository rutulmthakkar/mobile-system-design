# 29 — Navigation Architecture

> **Round type: Both (HLD + LLD)**
> **HLD:** Navigation strategy for large multi-module apps — graph structure, tab independence, deep linking
> **LLD:** Jetpack Navigation type-safe routes, back stack control, nested graphs, bottom nav, modals, ViewModel scoping

---

## 1. Why This Matters

Navigation is where architecture meets UX. Interviewers probe: how does your app handle the back stack when a user gets a notification deep link? How do tabs preserve their navigation state? How does the checkout flow share a ViewModel across 3 screens? Getting this wrong causes the most visible bugs — wrong screen on back press, state lost on tab switch, crash on deep link.

**Common interview angles:**
- "How does Jetpack Navigation manage the back stack?"
- "How do you preserve tab state when the user switches bottom nav items?"
- "Design the navigation graph for an app with auth + onboarding + main flow"
- "How do you share a ViewModel between screens in a checkout flow?"
- "How does a notification deep link restore the full back stack?"
- "What's the difference between `popUpTo` and `popBackStack`?"

---

## 2. Core Concepts

### 2.1 Components

```
NavController   → manages back stack, programmatic navigation
NavHost         → container that displays current destination
NavGraph        → directed graph of destinations + routes
NavBackStackEntry → holds each screen's lifecycle + ViewModel store

Back stack = stack of NavBackStackEntry objects
  [Home → ProductList → ProductDetail]
  popBackStack() removes the top entry → goes to ProductList
```

### 2.2 Type-Safe Routes (Navigation 2.8+ / Compose 1.7+)

```kotlin
// gradle
implementation("androidx.navigation:navigation-compose:2.8.5")

// ❌ Old: string routes — typos crash at runtime
navController.navigate("product/123")
composable("product/{id}") { }

// ✅ New: type-safe serializable objects — compiler catches errors
@Serializable data object Home
@Serializable data object ProductList
@Serializable data class ProductDetail(val productId: String)
@Serializable data class OrderConfirmation(val orderId: String, val total: Double)

// NavHost
NavHost(navController, startDestination = Home) {
    composable<Home> { HomeScreen() }
    composable<ProductList> { ProductListScreen() }
    composable<ProductDetail> { backStackEntry ->
        val route: ProductDetail = backStackEntry.toRoute()
        ProductDetailScreen(productId = route.productId)
    }
    composable<OrderConfirmation> { backStackEntry ->
        val route: OrderConfirmation = backStackEntry.toRoute()
        OrderConfirmationScreen(orderId = route.orderId, total = route.total)
    }
}

// Navigate
navController.navigate(ProductDetail(productId = "abc123"))

// Retrieve in ViewModel via SavedStateHandle (args injected automatically)
@HiltViewModel
class ProductDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val route = ProductDetail.fromSavedStateHandle(savedStateHandle)
    val productId = route.productId
}
```

---

## 3. Back Stack Control

```kotlin
// Simple navigate — adds to back stack
navController.navigate(ProductDetail("123"))
// Stack: [Home, ProductDetail]

// Navigate and clear to destination (e.g., after login, go to Home and clear auth screens)
navController.navigate(Home) {
    popUpTo<Login> { inclusive = true }  // remove Login and everything above it
    launchSingleTop = true               // don't create new Home if already at top
}

// Replace current screen (e.g., splash → home, no back to splash)
navController.navigate(Home) {
    popUpTo(navController.graph.id) { inclusive = true }
}

// Go back
navController.popBackStack()

// Go back to specific destination
navController.popBackStack<ProductList>(inclusive = false)

// Go back with result (pass data back to previous screen)
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set("selected_item_id", itemId)
navController.popBackStack()

// Receive result in calling screen
val result = navController.currentBackStackEntry
    ?.savedStateHandle
    ?.getStateFlow<String?>("selected_item_id", null)
    ?.collectAsStateWithLifecycle()
```

---

## 4. Nested Navigation Graphs

Group related screens into sub-graphs for modularity and ViewModel scoping.

```kotlin
// Route sealed classes per feature
@Serializable sealed class AuthGraph {
    @Serializable data object Graph  // the graph's own route
    @Serializable data object Login
    @Serializable data object Register
    @Serializable data object ForgotPassword
}

@Serializable sealed class CheckoutGraph {
    @Serializable data object Graph
    @Serializable data object Address
    @Serializable data object Payment
    @Serializable data object Confirmation
}

// Full NavHost
NavHost(navController, startDestination = AuthGraph.Graph) {

    // Auth nested graph
    navigation<AuthGraph.Graph>(startDestination = AuthGraph.Login) {
        composable<AuthGraph.Login> { LoginScreen() }
        composable<AuthGraph.Register> { RegisterScreen() }
        composable<AuthGraph.ForgotPassword> { ForgotPasswordScreen() }
    }

    // Main graph
    composable<Home> { HomeScreen() }
    composable<ProductList> { ProductListScreen() }
    composable<ProductDetail> { ProductDetailScreen() }

    // Checkout nested graph — shared ViewModel across all 3 screens
    navigation<CheckoutGraph.Graph>(startDestination = CheckoutGraph.Address) {
        composable<CheckoutGraph.Address> { AddressScreen() }
        composable<CheckoutGraph.Payment> { PaymentScreen() }
        composable<CheckoutGraph.Confirmation> { ConfirmationScreen() }
    }
}

// Shared ViewModel scoped to CheckoutGraph
@Composable
fun AddressScreen() {
    val navController = LocalNavController.current
    val checkoutEntry = remember(navController) {
        navController.getBackStackEntry(CheckoutGraph.Graph)
    }
    val viewModel: CheckoutViewModel = hiltViewModel(checkoutEntry)
    // All 3 checkout screens share the same CheckoutViewModel
    // ViewModel cleared when entire checkout graph is popped
}
```

---

## 5. Bottom Navigation with Independent Back Stacks

Each tab has its own back stack. Switching tabs preserves each tab's navigation state.

```kotlin
// Tab definitions
@Serializable data object HomeTab
@Serializable data object SearchTab
@Serializable data object ProfileTab

val tabs = listOf(
    TabItem(HomeTab, "Home", Icons.Default.Home),
    TabItem(SearchTab, "Search", Icons.Default.Search),
    TabItem(ProfileTab, "Profile", Icons.Default.Person)
)

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val currentEntry by navController.currentBackStackEntryAsState()
    val currentDestination = currentEntry?.destination

    Scaffold(
        bottomBar = {
            NavigationBar {
                tabs.forEach { tab ->
                    NavigationBarItem(
                        selected = currentDestination?.hierarchy?.any {
                            it.hasRoute(tab.route::class)
                        } == true,
                        onClick = {
                            navController.navigate(tab.route) {
                                // Pop back to start destination, avoid large back stacks
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true   // ← save tab's back stack state
                                }
                                launchSingleTop = true
                                restoreState = true    // ← restore saved state on tab switch
                            }
                        },
                        icon = { Icon(tab.icon, contentDescription = tab.label) },
                        label = { Text(tab.label) }
                    )
                }
            }
        }
    ) { padding ->
        NavHost(
            navController = navController,
            startDestination = HomeTab,
            modifier = Modifier.padding(padding)
        ) {
            // Each tab is a nested graph with its own start destination
            navigation<HomeTab>(startDestination = HomeScreen) {
                composable<HomeScreen> { HomeContent(navController) }
                composable<ProductDetail> { ProductDetailContent() }
                // Deep within Home tab — has its own back stack independent of Search tab
            }
            navigation<SearchTab>(startDestination = SearchScreen) {
                composable<SearchScreen> { SearchContent(navController) }
                composable<SearchResults> { SearchResultsContent() }
            }
            navigation<ProfileTab>(startDestination = ProfileScreen) {
                composable<ProfileScreen> { ProfileContent() }
                composable<Settings> { SettingsContent() }
            }
        }
    }
}

// Key flags:
// saveState = true  → when navigating away from a tab, save its back stack
// restoreState = true → when returning to a tab, restore its saved back stack
// launchSingleTop = true → don't create multiple copies of the root tab destination
```

---

## 6. Modal Bottom Sheets in Navigation

```kotlin
// Navigation 2.8+ — bottom sheet as a destination
import androidx.navigation.compose.dialog
// Use dialog() for modal dialogs
// For bottom sheets: use ModalBottomSheet + navigate pattern

@Serializable data object FilterSheet

// In NavHost
bottomSheet<FilterSheet> {  // requires accompanist-navigation-material or custom
    FilterBottomSheet(
        onDismiss = { navController.popBackStack() },
        onApply = { filters ->
            navController.previousBackStackEntry
                ?.savedStateHandle
                ?.set("filters", filters)
            navController.popBackStack()
        }
    )
}

// OR: simpler approach — manage sheet state locally, navigate only for full screens
@Composable
fun ProductListScreen() {
    var showFilter by rememberSaveable { mutableStateOf(false) }

    if (showFilter) {
        ModalBottomSheet(onDismissRequest = { showFilter = false }) {
            FilterContent(onApply = { showFilter = false })
        }
    }
}
// Rule: if the sheet has its own deep link or must survive process death → nav destination
// If it's ephemeral UI (filter picker, sort options) → local state
```

---

## 7. Deep Links into Nested Back Stacks

```kotlin
// Deep link restores the full back stack hierarchy
// e.g., notification tapped → opens ProductDetail inside HomeTab

composable<ProductDetail>(
    deepLinks = listOf(
        navDeepLink<ProductDetail>(basePath = "https://example.com/product")
        // generates: https://example.com/product?productId=123
    )
) { }

// In AndroidManifest.xml
<activity android:name=".MainActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="https" android:host="example.com"/>
    </intent-filter>
</activity>

// Navigate to deep link with back stack restored
val deepLinkIntent = NavDeepLinkRequest.Builder
    .fromUri("https://example.com/product?productId=123".toUri())
    .build()
navController.navigate(deepLinkIntent)

// From notification PendingIntent — restores full task back stack
val pendingIntent = navController.createDeepLinkPendingIntent(
    deepLink = navDeepLink { uriPattern = "https://example.com/product/{productId}" },
    args = bundleOf("productId" to "123")
)
// When tapped: opens app, restores [Home → ProductDetail] back stack automatically
```

---

## 8. Navigation in Multi-Module Apps

Feature modules can't import each other. Navigation uses routes (serializable objects) defined in shared `:core:navigation` module.

```
:app                    → NavHost, wires all graphs
:feature:home           → HomeGraph extension function
:feature:checkout       → CheckoutGraph extension function
:core:navigation        → Route sealed classes (shared between all modules)
```

```kotlin
// :core:navigation module
@Serializable sealed class AppRoute {
    @Serializable data object Home : AppRoute()
    @Serializable data class ProductDetail(val id: String) : AppRoute()
    @Serializable data object CheckoutGraph : AppRoute()
}

// :feature:home module
fun NavGraphBuilder.homeGraph(navController: NavController) {
    composable<AppRoute.Home> {
        HomeScreen(
            onProductClick = { id -> navController.navigate(AppRoute.ProductDetail(id)) }
        )
    }
    composable<AppRoute.ProductDetail> { backStackEntry ->
        val route: AppRoute.ProductDetail = backStackEntry.toRoute()
        ProductDetailScreen(productId = route.id)
    }
}

// :feature:checkout module
fun NavGraphBuilder.checkoutGraph(navController: NavController) {
    navigation<AppRoute.CheckoutGraph>(startDestination = CheckoutRoute.Address) {
        composable<CheckoutRoute.Address> { AddressScreen() }
        composable<CheckoutRoute.Payment> { PaymentScreen() }
    }
}

// :app module
NavHost(navController, startDestination = AppRoute.Home) {
    homeGraph(navController)
    checkoutGraph(navController)
}
```

---

## 9. React Native — React Navigation

```typescript
// react-native-navigation v7
import { NavigationContainer } from '@react-navigation/native'
import { createNativeStackNavigator } from '@react-navigation/native-stack'
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs'

// Type-safe routes
type RootStackParams = {
    Home: undefined
    ProductDetail: { productId: string }
    Checkout: undefined
}
const Stack = createNativeStackNavigator<RootStackParams>()
const Tab = createBottomTabNavigator()

// Nested navigators
function HomeTab() {
    return (
        <Stack.Navigator>
            <Stack.Screen name="Home" component={HomeScreen}/>
            <Stack.Screen name="ProductDetail" component={ProductDetailScreen}/>
        </Stack.Navigator>
    )
}

function MainTabs() {
    return (
        <Tab.Navigator screenOptions={{ headerShown: false }}>
            <Tab.Screen name="HomeTab" component={HomeTab}/>
            <Tab.Screen name="SearchTab" component={SearchStack}/>
            <Tab.Screen name="ProfileTab" component={ProfileStack}/>
        </Tab.Navigator>
    )
}

// Deep linking
const linking = {
    prefixes: ['https://example.com', 'myapp://'],
    config: {
        screens: {
            HomeTab: {
                screens: {
                    ProductDetail: 'product/:productId'
                }
            }
        }
    }
}
// <NavigationContainer linking={linking}>
```

---

## 10. HLD — Navigation Graph Design

### "Design the navigation architecture for a social commerce app"

```
Graph structure:

AuthGraph (unauthenticated users)
├── Splash (start)
├── Login
├── Register
└── ForgotPassword → popUpTo(Login)

MainGraph (authenticated)
├── BottomNav tabs (independent back stacks)
│   ├── HomeTab
│   │   ├── Home (start)
│   │   ├── ProductList
│   │   └── ProductDetail
│   ├── SearchTab
│   │   ├── Search (start)
│   │   └── SearchResults
│   └── ProfileTab
│       ├── Profile (start)
│       └── Settings
│
├── CheckoutGraph (nested, shared ViewModel)
│   ├── Address (start)
│   ├── Payment
│   └── Confirmation → popUpTo(Home, inclusive=false)
│
└── Modals / Dialogs
    ├── FilterSheet (local state, not nav destination)
    └── ErrorDialog (dialog destination)

Deep links:
    product/{id} → HomeTab → ProductDetail (back stack: Home → ProductDetail)
    order/{id}   → ProfileTab → OrderDetail

Auth guard:
    Entry point checks auth state
    if unauthenticated → navigate to AuthGraph, clear MainGraph
    on login success → navigate to MainGraph, clear AuthGraph
```

---

## 11. Common Misunderstandings & Pitfalls

**❌ Not using `saveState`/`restoreState` on bottom nav tab switches**
Without these flags, each tab switch creates a fresh instance of the tab's root screen — the user's scroll position, back stack within the tab, and any loaded data is lost. Always include `saveState = true` and `restoreState = true` in the tab `navigate` lambda.

**❌ Using string routes instead of type-safe objects (Navigation 2.8+)**
String routes like `"product/{id}"` cause runtime crashes from typos and have no compile-time checking. Type-safe routes with `@Serializable` objects catch these at compile time. Migrate all new code to type-safe routes.

**❌ Creating a NavController outside a Composable**
`NavController` must be created with `rememberNavController()` inside a Composable — not in a ViewModel or singleton. Holding a NavController reference in a ViewModel creates a memory leak (NavController holds a Context reference).

**❌ Navigating directly from ViewModel**
ViewModels should emit navigation events (via `Channel<Effect>`) and the Composable should call `navController.navigate()`. Never inject NavController into a ViewModel — it causes leaks and makes testing impossible.

**❌ `popUpTo` with wrong inclusive setting**
`popUpTo<Login>(inclusive = true)` removes Login from the back stack. `inclusive = false` keeps Login but removes everything above it. Confusing these causes either Login persisting (user can go back to it after logging in) or too many screens being removed.

**❌ Accessing `currentBackStackEntry` for results instead of `previousBackStackEntry`**
To pass a result back to the calling screen, set it on `previousBackStackEntry?.savedStateHandle`. Then on the receiving screen, observe from `currentBackStackEntry?.savedStateHandle`. Getting these backwards causes null pointer exceptions.

---

## 12. Best Practices

- **Type-safe routes (Navigation 2.8+)** — `@Serializable` objects, not strings
- **`saveState`/`restoreState` for all bottom nav items** — tab state preserved on switch
- **`launchSingleTop = true`** — prevent duplicate screens at top of back stack
- **Nested graphs for multi-screen flows** — checkout, onboarding, auth all get their own graph
- **NavGraph-scoped ViewModel for shared flow state** — checkout ViewModel lives across address/payment/confirmation
- **Navigation events from ViewModel via Channel** — never inject NavController into ViewModel
- **One NavController per NavHost** — don't share NavControllers across nested NavHosts
- **`popUpTo` with `inclusive = true` to clear auth screens** — after login, prevent back navigation to Login
- **Feature module navigation via `:core:navigation` shared routes** — modules never import each other directly
- **`NavigationSuiteScaffold`** — auto-switches between bottom bar (compact) and navigation rail (medium/expanded) based on window size

---

## 13. Interview Q&A

**Q1: How does the back stack work in Jetpack Navigation?**

> Jetpack Navigation maintains a stack of `NavBackStackEntry` objects — each entry represents a destination, holds a lifecycle, and owns a `ViewModelStore`. When you navigate to a new destination, an entry is pushed. `popBackStack()` removes the top entry and the associated ViewModel is cleared. The `NavController` ensures only the top entry is in the RESUMED state; entries below are STARTED. `popUpTo` lets you pop multiple entries at once — for example, after login you pop all auth screens so the user can't navigate back to the login screen.

---

**Q2: How do you preserve tab state in a bottom navigation bar?**

> With two flags on the `navigate()` call inside each tab's `onClick`: `saveState = true` saves the tab's current back stack and scroll positions when you navigate away, and `restoreState = true` restores that saved state when you return to the tab. Combined with `launchSingleTop = true` (prevents creating duplicate root tab screens), this gives the expected behavior: each tab independently maintains its own navigation history, and switching tabs doesn't reset progress within a tab.

---

**Q3: How would you structure the navigation graph for an app with auth + onboarding + main flow?**

> Three separate nested graphs. The `AuthGraph` contains Splash, Login, Register, ForgotPassword with Login as its start destination. The `OnboardingGraph` contains the onboarding screens. The `MainGraph` contains the bottom nav tabs and all main content. On app launch, I check auth state: if unauthenticated, navigate to AuthGraph and ensure MainGraph is cleared from the back stack. On successful login, navigate to MainGraph and `popUpTo(AuthGraph, inclusive = true)` so back press doesn't return to Login. If onboarding is needed, route through OnboardingGraph first. Each graph is a self-contained navigation unit, and the transition between them is handled by the auth state observer in the root composable.

---

**Q4: How do you share a ViewModel across a multi-screen checkout flow?**

> Scope the ViewModel to the checkout `NavGraph`'s back stack entry. Define the checkout as a `navigation<CheckoutGraph>(startDestination = Address)` block. In each checkout screen composable, retrieve the `NavBackStackEntry` for `CheckoutGraph` using `navController.getBackStackEntry(CheckoutGraph)`, then pass it to `hiltViewModel(checkoutEntry)`. All three screens (Address, Payment, Confirmation) get the same ViewModel instance because they all reference the same graph-level `ViewModelStore`. When the user completes checkout and the entire checkout graph is popped, the ViewModel is cleared automatically.

---

*Previous: [28 — Maps Integration](./28-maps-integration.md)*
*Next: [30 — In-App Purchases](./30-in-app-purchases.md)*
