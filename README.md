# Mobile System Design & LLD Reference Guide

A comprehensive reference for Android (Kotlin/Jetpack Compose) and React Native mobile interview preparation — covering both High-Level Design (HLD) system design rounds and Low-Level Design (LLD) coding/architecture rounds.

Modeled after Hello Interview but tailored specifically for mobile engineers targeting Android SDE II–III roles at FAANG-tier companies.

---

## Who Should Use This

**Primary audience:** Android engineers with 3–8 years of experience preparing for senior or staff-level interviews at Google, Amazon, Meta, Apple, Uber, Airbnb, Netflix, Lyft, or similar companies.

**You'll get the most value if:**
- You have production Android experience but haven't interviewed in 2–4+ years
- You've built features within existing apps and now need to design systems end-to-end
- You understand Kotlin and Jetpack Compose but want to articulate *why* behind every choice
- You need a reference that covers both "design WhatsApp" (HLD) and "design a ViewModel for a paginated feed" (LLD) questions

---

## How to Use This Guide

### Identifying your interview type

Every section is tagged:

| Badge | Meaning |
|---|---|
| **[HLD]** | System design round — "design X at scale", draw architecture diagrams |
| **[LLD]** | Coding/design round — class diagrams, implementation decisions, code |
| **[Both]** | Appears in both contexts |

### Study approach

**If you have 4+ weeks:**
Read all sections sequentially. Each section builds on concepts from earlier ones. Do the Q&A at the end of each section out loud without looking at the answers.

**If you have 2 weeks:**
Focus on Part A (sections 01–13) for system design rounds, and Part B sections specific to your target company's domain (media → 15–17, maps → 28, ML → 26).

**If you have 1 week:**
- Architecture (01), Networking (02), Offline-First (03), Threading (08) — fundamentals tested everywhere
- The section most relevant to the company's product
- Design Patterns (21) and Lifecycle (24) — asked in nearly every LLD round

**Daily prep pattern:**
- Morning: Read one section
- Midday: Answer the Q&A without looking
- Evening: Review where you stumbled

---

## Table of Contents

### Part A — Mobile System Design

| # | Section | Round | Key topics |
|---|---|---|---|
| 01 | [Architecture Patterns](01-architecture-patterns.md) | Both | MVC→MVI, Clean Architecture, Modularization, DFMs |
| 02 | [Networking & API Design](02-networking-api.md) | Both | REST/GraphQL/gRPC, OkHttp, retry, pagination, BFF |
| 03 | [Offline-First & Sync](03-offline-sync.md) | Both | SSOT, conflict resolution, outbox pattern, delta sync |
| 04 | [Data Layer — DB & Cache](04-data-layer.md) | Both | Room, DataStore (Tink), LRU, migrations, OkHttp cache |
| 05 | [Real-Time & Push](05-realtime-push.md) | Both | WebSocket, SSE, FCM, heartbeat, presence, scaling |
| 06 | [Performance](06-performance.md) | Both | Startup (TTID/TTFD), frame budget, memory, Baseline Profiles |
| 07 | [Background Work & Battery](07-background-battery.md) | Both | WorkManager, Doze, foreground services, App Standby |
| 08 | [Threading & Concurrency](08-threading-concurrency.md) | LLD | Coroutines, dispatchers, race conditions, Flow vs Channel |
| 09 | [Observability](09-observability.md) | HLD | Crashlytics, Bugsnag, Sentry, Firebase Perf, analytics, alerting |
| 10 | [Security](10-security.md) | Both | Tink (replaces EncryptedSharedPreferences), cert pinning, Widevine, Play Integrity |
| 11 | [Feature Flags, A/B & Remote Config](11-feature-flags.md) | HLD | Firebase Remote Config, kill switches, staged rollout, A/B testing |
| 12 | [App Distribution & Size](12-app-distribution.md) | HLD | AAB vs APK, DFMs, R8, size budget, in-app updates |
| 13 | [Cross-Cutting Concerns](13-cross-cutting.md) | HLD | i18n/RTL, App Links, accessibility (EAA 2025), fragmentation |

### Part B — Deep Technical Concepts

