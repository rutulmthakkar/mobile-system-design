# 05 — Real-Time & Push

> **Round type: Both (HLD + LLD)**
> **HLD:** Choosing a real-time strategy at scale — WebSocket vs SSE vs FCM, presence, delivery guarantees
> **LLD:** OkHttp WebSocket implementation, reconnection, heartbeat, FCM service setup, message types

---

## 1. Why This Matters in Interviews

Real-time is a system design staple. WhatsApp, Uber, live scores — all require the server to push data without polling. The challenge is keeping it reliable on flaky mobile networks, across app lifecycle transitions, and at scale with millions of concurrent connections.

**Common interview angles:**
- "How would you design the real-time layer for a chat app?"
- "What's the difference between WebSocket, SSE, and long polling?"
- "How does FCM work and when would you use it vs WebSocket?"
- "How do you handle a WebSocket connection when the user backgrounds the app?"
- "How do you implement presence — knowing who's online?"
- "What happens to undelivered messages when a device is offline?"
- "How do you scale WebSocket connections to millions of users?"

---

## 2. The Core Problem

HTTP is request-response: the client always initiates. For real-time, the **server needs to push** without being asked.

Four approaches:

```
Polling       → Client asks every N seconds: "anything new?" (wasteful)
Long Polling  → Client asks, server waits before replying (better, still HTTP)
SSE           → Server pushes over persistent HTTP connection (one-way)
WebSocket     → Persistent bidirectional TCP channel (full-duplex)
FCM / APNs    → OS-level push delivery (works even when app is killed)
```

---

## 3. Approach Comparison

### 3.1 Polling

```
Client → GET /messages?since=123  (every N seconds)
Server → 200 OK { messages: [] }  (empty most of the time)
```

**Pros:** Simplest, works everywhere, stateless server.
**Cons:** Wastes bandwidth on empty responses. Latency = poll interval. Server under load even when nothing changed.

**When to use:** Never for new apps. Last resort for legacy systems only.

---

### 3.2 Long Polling

```
Client → GET /messages?since=123
Server → (holds connection open up to 30s)
Server → 200 OK { messages: [...] }  (responds when data is ready, or on timeout)
Client → immediately sends next request
```

**Pros:** Near real-time, fewer empty responses than polling.
**Cons:** Head-of-line blocking, HTTP overhead per cycle, tricky timeout handling.

**When to use:** Fallback when WebSocket is blocked by proxies. Never as primary strategy for new systems.

---

### 3.3 Server-Sent Events (SSE)

```
Client → GET /events  (one request)
Server → Content-Type: text/event-stream  (keeps connection open)
         data: {"type":"message","text":"Hello"}\n\n
         data: {"type":"typing","userId":"123"}\n\n
```

**Pros:**
- Server→client push over standard HTTP — works through most proxies
- Auto-reconnect built into the browser spec with `Last-Event-ID`
- Much simpler to scale than WebSocket (plain HTTP)

**Cons:**
- One-way only — use REST for client→server
- No native Android support — needs `okhttp-eventsource` library
- Not suitable for bidirectional use cases like chat

```kotlin
// Android SSE with okhttp-eventsource
val request = Request.Builder().url("https://api.example.com/events").build()
val handler = object : EventSourceListener() {
    override fun onEvent(es: EventSource, id: String?, type: String?, data: String) {
        val event = Json.decodeFromString<ServerEvent>(data)
        handleEvent(event)
    }
    override fun onFailure(es: EventSource, t: Throwable?, response: Response?) {
        // library auto-reconnects with backoff
    }
}
EventSources.createFactory(okHttpClient).newEventSource(request, handler)
```

**When to use:** Live dashboards, news feeds, order status — anything server→client only.

---

### 3.4 WebSocket ✅ Primary for bidirectional real-time

```
Client → HTTP Upgrade: websocket (HTTP 101 Switching Protocols)
Server → Accepts upgrade
         ← Persistent full-duplex TCP channel →
Client ↔ Server: frames flow in both directions simultaneously
```

**Key properties:**
- Full-duplex: send and receive simultaneously
- Minimal frame overhead (2–10 bytes vs full HTTP headers per request)
- Sub-50ms latency
- Single persistent connection — no repeated handshakes

