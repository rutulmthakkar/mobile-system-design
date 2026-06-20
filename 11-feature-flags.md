# 11 — Feature Flags, A/B Testing & Remote Config

> **Round type: HLD (primary) + Both**
> **HLD:** Designing a feature flag and experimentation system for a mobile app
> **LLD:** Firebase Remote Config setup, flag abstraction layer, experiment tracking, targeting rules

---

## 1. Why This Matters in Interviews

Feature flags are how modern mobile teams ship safely. They separate **deployment** (code goes to production) from **release** (users see the feature). Interviewers at FAANG-level companies ask about this because it reflects production maturity — do you think about gradual rollouts, kill switches, and data-driven decisions, or do you just ship to everyone and hope for the best?

**Common interview angles:**
- "How would you safely roll out a new checkout flow to 100M users?"
- "What's the difference between a feature flag and A/B testing?"
- "How do you handle the case where a new feature causes a crash spike?"
- "How do you design a remote config system that doesn't break the app when offline?"
- "What is a kill switch and when would you use it?"
- "How do you ensure A/B test results are statistically valid?"

---

## 2. Core Concepts

### 2.1 Feature Flag Types

| Type | Purpose | Lifetime | Example |
|---|---|---|---|
| **Release flag** | Gradual rollout of new feature | Short-lived (weeks) | `new_checkout_enabled` |
| **Kill switch** | Emergency disable without hotfix | Permanent | `payment_fallback_enabled` |
| **Experiment flag** | A/B test to measure impact | Short-lived (experiment duration) | `checkout_cta_variant` |
| **Permission flag** | Gate feature by user tier/segment | Long-lived | `premium_analytics_enabled` |
| **Ops flag** | Control operational behavior | Variable | `max_retry_count`, `api_timeout_ms` |

### 2.2 The Core Value: Decouple Deployment from Release

```
Without feature flags:
  Code merge → Deploy → ALL users see new feature immediately
  Bug found → Emergency hotfix → Wait 24-72h for Play Store review

With feature flags:
  Code merge → Deploy (flag OFF) → No user impact
  Ready to ship → Turn flag ON for 1% → Monitor → 10% → 50% → 100%
  Bug found → Turn flag OFF instantly → Back to 0%, zero app update needed
```

This is the most important concept. Feature flags let you **ship code on your schedule** and **release features on the product team's schedule** — independently.

---

## 3. Firebase Remote Config

### 3.1 Setup

```kotlin
// gradle
implementation(platform("com.google.firebase:firebase-bom:33.7.0"))
implementation("com.google.firebase:firebase-config-ktx")

// Default values XML — CRITICAL: app must work without network using these
// res/xml/remote_config_defaults.xml
<?xml version="1.0" encoding="utf-8"?>
<defaultsMap>
    <entry><key>feature_new_checkout</key><value>false</value></entry>
    <entry><key>max_feed_items</key><value>20</value></entry>
    <entry><key>api_timeout_ms</key><value>30000</value></entry>
    <entry><key>checkout_cta_text</key><value>Place Order</value></entry>
    <entry><key>force_update_min_version</key><value>0</value></entry>
</defaultsMap>
```

### 3.2 Repository Pattern

