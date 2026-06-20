# 25 — Notifications

> **Round type: Both (HLD + LLD)**
> **HLD:** Push notification architecture — FCM + APNs, notification routing, targeting
> **LLD:** Notification channels, styles, actions, deep links, FCM payload handling, badges, DND

---

## 1. Why This Matters

Notifications are a primary re-engagement surface. At Amazon/Alexa, Google, and consumer apps interviewers probe: channel design, badge management, handling FCM vs APNs across platforms, and how notifications deep-link into specific app state.

---

## 2. Notification Channels (Android 8.0+, API 26+)

Channels are mandatory on API 26+. Users control sound, vibration, and importance per channel — not per notification.

```kotlin
// Create channels ONCE at app startup (idempotent — safe to call multiple times)
fun createNotificationChannels(context: Context) {
    val notificationManager = context.getSystemService(NotificationManager::class.java)

    val channels = listOf(
        NotificationChannel(
            "messages",
            "Messages",
            NotificationManager.IMPORTANCE_HIGH   // heads-up, sound, vibration
        ).apply {
            description = "Direct messages from contacts"
            enableVibration(true)
            enableLights(true)
            lightColor = Color.BLUE
        },
        NotificationChannel(
            "promotions",
            "Promotions",
            NotificationManager.IMPORTANCE_LOW    // no sound, no heads-up
        ).apply {
            description = "Offers and discounts"
        },
        NotificationChannel(
            "system",
            "System",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "App updates and account alerts"
        }
    )

    notificationManager.createNotificationChannels(channels)
}

// Importance levels:
// IMPORTANCE_HIGH    → makes sound, shows heads-up
// IMPORTANCE_DEFAULT → makes sound, no heads-up
// IMPORTANCE_LOW     → no sound, shows in shade
// IMPORTANCE_MIN     → no sound, no icon in status bar
// IMPORTANCE_NONE    → completely silent (user effectively blocked)
```

### 2.1 Check if Channel is Blocked

```kotlin
fun isChannelEnabled(context: Context, channelId: String): Boolean {
    val manager = context.getSystemService(NotificationManager::class.java)
    val channel = manager.getNotificationChannel(channelId)
    return channel?.importance != NotificationManager.IMPORTANCE_NONE
}

// If blocked, deep-link user to channel settings
fun openChannelSettings(context: Context, channelId: String) {
    context.startActivity(Intent(Settings.ACTION_CHANNEL_NOTIFICATION_SETTINGS).apply {
        putExtra(Settings.EXTRA_APP_PACKAGE, context.packageName)
        putExtra(Settings.EXTRA_CHANNEL_ID, channelId)
    })
}
```

---

## 3. Building Notifications

```kotlin
fun showMessageNotification(
    context: Context,
    sender: String,
    message: String,
    conversationId: String
) {
    val notificationManager = NotificationManagerCompat.from(context)

    // Deep link PendingIntent → opens specific conversation
    val openIntent = PendingIntent.getActivity(
        context, conversationId.hashCode(),
        Intent(context, MainActivity::class.java).apply {
            action = "open_conversation"
            putExtra("conversation_id", conversationId)
            flags = Intent.FLAG_ACTIVITY_SINGLE_TOP
        },
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    // Reply action (inline reply)
    val remoteInput = RemoteInput.Builder("reply_key")
        .setLabel("Reply...")
        .build()
    val replyPendingIntent = PendingIntent.getBroadcast(
        context, conversationId.hashCode(),
        Intent(context, ReplyReceiver::class.java).apply {
            putExtra("conversation_id", conversationId)
        },
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
    )
    val replyAction = NotificationCompat.Action.Builder(
        R.drawable.ic_reply, "Reply", replyPendingIntent
    ).addRemoteInput(remoteInput).build()

    // Messaging style (WhatsApp/Messages style)
    val messagingStyle = NotificationCompat.MessagingStyle("Me")
        .setConversationTitle(sender)
        .addMessage(message, System.currentTimeMillis(),
            Person.Builder().setName(sender).build())

    val notification = NotificationCompat.Builder(context, "messages")
        .setSmallIcon(R.drawable.ic_notification)
        .setStyle(messagingStyle)
        .setContentIntent(openIntent)
        .addAction(replyAction)
        .setAutoCancel(true)             // dismiss on tap
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .setCategory(NotificationCompat.CATEGORY_MESSAGE)
        .setShortcutId(conversationId)   // links to Dynamic Shortcut (long-press launcher)
        .build()

    if (ActivityCompat.checkSelfPermission(context, POST_NOTIFICATIONS) == PERMISSION_GRANTED) {
        // Group messages by conversation (Android 7+ notification grouping)
        notificationManager.notify(conversationId.hashCode(), notification)
    }
}
```

