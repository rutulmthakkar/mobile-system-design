# Mobile System Design Interview Questions

A comprehensive list of HLD and LLD questions asked at FAANG-tier companies for Android/mobile roles. Organized by category with the sections in this guide that cover each question.

---

## How to Use This List

**For HLD rounds:** Expect 1 broad question (45–60 min). Practice talking for 5 min on requirements, 10 min on architecture, 15 min on deep dives.

**For LLD rounds:** Expect 1–2 focused questions (30–45 min each). Practice drawing class diagrams and writing pseudocode before writing full code.

**Frequency guide:**
- ⭐⭐⭐ — Asked at almost every FAANG-tier mobile interview
- ⭐⭐ — Commonly asked, especially at product companies
- ⭐ — Niche, domain-specific (medical, automotive, fintech)

---

## Part I — HLD System Design Questions

### Messaging & Communication
| Question | Sections | Frequency |
|---|---|---|
| Design WhatsApp / a real-time messaging app | 02, 03, 05, 08, 25 | ⭐⭐⭐ |
| Design a group chat system (Slack/Teams) | 02, 05, 08, 25 | ⭐⭐⭐ |
| Design a read receipts system (delivered/seen) | 03, 05 | ⭐⭐ |
| Design push notifications for a messaging app | 05, 25 | ⭐⭐ |
| Design an in-app presence system (online/offline/typing) | 05 | ⭐⭐ |
| Design a VoIP/video calling feature (Zoom) | 05, 16 | ⭐⭐ |

### Social Media & Feeds
| Question | Sections | Frequency |
|---|---|---|
| Design Instagram (photo sharing + feed) | 01, 02, 03, 06, 20 | ⭐⭐⭐ |
| Design a social media feed (Twitter/X) | 02, 03, 05, 06 | ⭐⭐⭐ |
| Design Instagram Stories / Snapchat | 02, 05, 06 | ⭐⭐ |
| Design a news feed ranking system | 02, 11 | ⭐⭐ |
| Design a comments system with threading | 02, 03 | ⭐⭐ |
| Design a social media notification system | 05, 09, 25 | ⭐⭐ |

### Video & Media
| Question | Sections | Frequency |
|---|---|---|
| Design a video streaming app (YouTube/Netflix) | 02, 05, 06, 15, 20 | ⭐⭐⭐ |
| Design a music streaming app (Spotify) | 05, 06, 15, 16, 07 | ⭐⭐⭐ |
| Design video upload and processing pipeline | 02, 07, 12 | ⭐⭐ |
| Design a live streaming feature (Twitch) | 05, 15 | ⭐⭐ |
| Design an audio podcast player | 07, 15, 16 | ⭐⭐ |
| Design DRM-protected video playback | 10, 15 | ⭐ |

### Maps & Location
| Question | Sections | Frequency |
|---|---|---|
| Design a ride-sharing app (Uber/Lyft) | 05, 06, 07, 28 | ⭐⭐⭐ |
| Design real-time driver tracking | 05, 28 | ⭐⭐⭐ |
| Design a food delivery app (DoorDash) | 03, 05, 25, 28 | ⭐⭐ |
| Design nearby places search (Yelp) | 02, 28 | ⭐⭐ |
| Design turn-by-turn navigation | 07, 28 | ⭐⭐ |
| Design geofencing for a delivery app | 07, 28 | ⭐⭐ |

### E-Commerce & Payments
| Question | Sections | Frequency |
|---|---|---|
| Design an e-commerce app (Amazon) | 02, 03, 06, 11, 20 | ⭐⭐⭐ |
| Design a product catalog with search and filters | 02, 03, 20 | ⭐⭐ |
| Design a shopping cart (offline-capable) | 03, 04 | ⭐⭐ |
| Design a payment flow | 10, 08 | ⭐⭐ |
| Design a checkout funnel with analytics | 09, 11 | ⭐⭐ |
| Design a price alert / wishlist system | 03, 07 | ⭐ |