```kotlin
@Singleton
class RemoteConfigRepository @Inject constructor() {

    private val remoteConfig = Firebase.remoteConfig.apply {
        setDefaultsAsync(R.xml.remote_config_defaults)
        setConfigSettingsAsync(remoteConfigSettings {
            minimumFetchIntervalInSeconds = if (BuildConfig.DEBUG) 0L else 3600L
            // Debug: fetch every time. Production: max once per hour.
            // Default throttle is 12h — setting < 1h risks quota exhaustion
        })
    }

    // Fetch + activate on app start
    suspend fun initialize() {
        try {
            remoteConfig.fetchAndActivate().await()
        } catch (e: Exception) {
            // Failed to fetch — defaults are already active, app continues normally
            Timber.w(e, "Remote config fetch failed, using defaults")
        }
    }

    // Real-time listener (Firebase Remote Config v21.3+)
    // Server pushes updates without app restart
    val configUpdates: Flow<Set<String>> = callbackFlow {
        val listener = remoteConfig.addOnConfigUpdateListener(
            object : ConfigUpdateListener {
                override fun onUpdate(configUpdate: ConfigUpdate) {
                    remoteConfig.activate().addOnCompleteListener {
                        trySend(configUpdate.updatedKeys)
                    }
                }
                override fun onError(error: FirebaseRemoteConfigException) {
                    Timber.e(error, "Remote config update error")
                }
            }
        )
        awaitClose { listener.remove() }
    }

    // Typed accessors — never read raw strings at call sites
    fun isNewCheckoutEnabled(): Boolean = remoteConfig.getBoolean("feature_new_checkout")
    fun getMaxFeedItems(): Int = remoteConfig.getLong("max_feed_items").toInt()
    fun getApiTimeoutMs(): Long = remoteConfig.getLong("api_timeout_ms")
    fun getCheckoutCtaText(): String = remoteConfig.getString("checkout_cta_text")
    fun getForceUpdateMinVersion(): Int = remoteConfig.getLong("force_update_min_version").toInt()
}
```

### 3.3 Feature Flag Abstraction Layer ✅ Critical Pattern

Never call `remoteConfig.getBoolean("flag_name")` directly at use sites. Always abstract behind an interface:

```kotlin
// Interface — makes testing trivial and decouples from Firebase
interface FeatureFlags {
    fun isNewCheckoutEnabled(): Boolean
    fun isNewFeedAlgorithmEnabled(): Boolean
    fun getCheckoutVariant(): CheckoutVariant
    fun getMaxRetryCount(): Int
}

enum class CheckoutVariant { CONTROL, VARIANT_A, VARIANT_B }

// Production: reads from Remote Config
@Singleton
class RemoteFeatureFlags @Inject constructor(
    private val remoteConfig: RemoteConfigRepository
) : FeatureFlags {
    override fun isNewCheckoutEnabled() = remoteConfig.getBoolean("feature_new_checkout")
    override fun isNewFeedAlgorithmEnabled() = remoteConfig.getBoolean("feature_new_feed")
    override fun getCheckoutVariant() = when (remoteConfig.getString("checkout_variant")) {
        "variant_a" -> CheckoutVariant.VARIANT_A
        "variant_b" -> CheckoutVariant.VARIANT_B
        else -> CheckoutVariant.CONTROL
    }
    override fun getMaxRetryCount() = remoteConfig.getLong("max_retry_count").toInt()
}

// Test: override any flag for testing/QA
class TestFeatureFlags(
    private val overrides: Map<String, Any> = emptyMap()
) : FeatureFlags {
    override fun isNewCheckoutEnabled() = overrides["new_checkout"] as? Boolean ?: false
    override fun isNewFeedAlgorithmEnabled() = false
    override fun getCheckoutVariant() = CheckoutVariant.CONTROL
    override fun getMaxRetryCount() = 3
}

// Usage in ViewModel — depends on interface, not Firebase
@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val featureFlags: FeatureFlags,
    private val checkoutRepo: CheckoutRepository
) : ViewModel() {
    val useNewCheckout = featureFlags.isNewCheckoutEnabled()
}
```

**Why this matters:**
- Unit tests use `TestFeatureFlags` — no Firebase dependency, instant, deterministic
- QA team can override flags without changing code
- Swap Firebase for LaunchDarkly/Unleash without changing every call site
- Flag keys are defined in one place — no scattered string literals

---

## 4. Rollout Strategies

### 4.1 Staged Percentage Rollout

```
Day 0:   Internal team only (employees)
Day 1:   1% of users — watch crash rate, ANR rate, conversion
Day 3:   5%  — trending stable → expand
Day 5:   20% — monitor for 48h
Day 7:   50% — if metrics healthy, accelerate
Day 9:   100% — full rollout
         → Schedule ticket to remove flag from code
```

