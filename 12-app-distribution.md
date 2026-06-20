# 12 — App Distribution & Size

> **Round type: HLD (primary) + LLD**
> **HLD:** Distribution strategy, release pipeline, dynamic delivery architecture
> **LLD:** AAB internals, R8 configuration, DFM implementation, size reduction techniques

---

## 1. Why This Matters in Interviews

App size directly impacts install conversion rates and user retention. Google's research: every 6MB increase in download size → 1% drop in install conversion. In emerging markets, a 10MB reduction → ~2.5% more installs. Interviewers at consumer apps ask about this because at scale (millions of downloads), these percentages translate to real revenue.

**Common interview angles:**
- "What's the difference between APK and AAB?"
- "How would you reduce your app from 80MB to under 20MB?"
- "What are dynamic feature modules and when would you use them?"
- "How does R8 differ from ProGuard?"
- "How do you distribute a pre-release build for testing?"
- "What happens to app signing with AAB?"

---

## 2. APK vs AAB — Core Difference

### 2.1 APK (Android Package Kit)

A single self-contained file that includes **everything** — all ABIs, all screen densities, all languages.

```
myApp.apk (100MB)
├── classes.dex
├── lib/
│   ├── arm64-v8a/      ← only this runs on Pixel 9
│   ├── armeabi-v7a/    ← never used on modern device
│   └── x86_64/         ← only used on emulators
├── res/
│   ├── drawable-mdpi/   ← not used on xxhdpi screen
│   ├── drawable-hdpi/   ← not used on xxhdpi screen
│   └── drawable-xxhdpi/ ← the only one used
├── assets/
└── AndroidManifest.xml
```

A Pixel 9 (arm64, xxhdpi) downloads all of this but only uses ~30MB of it. The rest is wasted bandwidth and storage.

### 2.2 AAB (Android App Bundle)

A **publishing format**, not an installable file. Google Play generates optimized, device-specific APKs from it.

```
myApp.aab (submitted to Play Store)
    ↓ Google Play Dynamic Delivery
    
Pixel 9 (arm64, xxhdpi, English):       Galaxy A15 (arm32, hdpi, Spanish):
  base-arm64-v8a.apk (15MB)                base-armeabi-v7a.apk (10MB)
  base-xxhdpi.apk (8MB)                    base-hdpi.apk (4MB)
  base-en.apk (2MB)                        base-es.apk (2MB)
  Total download: 25MB ✅                   Total download: 16MB ✅
  vs 100MB universal APK                   vs 100MB universal APK
```

**Result:** AAB apps are typically 15–35% smaller on-device than universal APKs.

**Required for Play Store:** Since August 2021, all new apps must use AAB. Existing apps must migrate.

### 2.3 AAB Internals

```
myApp.aab
├── base/                    ← base module (always installed)
│   ├── manifest/AndroidManifest.xml (in proto format)
│   ├── dex/                 ← compiled Kotlin/Java
│   ├── res/                 ← resources (in proto format)
│   ├── assets/
│   └── lib/                 ← native libraries (all ABIs)
├── feature_camera/          ← dynamic feature module
├── feature_ar/              ← dynamic feature module
└── BUNDLE-METADATA/         ← ProGuard mappings, DEX list (not shipped to users)
```

Key difference from APK: resources and manifests are in Protocol Buffer format (not binary XML), enabling Play Store to restructure them during split APK generation.

### 2.4 App Signing with AAB

With APK: you sign the APK yourself and upload.
With AAB: you upload the AAB unsigned (or signed with your own key), and Play Store signs the delivery APKs with **Google Play App Signing**.

```
Your signing key (upload key) → signs the AAB → uploaded to Play Store
Google Play App Signing key   → signs the delivery APKs → installed on devices
```

**Why this matters:** If you lose your upload key, Google can still recover your app (they hold the app signing key). With APK-based signing, losing your key = you can never update the app.

**Opt-in to Play App Signing:** Strongly recommended. Go to Play Console → Setup → App Signing.

---

## 3. Dynamic Feature Modules (DFMs)

### 3.1 What They Are

DFMs let you split features into separate modules that are **not installed at app install time** but can be downloaded later, reducing the initial install size.

**When to use:**
- Feature is large (>5MB of code + assets)
- Feature is rarely used by most users (AR mode, QR scanner, video editor, map layer)
- Feature has device-specific requirements (AR requires ARCore)
- Game levels, additional content, premium features