**Android (OkHttp):**
```kotlin
class WebSocketManager(
    private val client: OkHttpClient,
    private val scope: CoroutineScope,
    private val tokenProvider: TokenProvider
) {
    private var webSocket: WebSocket? = null
    private var reconnectAttempts = 0
    private var lastUrl: String = ""

    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState = _connectionState.asStateFlow()

    private val _messages = MutableSharedFlow<ChatMessage>()
    val messages = _messages.asSharedFlow()

    fun connect(url: String) {
        lastUrl = url
        val request = Request.Builder()
            .url(url)
            .header("Authorization", "Bearer ${tokenProvider.getToken()}")
            .build()

        webSocket = client.newWebSocket(request, object : WebSocketListener() {
            override fun onOpen(ws: WebSocket, response: Response) {
                reconnectAttempts = 0
                _connectionState.value = ConnectionState.CONNECTED
            }

            override fun onMessage(ws: WebSocket, text: String) {
                scope.launch {
                    val msg = Json.decodeFromString<ChatMessage>(text)
                    _messages.emit(msg)
                }
            }

            override fun onFailure(ws: WebSocket, t: Throwable, response: Response?) {
                _connectionState.value = ConnectionState.DISCONNECTED
                scheduleReconnect()
            }

            override fun onClosed(ws: WebSocket, code: Int, reason: String) {
                _connectionState.value = ConnectionState.DISCONNECTED
                if (code != 1000) scheduleReconnect()  // 1000 = normal closure
                // Don't reconnect on 1008 (Policy Violation) — server rejected us
            }
        })
    }

    fun send(message: ChatMessage) {
        webSocket?.send(Json.encodeToString(message))
    }

    fun disconnect() {
        webSocket?.close(1000, "User disconnected")
        webSocket = null
    }

    private fun scheduleReconnect() {
        scope.launch {
            val delay = minOf((2.0.pow(reconnectAttempts) * 1000).toLong(), 30_000L)
            val jitter = Random.nextLong(0, 1000)
            delay(delay + jitter)
            reconnectAttempts++
            connect(lastUrl)
        }
    }
}
```

### 3.5 Decision Matrix

| | Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| Direction | C→S | C→S | S→C | Bidirectional |
| Latency | High | Low–Med | Low | Lowest |
| Server load | Highest | Medium | Low | Low |
| Complexity | Lowest | Medium | Low | High |
| Battery | Worst | Medium | Good | Good |
| Proxy support | Best | Good | Good | Can be blocked |
| Best for | Never | Fallback | Feeds, status | Chat, gaming, collab |

---

## 4. WebSocket — Deep Dive

### 4.1 Heartbeat / Ping-Pong

Mobile networks silently drop TCP connections — NAT timeouts, cell tower handoffs, tunnels. The connection appears alive but is dead ("half-open"). Without heartbeat, you won't know until you try to send.

```kotlin
// Protocol-level ping via OkHttp (recommended)
val client = OkHttpClient.Builder()
    .pingInterval(30, TimeUnit.SECONDS)  // sends Ping every 30s
    // if no Pong received, OkHttp closes connection → onFailure fires → reconnect
    .build()

// Application-level heartbeat (supplement for visibility)
private fun startHeartbeat() {
    heartbeatJob = scope.launch {
        while (isActive) {
            delay(30_000)
            val sent = webSocket?.send("""{"type":"ping"}""") ?: false
            if (!sent) scheduleReconnect()
        }
    }
}
```

**Key rule:** On app foreground (`onResume`), send an immediate ping — don't wait 30s. The connection likely died while backgrounded.

### 4.2 Reconnection with Exponential Backoff + Jitter

```kotlin
private fun scheduleReconnect() {
    scope.launch {
        // Exponential: 1s, 2s, 4s, 8s, 16s, 30s (capped)
        val delayMs = minOf((2.0.pow(reconnectAttempts) * 1000).toLong(), 30_000L)
        val jitter = Random.nextLong(0, 1000)  // desynchronize clients
        delay(delayMs + jitter)
        reconnectAttempts++
        connect(lastUrl)
    }
}
// Reset on successful open:
reconnectAttempts = 0
```

**Don't reconnect on close code 1008 (Policy Violation)** — the server explicitly rejected the connection; retrying will fail.