**Firebase Console:** Parameters → Add condition → "User in random percentile" → 0–5%.

**Automated guardrails:**
```
Define success criteria BEFORE rollout:
  ✅ Crash-free rate ≥ 99.5% (same as control group)
  ✅ ANR rate ≤ 0.47%
  ✅ Checkout completion rate ≥ baseline
  ✅ P95 screen load time ≤ 3s

Automated rollback trigger:
  If crash-free rate drops > 0.5% vs control → auto-flip flag to OFF
  Send alert to on-call
```

### 4.2 Targeting Rules (User Segmentation)

Firebase Remote Config conditions let you target:

```
User segments you can target:
├── Percentage of users (random 10%)
├── App version (>= 2.5.0 only)
├── Platform (Android vs iOS)
├── Country/region (US users first)
├── Language (en-US only)
├── Firebase Analytics audience (power users, churned users)
├── Custom user properties (subscription_tier = "premium")
└── Device type (tablet vs phone)
```

**Priority order:** Firebase evaluates conditions top-to-bottom, first match wins.

```
Example condition stack:
1. Internal beta testers → new_checkout = true (force-enabled for QA)
2. Country = US AND user percentile < 10% → new_checkout = true (US early access)
3. App version < 2.0.0 → new_checkout = false (old app can't support it)
4. Default → new_checkout = false
```

### 4.3 Kill Switch Pattern

A kill switch is a permanent flag that disables a risky feature instantly without an app update:

```kotlin
// Kill switch for payment processing
class PaymentProcessor @Inject constructor(
    private val flags: FeatureFlags,
    private val primaryProcessor: StripeProcessor,
    private val fallbackProcessor: BraintreeProcessor
) {
    suspend fun processPayment(request: PaymentRequest): PaymentResult {
        return if (flags.isStripeEnabled()) {
            try {
                primaryProcessor.charge(request)
            } catch (e: Exception) {
                // Stripe failed AND kill switch allows fallback
                if (flags.isPaymentFallbackEnabled()) {
                    fallbackProcessor.charge(request)
                } else throw e
            }
        } else {
            // Kill switch: Stripe disabled → use Braintree directly
            fallbackProcessor.charge(request)
        }
    }
}
```

**Kill switch best practices:**
- Permanent in the codebase — never remove them even after rollout
- Default should always be the SAFE behavior (e.g., `is_new_feature_enabled = false`)
- Keep the kill switch handler path well-tested — it's the most critical path
- Give kill switches clear ownership (on-call team who can flip them at 3am)

---

## 5. A/B Testing

### 5.1 What A/B Testing Is

Show variant A to group 1, variant B to group 2 (both randomly assigned), measure which performs better on a defined metric, make a statistically valid decision.

```
Control group (50%): existing checkout flow  → measure: checkout completion rate
Variant A    (50%): new checkout flow         → measure: checkout completion rate

After 2 weeks: variant A shows +3.2% conversion
Statistical significance: p < 0.05 ✅
Decision: roll out variant A to 100%
```

### 5.2 Firebase A/B Testing Setup

Firebase A/B Testing is built on Remote Config — it automates the assignment and analysis.

```kotlin
// A/B test is set up in Firebase Console:
// Experiments → Create Experiment → Remote Config
// Parameter: checkout_variant (values: "control", "new_ui")
// Target: 50/50 split
// Goal metric: in_app_purchase (or custom conversion event)
// Duration: minimum 2 weeks for statistical significance

// In code — read the variant the same way as any flag
val variant = featureFlags.getCheckoutVariant()
when (variant) {
    CheckoutVariant.CONTROL  -> showOldCheckout()
    CheckoutVariant.VARIANT_A -> showNewCheckout()
    CheckoutVariant.VARIANT_B -> showNewCheckoutWithOneClick()
}

// Track exposure — tell analytics which variant the user saw
analytics.trackEvent("checkout_experiment_exposure") {
    param("variant", variant.name.lowercase())
    param("experiment_id", "checkout_v2_q1_2025")
}

// Set user property for segmentation in Crashlytics / Analytics
FirebaseCrashlytics.getInstance().setCustomKey("checkout_variant", variant.name)
Firebase.analytics.setUserProperty("checkout_experiment", variant.name.lowercase())
```

