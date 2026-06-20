# 15 — Video Playback & DRM

> **Round type: Both (HLD + LLD)**
> **HLD:** Designing a streaming video architecture — CDN, adaptive bitrate, DRM licensing flow
> **LLD:** Media3/ExoPlayer setup, HLS/DASH, Widevine, MediaSession, background playback, offline downloads, Compose integration

---

## 1. Why This Matters in Interviews

Video playback is complex enough to fill an entire interview. Companies like Amazon (Prime Video), Netflix, YouTube, Spotify, Disney+, and any media-focused product test this deeply. You worked on Fire Tablet DRM encrypted audio streaming — this section formalizes that knowledge for interviews.

**Common interview angles:**
- "How does adaptive bitrate streaming work?"
- "What is the difference between HLS and DASH?"
- "How does Widevine DRM work?"
- "What is the difference between Widevine L1, L2, and L3?"
- "How do you implement background audio/video playback?"
- "How do you support offline video downloads?"
- "What causes buffering and how do you minimize it?"
- "How would you design the video playback system for a Netflix-like app?"

---

## 2. The Video Playback Stack

```
┌──────────────────────────────────────────────────────────┐
│                    Playback Stack                         │
├─────────────────────────────┬────────────────────────────┤
│   CONTENT DELIVERY          │   ANDROID PLAYER           │
│   CDN (CloudFront, Akamai)  │   Media3 / ExoPlayer       │
│   → Video segments          │   → Adaptive bitrate       │
│   → Manifest files          │   → Format parsing         │
│   → DRM license server      │   → DRM decryption         │
│   → Subtitle tracks         │   → Audio/video sync       │
│   → Multiple renditions     │   → MediaSession           │
│   (360p, 720p, 1080p, 4K)   │   → PlayerSurface          │
└─────────────────────────────┴────────────────────────────┘
```

---

## 3. Streaming Protocols

### 3.1 HLS vs DASH

| | HLS (HTTP Live Streaming) | DASH (Dynamic Adaptive Streaming over HTTP) |
|---|---|---|
| Origin | Apple (2009) | MPEG standard (2012) |
| Manifest | `.m3u8` (M3U8 playlist) | `.mpd` (XML descriptor) |
| Segment format | `.ts` (MPEG-TS) or `.fmp4` | `.m4s` (fragmented MP4) |
| DRM | FairPlay (native), Widevine via Media3 | Widevine, PlayReady, FairPlay |
| iOS/Safari support | ✅ Native | 🟡 Via Media Source Extensions |
| Android support | ✅ Media3 | ✅ Media3 |
| Latency (standard) | 6–30 seconds | 2–10 seconds |
| Low-latency | HLS LL (< 2s) | DASH LL (< 2s) |
| Codec flexibility | Good | Better (multiple codec groups) |
| Industry adoption | Netflix (migrated from), Apple, Twitch | YouTube, Netflix, Amazon Prime |
| Best for | Apple-first / cross-platform | Android-first, flexible encoding |

**Key difference in manifest:**
```
HLS .m3u8 (Master Playlist):
  #EXTM3U
  #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
  360p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=2400000,RESOLUTION=1280x720
  720p/playlist.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
  1080p/playlist.m3u8

DASH .mpd (Media Presentation Description):
  <AdaptationSet mimeType="video/mp4">
    <Representation id="1" bandwidth="800000" width="640" height="360"/>
    <Representation id="2" bandwidth="2400000" width="1280" height="720"/>
    <Representation id="3" bandwidth="5000000" width="1920" height="1080"/>
  </AdaptationSet>
```

### 3.2 Adaptive Bitrate (ABR) Streaming

ABR is the core technique that makes video streaming work on mobile:

```
Server encodes video at multiple bitrates (renditions):
  360p  @ 400  Kbps  → good for 3G
  480p  @ 800  Kbps
  720p  @ 2.5  Mbps  → good for LTE
  1080p @ 5    Mbps  → good for WiFi
  1080p @ 8    Mbps  → premium
  4K    @ 20   Mbps  → fast WiFi

Player algorithm (ABR algorithm) runs every segment (~2–6s):
  Measure current bandwidth → select rendition that fits → download next segment
  If bandwidth drops: switch down rendition mid-stream (seamless if buffer has margin)
  If bandwidth rises: switch up rendition

Buffer management:
  MIN_BUFFER_MS: keep at least this much buffered
  MAX_BUFFER_MS: don't buffer more than this (save memory + data)
  BUFFER_FOR_PLAYBACK_MS: minimum to start playing (affects startup latency)
```

---

## 4. Media3 / ExoPlayer — The Standard

### 4.1 Status