### 4.3 Lifecycle Scoping

```kotlin
// ❌ Bad: WebSocket in Activity — dies on rotation
class ChatActivity : AppCompatActivity() {
    private val ws = WebSocketManager(...)
}

// ✅ Good: WebSocket in ViewModel (survives rotation)
// Or application-scoped singleton for app-wide connections
@HiltViewModel
class ChatViewModel @Inject constructor(
    private val wsManager: WebSocketManager  // singleton scope
) : ViewModel() {
    override fun onCleared() = wsManager.disconnect()
}
```

**On background:** Don't disconnect immediately. Keep alive for ~60s (users switch apps constantly). Disconnect only on prolonged background, rely on FCM for wake-up.

### 4.4 Gap Sync on Reconnect

```kotlin
// Server assigns monotonically increasing sequence numbers
// Client tracks last seen in Room

fun onConnected() {
    val lastSeenId = messageDao.getLastMessageId(conversationId)
    webSocket.send("""{"type":"sync","since":$lastSeenId}""")
    // Server responds with all missed messages
    // Client upserts into Room → UI updates via Flow automatically
}
```

---

## 5. Firebase Cloud Messaging (FCM)

### 5.1 What FCM Solves

WebSocket requires a live connection — impossible when the app is killed. FCM uses Google's always-on connection (Google Play Services) to deliver wake-up messages even to a killed app.

```
Your Server → FCM Servers → Google Play Services → Your App
```

FCM is a **wake-up signal**, not a real-time message transport.

### 5.2 FCM Message Types — Critical Distinction

| | Notification Message | Data Message |
|---|---|---|
| Shows notification | System (automatic) | Your code (manual) |
| `onMessageReceived` called | **Foreground only** | **All states (fg + bg + killed)** |
| Max payload | 4KB | 4KB |
| Collapsible | Yes (by default) | With `collapse_key` |
| Best for | Simple alerts | Custom handling, sync trigger |

**The most common FCM mistake:** Sending a `notification` payload and expecting `onMessageReceived` to fire when the app is killed. It won't. Use data-only messages for any custom logic.

```json
// ❌ Notification message — your code doesn't run when app is killed
{
  "message": {
    "token": "...",
    "notification": { "title": "New message", "body": "Hello!" }
  }
}

// ✅ Data-only message — onMessageReceived always called
{
  "message": {
    "token": "...",
    "data": { "type": "new_message", "conversation_id": "123" }
  }
}
```

### 5.3 FirebaseMessagingService

```kotlin
class AppMessagingService : FirebaseMessagingService() {

    // Called on new token (install, token refresh)
    override fun onNewToken(token: String) {
        lifecycleScope.launch { userRepo.registerFcmToken(token) }
    }

    // Called for data messages in ALL states
    // Called for notification messages in FOREGROUND only
    override fun onMessageReceived(message: RemoteMessage) {
        when (message.data["type"]) {
            "new_message" -> {
                if (!isAppForeground()) showLocalNotification(message.data)
                triggerMessageSync(message.data["conversation_id"])
            }
            "sync" -> triggerFullSync()
            "call" -> showIncomingCallUI(message.data)
        }
    }

    private fun triggerMessageSync(convId: String?) {
        // Use WorkManager — gives us guaranteed execution + constraints
        WorkManager.getInstance(this).enqueue(
            OneTimeWorkRequestBuilder<MessageSyncWorker>()
                .setInputData(workDataOf("conv_id" to convId))
                .setConstraints(Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED).build())
                .build()
        )
    }
}
```

### 5.4 FCM + WebSocket Hybrid (Production Pattern)

```
App is FOREGROUND:
  WebSocket is connected → messages arrive in real-time
  FCM is ignored (or used as deduplication fallback)

App is BACKGROUND (< 60s):
  WebSocket may still be alive → message arrives via WS
  If WS dead → FCM wakes app → fetch from server

App is KILLED:
  FCM fires onMessageReceived → sync via WorkManager → show notification

App is OFFLINE:
  FCM queues message on Google servers (up to 4 weeks, normal priority)
  Delivered when device reconnects
  Use collapse_key → only last message delivered (not 50 individual ones)
```

### 5.5 FCM Token Management

