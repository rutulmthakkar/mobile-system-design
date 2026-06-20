# 13 — Cross-Cutting Concerns

> **Round type: HLD (primary) + LLD**
> **Topics:** Internationalization (i18n), Deep Links & App Links, Accessibility, OS/Device Fragmentation, User Targeting

---

## 1. Why This Matters in Interviews

These topics appear in system design interviews as "what else did you consider?" signals. A strong candidate designs for a global audience (i18n), thinks about how users arrive in the app (deep links), considers users with disabilities (accessibility), and handles the diversity of Android devices (fragmentation). Weak candidates only design for the happy path English-speaking user on a flagship device.

**Common interview angles:**
- "How would you design a feature for international markets?"
- "How does deep linking work and how do you secure it?"
- "What accessibility considerations would you include in your design?"
- "How do you support Android 8 through Android 15 with the same codebase?"
- "How do you target a feature to premium users in specific countries?"

---

## 2. Internationalization & Localization (i18n / l10n)

### 2.1 The Distinction

- **i18n (Internationalization):** Engineering work — designing the system to support multiple languages/regions (string externalization, RTL layouts, locale-aware formatting)
- **l10n (Localization):** Translation work — providing actual translations for each locale

Do i18n once at design time. l10n happens per language, continuously.

### 2.2 String Resources

```xml
<!-- res/values/strings.xml (default = English) -->
<resources>
    <!-- Simple string -->
    <string name="welcome_title">Welcome back</string>

    <!-- String with placeholder -->
    <string name="greeting">Hello, %1$s!</string>

    <!-- Plurals — handles "1 item" vs "2 items" -->
    <!-- Some languages (Arabic) have 6 plural forms; always provide "one" and "other" -->
    <plurals name="item_count">
        <item quantity="zero">No items</item>
        <item quantity="one">%d item</item>
        <item quantity="other">%d items</item>
    </plurals>

    <!-- String that should NOT be translated -->
    <string name="app_name" translatable="false">MyApp</string>

    <!-- Format string — avoid concatenation, order varies per language -->
    <!-- ❌ Bad: "Posted on " + date + " by " + user -->
    <!-- ✅ Good: -->
    <string name="post_attribution">Posted on %1$s by %2$s</string>
    <!-- In German: "Von %2$s am %1$s gepostet" — order reverses! -->
</resources>

<!-- res/values-de/strings.xml (German) -->
<resources>
    <string name="welcome_title">Willkommen zurück</string>
    <string name="greeting">Hallo, %1$s!</string>
    <plurals name="item_count">
        <item quantity="one">%d Artikel</item>
        <item quantity="other">%d Artikel</item>
    </plurals>
    <string name="post_attribution">Von %2$s am %1$s gepostet</string>
</resources>

<!-- res/values-ar/strings.xml (Arabic - RTL) -->
<!-- res/values-zh-rCN/strings.xml (Chinese Simplified) -->
<!-- res/values-b+sr+Latn/strings.xml (Serbian Latin script) -->
```

### 2.3 Compose String APIs

```kotlin
@Composable
fun WelcomeScreen(userName: String, itemCount: Int) {
    Column {
        // Simple string
        Text(text = stringResource(R.string.welcome_title))

        // String with argument
        Text(text = stringResource(R.string.greeting, userName))

        // Plurals
        Text(text = pluralStringResource(
            id = R.plurals.item_count,
            count = itemCount,
            itemCount          // passed as format argument
        ))
    }
}

// Quantity strings in ViewModel (not Composable) — needs Context
class CartViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel() {
    fun getItemCountText(count: Int): String =
        context.resources.getQuantityString(R.plurals.item_count, count, count)
}
```

### 2.4 RTL (Right-to-Left) Layout Support

Arabic, Hebrew, Farsi, Urdu — all read right-to-left. The OS automatically mirrors layouts when RTL is active, but you must design for it.