- **ExoPlayer 2.x (`com.google.android.exoplayer2`)** — deprecated, no new features
- **Media3 (`androidx.media3`)** — the current standard, actively developed
- **MediaPlayer (Android framework)** — obsolete for anything beyond basic playback

All development now happens in `androidx.media3`, which reached version 1.10 (stable as of March 2026).

### 4.2 Dependencies (Selective Import)

```kotlin
// Only include modules you actually use
val media3Version = "1.10.1"

// Core — always needed
implementation("androidx.media3:media3-exoplayer:$media3Version")
implementation("androidx.media3:media3-ui:$media3Version")            // PlayerView, StyledPlayerView
implementation("androidx.media3:media3-session:$media3Version")       // MediaSession integration

// Format support — import only what you stream
implementation("androidx.media3:media3-exoplayer-hls:$media3Version")    // .m3u8
implementation("androidx.media3:media3-exoplayer-dash:$media3Version")   // .mpd

// Optional
implementation("androidx.media3:media3-datasource-okhttp:$media3Version")// OkHttp for requests
implementation("androidx.media3:media3-exoplayer-smoothstreaming:$media3Version") // Microsoft SS
```

### 4.3 Player Architecture

```
ExoPlayer (top-level player interface)
├── Renderers          ← decode and render audio/video/text
│   ├── MediaCodecVideoRenderer  ← video decoding (hardware accelerated)
│   ├── MediaCodecAudioRenderer  ← audio decoding
│   └── TextRenderer             ← subtitles
├── TrackSelector      ← which track to play (quality, language, accessibility)
├── LoadControl        ← buffering strategy
└── DataSource         ← fetch bytes (HTTP, cache, local file)
    └── MediaSource    ← parse format (HLS, DASH, ProgressiveMedia)
```

### 4.4 Basic Setup with ViewModel

```kotlin
// ViewModel manages player lifecycle — survives rotation
@HiltViewModel
class VideoViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel() {

    val player: ExoPlayer = ExoPlayer.Builder(context)
        .setTrackSelector(
            DefaultTrackSelector(context).apply {
                setParameters(
                    buildUponParameters()
                        .setPreferredAudioLanguage("en")
                        .setPreferredTextLanguage("en")
                        .setMaxVideoSizeSd()  // cap at SD on metered network — override if needed
                )
            }
        )
        .setLoadControl(
            DefaultLoadControl.Builder()
                .setBufferDurationsMs(
                    DefaultLoadControl.DEFAULT_MIN_BUFFER_MS,  // 50s
                    DefaultLoadControl.DEFAULT_MAX_BUFFER_MS,  // 50s
                    DefaultLoadControl.DEFAULT_BUFFER_FOR_PLAYBACK_MS,  // 2.5s to start
                    DefaultLoadControl.DEFAULT_BUFFER_FOR_PLAYBACK_AFTER_REBUFFER_MS  // 5s after rebuffer
                )
                .build()
        )
        .build()

    fun playVideo(url: String, startPositionMs: Long = 0) {
        val mediaItem = MediaItem.fromUri(url)
        player.setMediaItem(mediaItem, startPositionMs)
        player.prepare()
        player.playWhenReady = true
    }

    fun playHlsStream(hlsUrl: String) {
        val mediaItem = MediaItem.Builder()
            .setUri(hlsUrl)
            .setMimeType(MimeTypes.APPLICATION_M3U8)
            .build()
        player.setMediaItem(mediaItem)
        player.prepare()
        player.playWhenReady = true
    }

    fun savePlaybackPosition(): Long = player.currentPosition

    override fun onCleared() {
        player.release()  // ALWAYS release — frees codec resources, audio focus
        super.onCleared()
    }
}
```

### 4.5 Compose Integration

Media3 1.10 introduced new Compose-first APIs: `PlayerSurface`, `ContentFrame`, and `rememberPresentationState`, along with Material3 composables like `PlayPauseButton`, `ProgressSlider`, and a full `Player` composable.