### 5.3 A/B Test Design Principles

**Sample size and duration:**
- Run the experiment for **at least 2 weeks** to account for day-of-week effects (behavior differs Mon vs Sun)
- Calculate required sample size before starting: use a power calculator (`n = 2 * (z_α/2 + z_β)² * σ² / δ²`)
- Never stop early because "it looks significant" — peeking inflates false positive rate

**Statistical significance:**
- `p < 0.05` is the standard threshold — 5% chance the result is due to random chance
- Firebase A/B Testing reports significance automatically
- **Practical significance** also matters: a 0.1% improvement that's statistically significant may not be worth shipping

**One change at a time:**
- Don't test two UI changes simultaneously in the same experiment — you can't attribute the result
- Exception: factorial design (2x2 experiments) with proper analysis

**Guardrail metrics:**
- Define metrics that must NOT degrade even if primary metric improves
- Example: testing a new checkout UI — primary metric is conversion, guardrail is crash rate and payment error rate
- If guardrail degrades, the variant fails even if conversion improves

### 5.4 Common A/B Testing Mistakes

```
❌ Running test for 3 days then declaring winner
   → Day-of-week effects, insufficient sample size

❌ Testing on weekday traffic only
   → Weekend users behave differently

❌ Not pre-registering your hypothesis and metric
   → HARKing (Hypothesizing After Results are Known) leads to p-hacking

❌ Same user seeing different variants on different sessions
   → Assignment must be sticky (same user always gets same variant)
   → Firebase handles this via user ID hashing

❌ Leakage between groups
   → If variant and control share data (e.g., social feed), behavior bleeds between groups

❌ Testing everything simultaneously
   → Running 20 experiments at once means many will interfere with each other
```

---

## 6. Remote Config for Operational Control

Beyond feature flags, Remote Config is invaluable for operational parameters you might need to tune in production:

```kotlin
// Dynamic operational parameters
object RemoteConfigKeys {
    const val API_TIMEOUT_MS = "api_timeout_ms"             // tune without hotfix
    const val MAX_RETRY_COUNT = "max_retry_count"           // adjust retry behavior
    const val FEED_PAGE_SIZE = "feed_page_size"             // pagination tuning
    const val VIDEO_BUFFER_SIZE_KB = "video_buffer_size_kb" // streaming performance
    const val RATE_LIMIT_REQUESTS_PER_MIN = "rate_limit_rpm"
    const val MIN_SUPPORTED_APP_VERSION = "min_supported_version"  // force update
    const val MAINTENANCE_MODE = "maintenance_mode"         // emergency maintenance
    const val IMAGE_QUALITY_PERCENT = "image_quality_percent" // bandwidth tuning
}

// Force update check
fun checkForceUpdate(context: Context) {
    val minVersion = featureFlags.getMinSupportedVersion()
    val currentVersion = BuildConfig.VERSION_CODE
    if (currentVersion < minVersion) {
        showForceUpdateDialog(context)
        // Don't dismiss — user must update to continue
    }
}

// Maintenance mode
fun checkMaintenanceMode() {
    if (featureFlags.isMaintenanceMode()) {
        _uiState.update { it.copy(showMaintenanceBanner = true) }
        // Optionally block all API calls during maintenance window
    }
}
```

---

## 7. Offline Behavior & Caching

Remote Config caches the last fetched values locally. The app always has values — from cache or defaults — even without network:

```
First launch (no network):
  → Default values from res/xml/remote_config_defaults.xml used
  → App works normally with conservative defaults

First launch (network available):
  → Fetch from server, cache locally
  → Activate (apply) the fetched values
  → App uses live config

Subsequent launches:
  → Activate cached values IMMEDIATELY (no delay)
  → Fetch fresh values in background
  → Next launch uses freshly fetched values
  (or use real-time updates to apply immediately)
```