```kotlin
// In Compose: use start/end semantics, not left/right
// ❌ Bad — doesn't mirror for RTL
Box(modifier = Modifier.padding(start = 16.dp))  // this IS start-aware ✅ in Compose
// But avoid:
Modifier.absolutePaddingLeft(16.dp)  // ❌ never mirrors

// Correct padding for RTL-aware layouts
Column(modifier = Modifier
    .padding(start = 16.dp, end = 8.dp)  // start/end, not left/right
) { }

// Row in Compose follows layout direction automatically
// If RTL: first item appears on RIGHT, last item on LEFT
Row {
    Icon(...)   // appears on right in Arabic
    Text(...)   // appears to the left of icon in Arabic
}

// Force specific direction when needed (e.g., media player controls always LTR)
CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Ltr) {
    Row {
        SkipPreviousButton()
        PlayButton()
        SkipNextButton()
    }
}

// In XML: use start/end not left/right
android:paddingStart="16dp"     // ✅ RTL-aware
android:layout_marginEnd="8dp"  // ✅ RTL-aware
android:paddingLeft="16dp"      // ❌ doesn't mirror
```

**Mirrored drawables:** Icons that have directional meaning (back arrow, forward arrow) must be mirrored for RTL.

```xml
<!-- Mark drawable as auto-mirrored -->
<vector android:autoMirrored="true" ...>
    <path android:pathData="M20,11H7.83l5.59-5.59L12,4l-8,8 8,8 1.41-1.41L7.83,13H20v-2z"/>
</vector>
```

### 2.5 Locale-Aware Formatting

Never hardcode date, time, or number formats — they vary significantly:

```kotlin
// ❌ Bad — hardcoded US format
val dateStr = String.format("%d/%d/%d", month, day, year)  // "12/25/2024" — wrong for EU

// ✅ Good — locale-aware
val date = LocalDate.of(2024, 12, 25)
val formatted = DateTimeFormatter
    .ofLocalizedDate(FormatStyle.MEDIUM)
    .withLocale(Locale.getDefault())
    .format(date)
// English: "Dec 25, 2024"
// German: "25.12.2024"
// Japanese: "2024年12月25日"

// Numbers and currency
val amount = 1234567.89
val numberFormat = NumberFormat.getCurrencyInstance(Locale.getDefault())
numberFormat.currency = Currency.getInstance("USD")
val formatted = numberFormat.format(amount)
// English: "$1,234,567.89"
// German: "1.234.567,89 $"    ← dot and comma SWAP
// French: "1 234 567,89 $"    ← space as thousands separator

// Measurement units
// Consider unit preference: US uses miles/°F, most countries use km/°C
```

### 2.6 Per-App Language Preferences (Android 13+)

Users can set a preferred language per app without changing the system language:

```kotlin
// Check and set in-app language (API 33+)
val appLocales = AppCompatDelegate.getApplicationLocales()

// Let user pick language in app settings
AppCompatDelegate.setApplicationLocales(
    LocaleListCompat.create(Locale.forLanguageTag("de"))
)
// This persists automatically — no Activity restart needed on API 33+
// On older APIs: manual Activity recreation required

// In AndroidManifest.xml — declare supported locales (API 33+)
<application>
    <service android:name="androidx.appcompat.app.AppLocalesMetadataHolderService"
        android:enabled="false" android:exported="false">
        <meta-data android:name="autoStoreLocales" android:value="true"/>
    </service>
</application>
```

### 2.7 Pseudolocalization (Testing)

Test your UI for layout issues before real translations arrive:

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        debug {
            isPseudoLocalesEnabled = true
        }
    }
}
```

Pseudolocale (`en-XA`) adds accents to all characters and prefixes `[` / `]` to expose untranslated strings. Pseudolocale (`ar-XB`) uses bidirectional text to test RTL layouts. Run your app in Android Studio with these locales before sending strings to translators — catch hardcoded strings and layout truncations cheaply.

### 2.8 Text Expansion

Languages are not the same length as English. Plan for expansion:

| Language | Typical expansion vs English |
|---|---|
| German | +30–35% |
| French | +15–20% |
| Finnish | +20–30% |
| Arabic | -20% (shorter but RTL) |
| Chinese/Japanese | -30–50% (more compact characters) |

**Design rule:** Never use `android:maxLines="1"` or fixed-width containers for translated strings without testing. Use `wrap_content` and ellipsize carefully.

---

## 3. Deep Links & App Links

### 3.1 Types of Links

| Type | Scheme | App installed? | Opens in | Disambiguation? |
|---|---|---|---|---|
| **Custom scheme deep link** | `myapp://` | Required | App | Only if another app claims same scheme |
| **Web link (implicit)** | `http://` or `https://` | Optional | Browser (or app picker) | Yes — user chooses |
| **Android App Link** | `https://` (verified) | Optional | App (no picker) | **No — opens directly** |
| **Instant App link** | `https://` | Not required | App inline | No |