```kotlin
// Modern approach — Media3 1.10 Compose APIs
@Composable
fun VideoPlayerScreen(viewModel: VideoViewModel = hiltViewModel()) {
    val player = viewModel.player

    // Track player state reactively
    var isPlaying by remember { mutableStateOf(player.isPlaying) }
    var currentPosition by remember { mutableLongStateOf(player.currentPosition) }
    var duration by remember { mutableLongStateOf(player.duration) }

    DisposableEffect(player) {
        val listener = object : Player.Listener {
            override fun onIsPlayingChanged(playing: Boolean) { isPlaying = playing }
            override fun onPlaybackStateChanged(state: Int) {
                currentPosition = player.currentPosition
                duration = player.duration
            }
        }
        player.addListener(listener)
        onDispose { player.removeListener(listener) }
    }

    Column {
        // Video surface — use AndroidView for older Media3 or PlayerSurface for 1.10+
        AndroidView(
            factory = { context ->
                PlayerView(context).apply {
                    this.player = player
                    useController = false  // use custom controls below
                    resizeMode = AspectRatioFrameLayout.RESIZE_MODE_FIT
                }
            },
            modifier = Modifier
                .fillMaxWidth()
                .aspectRatio(16f / 9f)
        )

        // Custom controls
        Row(
            modifier = Modifier.fillMaxWidth().padding(16.dp),
            horizontalArrangement = Arrangement.SpaceEvenly
        ) {
            IconButton(onClick = { player.seekBack() }) {
                Icon(Icons.Default.Replay10, contentDescription = "Rewind 10s")
            }
            IconButton(onClick = {
                if (player.isPlaying) player.pause() else player.play()
            }) {
                Icon(
                    if (isPlaying) Icons.Default.Pause else Icons.Default.PlayArrow,
                    contentDescription = if (isPlaying) "Pause" else "Play"
                )
            }
            IconButton(onClick = { player.seekForward() }) {
                Icon(Icons.Default.Forward10, contentDescription = "Forward 10s")
            }
        }

        // Progress slider
        if (duration > 0) {
            Slider(
                value = currentPosition.toFloat() / duration.toFloat(),
                onValueChange = { player.seekTo((it * duration).toLong()) },
                modifier = Modifier.fillMaxWidth().padding(horizontal = 16.dp)
            )
        }
    }
}
```

---

## 5. DRM — Digital Rights Management

### 5.1 What DRM Does

DRM prevents unauthorized copying and playback of premium content. The video stream is encrypted on the server. The device must obtain a license key to decrypt it. Without the key, the stream is unplayable.

```
Content Flow:
  Studio content → encode → encrypt with content key → upload to CDN
  License server holds content keys, issues them to authorized devices

Playback Flow:
  1. App requests video URL → receives encrypted stream URL + license server URL
  2. ExoPlayer fetches encrypted video segments from CDN
  3. ExoPlayer sends license request to license server (with device attestation)
  4. License server validates request → returns encrypted content key
  5. Content key decrypted using device's hardware-backed key
  6. Video segments decrypted → decoded → rendered
```

### 5.2 DRM Systems

| | Widevine (Google) | FairPlay (Apple) | PlayReady (Microsoft) |
|---|---|---|---|
| Platform | Android, Chrome | iOS, Safari, tvOS | Windows, Xbox, some Android |
| Standard | MPEG-DASH/HLS | HLS only | DASH/HLS |
| Android support | ✅ Built-in | ❌ | 🟡 Via plugin |
| Key exchange | MPEG-CENC | Custom | MPEG-CENC |
| Used by | Netflix, Amazon, YouTube | Netflix iOS, Apple TV+ | Microsoft, Xbox |

**On Android: Widevine is the standard.** Every Android device ships with Widevine CDM (Content Decryption Module).

### 5.3 Widevine Security Levels

| Level | Security | Hardware TEE? | Use case |
|---|---|---|---|
| **L1** | Highest | ✅ Yes (hardware-backed) | HD/4K protected content |
| **L2** | Medium | Partial | Limited content protection |
| **L3** | Software only | ❌ No | SD content, emulators, unverified devices |

**L1 requirements:**
- Content decryption happens in hardware Trusted Execution Environment (TEE)
- Decoded video never passes through application memory
- Required by Netflix, Prime Video, Disney+ for HD/4K streams
- Most flagship devices are L1; some budget devices are L3

```kotlin
// Check Widevine security level at runtime
fun getWidevineSecurityLevel(): String {
    return try {
        val mediaDrm = MediaDrm(UUID.fromString("edef8ba9-79d6-4ace-a3c8-27dcd51d21ed"))
        val level = mediaDrm.getPropertyString("securityLevel")
        mediaDrm.close()
        level  // "L1", "L2", or "L3"
    } catch (e: Exception) {
        "Unknown"
    }
}

// Adapt content based on security level
val maxQuality = when (getWidevineSecurityLevel()) {
    "L1" -> "4K"    // or 1080p with HDR
    "L3" -> "480p"  // SD only for software-only decryption
    else -> "720p"
}
```

### 5.4 DRM with Media3