### 3.2 Delivery Types

| Type | When installed | Use case |
|---|---|---|
| **On-demand** | User triggers download | AR feature, heavy editor, optional tool |
| **Conditional** | Automatically at install if conditions met | AR module only on ARCore-capable devices |
| **Install-time** | Always at install (like base module) | Mandatory feature that benefits from separate update cycle |
| **Instant** | No install required, runs from Play instantly | Instant app / Try Now in Play Store |

### 3.3 Creating a DFM

```kotlin
// 1. Create new module in Android Studio: File → New → New Module → Dynamic Feature Module

// 2. DFM build.gradle.kts
plugins { id("com.android.dynamic-feature") }
android { /* same as regular module */ }
dependencies { implementation(project(":app")) }  // depends on base app, not other features

// 3. DFM AndroidManifest.xml
<manifest xmlns:dist="http://schemas.android.com/apk/distribution">
    <dist:module dist:instant="false" dist:title="@string/title_ar_feature">
        <dist:delivery>
            <dist:on-demand/>   <!-- download when user requests it -->
        </dist:delivery>
        <dist:fusing dist:include="true"/>  <!-- include in pre-Lollipop fallback APK -->
    </dist:module>
</manifest>

// 4. Conditional delivery (ARCore example)
<dist:module>
    <dist:delivery>
        <dist:install-time>
            <dist:conditions>
                <dist:device-feature dist:name="android.hardware.camera.ar"/>
                <dist:min-sdk dist:value="24"/>
                <dist:user-countries dist:exclude="false">
                    <dist:country dist:code="US"/>
                    <dist:country dist:code="GB"/>
                </dist:user-countries>
            </dist:conditions>
        </dist:install-time>
    </dist:delivery>
</dist:module>
```

### 3.4 Requesting On-Demand DFM Download

```kotlin
// gradle (base app module)
implementation("com.google.android.play:feature-delivery:2.1.0")
implementation("com.google.android.play:feature-delivery-ktx:2.1.0")

class ArFeatureLauncher @Inject constructor(private val context: Context) {

    private val splitInstallManager = SplitInstallManagerFactory.create(context)

    fun downloadArModule(
        onDownloading: (progress: Int) -> Unit,
        onInstalled: () -> Unit,
        onFailed: (error: Int) -> Unit
    ) {
        // Check if already installed
        if (splitInstallManager.installedModules.contains("feature_ar")) {
            onInstalled()
            return
        }

        val request = SplitInstallRequest.newBuilder()
            .addModule("feature_ar")
            .build()

        val listener = SplitInstallStateUpdatedListener { state ->
            when (state.status()) {
                SplitInstallSessionStatus.DOWNLOADING -> {
                    val progress = (state.bytesDownloaded() * 100 / state.totalBytesToDownload()).toInt()
                    onDownloading(progress)
                }
                SplitInstallSessionStatus.INSTALLED -> onInstalled()
                SplitInstallSessionStatus.FAILED -> onFailed(state.errorCode())
                SplitInstallSessionStatus.REQUIRES_USER_CONFIRMATION -> {
                    // Large download requires Play Store confirmation dialog
                    splitInstallManager.startConfirmationDialogForResult(state, activity, REQUEST_CODE)
                }
            }
        }

        splitInstallManager.registerListener(listener)
        splitInstallManager.startInstall(request)
    }

    // Important: DFM code accessed via reflection or navigation contract
    // The base app can't directly import DFM classes (reversed dependency)
    fun launchArScreen(navController: NavController) {
        navController.navigate("ar_screen_route")  // defined in DFM nav graph
    }
}
```

### 3.5 DFM Navigation

Since the base app can't import DFM classes (reversed dependency), navigation uses route strings:

```kotlin
// In base app: navigate by route string
navController.navigate("ar://start")

// In DFM: registers its own navigation graph
// feature_ar/src/main/res/navigation/ar_nav.xml
<navigation app:startDestination="@id/arFragment">
    <deepLink app:uri="ar://start"/>  <!-- matches route from base app -->
</navigation>

// OR: Use a shared :feature:ar:api module with interface
// :feature:ar:api exposes ArFeatureLauncher interface
// :feature:ar implements it
// :app depends on :feature:ar:api only
```

---

