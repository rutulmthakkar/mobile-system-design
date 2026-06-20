# 30 — In-App Purchases & Subscriptions

> **Round type: Both (HLD + LLD)**
> **HLD:** Monetization architecture — entitlement management, server-side validation, subscription states
> **LLD:** Google Play Billing Library 7, product types, purchase flow, acknowledgment, restore purchases

---

## 1. Why This Matters

IAP is the revenue engine for most consumer apps. Interviewers at companies with consumer monetization (streaming, gaming, productivity) expect you to know the product types, the purchase lifecycle, why server-side validation is mandatory, and what happens to subscriptions when payment fails.

**⚠️ Compliance:** All new apps and updates to Play Store must use **Billing Library 7+** as of August 31, 2025.

---

## 2. Product Types

```
One-Time Products (formerly "in-app items"):
├── Consumable    → purchased and consumed repeatedly
│                   (in-game currency, lives, power-ups)
│                   Must consume() before user can buy again
│
└── Non-consumable → purchased once, owned forever
                    (premium unlock, ad removal, lifetime access)
                    Must acknowledge() within 72 hours

Subscriptions:
  Subscription Product (e.g., "premium_access")
  └── Base Plan (e.g., "monthly", "yearly", "weekly")
      └── Offers (e.g., "7-day free trial", "50% off first month")
  
  States: Active → Cancelled → Grace Period → On Hold → Paused → Expired
```

### 2.1 Subscription Model (BL5+ new structure)

```
OLD (BL4): one subscription = one price + one trial
NEW (BL5+): subscription → multiple base plans → multiple offers per plan

Example: "premium_access" subscription
├── base_plan: "monthly"
│   ├── offer: "free_trial_7_day" (new users only)
│   └── offer: "introductory_50_off" (first month 50% off)
├── base_plan: "yearly"
│   └── offer: "free_trial_30_day"
└── base_plan: "weekly" (no offers)

Prepaid plans (BL6+): pay upfront for fixed period, no auto-renewal
```

---

## 3. Setup

```kotlin
// gradle
implementation("com.android.billingclient:billing-ktx:7.1.1")
```

---

## 4. BillingClient — Connection

```kotlin
class BillingRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private var billingClient: BillingClient? = null
    private val _purchaseState = MutableStateFlow<PurchaseState>(PurchaseState.Idle)
    val purchaseState = _purchaseState.asStateFlow()

    // PurchasesUpdatedListener — called for all purchase events
    private val purchasesUpdatedListener = PurchasesUpdatedListener { billingResult, purchases ->
        when (billingResult.responseCode) {
            BillingClient.BillingResponseCode.OK -> {
                purchases?.forEach { handlePurchase(it) }
            }
            BillingClient.BillingResponseCode.USER_CANCELED -> {
                _purchaseState.value = PurchaseState.Cancelled
            }
            BillingClient.BillingResponseCode.ITEM_ALREADY_OWNED -> {
                _purchaseState.value = PurchaseState.AlreadyOwned
                // Trigger restore to grant entitlement
            }
            else -> {
                _purchaseState.value = PurchaseState.Error(billingResult.debugMessage)
            }
        }
    }

    suspend fun connect(): Boolean = suspendCancellableCoroutine { continuation ->
        billingClient = BillingClient.newBuilder(context)
            .setListener(purchasesUpdatedListener)
            .enablePendingPurchases(
                PendingPurchasesParams.newBuilder().enableOneTimeProducts().build()
            )
            .build()

        billingClient!!.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                continuation.resume(result.responseCode == BillingClient.BillingResponseCode.OK)
            }
            override fun onBillingServiceDisconnected() {
                // Reconnect — Google Play disconnects periodically
                // Use exponential backoff here
            }
        })
    }
}
```

---

## 5. Query Products