### 3.2 Deep Links — Custom Scheme

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="myapp"
              android:host="product"
              android:pathPattern="/.*"/>
        <!-- Handles: myapp://product/123 -->
    </intent-filter>
</activity>
```

```kotlin
// Handle deep link in Activity/Compose Navigation
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handleDeepLink(intent)
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        handleDeepLink(intent)
    }

    private fun handleDeepLink(intent: Intent) {
        intent.data?.let { uri ->
            // myapp://product/123
            val productId = uri.lastPathSegment
            navigateToProduct(productId)
        }
    }
}

// With Jetpack Navigation — declare deep link in nav graph
<fragment android:id="@+id/productFragment">
    <deepLink
        app:uri="myapp://product/{productId}"
        app:action="android.intent.action.VIEW"/>
</fragment>
```

**Custom scheme limitation:** Any app can claim `myapp://` — another malicious app could intercept your deep links. Don't pass sensitive data (auth tokens, payment info) via custom scheme deep links.

### 3.3 Android App Links — Verified HTTPS Deep Links ✅

App Links use HTTPS URLs and require domain verification. Only your verified app opens these links — no disambiguation dialog.

**Step 1: Declare in manifest**
```xml
<activity android:name=".MainActivity">
    <intent-filter android:autoVerify="true">
        <!-- autoVerify triggers domain ownership verification -->
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="https"
              android:host="www.example.com"
              android:pathPrefix="/product"/>
        <!-- Handles: https://www.example.com/product/123 -->
    </intent-filter>
</activity>
```

**Step 2: Host Digital Asset Links JSON**
```json
// https://www.example.com/.well-known/assetlinks.json
[{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        "package_name": "com.example.myapp",
        "sha256_cert_fingerprints": [
            "AB:CD:EF:..."  // SHA-256 of your app's signing certificate
        ]
    }
}]
```

```bash
# Get your signing certificate fingerprint
keytool -list -v -keystore my-release-key.jks
# OR from Play Console → App Signing → SHA-256 certificate fingerprint
```

**Step 3: Verify it works**
```bash
# Test domain verification
adb shell pm get-app-links com.example.myapp

# Test specific link
adb shell am start -a android.intent.action.VIEW \
    -c android.intent.category.BROWSABLE \
    -d "https://www.example.com/product/123"
```

### 3.4 Deep Links from Notifications

```kotlin
// Build notification with deep link PendingIntent
val deepLinkIntent = Intent(
    Intent.ACTION_VIEW,
    "https://example.com/order/456".toUri(),
    context,
    MainActivity::class.java
)
val pendingIntent = PendingIntent.getActivity(
    context, 0, deepLinkIntent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)

val notification = NotificationCompat.Builder(context, CHANNEL_ID)
    .setContentTitle("Order shipped!")
    .setContentText("Tap to track your order")
    .setContentIntent(pendingIntent)
    .setAutoCancel(true)
    .build()
```

### 3.5 Deep Link Security

```kotlin
// ❌ Never pass sensitive data via deep link
// myapp://auth?token=eyJhbGc...  ← token in URL = logged by every proxy, analytics tool

// ✅ Pass a short-lived opaque code, exchange for token server-side
// myapp://auth?code=one_time_code_123  ← code useless after single use

// ❌ Never trust deep link parameters without validation
fun handleProductDeepLink(uri: Uri) {
    val id = uri.getQueryParameter("id")  // could be malicious
    loadProduct(id)  // SQL injection? Path traversal?
}

// ✅ Validate and sanitize
fun handleProductDeepLink(uri: Uri) {
    val id = uri.getQueryParameter("id")?.let {
        it.toLongOrNull()?.takeIf { id -> id > 0 }  // only positive integers
    } ?: return  // reject malformed IDs
    loadProduct(id)
}

// For App Links: verify the calling app's package if needed
// The system auto-verifies domain ownership, so you can trust HTTPS App Links more than custom schemes
```

---

## 4. Accessibility

### 4.1 Why Accessibility Matters

Over 1 billion people worldwide have a disability. In the EU, the European Accessibility Act (EAA) took effect June 28, 2025 — non-compliant apps face legal risk in EU markets. Beyond compliance: accessible apps score better in Play Store, are more usable in difficult conditions (bright sunlight, one-handed, noisy environments), and Google's ranking algorithms consider accessibility.