## 4. R8 — Code Shrinking, Obfuscation & Optimization

R8 is the default compiler in AGP 3.4+. It replaces ProGuard and combines shrinking, obfuscation, and optimization in a single pass — faster and produces smaller DEX.

### 4.1 Enabling R8

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true       // enables R8 shrinking + obfuscation
            isShrinkResources = true     // removes unused resources (requires minifyEnabled)
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),  // use -optimize, not default
                "proguard-rules.pro"
            )
        }
        debug {
            isMinifyEnabled = false      // fast builds in debug
        }
    }
}
```

### 4.2 What R8 Does

```
1. TREE SHAKING   → removes unused classes, methods, fields
   Example: you import Gson but only use JsonParser → other Gson classes removed

2. OBFUSCATION    → renames identifiers to short names
   UserRepository → a.b.c; fetchUser() → a()
   Reduces DEX size (shorter names = fewer bytes in string pool)

3. OPTIMIZATION   → rewrites code for efficiency
   Method inlining: replaces method call with method body (eliminates call overhead)
   Class merging: combines small related classes into one
   Dead code removal: removes unreachable branches

4. RESOURCE SHRINKING (when isShrinkResources = true)
   → removes unused drawables, layouts, strings
   → must run after code shrinking (needs to know which resources code references)
```

**R8 Full Mode (AGP 8.0+):** More aggressive optimizations (class hierarchies, interface removal). Enable with:
```kotlin
// gradle.properties
android.enableR8.fullMode=true
```
Disney+ and Reddit reported 15–40% performance improvements after enabling full mode. Test thoroughly — some reflection-based code needs explicit keep rules.

### 4.3 Essential ProGuard Rules

```proguard
# proguard-rules.pro

# Keep data classes used for serialization
# (Kotlin Serialization and Moshi codegen handle this automatically)
# If using Gson: add @SerializedName and this rule:
-keepclassmembers class com.example.model.** { *; }

# Keep enum values (used in TypeConverters for Room)
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# Keep Parcelable CREATOR
-keepclassmembers class * implements android.os.Parcelable {
    static ** CREATOR;
}

# Keep custom Application class
-keep class com.example.App { *; }

# Don't warn about missing optional dependencies
-dontwarn okio.**
-dontwarn javax.annotation.**

# Keep crash reporter stack traces readable
# (Don't obfuscate exception names — Crashlytics deobfuscates via mapping file anyway)
-keepattributes SourceFile, LineNumberTable
-renamesourcefileattribute SourceFile

# R8 Full Mode: keep annotations used at runtime
-keepattributes *Annotation*
-keepattributes Signature

# NEVER add this — it disables R8 for your entire app:
# -keep class ** { *; }
```

### 4.4 Debugging R8 Issues

```bash
# Generate a report of what R8 kept/removed
# Add to proguard-rules.pro:
-printusage build/outputs/mapping/release/usage.txt    # removed code
-printseeds build/outputs/mapping/release/seeds.txt    # kept code entry points

# Check why a class was kept
# In Android Studio: Build → Analyze APK → Compare APK

# Use R8 configuration analyzer (AGP 9+):
./gradlew :app:assembleRelease --info | grep "R8 configuration"

# dependencyInsight: trace where a dependency comes from
./gradlew :app:dependencyInsight --configuration releaseRuntimeClasspath --dependency com.google.gson:gson
```

### 4.5 Resource Shrinking

```kotlin
// Exclude specific resources from shrinking (if dynamically referenced)
// res/raw/keep.xml
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/l_used*_c,@layout/l_used_a,@layout/l_used_b*"
    tools:discard="@layout/unused2" />

// Problem: resources.getIdentifier("img_" + id, "drawable", packageName)
// R8 doesn't know this dynamically references resources → may shrink them
// Fix: use tools:keep on the resource, or switch to explicit references
```

---

## 5. Image Optimization

### 5.1 WebP Conversion

WebP provides 25–35% smaller file size vs PNG with lossless compression, and 25–34% smaller than JPEG at equivalent quality.

```
Android Studio: Right-click PNG in Project view → Convert to WebP
- Lossy WebP: use for photos (equivalent to JPEG)
- Lossless WebP: use for icons/UI with transparency (replaces PNG)
- Minimum Android: 4.0 (API 14) for lossy; 4.3 (API 18) for lossless