**Activation timing matters:**
```kotlin
// Option A: Fetch-then-activate (values apply on NEXT launch)
// Safe but values may be stale for current session
remoteConfig.fetch().await()
// Don't activate yet — will activate on next launch

// Option B: fetchAndActivate (values apply THIS session)
// May cause mid-session UI changes if user is active
remoteConfig.fetchAndActivate().await()

// Option C: Activate cached first, fetch in background ✅ Best for most cases
remoteConfig.activate().await()  // instant, no network
scope.launch { remoteConfig.fetch().await() }  // background refresh for next time
```

---

## 8. Tools Beyond Firebase Remote Config

| Tool | Best for | Pricing |
|---|---|---|
| **Firebase Remote Config** | Mobile-first, Firebase ecosystem, simple use cases | Free |
| **LaunchDarkly** | Enterprise, complex targeting, real-time flags, feature management at scale | Paid |
| **Unleash** | Open source, self-hosted, no vendor lock-in | Free (self-hosted) |
| **Split.io** | Experimentation-focused teams, built-in stats engine | Paid |
| **Flagsmith** | Open source alternative to LaunchDarkly | Free + paid cloud |
| **Statsig** | Product analytics + experimentation combined | Paid |

**Firebase Remote Config limitations:**
- 2,000 parameters max
- 500 conditions max
- No rollback history or change log in the UI
- Fetch throttle: default 12h (adjustable but quota-limited)
- No native flag ownership / TTL / cleanup tooling

For teams doing serious experimentation at scale, LaunchDarkly or Statsig provide better tooling for experiment analysis, flag lifecycle management, and team workflow.

---

## 9. HLD — Feature Flag System Architecture

### "Design a feature flag system for a mobile app with 50M users"

```
Components:

1. Flag Management Service (backend)
   - Store flag definitions: name, type, default, targeting rules, owner
   - API: GET /flags (returns all flags for client), POST /flags/{id}/enable
   - Audit log: who changed what flag when
   - TTL enforcement: flag must be cleaned up within N days of full rollout

2. Client SDK (Android/iOS)
   - Fetch flags on app start (background, non-blocking)
   - Cache locally (DataStore) for offline use
   - Evaluate targeting rules locally for faster response
   - Send flag exposure events to analytics

3. Targeting Engine
   - Stable user assignment (same user always gets same variant)
   - Assignment based on: user_id hash % 100 for percentage rollouts
   - Segment rules: user properties, device, location, A/B group

4. Experiment Analysis
   - Collect metric data per variant via analytics
   - Run statistical tests (t-test, z-test for proportions)
   - Dashboard: real-time metrics per variant, significance indicators
   - Automated winner detection + notification

5. Safety Controls
   - Rate of change alerts (if flag causes metric spike, auto-notify)
   - Rollback in one click
   - Default values shipped in APK (never broken without network)

Data flow:
  App start → fetch flags (async) → cache → render with current flags
  Flag changes → real-time push (WebSocket or Firebase Real-time) or next fetch
  User actions → log events with variant tag → analysis pipeline
```

---

## 10. React Native

```typescript
// Firebase Remote Config in React Native
import remoteConfig from '@react-native-firebase/remote-config'

// Initialize with defaults
await remoteConfig().setDefaults({
    feature_new_checkout: false,
    max_feed_items: 20,
    checkout_cta_text: 'Place Order',
})

await remoteConfig().setConfigSettings({
    minimumFetchIntervalMillis: __DEV__ ? 0 : 3600000,  // 1h prod, instant dev
})

// Fetch and activate
await remoteConfig().fetchAndActivate()

// Read values
const isNewCheckout = remoteConfig().getBoolean('feature_new_checkout')
const ctaText = remoteConfig().getString('checkout_cta_text')

// Real-time listener
const unsubscribe = remoteConfig().onConfigUpdated(async () => {
    await remoteConfig().activate()
    // Re-render with new values
    setFlags(getCurrentFlags())
})

// Feature flag hook
function useFeatureFlag(key: string, defaultValue: boolean): boolean {
    const [value, setValue] = useState(
        () => remoteConfig().getBoolean(key) ?? defaultValue
    )
    useEffect(() => {
        const unsub = remoteConfig().onConfigUpdated(async () => {
            await remoteConfig().activate()
            setValue(remoteConfig().getBoolean(key))
        })
        return () => unsub()
    }, [key])
    return value
}
```