### 4.2 TalkBack & Screen Readers

TalkBack (Android's screen reader) reads UI aloud. It navigates by swiping and double-tapping. Your app must make sense when read aloud.

```kotlin
// Compose accessibility
@Composable
fun ProductCard(product: Product, onAddToCart: () -> Unit) {
    Card(
        modifier = Modifier
            .semantics {
                // Group related elements so TalkBack reads them as one unit
                contentDescription = "${product.name}, ${product.price}, ${product.rating} stars"
            }
    ) {
        AsyncImage(
            model = product.imageUrl,
            contentDescription = "${product.name} product image"  // ✅ describe image
            // null for decorative images that add no information
        )
        Text(product.name)
        Text(product.price)

        IconButton(
            onClick = onAddToCart,
            modifier = Modifier.semantics {
                contentDescription = "Add ${product.name} to cart"  // ✅ describe action
            }
        ) {
            Icon(
                imageVector = Icons.Default.Add,
                contentDescription = null  // ✅ null: label is on parent IconButton
            )
        }
    }
}

// Custom action for complex gestures
@Composable
fun SwipeableItem(onDelete: () -> Unit) {
    Box(modifier = Modifier.semantics {
        customActions = listOf(
            CustomAccessibilityAction(
                label = "Delete item",
                action = { onDelete(); true }
            )
        )
    }) {
        // swipe-to-delete UI...
    }
}
```

### 4.3 Touch Target Size

Minimum 48×48dp for any interactive element (Google Material Design + WCAG 2.1 AA):

```kotlin
// Small icon button — expand touch target without changing visual size
IconButton(
    onClick = { },
    modifier = Modifier
        .size(48.dp)  // ✅ minimum touch target
) {
    Icon(
        modifier = Modifier.size(24.dp),  // visual size (smaller)
        imageVector = Icons.Default.Favorite,
        contentDescription = "Add to favorites"
    )
}

// Or use minimumInteractiveComponentSize (Compose Material 3)
Icon(
    modifier = Modifier
        .minimumInteractiveComponentSize()  // enforces 48dp minimum
        .clickable { }
)
```

### 4.4 Color Contrast

WCAG 2.1 AA requires:
- **Normal text:** 4.5:1 contrast ratio
- **Large text (18sp+ or 14sp+ bold):** 3:1 ratio
- **UI components and icons:** 3:1 ratio

```kotlin
// Material 3 Design System handles this automatically
// If custom colors: verify with contrast checker
// Material Theme Builder: https://m3.material.io/theme-builder

// Don't rely on color alone to convey information
// ❌ Red text for error, green for success — colorblind users can't distinguish
// ✅ Error icon + red text + "Error:" prefix — multiple indicators
@Composable
fun ErrorText(message: String) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Icon(
            imageVector = Icons.Default.Error,
            contentDescription = null,  // described by text
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(Modifier.width(4.dp))
        Text(
            text = "Error: $message",   // "Error:" prefix makes meaning clear without color
            color = MaterialTheme.colorScheme.error
        )
    }
}
```

### 4.5 Content Descriptions Best Practices

```kotlin
// ❌ Bad descriptions
contentDescription = "image"           // says nothing
contentDescription = "ic_profile_pic"  // internal resource name leaks
contentDescription = "button"          // redundant — TalkBack already says "button"

// ✅ Good descriptions — describe PURPOSE, not appearance
contentDescription = "Alice's profile picture"
contentDescription = "Search"           // what it does
contentDescription = "Close dialog"     // clear action

// Null for decorative content
Icon(imageVector = Icons.Default.Star, contentDescription = null)  // if rating text already present

// Compose: merge semantics for complex components
Row(modifier = Modifier.semantics(mergeDescendants = true) { }) {
    // TalkBack reads this entire row as one item
    Icon(...)
    Text("4.5 stars")
    Text("(2,341 reviews)")
    // TalkBack: "4.5 stars (2,341 reviews)" — one announcement
}
```

### 4.6 Accessibility Testing

```kotlin
// 1. Enable TalkBack: Settings → Accessibility → TalkBack
// Navigate your entire app using only TalkBack gestures

// 2. Automated testing with UIAutomator
@Test
fun checkAccessibility() {
    val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())
    val context = ApplicationProvider.getApplicationContext<Context>()

    // Use AccessibilityNodeInfo to verify descriptions
    val info = device.findObject(By.desc("Add to cart"))
    assertNotNull("Add to cart button must have content description", info)
}

// 3. Compose UI test — semantic assertions
@Test
fun productCard_hasCorrectAccessibility() {
    composeTestRule.setContent { ProductCard(product = testProduct, onAddToCart = {}) }
    composeTestRule.onNodeWithContentDescription("Add ${testProduct.name} to cart")
        .assertHasClickAction()
        .assertIsEnabled()
}

// 4. Accessibility Scanner (Google app)
// Install on device → run → scans screen and reports issues

// 5. Lint checks
// AndroidStudio has accessibility lint checks:
// "Missing contentDescription on image" warnings
```

---

## 5. Device & OS Fragmentation

### 5.1 Choosing Minimum SDK

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        minSdk = 26   // Android 8.0 Oreo — covers ~97% of active devices (2024)
        targetSdk = 35 // Android 15 — always target latest stable
        compileSdk = 35
    }
}
```

**minSdk selection guide (2024 data):**
| minSdk | API level | Android version | Active device coverage |
|---|---|---|---|
| 21 | Lollipop | 5.0 | ~99.8% |
| 24 | Nougat | 7.0 | ~99.5% |
| **26** | **Oreo** | **8.0** | **~97%** |
| 28 | Pie | 9.0 | ~95% |
| 31 | Android 12 | 12 | ~80% |

**Rule:** minSdk 26 is the sweet spot for most consumer apps — covers ~97% of devices and unlocks Channels API for notifications, background execution improvements, and Autofill API.

### 5.2 Handling API Level Differences

```kotlin
// Check API level at runtime for APIs not available on minSdk
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {  // Android 12+
    val splashScreen = installSplashScreen()
    splashScreen.setKeepOnScreenCondition { !viewModel.isReady.value }
}