```kotlin
// DRM-protected DASH stream
fun playDrmContent(
    dashUrl: String,
    licenseServerUrl: String,
    licenseToken: String     // auth token for license server
) {
    val drmConfiguration = MediaItem.DrmConfiguration.Builder(
        C.WIDEVINE_UUID  // edef8ba9-79d6-4ace-a3c8-27dcd51d21ed
    )
    .setLicenseUri(licenseServerUrl)
    .setLicenseRequestHeaders(
        mapOf(
            "Authorization" to "Bearer $licenseToken",
            "Content-Type" to "application/octet-stream"
        )
    )
    // Optional: multi-session for offline
    .setMultiSession(false)
    // Force software decryption for testing (L3 mode — never in production)
    // .forceDefaultLicenseUri(true)
    .build()

    val mediaItem = MediaItem.Builder()
        .setUri(dashUrl)
        .setMimeType(MimeTypes.APPLICATION_MPD)
        .setDrmConfiguration(drmConfiguration)
        .build()

    player.setMediaItem(mediaItem)
    player.prepare()
    player.playWhenReady = true
}

// Custom license interceptor (to handle token refresh, custom headers)
class CustomDrmSessionManagerProvider(
    private val tokenProvider: () -> String
) : DrmSessionManagerProvider {
    override fun get(mediaItem: MediaItem): DrmSessionManager {
        val httpDataSourceFactory = DefaultHttpDataSource.Factory()
            .setDefaultRequestProperties(mapOf(
                "Authorization" to "Bearer ${tokenProvider()}"
            ))

        return DefaultDrmSessionManager.Builder()
            .setMultiSession(false)
            .build(
                FrameworkMediaDrm.DEFAULT_PROVIDER,
                DefaultDrmCallback(
                    mediaItem.localConfiguration?.drmConfiguration?.licenseUri?.toString() ?: "",
                    HttpMediaDrmCallback(
                        mediaItem.localConfiguration?.drmConfiguration?.licenseUri?.toString(),
                        httpDataSourceFactory
                    )
                )
            )
    }
}
```

### 5.5 Secure Decoder Surface (L1 Requirement)

For L1 DRM, video must be rendered on a secure surface — decoded frames never enter app memory:

```kotlin
// PlayerView automatically handles secure surface for DRM content
// If using SurfaceView directly:
val surfaceView = SurfaceView(context)
surfaceView.holder.setSecure(true)  // marks surface as secure
player.setVideoSurfaceView(surfaceView)

// NEVER use TextureView for DRM content
// TextureView copies frames through app memory → DRM violation, black screen
// Always use SurfaceView (or PlayerView which uses SurfaceView internally)
```

---

## 6. MediaSession & Background Playback

### 6.1 Why MediaSession

`MediaSession` connects your player to the Android system:
- **Lock screen controls** — play/pause from lock screen
- **Notification with media controls** — persistent notification while playing
- **Bluetooth/headset button handling** — media button commands
- **Assistant integration** — "Hey Google, pause" works on your app
- **Android Auto / Wear OS** — playback control from car/watch

```kotlin
// In a MediaSessionService (recommended for background audio)
class PlaybackService : MediaSessionService() {
    private lateinit var player: ExoPlayer
    private lateinit var mediaSession: MediaSession

    override fun onCreate() {
        super.onCreate()
        player = ExoPlayer.Builder(this).build()

        mediaSession = MediaSession.Builder(this, player)
            .setCallback(object : MediaSession.Callback {
                override fun onAddMediaItems(
                    mediaSession: MediaSession,
                    controller: MediaSession.ControllerInfo,
                    mediaItems: List<MediaItem>
                ): ListenableFuture<List<MediaItem>> {
                    // Resolve MediaItem URIs here if needed
                    val resolvedItems = mediaItems.map { item ->
                        item.buildUpon().setUri(resolveUri(item.requestMetadata.mediaUri)).build()
                    }
                    return Futures.immediateFuture(resolvedItems)
                }
            })
            .build()
    }

    // MediaSessionService creates notification automatically from MediaSession
    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo) = mediaSession

    override fun onDestroy() {
        mediaSession.release()
        player.release()
        super.onDestroy()
    }
}

// Connect from UI layer
@Composable
fun VideoScreen() {
    val context = LocalContext.current
    var player by remember { mutableStateOf<Player?>(null) }

    DisposableEffect(Unit) {
        val sessionToken = SessionToken(context,
            ComponentName(context, PlaybackService::class.java))
        val controllerFuture = MediaController.Builder(context, sessionToken)
            .buildAsync()
        controllerFuture.addListener({
            player = controllerFuture.get()
        }, ContextCompat.getMainExecutor(context))

        onDispose {
            MediaController.releaseFuture(controllerFuture)
            player = null
        }
    }
    // Use player in PlayerView
}
```