### 3.1 Notification Styles

```kotlin
// BigTextStyle — for long text (email, article preview)
NotificationCompat.BigTextStyle()
    .bigText("Full email body text here...")
    .setBigContentTitle("Email Subject")
    .setSummaryText("sender@email.com")

// BigPictureStyle — image preview (social media, photo share)
NotificationCompat.BigPictureStyle()
    .bigPicture(bitmap)
    .bigLargeIcon(null as Bitmap?)  // hide large icon when expanded

// InboxStyle — list of items (multiple emails, multiple messages)
NotificationCompat.InboxStyle()
    .addLine("Alice: Hey!")
    .addLine("Bob: Are you coming?")
    .addLine("Carol: Check this out")
    .setSummaryText("+2 more")

// MessagingStyle — conversation (recommended for chat apps, required for bubbles)
NotificationCompat.MessagingStyle(myPerson)
    .addMessage("Hey!", timestamp1, alicePerson)
    .addMessage("What's up?", timestamp2, myPerson)
```

---

## 4. POST_NOTIFICATIONS Permission (Android 13+)

```kotlin
// AndroidManifest.xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

// Runtime request
val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) enableNotifications()
    else showPermissionRationale()
}

// When to request: after onboarding, when user explicitly enables notifications
// Don't request on first launch — ask contextually
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    when {
        ContextCompat.checkSelfPermission(this, POST_NOTIFICATIONS) == PERMISSION_GRANTED -> { }
        shouldShowRequestPermissionRationale(POST_NOTIFICATIONS) -> showRationale()
        else -> requestPermissionLauncher.launch(POST_NOTIFICATIONS)
    }
}
```

---

## 5. FCM Integration (Data Message → Notification)

```kotlin
class AppMessagingService : FirebaseMessagingService() {

    override fun onNewToken(token: String) {
        // Send to server — multiple devices per user, token can change
        lifecycleScope.launch { userRepo.registerFcmToken(token) }
    }

    override fun onMessageReceived(message: RemoteMessage) {
        // data-only messages → called in ALL app states
        when (message.data["type"]) {
            "message" -> {
                val convId = message.data["conversation_id"] ?: return
                val sender = message.data["sender"] ?: "Someone"
                val text = message.data["text"] ?: "New message"

                if (isAppForeground()) {
                    // App is visible — update UI via ViewModel/WebSocket
                    // Don't show notification — redundant
                } else {
                    showMessageNotification(this, sender, text, convId)
                }
            }
            "order_update" -> showOrderNotification(message.data)
            "promo" -> if (isPromoChannelEnabled()) showPromoNotification(message.data)
        }
    }

    private fun isAppForeground(): Boolean {
        val am = getSystemService(ActivityManager::class.java)
        return am.runningAppProcesses?.any {
            it.importance == RunningAppProcessInfo.IMPORTANCE_FOREGROUND
        } == true
    }
}
```

### 5.1 Notification → Deep Link → Screen

```kotlin
// In MainActivity.onCreate and onNewIntent
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    handleNotificationIntent(intent)
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    handleNotificationIntent(intent)
}

private fun handleNotificationIntent(intent: Intent?) {
    when (intent?.action) {
        "open_conversation" -> {
            val convId = intent.getStringExtra("conversation_id") ?: return
            navController.navigate("conversation/$convId")
        }
        "open_order" -> {
            val orderId = intent.getStringExtra("order_id") ?: return
            navController.navigate("order/$orderId")
        }
    }
}
```

---

## 6. Notification Badges

```kotlin
// Badge count on launcher icon (API 26+)
val notificationManager = getSystemService(NotificationManager::class.java)

// Automatic: notification count shown as badge
// Works automatically when notifications are posted

// Manual badge count (Samsung/some OEMs)
val shortcutBadger = ShortcutBadger.applyCount(context, unreadCount)

// Clear badge on open
fun clearBadge(context: Context) {
    ShortcutBadger.removeCount(context)
    // Also cancel any active notifications
    NotificationManagerCompat.from(context).cancelAll()
}
```

---

## 7. Notification Groups (Android 7+)

```kotlin
// Group multiple notifications from same conversation
val GROUP_KEY = "com.example.MESSAGES"

// Individual notifications
NotificationCompat.Builder(context, "messages")
    .setGroup(GROUP_KEY)
    .setGroupAlertBehavior(NotificationCompat.GROUP_ALERT_CHILDREN)
    .build()

// Summary notification (required for groups)
val summaryNotification = NotificationCompat.Builder(context, "messages")
    .setSmallIcon(R.drawable.ic_notification)
    .setStyle(NotificationCompat.InboxStyle()
        .addLine("Alice: Hey!")
        .addLine("Bob: You there?")
        .setSummaryText("2 new messages"))
    .setGroup(GROUP_KEY)
    .setGroupSummary(true)   // ← marks as summary
    .build()

notificationManager.notify(SUMMARY_ID, summaryNotification)
```