// AndroidX compatibility libraries handle most of this automatically
// Use Compat variants over direct API calls:
ContextCompat.getColor(context, R.color.primary)           // vs context.getColor()
ActivityCompat.requestPermissions(...)                      // vs requestPermissions()
NotificationManagerCompat.from(context).notify(...)        // vs getSystemService()
WindowCompat.setDecorFitsSystemWindows(window, false)      // vs window.setDecorFitsSystemWindows()

// Compose handles API differences internally
// But check: some Compose APIs require minSdk 21+; Material3 requires 21+
```

### 5.3 Screen Size & Adaptive Layouts

```kotlin
// Compose adaptive layout
@Composable
fun AdaptiveProductScreen() {
    val windowSizeClass = calculateWindowSizeClass(activity)

    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // Phone portrait — single column
            ProductListScreen()
        }
        WindowWidthSizeClass.Medium -> {
            // Tablet portrait / foldable — two columns or list-detail
            TwoColumnProductLayout()
        }
        WindowWidthSizeClass.Expanded -> {
            // Tablet landscape / desktop — full list-detail
            ListDetailProductLayout()
        }
    }
}

// LazyVerticalGrid for responsive grid
LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 160.dp),  // auto-adapts column count
    contentPadding = PaddingValues(16.dp)
) {
    items(products) { product -> ProductCard(product) }
}
```

### 5.4 Foldable Support

```kotlin
// Detect fold state
implementation("androidx.window:window:1.3.0")

val windowInfoTracker = WindowInfoTracker.getOrCreate(context)
windowInfoTracker.windowLayoutInfo(activity).collect { layoutInfo ->
    val fold = layoutInfo.displayFeatures
        .filterIsInstance<FoldingFeature>()
        .firstOrNull()

    when {
        fold == null -> normalLayout()
        fold.state == FoldingFeature.State.HALF_OPENED -> tableTopLayout()
        fold.isSeparating -> dualPaneLayout(fold.bounds)
        else -> normalLayout()
    }
}
```

### 5.5 OEM Customizations & Fragmentation Issues

Different OEMs customize Android in ways that break apps:

```kotlin
// Common fragmentation issues:

// 1. Battery optimization aggressively kills background work
// Samsung, Xiaomi, Huawei have aggressive doze policies
// WorkManager handles this better than direct JobScheduler
// Guide users to exempt app from battery optimization for critical features

// 2. Background location access varies by OEM
// Some OEMs require additional permission grants beyond system dialogs

// 3. Notification channels — some OEMs show their own notification management UI
// Always create channels on API 26+ but don't assume they appear as designed