```kotlin
// Get token — always send to your server after login
FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
    sendTokenToServer(token)
}

// Token invalidated when:
// - App uninstalled/reinstalled
// - User clears app data
// - Token unused for 270 days
// → Always handle onNewToken and delete stale tokens on FCM 404 response
```

**Multi-device:** Store ALL FCM tokens per user. Fan out to all tokens when sending a push. Remove tokens that return `UNREGISTERED` error from FCM.

---

## 6. Presence System Design (HLD)

### 6.1 Heartbeat-Based (Recommended)

```
1. Client connects → WebSocket server sets Redis key:
   SET presence:{userId} 1 EX 60  (TTL = 60s)

2. Client sends heartbeat every 30s:
   WebSocket server renews: EXPIRE presence:{userId} 60

3. Client disconnects or heartbeat stops:
   Redis key expires after 60s → user is "offline"

4. Any server can query presence:
   GET presence:{userId} → 1 (online) or nil (offline)
```

### 6.2 Presence Change Notifications

```
On connect: publish to Redis Pub/Sub → "user:{id}:online"
On expire/disconnect: publish → "user:{id}:offline"
WebSocket servers subscribe and push presence updates to connected clients
```

### 6.3 Gotchas

- **False offline:** User switches apps for 5s → WebSocket pauses → shows offline. Add 30–60s grace period before surfacing "offline"
- **False online:** App crashes without clean disconnect → heartbeat stops → TTL expires naturally (self-healing)
- **Scale:** Don't query presence for every user in a list simultaneously. Batch: `MGET presence:u1 presence:u2 presence:u3`

---

## 7. React Native

```typescript
// Built-in WebSocket API
const ws = new WebSocket('wss://api.example.com/ws', [], {
    headers: { Authorization: `Bearer ${token}` }
})
ws.onopen = () => { /* connected */ }
ws.onmessage = (e) => handleMessage(JSON.parse(e.data))
ws.onerror = (e) => scheduleReconnect()
ws.onclose = (e) => { if (e.code !== 1000) scheduleReconnect() }
ws.send(JSON.stringify({ type: 'message', content: 'Hello' }))

// FCM with @react-native-firebase/messaging
import messaging from '@react-native-firebase/messaging'

// Register background handler — outside component, at module level
messaging().setBackgroundMessageHandler(async msg => {
    // runs when app is background or killed
    await syncMessages(msg.data.conversation_id)
})

// Foreground handler
useEffect(() => {
    return messaging().onMessage(async msg => {
        showLocalNotification(msg)
    })
}, [])

// Get FCM token
const token = await messaging().getToken()
```

---

## 8. Common Misunderstandings & Pitfalls

**❌ Expecting `onMessageReceived` for notification messages when app is killed**
Only data-only messages fire `onMessageReceived` in all app states. With a `notification` key, the system handles it — your code never runs in background/killed state.

**❌ WebSocket in Activity**
Activity is recreated on rotation → WebSocket reconnects every time. Use ViewModel or application-scoped singleton.

**❌ No heartbeat**
Mobile networks silently kill TCP connections. Without ping-pong, you won't detect a dead socket until you try to send — which can be minutes later.

**❌ Reconnecting immediately without backoff**
Million clients down → all reconnect at once → thundering herd → server can't recover. Always exponential backoff + jitter.

**❌ Not tracking gap sync on reconnect**
Any WebSocket reconnect = potential message gap. Always fetch missed messages since last seen ID on reconnect.

**❌ One FCM token per user**
Users have multiple devices. Store and fan out to all tokens. Remove `UNREGISTERED` tokens returned by FCM.

**❌ Not using `collapse_key`**
Device offline → 50 "new message" FCMs queued → device comes online → 50 push notifications at once. Use `collapse_key` so only the latest is delivered.

**❌ Using WebSocket for static or infrequent data**
WebSocket has overhead — persistent connection, heartbeat, reconnect logic. For a dashboard that updates every 60s, SSE or even polling is simpler and cheaper.

---

## 9. Best Practices