```xml
<!-- AndroidManifest.xml -->
<service android:name=".PlaybackService"
    android:foregroundServiceType="mediaPlayback"  <!-- required Android 14+ -->
    android:exported="true">
    <intent-filter>
        <action android:name="androidx.media3.session.MediaSessionService"/>
    </intent-filter>
</service>
```

---

## 7. Offline Downloads

### 7.1 Download Architecture

```kotlin
// Setup download manager
val downloadCache = SimpleCache(
    File(context.filesDir, "video_downloads"),
    LeastRecentlyUsedCacheEvictor(500L * 1024 * 1024),  // 500MB max
    StandaloneDatabaseProvider(context)
)

val downloadManager = DownloadManager(
    context,
    StandaloneDatabaseProvider(context),
    downloadCache,
    DefaultHttpDataSource.Factory(),
    Executors.newFixedThreadPool(6)  // parallel download threads
)

// Download a DASH stream
val mediaItem = MediaItem.Builder()
    .setUri(dashUrl)
    .setMimeType(MimeTypes.APPLICATION_MPD)
    .build()

val downloadRequest = DownloadRequest.Builder(
    contentId,                         // unique ID for this download
    mediaItem.requestMetadata.mediaUri!!
)
.setMimeType(MimeTypes.APPLICATION_MPD)
.setStreamKeys(
    listOf(
        StreamKey(0, 0),    // video track 0 (SD rendition for storage efficiency)
        StreamKey(1, 0)     // audio track 0
    )
)
.build()

DownloadService.sendAddDownload(context, VideoDownloadService::class.java, downloadRequest, false)
```

```kotlin
// DownloadService
class VideoDownloadService : DownloadService(
    FOREGROUND_NOTIFICATION_ID,
    DEFAULT_FOREGROUND_NOTIFICATION_UPDATE_INTERVAL,
    CHANNEL_ID,
    R.string.download_channel_name,
    R.string.download_channel_description
) {
    override fun getDownloadManager(): DownloadManager = downloadManager
    override fun getScheduler(): Scheduler? = null
    override fun getForegroundNotification(
        downloads: List<Download>,
        notMetRequirements: Int
    ): Notification {
        return DownloadNotificationHelper(this, CHANNEL_ID)
            .buildProgressNotification(
                this, R.drawable.ic_download, null, null, downloads, notMetRequirements
            )
    }
}

// Play offline content
val cacheDataSourceFactory = CacheDataSource.Factory()
    .setCache(downloadCache)
    .setUpstreamDataSourceFactory(DefaultHttpDataSource.Factory())
    .setCacheWriteDataSinkFactory(null)  // read-only (already downloaded)
    .setFlags(CacheDataSource.FLAG_IGNORE_CACHE_ON_ERROR)

val offlinePlayer = ExoPlayer.Builder(context)
    .setMediaSourceFactory(DefaultMediaSourceFactory(cacheDataSourceFactory))
    .build()
```

---

## 8. Track Selection & Quality Control

```kotlin
// Force specific quality (e.g., user selects 720p)
val trackSelector = DefaultTrackSelector(context)

fun setVideoQuality(maxWidth: Int, maxHeight: Int) {
    trackSelector.setParameters(
        trackSelector.buildUponParameters()
            .setMaxVideoSize(maxWidth, maxHeight)      // cap resolution
            .setMaxVideoBitrate(4_000_000)             // cap bitrate
    )
}

// Get available tracks for quality picker UI
val mappedTrackInfo = trackSelector.currentMappedTrackInfo
val videoGroup = mappedTrackInfo?.getTrackGroups(/* rendererIndex= */ 0)
val availableResolutions = videoGroup?.let { groups ->
    (0 until groups.length).flatMap { groupIndex ->
        val group = groups[groupIndex]
        (0 until group.length).map { trackIndex ->
            val format = group.getFormat(trackIndex)
            "${format.width}x${format.height} @ ${format.bitrate / 1000}kbps"
        }
    }
}

// Adaptive — let ExoPlayer decide (default)
trackSelector.setParameters(
    trackSelector.buildUponParameters()
        .clearVideoSizeConstraints()
        .clearVideoBitrateConstraints()
)
```

---

## 9. Player Events & Buffering Metrics