Impact: 10–30MB saved in image-heavy apps
```

### 5.2 Vector Drawables

Replace density-specific PNGs with a single vector file:

```xml
<!-- One vector file replaces 5 PNG densities (mdpi, hdpi, xhdpi, xxhdpi, xxxhdpi) -->
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="#FF000000"
        android:pathData="M12,2L2,7l10,5 10-5-10-5z"/>
</vector>

<!-- For API < 21 support, use VectorDrawableCompat -->
app:srcCompat="@drawable/ic_home"  <!-- not android:src -->
```

**Rule:** Use vectors for all icons and simple illustrations. Use WebP for photos and complex images.

### 5.3 Lottie for Animations

Replace large animated GIF or multi-frame PNG sequences with lightweight JSON animation files:

```kotlin
implementation("com.airbnb.android:lottie:6.4.0")

// In layout
<com.airbnb.lottie.LottieAnimationView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:lottie_rawRes="@raw/loading_animation"  // ~20KB vs 500KB animated GIF
    app:lottie_autoPlay="true"
    app:lottie_loop="true"/>
```

---

## 6. ABI & Resource Splitting

### 6.1 ABI Filters (Native Libraries)

Native `.so` files for multiple ABIs can be 30–50% of total app size.

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        ndk {
            // arm64-v8a covers ~95% of active Android devices (2024)
            // x86_64 covers emulators
            // armeabi-v7a covers older 32-bit devices
            abiFilters += listOf("arm64-v8a", "x86_64")
            // Dropping armeabi-v7a and x86 saves 5–20MB depending on native code
        }
    }
}
// Note: With AAB, Play Store does this automatically per device — you don't need abiFilters
// Only needed for APK builds (sideloading, enterprise, testing)
```

### 6.2 Language Resource Configuration

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        // Only include resources for languages your app supports
        resourceConfigurations += listOf("en", "es", "fr", "de", "ja", "zh-rCN")
        // Removes string resources for all other languages (including from libraries)
    }
}
```

---

## 7. APK Analyzer

Built into Android Studio: **Build → Analyze APK**

```
What to look for:
├── classes.dex — which libraries are biggest? (use dex method count)
├── lib/ — native libraries (often 30-50% of total)
├── res/ — unused densities? Large images?
├── assets/ — bundled fonts, JSON, models taking space?
└── META-INF/ — unusually large? (may contain duplicate signatures)

Useful commands:
# Compare two APKs (before/after optimization)
./gradlew :app:assembleRelease
# Then: Analyze APK → File → Compare with previous APK

# Check DEX method count (64K limit)
# APK Analyzer shows method count per package
# If near 64K: check for duplicated library versions

# bundletool — analyze AAB as if Play delivered it
bundletool build-apks --bundle=app-release.aab --output=app.apks
bundletool get-size total --apks=app.apks
```

---

## 8. Play Asset Delivery (PAD)

For game-level large assets (textures, audio, video) that would exceed the 150MB limit:

```kotlin
// Replaces OBB (APK Expansion Files)
// Three delivery modes matching DFM:
// install-time: downloaded with app
// fast-follow: downloaded right after install, in background
// on-demand: downloaded when requested

// Asset pack manifest
// src/main/assets/pack_metadata/level_1_textures.json
{
  "delivery": "on-demand"
}

// Usage
val assetPackManager = AssetPackManagerFactory.getInstance(context)
val location = assetPackManager.getPackLocation("level_1_textures")
val assetPath = location?.assetsPath()  // null if not yet downloaded
```

---

## 9. Distribution Channels

### 9.1 Google Play Store (Primary)

```
Internal Testing → Alpha → Beta → Production (gradual rollout)

Tracks:
- Internal testing: up to 100 testers, instant publish
- Closed testing (Alpha): defined list of testers
- Open testing (Beta): any user can opt-in
- Production: staged rollout (1% → 5% → 20% → 50% → 100%)

Staged rollout allows:
- Monitoring crash rate per release
- Halting rollout if metrics degrade
- Gradual exposure to find device-specific issues
```

### 9.2 Firebase App Distribution

For pre-release builds to QA and stakeholders:

```kotlin
// Distribute via Gradle plugin
plugins { id("com.google.firebase.appdistribution") }