// 4. Font scaling — users can set system font scale (0.85x to 3.0x)
// Use sp for text sizes, test at largest scale
Text(text = "Title", fontSize = 24.sp)  // ✅ respects user's font scaling
// Avoid fixed-height containers for text

// 5. Dark mode implementation varies
// Some OEM dark modes invert colors rather than using dark theme
// Test both Material3 dark theme AND forced night mode
```

---

## 6. User Targeting

### 6.1 Capabilities-Based Targeting

```kotlin
// Target features based on hardware capabilities
fun canUseARFeature(): Boolean {
    return ArCoreApk.getInstance().checkAvailability(context) ==
            ArCoreApk.Availability.SUPPORTED_INSTALLED
}

fun canUseHighQualityCamera(): Boolean {
    val cameraManager = context.getSystemService(CameraManager::class.java)
    return cameraManager.cameraIdList.any { id ->
        val chars = cameraManager.getCameraCharacteristics(id)
        chars.get(CameraCharacteristics.INFO_SUPPORTED_HARDWARE_LEVEL) ==
                CameraCharacteristics.INFO_SUPPORTED_HARDWARE_LEVEL_3
    }
}

// In Play Store manifest — declare hardware requirements
<uses-feature android:name="android.hardware.camera.ar" android:required="false"/>
// required="false" = app appears in Play Store for all devices
// check at runtime and gracefully degrade
```

### 6.2 Subscription & Permission-Based Gating

```kotlin
// Gate premium features
class PremiumGate @Inject constructor(
    private val userRepo: UserRepository,
    private val featureFlags: FeatureFlags
) {
    suspend fun canAccessFeature(feature: Feature): Boolean {
        val user = userRepo.getCurrentUser() ?: return false
        return when (feature) {
            Feature.ADVANCED_ANALYTICS -> user.subscription == SubscriptionTier.PREMIUM
            Feature.EXPORT_PDF -> user.subscription in listOf(SubscriptionTier.PRO, SubscriptionTier.PREMIUM)
            Feature.DARK_THEME -> featureFlags.isDarkThemeForAll()
                    || user.subscription != SubscriptionTier.FREE
            Feature.BETA_FEED -> featureFlags.isBetaFeedEnabled()
                    && user.isBetaTester
        }
    }
}

// Gated composable
@Composable
fun PremiumFeatureContent(user: User, onUpgrade: () -> Unit) {
    if (user.isPremium) {
        AdvancedAnalyticsDashboard()
    } else {
        PremiumUpsellPrompt(
            featureName = "Advanced Analytics",
            onUpgrade = onUpgrade
        )
    }
}
```

### 6.3 Country & Region Targeting

```kotlin
// Get user's locale/country
val locale = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    context.resources.configuration.locales[0]
} else {
    @Suppress("DEPRECATION")
    context.resources.configuration.locale
}

val countryCode = locale.country  // "US", "DE", "JP"
val languageCode = locale.language  // "en", "de", "ja"

// Country-specific features
fun isUpiPaymentEnabled(): Boolean = countryCode == "IN"  // India-only
fun isSEPAPaymentEnabled(): Boolean = countryCode in EU_COUNTRY_CODES
fun usesImperialUnits(): Boolean = countryCode in listOf("US", "LR", "MM")

// Don't rely solely on device locale — user may be traveling
// Better: server-side targeting based on IP + user preference
```

---

## 7. HLD — Designing for a Global Audience

### "Design a social media app for 100M users in 20 countries"

```
Internationalization:
  - All strings in res/values-{locale}/strings.xml
  - Date/time via DateTimeFormatter with system locale
  - Currency via NumberFormat.getCurrencyInstance(locale)
  - RTL layouts using start/end semantics throughout
  - Per-app language preference (API 33+)
  - Pseudolocalization in debug builds
  - Text expansion budget: design all UI for +35% text length

Deep Links:
  - Android App Links (verified HTTPS) for all shareable content
  - Push notification deep links → App Link format
  - Deep link handling in NavHost via <deepLink> declarations
  - Validate and sanitize all deep link parameters

Accessibility:
  - All images have contentDescription
  - All interactive elements ≥ 48×48dp
  - Color contrast ≥ 4.5:1 for text
  - Content semantic grouping with mergeDescendants
  - Dark mode support (Material3 dynamic color)
  - Font scale testing at 3x