```kotlin
player.addListener(object : Player.Listener {

    override fun onPlaybackStateChanged(state: Int) {
        when (state) {
            Player.STATE_IDLE     -> handleIdle()       // not prepared
            Player.STATE_BUFFERING -> handleBuffering() // loading data
            Player.STATE_READY    -> handleReady()      // can play
            Player.STATE_ENDED    -> handleEnded()      // finished
        }
    }

    override fun onPlayerError(error: PlaybackException) {
        when (error.errorCode) {
            PlaybackException.ERROR_CODE_IO_NETWORK_CONNECTION_FAILED ->
                showError("No network connection")
            PlaybackException.ERROR_CODE_BEHIND_LIVE_WINDOW ->
                player.seekToDefaultPosition()  // live stream: seek to live edge
            PlaybackException.ERROR_CODE_DRM_LICENSE_EXPIRED ->
                refreshDrmLicense()
            PlaybackException.ERROR_CODE_DRM_CONTENT_ERROR ->
                showError("Cannot play protected content on this device")
            else ->
                showError("Playback error: ${error.message}")
        }
    }

    override fun onVideoSizeChanged(videoSize: VideoSize) {
        // Adjust aspect ratio container when video dimensions are known
        updateAspectRatio(videoSize.width.toFloat() / videoSize.height)
    }

    override fun onTracksChanged(tracks: Tracks) {
        // Detect if content is DRM-protected
        val isDrmProtected = tracks.groups.any { group ->
            group.type == C.TRACK_TYPE_VIDEO &&
            group.mediaTrackGroup.getFormat(0).drmInitData != null
        }
    }
})

// Analytics: track buffering events
class BufferingAnalytics(private val analytics: Analytics) : AnalyticsListener {
    private var bufferStartTime = 0L

    override fun onLoadStarted(
        eventTime: AnalyticsListener.EventTime,
        loadEventInfo: LoadEventInfo,
        mediaLoadData: MediaLoadData
    ) {
        if (mediaLoadData.dataType == C.DATA_TYPE_MEDIA) {
            bufferStartTime = System.currentTimeMillis()
        }
    }

    override fun onLoadCompleted(
        eventTime: AnalyticsListener.EventTime,
        loadEventInfo: LoadEventInfo,
        mediaLoadData: MediaLoadData
    ) {
        val loadDurationMs = System.currentTimeMillis() - bufferStartTime
        analytics.trackEvent("video_segment_loaded", mapOf(
            "duration_ms" to loadDurationMs,
            "bytes" to loadEventInfo.bytesLoaded,
            "uri" to loadEventInfo.uri.toString()
        ))
    }
}
player.addAnalyticsListener(BufferingAnalytics(analytics))
```

---

## 10. Picture-in-Picture (PiP)

```kotlin
// Enable PiP when user presses home during video
class VideoActivity : AppCompatActivity() {
    override fun onUserLeaveHint() {
        enterPiPMode()
    }

    private fun enterPiPMode() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val params = PictureInPictureParams.Builder()
                .setAspectRatio(Rational(16, 9))
                .setActions(listOf(
                    RemoteAction(
                        Icon.createWithResource(this, R.drawable.ic_pause),
                        "Pause", "Pause playback",
                        PendingIntent.getBroadcast(this, 0,
                            Intent(ACTION_PAUSE), PendingIntent.FLAG_IMMUTABLE)
                    )
                ))
                .build()
            enterPictureInPictureMode(params)
        }
    }

    override fun onPictureInPictureModeChanged(isInPiPMode: Boolean, newConfig: Configuration) {
        // Hide UI controls in PiP mode (they're too small to tap)
        viewModel.setShowControls(!isInPiPMode)
    }
}
```

---

## 11. HLD — Netflix-Like Video Streaming Architecture

```
CONTENT SIDE:
  Studio master file → encoding pipeline → multiple renditions (360p–4K)
  Encryption: server-side encryption per content key → stored on CDN
  Key management: key stored in license server (KMS)
  Manifest: .mpd or .m3u8 generated with segment URLs + DRM info

CDN:
  Video segments served from edge nodes (closest to user)
  Manifest files served with short TTL (dynamic ABR)
  License server is NOT on CDN (security) — served from origin with auth

MOBILE CLIENT:
  1. Auth: user logs in → receives access token
  2. Catalog API: user selects video → receive stream URL + license URL
  3. License request: ExoPlayer → license server (with token + device attestation)
  4. License response: encrypted content key → stored in MediaDrm
  5. Stream: ExoPlayer fetches segments → decrypts in TEE (L1) → renders
  
QUALITY FLOW:
  Network probe → ABR algorithm selects initial rendition (start low, ramp up)
  Buffer health monitor: < 10s buffer → step down quality
  Bandwidth estimate: sliding window average of recent download speeds
  
OFFLINE:
  User downloads at selected quality
  License pre-fetched with offline flag → stored in MediaDrm key store
  Download via WorkManager (resumable, constraint-aware)
  
OBSERVABILITY:
  Startup time (time-to-first-frame)
  Rebuffer rate (% of playback time in buffering state)
  Quality switches (up/down per session)
  Error rate by error code
  Watchtime per session
```

---

## 12. React Native