### Productivity & Storage
| Question | Sections | Frequency |
|---|---|---|
| Design a note-taking app (Notion/Evernote) that works offline | 03, 04, 08 | ⭐⭐⭐ |
| Design a file sync app (Dropbox/Google Drive) | 02, 03, 07, 10 | ⭐⭐ |
| Design a document collaboration editor | 03, 05, 08 | ⭐⭐ |
| Design a task manager (Todoist) | 03, 04 | ⭐⭐ |
| Design a calendar app with sync | 03, 05, 24 | ⭐ |

### Search & Discovery
| Question | Sections | Frequency |
|---|---|---|
| Design a search feature with autocomplete | 02, 26 | ⭐⭐⭐ |
| Design a type-ahead search (offline-first) | 03, 26 | ⭐⭐ |
| Design content recommendation system | 11, 26 | ⭐⭐ |
| Design a barcode / QR code scanner | 14 | ⭐⭐ |

### IoT & Hardware
| Question | Sections | Frequency |
|---|---|---|
| Design a smart home control app | 05, 14 | ⭐⭐ |
| Design a fitness tracker companion app (Fitbit) | 07, 14, 17 | ⭐⭐ |
| Design a BLE device pairing and communication system | 14 | ⭐⭐ |
| Design a contactless payment system (NFC) | 10, 14 | ⭐ |
| Design an audio processing pipeline (Alexa) | 16, 17, 26 | ⭐ |

### Platform & Infrastructure
| Question | Sections | Frequency |
|---|---|---|
| Design an A/B testing system for mobile | 11 | ⭐⭐⭐ |
| Design a feature flag system | 11 | ⭐⭐⭐ |
| Design a crash reporting system | 09 | ⭐⭐ |
| Design a mobile analytics system | 09 | ⭐⭐ |
| Design an image loading library | 20, 21 | ⭐⭐ |
| Design a mobile CI/CD pipeline | 12 | ⭐ |
| Design an app update mechanism | 12 | ⭐ |

---

## Part II — LLD Design Questions

### Architecture & Patterns
| Question | Sections | Frequency |
|---|---|---|
| Design the ViewModel for a paginated feed | 01, 02, 08 | ⭐⭐⭐ |
| Explain MVVM vs MVI — when would you choose each? | 01 | ⭐⭐⭐ |
| Design a clean architecture for a social media app | 01 | ⭐⭐⭐ |
| How would you modularize a large Android app? | 01, 12 | ⭐⭐⭐ |
| Design the Repository pattern with caching | 03, 04, 21 | ⭐⭐⭐ |
| Walk me through your dependency injection strategy | 01, 27 | ⭐⭐ |
| Design a Use Case layer — when is it worth it? | 01, 21 | ⭐⭐ |

### Networking & Data
| Question | Sections | Frequency |
|---|---|---|
| Design a network layer with retry and token refresh | 02 | ⭐⭐⭐ |
| How do you implement cursor-based pagination? | 02 | ⭐⭐⭐ |
| Design an offline-first architecture | 03 | ⭐⭐⭐ |
| Design a conflict resolution strategy for synced notes | 03 | ⭐⭐⭐ |
| Design a Room database schema for a messaging app | 04 | ⭐⭐ |
| Implement an LRU cache from scratch | 04, 21 | ⭐⭐⭐ |
| Design a cache invalidation strategy | 20 | ⭐⭐ |
| Design the image loading pipeline | 20 | ⭐⭐ |

### Concurrency & Performance
| Question | Sections | Frequency |
|---|---|---|
| Explain structured concurrency in Kotlin | 08 | ⭐⭐⭐ |
| What's the difference between launch and async? | 08 | ⭐⭐⭐ |
| How do you handle a race condition in coroutines? | 08 | ⭐⭐⭐ |
| Design a background sync that works in Doze mode | 07 | ⭐⭐ |
| What causes ANRs and how do you prevent them? | 06, 22 | ⭐⭐⭐ |
| Your app uses 800MB of RAM. How do you debug it? | 06, 22 | ⭐⭐⭐ |
| How do you make a scroll-driven animation not cause recomposition? | 23 | ⭐⭐ |
| What makes a Compose composable unskippable? | 23 | ⭐⭐ |