---

## 8. Notification Channels — HLD Design

```
Channel architecture for a social app:

MESSAGES (HIGH)           → DMs and group chats
COMMENTS (DEFAULT)        → replies to your posts
LIKES (LOW)               → post reactions (aggregated, not per-like)
FOLLOWS (DEFAULT)         → new follower alerts
LIVE_UPDATES (HIGH)       → live events the user is watching
PROMOTIONS (LOW)          → offers (separate so users can mute without missing messages)
SYSTEM (DEFAULT)          → account, security, app updates

Design principles:
  → Separate channel per distinct user concern
  → Don't create channels per user or per content — per notification TYPE
  → Critical paths (messages, security) get HIGH importance
  → Marketing always gets LOW importance — it can never be HIGH
  → Server controls which channel a notification goes to via FCM data payload
```

---

## 9. Common Misunderstandings & Pitfalls

**❌ Sending `notification` payload instead of `data` payload for custom handling**
Only `data` messages call `onMessageReceived` in all app states. `notification` payloads are shown by the system automatically when the app is in background — your code never runs.

**❌ Not grouping notifications for high-volume apps**
A messaging app that shows 10 individual notifications without grouping is a terrible UX. Always group by conversation and show a summary notification.

**❌ Creating notification channels in a loop per conversation or per user**
Channels should represent notification *types*, not individual items. Creating a channel per conversation creates thousands of channels that clutter the notification settings UI.

**❌ Requesting `POST_NOTIFICATIONS` on cold launch**
Asking for notification permission before the user sees any value from the app gets rejected. Ask contextually — after the user performs an action that would benefit from notifications.

**❌ Not handling the case where notification channel is blocked**
Users can block individual channels. Don't assume your notification was shown. Check `isChannelEnabled()` before showing permission rationale or notification-related UI.

---

## 10. Best Practices

- **Data-only FCM messages** for all custom logic — never notification payload with custom handling expectations
- **One channel per notification type** — not per user/item/conversation
- **Group notifications** by conversation/category — prevent notification spam
- **`MessagingStyle`** for chat — required for notification bubbles, looks better in all cases
- **Inline reply action** for messaging apps — reduces friction
- **`POST_NOTIFICATIONS` contextually** — after demonstrating value, not on first launch
- **Cancel notifications when user opens related screen** — avoid stale notifications
- **Test with app in background AND killed** — behavior differs between states

---

## 11. Interview Q&A

**Q1: How do you design notification channels for a social media app?**

> I'd start by mapping out distinct user concerns: direct messages (high importance — user expects to see these immediately), mentions and replies (default importance), likes and reactions (low importance — these should be aggregated, not one per like), new follower alerts (default), promotional offers (always low — marketing can never be high), and system alerts like security or account changes (high). Each concern gets its own channel. The key principle is that channels represent types of user attention, not data categories. Users will mute promotions without affecting messages. The FCM payload includes a `channel_id` field so the server controls routing without an app update.

---

**Q2: What's the difference between FCM notification and data messages for notifications?**

> A notification message has a `notification` key in the FCM payload. When the app is in the background, the FCM SDK intercepts it and shows the notification automatically without calling `onMessageReceived`. Your code never runs — you can't customize the behavior, add an action, or handle the tap. A data message has only a `data` key. `onMessageReceived` is called in all app states — foreground, background, and killed. Your code builds and shows the notification with full customization: channel routing, styling, actions, deep link intent. For any production app with custom notification behavior, always use data-only messages.

---

**Q3: How do you handle notification deep links correctly?**

> The notification's `contentIntent` is a `PendingIntent` that fires when the user taps it. I construct it to deliver an explicit Intent to `MainActivity` with an action and extras identifying what the user tapped ("open_conversation", "conversation_id = 123"). In both `onCreate` (cold start from notification) and `onNewIntent` (app already running, brought to foreground), I handle this intent and navigate to the correct screen using `NavController`. I use `FLAG_ACTIVITY_SINGLE_TOP` so if the Activity is already running, `onNewIntent` is called instead of creating a new instance. I also handle the case where the user is already on that screen — navigate should be a no-op, not add a duplicate to the back stack.

---

*Previous: [24 — Lifecycle Management](./24-lifecycle.md)*
*Next: [26 — On-Device ML & Local LLMs](./26-on-device-ml-llm.md)*