Device support:
  - minSdk 26 (covers ~97% active devices)
  - Adaptive layout for phone/tablet/foldable
  - LazyVerticalGrid(GridCells.Adaptive) for feed
  - WorkManager for background sync (handles OEM battery optimizations)
  - Test on low-end device (2GB RAM, slow CPU) before each release

Country targeting:
  - Payment methods per country (UPI in India, SEPA in EU, etc.)
  - Content moderation rules per jurisdiction
  - Data residency: EU users' data in EU data centers (GDPR)
  - Feature flags: US-only beta features, staged by country
```

---

## 8. Common Misunderstandings & Pitfalls

**❌ Using `left`/`right` instead of `start`/`end` in layouts**
In RTL languages, `paddingLeft` doesn't mirror — content stays on the left instead of moving to the right. Always use `paddingStart`/`paddingEnd` and `marginStart`/`marginEnd`. In Compose, all standard modifiers use start/end correctly — only `absolutePaddingLeft()` is wrong.

**❌ String concatenation for multi-language text**
`"Hello " + userName + "!"` works in English but breaks in languages where the word order changes. Always use format strings: `stringResource(R.string.greeting, userName)`.

**❌ Hardcoded date/time formats**
`SimpleDateFormat("MM/dd/yyyy")` is US-only. Germany uses `dd.MM.yyyy`, Japan uses `yyyy年MM月dd日`. Use `DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)`.

**❌ Using custom scheme deep links for sensitive flows**
`myapp://reset-password?token=abc123` — any app claiming the `myapp://` scheme can intercept this. Use Android App Links (HTTPS + DAL verification) for any flow that involves authentication or sensitive data.

**❌ Missing `autoVerify="true"` on App Link intent filters**
Without `autoVerify`, Android asks users to choose which app handles the URL. With it, your app opens automatically. Without the `assetlinks.json` file on your domain, verification fails even with `autoVerify="true"`.

**❌ Empty or generic `contentDescription` values**
`contentDescription = "image"` or `contentDescription = "button"` tells screen reader users nothing. Describe the purpose: "User profile photo" or "Submit payment". Never describe appearance; always describe purpose.

**❌ Using `px` or `dp` for font sizes**
Font sizes in `px` don't respect user's system font scale setting. Font sizes in `dp` also don't. Always use `sp` for text — it scales with both display density AND user font size preference.

**❌ Testing accessibility only with Accessibility Scanner**
Accessibility Scanner catches some issues but misses others (logical reading order, meaningful content descriptions, custom action accessibility). Manually navigate the entire app with TalkBack enabled — there's no substitute.

---

## 9. Best Practices

**Internationalization:**
- All strings in `strings.xml` — never hardcode user-visible text
- Format strings with positional arguments: `%1$s %2$s` not `%s %s` (order varies by language)
- Design for +35% text expansion — test with German/Finnish localization
- `start`/`end` everywhere, never `left`/`right`
- `sp` for all font sizes, never `px` or `dp` for text
- `pseudoLocalesEnabled = true` in debug builds
- Locale-aware date, time, and number formatting

**Deep Links:**
- Android App Links (HTTPS + `autoVerify`) over custom schemes for important flows
- Host `assetlinks.json` at `https://yourdomain.com/.well-known/assetlinks.json`
- Never pass secrets (tokens, PII) in deep link URLs
- Validate all deep link parameters before using them
- Test with `adb shell am start -d "your://deeplink"`

**Accessibility:**
- `contentDescription` on all non-decorative images and icon buttons
- Minimum 48×48dp touch targets for all interactive elements
- 4.5:1 color contrast ratio for text (3:1 for large text and UI components)
- Don't rely on color alone — always add icon, label, or text reinforcement
- `semantics(mergeDescendants = true)` for complex multi-element components
- Test with TalkBack manually, not just automated tools
- European Accessibility Act compliance for EU market apps (June 2025)

**Fragmentation:**
- minSdk 26 is the 2024 sweet spot — covers ~97% of active devices
- Always use AndroidX Compat APIs over direct SDK calls
- Adaptive layouts with `WindowSizeClass` for phone/tablet/foldable
- Test on a low-end device before every release
- `GridCells.Adaptive` for responsive grids

---

## 10. Interview Q&A

**Q1: How would you design a feature for international markets?**