- **Hybrid: WebSocket for foreground, FCM for background/killed** — best of both worlds
- **Data-only FCM messages for any custom handling** — never mix notification key with custom logic expectations
- **Gap sync on every reconnect** — track `lastSeenId`, request delta immediately after `onOpen`
- **Protocol-level ping via OkHttp `pingInterval`** — simpler and more reliable than application-level heartbeat alone
- **Immediate ping on app foreground** — don't wait for next heartbeat interval
- **Exponential backoff + jitter on reconnect** — cap at 30s, add random 0–1s jitter
- **Surface connection state to UI** — show "Reconnecting…" banner after 2+ failed attempts, not the first
- **`collapse_key` for FCM batching** — always for "new content" notifications
- **`wss://` always in production** — never send auth tokens or content unencrypted
- **Validate all WebSocket messages** — malformed JSON or unknown types should not crash; log and discard

---

## 10. Interview Q&A

**Q1: What's the difference between WebSocket, SSE, and long polling?**

> Long polling is still HTTP request-response — the server just delays the response until data is ready. Still fundamentally one round trip per event. SSE is a persistent HTTP connection where the server streams events — server-to-client only, works well for feeds and notifications, simpler to scale than WebSocket. WebSocket upgrades the HTTP connection to a full-duplex TCP channel — both sides can send simultaneously at any time, lowest latency, ideal for interactive bidirectional communication. I use SSE for server-push-only use cases (live dashboards, order tracking) and WebSocket for chat, collaboration, or anything where the client also sends frequently. Long polling only as a compatibility fallback.

---

**Q2: How does FCM work and what's the difference between notification and data messages?**

> FCM piggybacks on Google Play Services' always-on connection to deliver messages even to killed apps. Notification messages contain a `notification` key — FCM SDK displays the notification automatically when the app is in the background; your `onMessageReceived` is only called in the foreground. Data messages contain only a `data` key — `onMessageReceived` fires in all app states. This is the most common FCM pitfall. In production I always use data-only messages so I control the handling in all states, and I build the local notification manually inside `onMessageReceived`.

---

**Q3: How do you handle a WebSocket when the user backgrounds the app?**

> I don't disconnect immediately — users switch apps constantly. I keep the connection alive for up to 60 seconds. If the app stays backgrounded longer, I disconnect and rely on FCM for wake-up. On foreground, I immediately send a ping — the connection may have silently died during background time. If no pong returns within 10 seconds, I reconnect. On reconnect I also trigger a gap sync to fetch any messages missed during the disconnection window.

---

**Q4: Design the presence system for a chat app.**

> Heartbeat-based with Redis TTL. On WebSocket connect, set `presence:{userId}` in Redis with 60-second TTL. The client sends a heartbeat every 30s; the server renews the TTL. If the heartbeat stops (app killed, crash, network loss), the key expires after 60s and the user is effectively offline — self-healing with no explicit disconnect needed. For real-time presence updates to friends, I publish to a Redis Pub/Sub channel on presence changes. WebSocket servers subscribed to those channels push updates to connected clients. To avoid false-offline flickers when users briefly switch apps, the UI waits 30s before showing "offline" after the last known-online signal.

---

**Q5: How do you handle message gaps when a WebSocket reconnects?**

> Every message from the server carries a monotonically increasing sequence number. The client tracks the highest received sequence number in Room. On every reconnect, immediately after `onOpen`, the client sends `{"type":"sync","since":lastSeqNum}`. The server responds with all messages since that point. The client upserts them into Room using `@Upsert` — duplicates are handled naturally. The UI observes Room via Flow, so it auto-refreshes with the filled gap. The same delta fetch is also triggered by FCM data messages to recover from offline gaps.

---

**Q6: How do you scale WebSocket connections to millions of users?**

> WebSocket connections are stateful — pinned to a server. The key insight is to separate connection management from message routing. WebSocket gateway servers each hold N persistent connections. When a message arrives for User B on Server A, Server A publishes to a Redis Pub/Sub topic for User B's conversation. Server B, which holds User B's connection, subscribes to that topic and forwards the message. The load balancer uses sticky sessions to keep a client routed to the same gateway server for the session duration. For very large scale, dedicated connection gateways (AWS API Gateway WebSocket, or custom) handle the stateful layer, while message services remain stateless and scale independently.

---

*Previous: [04 — Data Layer](./04-data-layer.md)*
*Next: [06 — Performance](./06-performance.md)*