```kotlin
suspend fun queryProducts(productIds: List<String>, type: String): List<ProductDetails> {
    val params = QueryProductDetailsParams.newBuilder()
        .setProductList(
            productIds.map { id ->
                QueryProductDetailsParams.Product.newBuilder()
                    .setProductId(id)
                    .setProductType(type)  // ProductType.INAPP or ProductType.SUBS
                    .build()
            }
        ).build()

    val result = billingClient!!.queryProductDetails(params)
    return if (result.billingResult.responseCode == BillingClient.BillingResponseCode.OK) {
        result.productDetailsList ?: emptyList()
    } else emptyList()
}

// Display subscription offers
fun getOfferToken(productDetails: ProductDetails): String? {
    // Find the best offer for this user (e.g., trial offer for new users)
    return productDetails.subscriptionOfferDetails
        ?.firstOrNull { it.offerTags.contains("trial") }  // prefer trial
        ?.offerToken
        ?: productDetails.subscriptionOfferDetails?.firstOrNull()?.offerToken  // fallback to base
}
```

---

## 6. Launch Purchase Flow

```kotlin
fun launchPurchaseFlow(activity: Activity, productDetails: ProductDetails, offerToken: String? = null) {
    val productDetailsParamsList = listOf(
        BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(productDetails)
            .apply {
                // For subscriptions: must include offerToken
                offerToken?.let { setOfferToken(it) }
            }
            .build()
    )

    val billingFlowParams = BillingFlowParams.newBuilder()
        .setProductDetailsParamsList(productDetailsParamsList)
        .build()

    // Launches Google Play purchase UI — result comes via PurchasesUpdatedListener
    billingClient!!.launchBillingFlow(activity, billingFlowParams)
}
```

---

## 7. Handle Purchase — The Critical Part

```kotlin
// RULE: Acknowledge within 72 hours or Google auto-refunds
private fun handlePurchase(purchase: Purchase) {
    when (purchase.purchaseState) {
        Purchase.PurchaseState.PURCHASED -> {
            // Validate on server FIRST, then grant entitlement
            scope.launch {
                val valid = validatePurchaseOnServer(purchase.purchaseToken)
                if (valid) {
                    grantEntitlement(purchase)
                    acknowledgePurchase(purchase)
                }
            }
        }
        Purchase.PurchaseState.PENDING -> {
            // User chose PENDING payment (cash payment, bank transfer)
            // Don't grant entitlement yet — wait for PURCHASED state
            // Show "payment pending" UI
            _purchaseState.value = PurchaseState.Pending
        }
    }
}

// For non-consumables and subscriptions: acknowledge
private suspend fun acknowledgePurchase(purchase: Purchase) {
    if (!purchase.isAcknowledged) {
        val params = AcknowledgePurchaseParams.newBuilder()
            .setPurchaseToken(purchase.purchaseToken)
            .build()
        billingClient!!.acknowledgePurchase(params)
    }
}

// For consumables: consume (allows re-purchase)
private suspend fun consumePurchase(purchase: Purchase) {
    val params = ConsumeParams.newBuilder()
        .setPurchaseToken(purchase.purchaseToken)
        .build()
    billingClient!!.consumePurchase(params)
    // Now user can buy this consumable again
}
```

---

## 8. Server-Side Validation — MANDATORY

**Never grant entitlements based solely on client-side purchase state.** Purchases can be faked on rooted devices.

```
Client Purchase Flow:
  User taps Buy → Google Play UI
  → Purchase complete → purchase.purchaseToken returned to app
  → App sends purchaseToken to YOUR server
  → Your server calls Google Play Developer API to verify
  → Server returns: valid/invalid + entitlement to grant
  → App grants entitlement (local cache + UI)
  → App acknowledges purchase via BillingClient

Google Play Developer API (server-side):
  GET https://androidpublisher.googleapis.com/androidpublisher/v3/
      applications/{packageName}/purchases/subscriptions/{subscriptionId}/tokens/{token}

  Response includes:
  - paymentState (payment received, free trial, pending)
  - expiryTimeMillis (when subscription expires)
  - autoRenewing (will it renew?)
  - orderId (for refund tracking)
```