android {
    buildTypes {
        debug {
            firebaseAppDistribution {
                releaseNotes = "Fixed login bug, new onboarding flow"
                testers = "qa@company.com,pm@company.com"
                // or groups = "qa-team,internal-stakeholders"
            }
        }
    }
}

// Distribute via CI
./gradlew assembleDebug appDistributionUploadDebug
```

### 9.3 Enterprise / MDM Distribution

For B2B apps not on Play Store:

```kotlin
// Private Google Play channel (Managed Google Play)
// OR: APK sideloading via MDM (Mobile Device Management)
// Note: Side-loaded APKs bypass Play Store security scanning

// For enterprise: separate signing key, packageId suffix
applicationId = "com.example.app.enterprise"
signingConfig = signingConfigs.getByName("enterprise")
```

### 9.4 In-App Updates

Prompt users to update without leaving the app:

```kotlin
implementation("com.google.android.play:app-update-ktx:2.1.0")

val appUpdateManager = AppUpdateManagerFactory.create(context)
val appUpdateInfoTask = appUpdateManager.appUpdateInfo

appUpdateInfoTask.addOnSuccessListener { info ->
    if (info.updateAvailability() == UpdateAvailability.UPDATE_AVAILABLE) {
        when {
            // Flexible: user can continue using app while update downloads
            info.isUpdateTypeAllowed(AppUpdateType.FLEXIBLE) -> {
                appUpdateManager.startUpdateFlowForResult(
                    info, AppUpdateType.FLEXIBLE, activity, UPDATE_REQUEST_CODE
                )
            }
            // Immediate: blocks the app until user updates (use for critical fixes)
            info.isUpdateTypeAllowed(AppUpdateType.IMMEDIATE) -> {
                appUpdateManager.startUpdateFlowForResult(
                    info, AppUpdateType.IMMEDIATE, activity, UPDATE_REQUEST_CODE
                )
            }
        }
    }
}
```

**Immediate vs Flexible:**
- **Flexible:** Downloads in background, user installs when convenient. Use for non-critical updates.
- **Immediate:** Blocks the entire app with a fullscreen overlay until update is complete. Use for critical security fixes or mandatory feature updates.

---

## 10. Size Budget & Monitoring

### 10.1 Setting a Size Budget

```kotlin
// app/build.gradle.kts — fail the build if app exceeds budget
android {
    buildTypes {
        release {
            // Use Android Gradle Plugin's size checks
        }
    }
}

// Or: custom task to check AAB size
tasks.register("checkAppSize") {
    doLast {
        val aabFile = file("build/outputs/bundle/release/app-release.aab")
        val sizeMb = aabFile.length() / (1024 * 1024)
        if (sizeMb > 50) {
            throw GradleException("App bundle size ${sizeMb}MB exceeds 50MB budget!")
        }
    }
}
tasks.named("bundleRelease").configure {
    finalizedBy("checkAppSize")
}
```

### 10.2 CI Size Monitoring

```bash
# In CI pipeline (GitHub Actions, etc.)
# 1. Build AAB
./gradlew bundleRelease

# 2. Generate device-specific APK size with bundletool
bundletool build-apks \
    --bundle=app/build/outputs/bundle/release/app-release.aab \
    --output=app-release.apks \
    --connected-device  # measure for currently connected device spec

# 3. Report size
bundletool get-size total --apks=app-release.apks

# 4. Compare with previous release and alert if delta > threshold
```

---

## 11. HLD — App Distribution Architecture

### "Design the release pipeline for an Android app with 10M users"

```
Build Phase (CI/CD):
  PR merged → CI builds debug APK (tests run)
  Release branch → CI builds release AAB + signs with upload key
  R8 mapping file → uploaded to Crashlytics automatically
  Size check → fail if bundle > 50MB budget

Distribution Phase:
  Release AAB → Google Play Console (app signing via Play)
  Debug APK → Firebase App Distribution (QA team, 100 testers)
  Hotfix APK → Directly to internal testing track (instant)

Rollout Phase:
  Internal → Alpha (known issues?) → Beta (opt-in users)
  → Production: 1% → 10% → 50% → 100%
  Monitoring at each stage: crash rate, ANR rate, rating
  Halt criteria: crash-free rate drops > 0.5% vs previous version

Dynamic Content:
  Large assets (>5MB) → Dynamic Feature Modules or Play Asset Delivery
  Feature code → DFMs for optional features (AR, video editor)
  Config → Firebase Remote Config (no release needed)
  Critical fixes → In-App Updates (IMMEDIATE type for security)