```typescript
// react-native-video — most popular RN video library
import Video from 'react-native-video'

// Basic playback
<Video
    source={{ uri: 'https://cdn.example.com/video.m3u8' }}
    style={{ width: '100%', aspectRatio: 16/9 }}
    controls={true}
    resizeMode="contain"
    paused={isPaused}
    onBuffer={({ isBuffering }) => setBuffering(isBuffering)}
    onError={(error) => handleError(error)}
    onProgress={({ currentTime, playableDuration }) => updateProgress(currentTime)}
    onEnd={() => handleEnd()}
/>

// DRM in react-native-video
<Video
    source={{
        uri: 'https://cdn.example.com/protected.mpd',
        type: 'mpd',
        drm: {
            type: 'widevine',
            licenseServer: 'https://license.example.com/widevine',
            headers: {
                Authorization: `Bearer ${token}`
            }
        }
    }}
/>

// expo-video (for Expo projects — newer API)
import { VideoView, useVideoPlayer } from 'expo-video'

const player = useVideoPlayer('https://cdn.example.com/video.m3u8', player => {
    player.loop = false
    player.play()
})
<VideoView player={player} style={{ width: '100%', aspectRatio: 16/9 }} />
```

---

## 13. Common Misunderstandings & Pitfalls

**❌ Using `TextureView` for DRM content**
`TextureView` routes decoded frames through app memory — a DRM violation. Widevine L1 requires decoded video to stay in the secure hardware path. Use `SurfaceView` (or `PlayerView`, which uses `SurfaceView` internally). Using `TextureView` with DRM content results in a black screen.

**❌ Not releasing the player in `onCleared()`**
`ExoPlayer` holds MediaCodec instances, audio focus, and a wakelock while playing. Failing to call `player.release()` leaks these resources, causing audio focus issues in other apps and eventual OOM. Always release in `ViewModel.onCleared()` when scoped to ViewModel.

**❌ Creating the player in a Composable**
Player creation is expensive (~100ms). Creating it in a Composable means it's recreated on every recomposition or configuration change. Always create the player in a ViewModel and pass it down.

**❌ Assuming all devices are Widevine L1**
Budget devices, emulators, and some manufacturer-specific builds are L3 (software-only decryption). If your app requires L1 for HD content (like Netflix does), check at runtime and offer a lower quality tier for L3 devices rather than failing playback entirely.

**❌ Calling `player.prepare()` without `setMediaItem()` first**
`prepare()` does nothing without a media item. Always: `setMediaItem()` → `prepare()` → `playWhenReady = true`.

**❌ Caching entire video in memory**
Large video buffers cause OOM. Set `MaxBufferMs` to a reasonable value (30–50 seconds). ExoPlayer's `DefaultLoadControl` defaults are conservative — don't change them unless you have a specific measured reason.

**❌ Not handling `ERROR_CODE_BEHIND_LIVE_WINDOW` for live streams**
For live streams, if playback falls too far behind the live edge (e.g., device was offline), this error fires. The fix is `player.seekToDefaultPosition()` to jump to the live edge — handle this in `onPlayerError`.

**❌ Requesting DRM license from client with hardcoded keys**
Never put DRM license server credentials or content keys in the client app. The license request should be authenticated with a short-lived token obtained from your auth backend. The license server validates the token before issuing content keys.

---

## 14. Best Practices

- **Media3 always, ExoPlayer 2 never for new code** — ExoPlayer 2 is deprecated
- **`SurfaceView` for DRM, never `TextureView`** — hardware secure path requirement
- **Player in ViewModel** — survives rotation, released properly in `onCleared()`
- **`MediaSessionService` for background audio** — connects to system for lock screen controls
- **Selective module imports** — only add `media3-exoplayer-hls` if you stream HLS; saves APK size
- **Check Widevine L1/L3 at runtime** — serve appropriate quality for the device capability
- **Short-lived DRM license tokens** — auth token from your backend, not hardcoded credentials
- **`DefaultLoadControl` buffer tuning** — increase `bufferForPlaybackMs` for startup, `maxBufferMs` for uninterrupted playback
- **Adaptive startup quality** — start at a low rendition, ramp up quickly — better perceived UX than stalling at HD
- **`WorkManager` for downloads** — resumable across sessions, respects WiFi constraint
- **DRM license pre-fetch for offline** — fetch offline license before device goes offline; store in MediaDrm
- **Track `rebuffer rate` in production** — % of playback time in buffering state; alert if > 2%
- **`PlayerView` with `keepContentOnPlayerReset = true`** — prevents screen flash on track change
- **Disable screensaver during video** — `window.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON)`

---

## 15. Interview Q&A

**Q1: What is the difference between HLS and DASH?**