```kotlin
// Your server validates and stores the subscription state
// App queries YOUR server to check current entitlement
// This way: refunds, cancellations, billing holds are handled server-side

suspend fun checkEntitlement(): SubscriptionStatus {
    return yourApi.getSubscriptionStatus(currentUserId)
    // Server checks its DB which is synced with Google Play Developer API
    // via real-time developer notifications (Pub/Sub)
}
```

---

## 9. Restore Purchases

Required for non-consumables and subscriptions so users can restore after reinstall:

```kotlin
suspend fun restorePurchases() {
    // Query one-time purchases
    val inAppResult = billingClient!!.queryPurchasesAsync(
        QueryPurchasesParams.newBuilder()
            .setProductType(BillingClient.ProductType.INAPP)
            .build()
    )

    // Query subscriptions
    val subsResult = billingClient!!.queryPurchasesAsync(
        QueryPurchasesParams.newBuilder()
            .setProductType(BillingClient.ProductType.SUBS)
            .build()
    )

    val allPurchases = (inAppResult.purchasesList + subsResult.purchasesList)
        .filter { it.purchaseState == Purchase.PurchaseState.PURCHASED }

    allPurchases.forEach { purchase ->
        // Re-validate and re-grant entitlements
        val valid = validatePurchaseOnServer(purchase.purchaseToken)
        if (valid) grantEntitlement(purchase)
    }
}

// Call on app start + after user explicitly taps "Restore Purchases"
// Required by App Store / Play Store policy — users must be able to restore
```

---

## 10. Subscription States & Grace Period

```
Active:        User has access. subscription.paymentState = 1
               autoRenewing = true

Grace Period:  Payment failed, Google retrying (default: 3 days for cards)
               User STILL has access during grace period
               Show "update payment method" banner

On Hold:       Grace period ended, payment still failed
               User LOSES access
               Show paywall — "renew subscription to continue"

Paused:        User explicitly paused (Play Console feature)
               User loses access
               expiryTimeMillis = when pause ends

Cancelled:     User cancelled but access valid until expiryTimeMillis
               autoRenewing = false
               Show "resubscribe" option, don't cut off access yet

Expired:       expiryTimeMillis passed, access revoked
               Show paywall
```

```kotlin
// Handle via Real-Time Developer Notifications (server Pub/Sub)
// Google sends Pub/Sub message to your server on any subscription event:
// SUBSCRIPTION_RENEWED, SUBSCRIPTION_CANCELED, SUBSCRIPTION_ON_HOLD,
// SUBSCRIPTION_IN_GRACE_PERIOD, SUBSCRIPTION_RESTARTED, etc.

// Your server updates user's entitlement status in DB
// App polls or receives push notification to refresh entitlement
```

---

## 11. HLD — Entitlement Architecture

```
Purchase Flow:
  App → Google Play UI → Purchase token
  App → Your backend: POST /purchases/validate { token, userId }
  Backend → Google Play Developer API: verify token
  Backend → DB: store subscription { userId, productId, expiresAt, status }
  Backend → App: { entitled: true, expiresAt: ... }
  App → BillingClient: acknowledgePurchase(token)
  App → UI: show premium content

Ongoing Entitlement Check:
  App launch → query your backend for current entitlement
  Never trust local cache alone (subscriptions can be cancelled/refunded)

Subscription Event Handling (server-side):
  Google Pub/Sub → your webhook
  → Parse notification type (RENEWED, CANCELLED, ON_HOLD)
  → Update user entitlement in DB
  → Send push notification to user if access changed

Key DB fields:
  subscription { userId, productId, purchaseToken, status,
                 expiresAt, autoRenewing, gracePeriodEnd }
```

---

## 12. Common Misunderstandings & Pitfalls