> Three layers: internationalization (engineering), localization (translation), and regional adaptation (product). For i18n: all strings in `strings.xml` with named placeholders (not concatenation), dates/numbers formatted with system locale, layouts using start/end semantics, pseudolocalization enabled in debug builds to catch hardcoded strings and layout overflow. For localization: export strings to a translation management system (Phrase, Lokalise, Crowdin), design with +35% text expansion budget, test RTL with Arabic and Hebrew. For regional adaptation: country-specific payment methods, content moderation rules per jurisdiction, date format conventions (ISO 8601 for APIs, localized for display), unit systems (imperial vs metric), and GDPR data residency for EU users.

---

**Q2: What's the difference between a deep link and an App Link?**

> A deep link with a custom scheme like `myapp://product/123` works only when the app is installed and can be claimed by any app registering the same scheme — opening it triggers a disambiguation dialog if multiple apps claim it, or worse, a malicious app could register your scheme and intercept sensitive flows. An Android App Link uses a verified HTTPS URL like `https://example.com/product/123`. Verification works via a `assetlinks.json` file hosted at `/.well-known/` on your domain, linking your app's signing certificate to the domain. Once verified with `autoVerify="true"` in the manifest, clicking these links opens your app directly — no disambiguation dialog, no risk of interception. App Links are the recommended approach for any link that handles authentication, payments, or shared content.

---

**Q3: What accessibility standards apply to Android apps?**

> WCAG 2.1 (Web Content Accessibility Guidelines) at Level AA is the standard Google references for Android. Key requirements: all non-decorative images have meaningful content descriptions; interactive elements have minimum 48×48dp touch targets; text has 4.5:1 color contrast ratio (3:1 for large text); color is not the only way to convey information; content is navigable and usable with TalkBack. Since June 2025, the European Accessibility Act requires apps targeting EU markets to meet accessibility requirements — non-compliance can result in fines. Testing: Accessibility Scanner app for automated checks, then mandatory manual TalkBack walkthrough for logical reading order and action discovery that automation can't check.

---

**Q4: How do you handle an app that needs to support Android 8 through Android 15?**

> Three strategies. First, set `minSdk = 26` (Android 8) and `targetSdk = 35` (Android 15). Second, use AndroidX compatibility libraries — they handle API differences internally; `ContextCompat`, `ActivityCompat`, `NotificationManagerCompat` choose the right API for the running OS version automatically. Third, for APIs that don't have a compat wrapper, guard with `Build.VERSION.SDK_INT >= Build.VERSION_CODES.X`. I avoid `@RequiresApi` annotations on public methods unless the entire feature is restricted to that API level. I test on API 26 in CI (lowest supported) and on current Android in release. WorkManager is my standard for background work — it handles the evolving background execution restrictions from Oreo through Android 15 without me needing to know each OS's specific rules.

---

**Q5: How do you target a feature to premium users in specific countries?**

> Layered targeting using Firebase Remote Config plus server-side user state. Remote Config condition: "User is in US or CA AND subscription_tier property = 'premium'" → feature flag ON. Client reads the flag through the typed `FeatureFlags` abstraction — the ViewModel doesn't know whether it's a Remote Config flag, a server-side flag, or a hardcoded value. Server-side I also maintain this gate in API responses: even if a client bypasses the UI gate, the API returns 403 for premium features to free users. Country detection: I use the user's billing country from their payment method (most accurate) over device locale, since a user may be traveling. I also ensure data residency compliance — EU users' data is processed in EU data centers regardless of which country the feature is being targeted in.

---

**Q6: A non-English user reports that your app is displaying garbled text. How do you debug it?**

> Check three things: encoding, rendering, and locale. Encoding: verify all strings in `strings.xml` are UTF-8 encoded — the default, but sometimes corrupted by editors; check for BOM markers. Rendering: check if the font supports the character set — some custom fonts don't include Arabic or CJK glyphs; the system falls back to a default font which may not match the design. Locale: check if the translation file for their locale exists at the correct path (`res/values-{locale}/strings.xml` for exact match, `res/values-{language}/strings.xml` for language-only fallback) and whether the string key is present in that file. If missing, Android falls back to the default `res/values/strings.xml`. Enable pseudolocales in debug to reproduce locale issues without needing a real translation. Finally, check if the text is assembled via string concatenation rather than format strings — concatenation breaks RTL text direction and word order.

---

*Previous: [12 — App Distribution & Size](./12-app-distribution.md)*
*Next: [14 — Bluetooth & NFC](./14-bluetooth-nfc.md)*