### Real-Time & State
| Question | Sections | Frequency |
|---|---|---|
| Design a WebSocket reconnection strategy | 05 | ⭐⭐ |
| How would you implement message ordering guarantees? | 05, 08 | ⭐⭐ |
| Design state management for a checkout flow (MVI) | 01, 18 | ⭐⭐ |
| What's StateFlow vs SharedFlow vs Channel? | 08, 18 | ⭐⭐⭐ |
| Design a ViewModel for a real-time collaborative document | 03, 05, 08 | ⭐⭐ |

### Security & Auth
| Question | Sections | Frequency |
|---|---|---|
| Where do you store auth tokens on Android? | 10 | ⭐⭐⭐ |
| What is certificate pinning and when would you use it? | 10 | ⭐⭐⭐ |
| How does Widevine DRM work? | 10, 15 | ⭐⭐ |
| How do you prevent MITM attacks? | 10 | ⭐⭐ |
| Design biometric authentication tied to Keystore | 10 | ⭐⭐ |

### Platform-Specific
| Question | Sections | Frequency |
|---|---|---|
| What's the difference between config change and process death? | 24 | ⭐⭐⭐ |
| How does ViewModel survive rotation? | 24 | ⭐⭐⭐ |
| When would you use SavedStateHandle? | 24 | ⭐⭐⭐ |
| What is repeatOnLifecycle and why was it introduced? | 24 | ⭐⭐⭐ |
| Design notification channels for a social media app | 25 | ⭐⭐ |
| How does FCM data message differ from notification message? | 05, 25 | ⭐⭐⭐ |
| How do you handle audio focus in a music app? | 16 | ⭐⭐ |
| Walk me through the AEC pipeline for a voice assistant | 17 | ⭐ |
| Design smooth driver marker animation for a ride-sharing app | 28 | ⭐⭐ |

### Design Patterns (LLD Classic)
| Question | Sections | Frequency |
|---|---|---|
| Draw the class diagram for the Repository pattern | 21 | ⭐⭐⭐ |
| What design patterns does OkHttp use? | 21 | ⭐⭐ |
| How would you implement undo/redo in a drawing app? | 21 | ⭐⭐ |
| Explain how StateFlow relates to the Observer pattern | 21 | ⭐⭐ |
| Design an image loading library (class diagram) | 20, 21 | ⭐⭐ |
| When would you use Strategy vs Chain of Responsibility? | 21 | ⭐⭐ |

### Testing
| Question | Sections | Frequency |
|---|---|---|
| How do you test a ViewModel? | 01, 27 | ⭐⭐⭐ |
| Espresso vs Robolectric — when would you use each? | 27 | ⭐⭐ |
| How do you test a Flow emission sequence? | 08, 27 | ⭐⭐ |
| How do you write a fake repository for testing? | 01, 21 | ⭐⭐⭐ |

---

## Part III — Deep Dive Questions (Asked at Senior / Staff Level)

These are follow-ups after you've given a good initial design:

### Architecture deep dives
- "How would your offline-first architecture handle a clock skew between client and server?"
- "Your modular app has 20 feature modules. Build times are 8 minutes. What do you do?"
- "How do you prevent circular dependencies between modules?"
- "How would you migrate a monolith Activity to Clean Architecture incrementally?"

### Scaling & performance deep dives
- "Your feed loads 50 items. A user has 2GB RAM. How do you keep memory under 200MB?"
- "How do you quantify the battery impact of your background sync?"
- "Your cold start is 3s. Walk me through exactly what's slow using Perfetto."
- "You have a memory leak that only happens after navigating 10 screens. How do you find it?"
- "How would you detect performance regressions before they ship?"

### Concurrency deep dives
- "Two coroutines read-modify-write the same counter. What are your options and trade-offs?"
- "You have a Mutex and a coroutine deadlocks. How did it happen and how do you fix it?"
- "Why can't you catch CancellationException with `catch (e: Exception)`?"
- "Your ViewModel launches a coroutine that reads a stale lambda. What's the fix?"

### Real-time deep dives
- "Your WebSocket server restarts. 10,000 clients reconnect simultaneously. What happens?"
- "How do you guarantee exactly-once message delivery in a chat app?"
- "A user goes offline for 3 days. How do you sync their state when they come back?"
- "How do you handle the token refresh race condition when 50 requests fail with 401 simultaneously?"