**❌ Not acknowledging within 72 hours**
Google auto-refunds unacknowledged purchases after 72 hours. Always acknowledge immediately after server validation — don't defer.

**❌ Granting entitlement client-side without server validation**
Rooted devices can fake purchase receipts. Always validate `purchaseToken` against the Google Play Developer API server-side before granting premium access.

**❌ Not handling PENDING purchases**
Some markets support offline payment methods (cash vouchers, bank transfers). These arrive as `PENDING`. Don't grant access; show "payment processing" UI and wait for the state to change to `PURCHASED` (delivered via `PurchasesUpdatedListener`).

**❌ Not calling `restorePurchases()` on app start**
Users who reinstall need their purchases restored. The Billing Library won't automatically restore — you must call `queryPurchasesAsync()` on connection.

**❌ Showing wrong price from hardcoded strings**
Price is locale-specific. Always use `productDetails.oneTimePurchaseOfferDetails?.formattedPrice` — never hardcode "$9.99". The Play Store handles currency conversion and local pricing.

**❌ Not implementing `onBillingServiceDisconnected()`**
The billing service disconnects after inactivity. Implement retry logic with exponential backoff in `onBillingServiceDisconnected()` so purchases continue to work.

---

## 13. Best Practices

- **Billing Library 7+ mandatory** — all Play Store apps since August 2025
- **Server-side validation always** — never trust client-only purchase state
- **Acknowledge immediately** — within 72 hours, auto-refund otherwise
- **Consume before re-purchase** — consumables require consume() before repurchase
- **Real-time developer notifications** — Pub/Sub for subscription state changes
- **Show formattedPrice from API** — never hardcode prices
- **Restore on app start** — call `queryPurchasesAsync()` on `BillingClient` connected
- **Handle pending purchases** — show "processing" UI, don't grant access
- **Grace period: keep access** — don't revoke immediately on payment failure
- **On Hold: revoke access, show paywall** — different from grace period

---

## 14. Interview Q&A

**Q1: What's the difference between consumable and non-consumable products?**

> Consumable products can be purchased multiple times — in-game currency, lives, power-ups. After purchase, the app calls `consumePurchase()` which removes the item from the user's ownership, allowing them to buy it again. Non-consumable products are purchased once and owned permanently — premium unlocks, ad removal, lifetime access. They require `acknowledgePurchase()` within 72 hours but are never consumed. Subscriptions are a third category — recurring purchases with automatic renewal managed by Google Play, with a complex state machine covering active, grace period, on-hold, and cancelled states.

---

**Q2: Why must you validate purchases server-side?**

> The `purchaseToken` returned by Google Play is just a string that the client received. On a rooted device, an attacker can generate or replay fake tokens. If your app grants entitlements based solely on `purchase.purchaseState == PURCHASED` from the client, you're vulnerable to fraud. The correct flow: client receives token, sends it to your server, your server calls the Google Play Developer API with your service account credentials to verify it's genuine and associated with your app and the correct product. The server then updates the entitlement in your database and returns the result. The app acknowledges only after server confirmation. This way, even if the client is compromised, your backend is the authoritative source of entitlement.

---

**Q3: What happens when a subscription payment fails?**

> Google Play has a multi-stage retry process. First, a grace period — typically 3 days — where Google retries the payment and the user retains access. Show a non-intrusive banner prompting them to update their payment method. If payment still fails after the grace period, the subscription enters "on hold" — access is revoked. Show a paywall and a prompt to update payment. If the user never updates, the subscription eventually expires. Throughout, Google sends real-time developer notifications to your server via Pub/Sub, so you should update your entitlement DB and optionally send push notifications to the user at each state change. Never rely on polling — the Pub/Sub webhook is the source of truth.

---

*Previous: [29 — Navigation Architecture](./29-navigation.md)*
*Next: [31 — WebRTC & Video Calling](./31-webrtc.md)*