---

## 11. Common Misunderstandings & Pitfalls

**❌ No default values defined**
If the app fetches config for the first time and the network is unavailable, Firebase returns empty/zero values. Without defaults in `remote_config_defaults.xml`, `getBoolean()` returns `false` and `getLong()` returns `0` — potentially breaking the app. Always define safe conservative defaults.

**❌ Reading flags directly at call sites**
`remoteConfig.getBoolean("feature_new_checkout")` scattered across the codebase creates string literal duplication, makes testing impossible (can't inject fake values), and makes refactoring flags dangerous. Always use a typed abstraction layer.

**❌ Default value is the UNSAFE behavior**
`payment_new_processor_enabled = true` as default means the first time an old app version starts without network, it uses the new processor. Default should always be the safe/old behavior: `payment_new_processor_enabled = false`.

**❌ Stopping A/B tests too early**
A test that looks like a winner after 3 days is often not — it's the novelty effect (users engage with anything new initially) or it's statistically underpowered. Always define minimum duration (2 weeks) and sample size before starting.

**❌ Not cleaning up flags after full rollout**
Dead flags accumulate as tech debt. Code becomes cluttered with `if (flags.isXEnabled())` branches that are always true. Schedule a cleanup ticket the moment a flag hits 100% rollout. Remove the flag key from Firebase AND the conditional from code.

**❌ Fetching remote config synchronously on the main thread**
Blocking app start on a network call makes every launch slow. Fetch asynchronously in background, use cached values immediately, apply fresh values on next launch or via real-time listener.

**❌ No kill switch for critical features**
Any feature that touches payments, auth, or data integrity needs a kill switch. Discovering you need one after a production incident — when a store review takes 24h to fix it — is a painful lesson.

**❌ Same user seeing different variants across sessions**
Variant assignment must be sticky. If user 123 gets variant A on Monday and variant B on Tuesday, the experiment is contaminated. Firebase uses a hash of user ID + experiment name to ensure consistent assignment.

---

## 12. Best Practices

- **Always define defaults in XML** — app must work without ever reaching the network
- **Abstract behind a typed interface** — never call `remoteConfig.getString("key")` at use sites
- **Default = safe behavior** — feature OFF by default; turn ON during rollout
- **Staged rollout: 1% → 5% → 20% → 50% → 100%** — with 24–48h pause between stages
- **Define success AND guardrail metrics before starting A/B test** — not after
- **Minimum 2-week A/B test duration** — accounts for weekly behavioral cycles
- **Make flag assignment sticky** — same user always gets same variant (hash of user ID)
- **Kill switches for every critical path** — payments, auth, data integrity
- **Real-time config updates for kill switches** — don't rely on next fetch for emergencies
- **Set user property for variant** — enables per-variant analysis in Crashlytics, Analytics
- **Log exposure event** — when did the user actually SEE the variant, not just assignment
- **Clean up flags after 100% rollout** — schedule it; flag debt is real
- **Audit trail** — log who changed what flag and when (Firebase Console shows this)
- **Minimum fetch interval = 3600s in production** — avoid Firebase quota exhaustion from aggressive fetching

---

## 13. Interview Q&A

**Q1: What is the difference between a feature flag and A/B testing?**

> A feature flag is a switch that controls whether a feature is visible to users — primarily used for safe rollouts and kill switches. A/B testing is a controlled experiment where you show different variants to different user groups and measure which performs better on a defined metric. Feature flags are the mechanism; A/B testing is a use case of feature flags. You can use a feature flag for a simple on/off rollout with no statistical analysis. An A/B test uses a flag to assign users to variants, then ties those assignments to metric data to make a statistically informed decision. Many teams use the same technical infrastructure (Remote Config, LaunchDarkly) for both.

---

**Q2: How would you safely roll out a new checkout flow to 100M users?**

> Deploy the code with the flag off — no user impact. Define success criteria: checkout completion rate, crash-free rate, payment error rate. Start with internal users and 1% of production traffic. Monitor for 24 hours — watch crash rate, ANR rate, checkout conversion vs control group. If healthy, expand to 5%, then 20%, then 50%, then 100%, with 24–48h hold at each stage. At every stage, the control group is the other users — so I have a live comparison. If metrics degrade at any stage, I flip the flag off instantly — no hotfix, no Play Store wait. After full rollout, I schedule a ticket to remove the flag from Firebase console and the `if` statement from code. The entire rollout takes 1–2 weeks but carries almost zero risk.

---

**Q3: What is a kill switch and how is it different from a release flag?**

> A kill switch is a permanent remote toggle that disables a specific feature or code path instantly. A release flag is temporary — it controls the gradual rollout of a feature and gets removed from code after full rollout. A kill switch lives permanently in the codebase because the feature it protects is always at risk of needing emergency disablement. For example: a kill switch for a new payment processor means if the processor has an outage, the on-call engineer flips `stripe_enabled = false` in the Firebase Console and all users fall back to the legacy processor in seconds — no hotfix, no deploy, no Play Store review. The default value is always the safe/fallback behavior. Kill switches are most critical for: payments, auth, real-time features, and any third-party integration.

---

**Q4: How does Remote Config behave when the device is offline?**

> Remote Config caches the last successfully fetched values in local storage. On app start, it immediately activates the cached values — no network wait. Then it tries to fetch fresh values in the background. If fetch fails (offline), the cached values continue to be used. If there's never been a successful fetch (first install, no network), the app uses the default values defined in `remote_config_defaults.xml` bundled in the APK. This is why defaults are critical — define them conservatively so the app works correctly even with zero network access. The fetch throttle (default 12h, minimum 1h in production) means frequent relaunches don't all hit the network — they use cached values and one background fetch serves multiple launches.

---

**Q5: How do you make A/B test assignments sticky so the same user always sees the same variant?**

> Sticky assignment means: given the same user ID and experiment name, always produce the same variant. The standard approach is consistent hashing: `hash(userId + experimentId) % 100`. If this value falls in 0–49, the user gets variant A; 50–99, variant B. The hash function is deterministic — same inputs always produce same output — so the user always gets the same variant regardless of which server handles their request or which device they use. Firebase A/B Testing handles this automatically using the app instance ID. For custom implementations, I use MurmurHash3 (fast, good distribution). The key requirement: the user ID used for hashing must be stable (don't use session ID or device ID that changes on reinstall).

---

**Q6: What happens if you don't clean up feature flags after full rollout?**

> Technical debt accumulates quickly. The code becomes full of dead conditionals — `if (flags.isFeatureXEnabled())` branches where the flag is always true. Engineers lose context on which flags are active, which are experiments, and which are kill switches — they become scared to remove them. Firebase Console fills with hundreds of stale parameters. New engineers reading the code can't tell which branches are ever taken. Eventually, someone accidentally toggles an old flag thinking it's inactive and breaks production. The fix: enforce a flag TTL. When a flag hits 100% rollout, automatically create a cleanup ticket. The ticket closes when both the Firebase parameter is deleted AND the conditional is removed from code. Some teams run a linter that fails CI if a flag has been in the codebase more than 90 days without being cleaned up.

---

*Previous: [10 — Security](./10-security.md)*
*Next: [12 — App Distribution & Size](./12-app-distribution.md)*