### Security deep dives
- "A user's device is rooted. What can an attacker do to your app and how do you mitigate it?"
- "Your cert pinning breaks because your CDN rotated its certificate. How do you recover without a hotfix?"
- "Walk me through exactly what happens during Widevine L1 video decryption."

---

## Part IV — Behavioural / Googleyness Questions with Technical Edge

These blur the line between behavioral and technical:

- "Tell me about a time you caught a production bug no one else could reproduce. How did you debug it?"
- "Describe a time you had to make a performance trade-off. What did you measure and how?"
- "Tell me about a system you designed that had to be changed significantly. What would you do differently?"
- "How do you approach a problem where you don't know which tool to use?"
- "Describe a time you pushed back on a feature because of technical concerns. What was the outcome?"

---

## Part V — Company-Specific Focus Areas

### Google
- Jetpack Compose internals, recomposition, stability
- Android Vitals thresholds and optimization
- Baseline Profiles and startup performance
- Accessibility (WCAG 2.1 AA, EAA 2025)
- Material 3 design system compliance

### Amazon / Alexa
- Audio pipeline (AEC, NS, AGC, VAD)
- DRM encrypted media streaming
- Offline-first with guaranteed sync
- WorkManager and battery optimization
- Fire OS compatibility (AOSP fork nuances)

### Meta / Facebook
- React Native performance (Reanimated, New Architecture, JSI)
- Feed rendering at scale (Litho, FlatList optimization)
- Real-time messaging (WebSocket, presence)
- A/B testing at scale

### Uber / Lyft
- Real-time location tracking
- Map rendering and smooth marker animation
- Surge pricing UI (real-time updates)
- Offline-capable booking flow

### Netflix / Streaming companies
- HLS/DASH adaptive bitrate
- Widevine DRM L1/L2/L3
- Offline download with DRM
- Startup time optimization (time-to-first-frame)

### Fintech (Stripe, Square, PayPal)
- Security: certificate pinning, token storage, biometric auth
- Offline-capable payment flows
- PCI compliance constraints on local storage
- Fraud detection signals

---

## HLD Answer Framework (for any design question)

```
1. CLARIFY (2–3 min)
   → Functional requirements: core features (top 3)
   → Non-functional: scale (users, requests/sec), offline?, real-time?
   → Platforms: Android only? iOS? Web?
   → Constraints: latency targets, battery sensitivity, data limits

2. DEFINE COMPONENTS (5 min)
   → Data models / entities (draw on whiteboard)
   → API contract (REST/GraphQL/WebSocket — justify)
   → Storage strategy (local DB, cache, offline policy)

3. HIGH-LEVEL ARCHITECTURE (10 min)
   → Client layers: UI → ViewModel → Repository → DataSource
   → Networking: API, real-time channel, CDN
   → Local storage: Room schema (sketch), caching strategy
   → Background work: sync, notifications

4. DEEP DIVES (15 min)
   → Pick 2–3 of: offline sync, real-time, performance, security, error handling
   → Show tradeoffs, not just happy path
   → Mention production failure modes proactively

5. TRADEOFFS (5 min)
   → What did you deprioritize and why?
   → What would you do differently at 10x scale?
   → What's the biggest risk in your design?
```

---

## LLD Answer Framework

```
1. UNDERSTAND THE SCOPE (2 min)
   → What class/module are you designing?
   → What are the inputs and outputs?
   → What does the caller look like?

2. DEFINE THE INTERFACE (5 min)
   → Public API: method signatures, return types
   → Interface vs abstract class vs concrete class decision

3. DRAW CLASS DIAGRAM (5–10 min)
   → Key classes and their relationships
   → Identify which design patterns you're applying

4. WALK THROUGH A SCENARIO (10 min)
   → "When the user taps X, here's what happens..."
   → Sequence diagram for async flows
   → Error cases (not just happy path)

5. CODE KEY PARTS (10 min)
   → Don't try to code everything — code the interesting parts
   → State management, error handling, concurrency

6. DISCUSS TRADEOFFS (5 min)
   → Thread safety? Testability? Extensibility?
   → What would change at larger scale?
```