```

---

## 12. React Native Distribution

```
React Native specific:
  - Metro bundler produces JS bundle (hermes bytecode in production)
  - Native code compiled per platform

App size reduction for RN:
  1. Enable Hermes — compiled bytecode is smaller than JS source
  2. Separate per-ABI APKs (in build.gradle: splits { abi { enable true } })
  3. Enable ProGuard for Java/Kotlin native layer
  4. Lazy imports for heavy screens: const HeavyScreen = React.lazy(() => import('./HeavyScreen'))
  5. Optimize Metro bundle: purge unused node_modules

OTA updates (CodePush / Expo Updates):
  // Update JS bundle without Play Store review
  // Only JS/assets can be updated — native code changes require a store release
  import codePush from 'react-native-code-push'
  const App = codePush({ checkFrequency: codePush.CheckFrequency.ON_APP_RESUME })(AppRoot)
  // Limitation: cannot update anything that touches native modules
```

---

## 13. Common Misunderstandings & Pitfalls

**❌ Uploading APK to Play Store when AAB is available**
APK-based releases don't get Dynamic Delivery — users download the entire app including unused ABIs and densities. Switch to AAB immediately. It's been mandatory for new apps since August 2021.

**❌ Using `proguard-android.txt` instead of `proguard-android-optimize.txt`**
The non-optimize file disables R8's code optimization passes, leaving significant size and performance gains on the table. Always use `proguard-android-optimize.txt` as your base.

**❌ Adding overly broad keep rules like `-keep class ** { *; }`**
This tells R8 to keep everything — you lose all shrinking benefits. Only keep what actually needs to be preserved (classes accessed via reflection, serialization, JNI). Broad keep rules are a sign of cargo-culting ProGuard config.

**❌ Forgetting `shrinkResources = true`**
Enabling `minifyEnabled` without `shrinkResources` only shrinks code, not unused resources (images, strings, layouts). Both must be true for maximum size reduction.

**❌ DFM navigation using direct class imports from base app**
The dependency direction reverses in DFMs — the feature depends on app, not the other way around. Importing DFM classes in the base app creates a circular dependency. Use navigation route strings or a shared `:feature:xyz:api` interface module.

**❌ Not testing DFM download on slow networks**
DFM downloads fail silently on slow or metered connections if you don't handle `REQUIRES_USER_CONFIRMATION` and `FAILED` states. Always show a progress UI and handle the full `SplitInstallSessionStatus` state machine.

**❌ No size budget in CI**
App size grows invisibly — one dependency upgrade adds 3MB, a font is bundled unnecessarily, a PNG isn't converted to WebP. Without automated size checks in CI, you discover the problem when users complain. Add a size budget check to every release build.

**❌ Using `AppUpdateType.IMMEDIATE` for every update**
Immediate updates block the entire app — terrible UX if used for minor updates. Reserve `IMMEDIATE` for critical security patches or mandatory breaking changes. Use `FLEXIBLE` for regular feature updates.

---

## 14. Best Practices

- **AAB for all Play Store releases** — required since Aug 2021; use APK only for sideloading/testing
- **Play App Signing opt-in** — Google holds the signing key; losing your upload key is recoverable
- **R8 + proguard-android-optimize.txt** — always use the optimize variant for maximum benefit
- **`isMinifyEnabled = true` + `isShrinkResources = true`** — both in release, never in debug
- **Size budget in CI** — catch size regressions before they reach users
- **Convert PNGs to WebP** — 25–35% size reduction, zero quality loss for UI assets
- **Vector drawables for icons** — one file replaces 5 density variants
- **DFMs for features > 5MB that aren't used by all users** — AR, video editor, maps, scanner
- **In-app updates for critical fixes** — `IMMEDIATE` type for security patches, `FLEXIBLE` for features
- **Firebase App Distribution for pre-release** — fast QA distribution without Play Store review
- **Save R8 mapping.txt per release** — required to deobfuscate production crash stacktraces
- **Staged rollout: always** — never ship to 100% without monitoring period at 1–5%
- **`dependencyInsight` to audit transitive dependencies** — find unexpected library weight
- **CDN for large assets** — move large media, fonts, ML models to CDN; download on first use

---

## 15. Interview Q&A

**Q1: What's the difference between APK and AAB, and why does it matter?**

> An APK is a self-contained installable file containing everything — all CPU architectures, all screen densities, all language strings. A user on a Pixel 9 downloads the x86 libraries and the ldpi drawables even though they'll never use them. An AAB is a publishing format submitted to Google Play; Play generates optimized, device-specific split APKs from it — delivering only the code and resources the specific device needs. The result is typically 15–35% smaller downloads. AAB also unlocks Dynamic Feature Modules which let you defer optional features until the user needs them. It's been mandatory for all new Play Store apps since August 2021.

---

**Q2: How would you reduce an 80MB app to under 20MB?**

> In priority order: First, switch from APK to AAB — immediate 15–25% reduction from device-specific delivery, zero code changes. Second, enable R8 with `proguard-android-optimize.txt`, `isMinifyEnabled = true`, and `isShrinkResources = true` — typically removes 20–40% of code and unused resources. Third, convert PNGs to WebP — 25–35% image size reduction. Fourth, replace icon PNGs with vector drawables — one file per icon instead of 5 densities. Fifth, audit native libraries with APK Analyzer — `lib/` is often 30–50% of total; ABI filters (`arm64-v8a` only) or ensuring AAB is used cuts this significantly. Sixth, move large rarely-used features to Dynamic Feature Modules — defer download until needed. Finally, move large assets (fonts, ML models, audio) to CDN with on-demand download.

---

**Q3: When would you use Dynamic Feature Modules?**

> When a feature is large enough (>5MB) that not all users will use it, and it's not needed at first launch. Examples: an AR try-on feature in a shopping app (needs ARCore, not all devices support it, most users never use it), a video editor (heavy FFmpeg native code, only power users need it), premium features gated behind subscription (download only on purchase), additional game levels. The tradeoff: DFMs add complexity — the base app can't directly import DFM code (reversed dependency), navigation requires route strings or shared API modules, and you must handle the full download state machine including failures and user confirmation for large downloads. Worth it when the feature is genuinely optional and large.

---

**Q4: What does R8 do and how is it different from ProGuard?**

> R8 does four things: tree shaking (removes unused classes and methods), obfuscation (renames identifiers to short names like `a`, `b`, `c`), optimization (method inlining, class merging, dead code removal), and resource shrinking (removes unused drawables and strings). ProGuard did the first three but was a separate tool from the dexer. R8 combines shrinking, obfuscation, optimization, and dexing into a single compilation pass — faster builds and often better optimization because it can make decisions across the full program. R8 in full mode (enabled via `android.enableR8.fullMode=true`) goes further with whole-program optimizations. Reddit saw 40% size reduction after enabling full mode. The main operational difference: ProGuard is effectively legacy in modern AGP; R8 is the default and you configure it with the same ProGuard rule syntax.

---

**Q5: How does app signing work differently with AAB vs APK?**

> With APK, you sign the file yourself and upload the signed APK. With AAB, you upload to Google Play and opt into Play App Signing. Google generates a new app signing key which they store securely, and you retain an upload key to authenticate your uploads. When Play generates split APKs for each device, they sign them with the Play-held app signing key. This has an important benefit: if you lose your upload key, Google can rekey you with a new upload key while the app signing key remains the same — users don't notice any difference. With the old APK model, losing your signing key meant you could never publish an update to that app again. The downside: you're trusting Google with your signing key, but given their security infrastructure, this is the right trade-off for almost every team.

---

**Q6: How do you handle in-app updates and what's the difference between flexible and immediate?**

> In-app updates via the Play Core library let you prompt users to update without leaving the app. Flexible update: the new version downloads in the background while the user continues using the app; you show a snackbar asking them to restart when it's ready. Immediate update: shows a fullscreen overlay that blocks the app entirely until the update is downloaded and installed; the user can't use the app during this time. I use flexible for regular feature updates — low friction, users appreciate that they can keep working. I use immediate for critical security fixes or when an old version of the app will break due to backend API changes — situations where using the old version is worse than being blocked temporarily. I also use Remote Config's minimum version check as a complementary mechanism to force-update extremely old versions that can't receive in-app update prompts.

---

*Previous: [11 — Feature Flags, A/B Testing & Remote Config](./11-feature-flags.md)*
*Next: [13 — Cross-Cutting Concerns](./13-cross-cutting.md)*