| # | Section | Round | Key topics |
|---|---|---|---|
| 14 | [Bluetooth & NFC](14-bluetooth-nfc.md) | LLD | BLE GATT, scanning, status 133, NDEF, HCE |
| 15 | [Video Playback & DRM](15-video-drm.md) | Both | Media3/ExoPlayer, HLS/DASH, Widevine L1/L2/L3, offline downloads |
| 16 | [Audio Management & Focus](16-audio-management.md) | LLD | AudioFocus, ducking, MediaSession, headphone disconnect, SoundPool |
| 17 | [Audio Input Stream Manipulation](17-audio-input.md) | LLD | Recording pipeline, AEC, NS, AGC, VAD, hesitation filtering |
| 18 | [State Management](18-state-management.md) | LLD | UDF, Compose side effects, Redux/Zustand/Jotai (RN) |
| 19 | [Animation](19-animation.md) | LLD | Compose APIs, graphicsLayer, Shared Elements, Reanimated 3 |
| 20 | [Caching Deep Dive](20-caching.md) | Both | HTTP cache, Coil/Glide pipeline, DiskLruCache, invalidation strategies |
| 21 | [Design Patterns & Diagrams](21-design-patterns.md) | LLD | Repository, Observer, Strategy, Command, class diagrams, sequence diagrams |
| 22 | [Debugging, Profiling & Tooling](22-debugging.md) | LLD | Profiler, Perfetto, LeakCanary, StrictMode, ADB, Flipper |
| 23 | [Jetpack Compose Optimization](23-compose-optimization.md) | LLD | Stability, Compiler Reports, Strong Skipping, derivedStateOf, LazyList |
| 24 | [Lifecycle Management](24-lifecycle.md) | LLD | Activity/Fragment lifecycle, ViewModel scoping, SavedStateHandle, repeatOnLifecycle |
| 25 | [Notifications](25-notifications.md) | Both | Channels, FCM data messages, styles, deep links, POST_NOTIFICATIONS |
| 26 | [On-Device ML & Local LLMs](26-on-device-ml-llm.md) | Both | TFLite, ML Kit, Gemini Nano/AICore, VAD, predictive text |
| 27 | [Common Libraries & Dependency Management](27-common-libraries.md) | LLD | Gson/Moshi/KotlinX, Hilt/Koin, Coil/Glide, testing stack, Gradle, AAB size |
| 28 | [Maps Integration](28-maps-integration.md) | Both | Google Maps/HERE/Mapbox comparison, ride-sharing pipeline, smooth marker animation, geofencing |

---

## Section Format

Every section follows a consistent structure:

1. **Round type badge** — HLD / LLD / Both
2. **Why this matters** — what interviewers are really testing
3. **Common interview angles** — exact questions you'll be asked
4. **Concept explanation** — internals, not just API surface
5. **Android (Kotlin/Compose) implementation** — production-quality code
6. **React Native equivalent** — where applicable
7. **HLD framing** — how to answer system design questions
8. **LLD framing** — how to answer implementation questions
9. **Common misunderstandings & pitfalls** — what trips up candidates
10. **Best practices** — what senior engineers do differently
11. **Q&A** — 5–8 questions with full answers

---

## Contributing / Updating

- Each section is a standalone `.md` file — update independently
- Code examples target: Kotlin 2.0+, Compose 1.7+, Media3 1.10+, AGP 8.x
- RN examples target: React Native 0.76+, Reanimated 3
- Flag outdated APIs with `⚠️ Deprecated` notices (see section 10 for example)

---

## Quick Reference — Interview Question to Section Mapping

| "Design a..." | Section(s) |
|---|---|
| Chat / messaging app | 02, 03, 05, 08, 25 |
| Video streaming app (Netflix/YouTube) | 02, 05, 15, 20 |
| Ride-sharing app (Uber) | 05, 06, 28 |
| Music player (Spotify) | 15, 16, 07 |
| Social media feed (Instagram) | 01, 02, 03, 06, 20 |
| E-commerce app (Amazon) | 02, 03, 11, 20 |
| Voice assistant (Alexa) | 16, 17, 26 |
| Maps / navigation app | 07, 28 |
| Note-taking app (offline-first) | 03, 04, 08 |
| File sync (Dropbox) | 03, 07, 10 |
| Authentication flow | 10, 24 |
| Image loading library | 20, 21 |
| A/B testing system | 11 |
| Crash reporting system | 09 |