> Both are adaptive bitrate streaming protocols that deliver video in small segments with a manifest file describing available quality levels. HLS was created by Apple — uses `.m3u8` playlists and originally `.ts` segments (now fMP4). It has native support on iOS/Safari. DASH is an open standard — uses XML `.mpd` manifests and `.m4s` segments. It's more flexible in codec and DRM support. Both are supported by Media3 on Android. For Android-first apps, DASH is slightly preferred for its open standard and flexibility. For cross-platform (including Safari), HLS is simpler since it works natively. Netflix migrated from their own format to both, serving DASH to Chrome/Android and HLS to iOS/Safari.

---

**Q2: Explain how Widevine DRM works.**

> Widevine is Google's DRM system built into every Android device. Content is encrypted on the server using AES-128 or AES-256. When playback starts, ExoPlayer detects the encryption signaling in the manifest and sends a license request to the Widevine license server — this request contains a device certificate proving the device's identity and security level. The license server validates the request, checks authorization, and returns a license containing the content decryption key, encrypted with the device's public key. On L1 devices (hardware TEE), the key is decrypted and video segments are decrypted entirely inside the hardware secure enclave — decoded frames never enter app memory, preventing screen recording. On L3 devices (software only), decryption happens in software, offering weaker protection but still functional playback.

---

**Q3: What is the difference between Widevine L1, L2, and L3?**

> They represent different security levels of the Widevine Content Decryption Module. L1 is hardware-backed — decryption and decoding happen in the device's Trusted Execution Environment; decoded video frames never enter application memory, preventing screen capture. This is required by services like Netflix and Amazon Prime for HD/4K content. L2 is a partial hardware implementation — not widely deployed. L3 is software-only — decryption happens in a software process, and decoded frames can pass through normal memory. L3 allows screen recording of protected content on a sufficiently compromised device, so content providers restrict L3 devices to SD quality. Most flagship Android devices are L1. Emulators, some budget devices, and rooted phones are L3. Always check at runtime with `MediaDrm.getPropertyString("securityLevel")` and adapt content quality accordingly.

---

**Q4: How do you implement background audio/video playback?**

> Using `MediaSessionService`. The service hosts the `ExoPlayer` instance and creates a `MediaSession` that connects the player to the Android system. The `MediaSessionService` automatically creates a foreground notification with playback controls derived from the `MediaSession` state — required since Android 14 with `foregroundServiceType="mediaPlayback"`. The UI connects to the service via `MediaController` — a thin proxy that sends playback commands to the service-hosted player. This architecture means the player lives in the service, survives the Activity being destroyed, and the lock screen controls, Bluetooth headset buttons, and Google Assistant all work through the `MediaSession` abstraction. The key lifecycle rule: release `MediaSession` and `ExoPlayer` in the service's `onDestroy()`, not tied to any Activity lifecycle.

---

**Q5: How do you minimize buffering / reduce time-to-first-frame?**

> Three areas: network efficiency, initial quality selection, and buffer pre-loading. For network efficiency: put video segments on a CDN close to users (edge caching), enable HTTP/2 for multiplexed segment requests, and use GZIP compression for manifest files. For initial quality: start at a conservatively low rendition (e.g., 360p) and ramp up quickly based on bandwidth estimation — users accept a brief moment of lower quality better than a stall. For pre-loading: when the user hovers over a thumbnail or is likely to play next, call `player.setMediaItem()` and `player.prepare()` before they tap play — the player starts buffering in the background. Tune `bufferForPlaybackMs` (how much to buffer before starting — default 2.5s) and ensure the manifest is cached. For live streams, use shorter segment durations (2s vs 6s) to reduce startup latency.

---

**Q6: How do you design offline video downloads for a streaming app?**

> Three components: download management, DRM offline licensing, and offline playback. For download management, use Media3's `DownloadManager` with a `DownloadService` running as a foreground service — it handles resumable downloads, queuing, and progress tracking. Schedule downloads through `WorkManager` with WiFi + charging constraints for large files. For storage, use `SimpleCache` with `LeastRecentlyUsedCacheEvictor` capped at the device's available storage. For DRM, fetch an offline Widevine license before the download using `setKeyRequestParameter("offline", "true")` — this stores the content key in the device's MediaDrm key store so it's available without network. For playback, use `CacheDataSource.Factory` backed by the download cache, with read-only flag set — ExoPlayer reads from cache first, falls back to network for any uncached segments. Always show download progress via the notification and allow the user to manage (delete) downloads from a settings screen.

---

*Previous: [14 — Bluetooth & NFC](./14-bluetooth-nfc.md)*
*Next: [16 — Audio Management & Focus](./16-audio-management.md)*
