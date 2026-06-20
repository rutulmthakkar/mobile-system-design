# 16 — Audio Management & Focus

> **Round type: LLD (primary)**
> **Focus:** Audio streams, focus management, ducking, interruptions, routing, AudioAttributes, MediaSession — everything needed to be a good citizen in Android's audio ecosystem

---

## 1. Why This Matters in Interviews

Audio management is a hidden complexity that separates production-grade media apps from hobbyist ones. A music app that keeps playing through a phone call, or a game that silences a podcast without ducking, is a bad citizen. Interviewers at Amazon (Alexa), Spotify, Google, and any media company test this because getting it wrong causes real user complaints.

**Common interview angles:**
- "What happens to your music app when a phone call comes in?"
- "What is audio ducking and when should you duck vs pause?"
- "How do you handle headphone disconnect?"
- "What's the difference between `AUDIOFOCUS_LOSS` and `AUDIOFOCUS_LOSS_TRANSIENT`?"
- "How does Media3 handle audio focus automatically?"
- "What are `AudioAttributes` and why do they matter?"

---

## 2. Android Audio Streams

Android routes audio through logical streams, each with independent volume control:

```
STREAM_MUSIC       → Music, video, games (most apps use this)
STREAM_RING        → Ringtone
STREAM_NOTIFICATION→ Notification sounds
STREAM_ALARM       → Alarm sounds (always plays even in Do Not Disturb)
STREAM_VOICE_CALL  → In-call voice
STREAM_SYSTEM      → System UI sounds (clicks, keyboard)
STREAM_DTMF        → Dial-tone multi-frequency (phone keypad tones)
```

**Critical rule:** Call `setVolumeControlStream(AudioManager.STREAM_MUSIC)` in your `Activity.onCreate()` so that hardware volume buttons control the right stream:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Without this, volume buttons control STREAM_RING (ringer volume) by default
    volumeControlStream = AudioManager.STREAM_MUSIC
}
```

---

## 3. AudioAttributes

`AudioAttributes` replaced stream types for describing audio in API 21+. They tell the system **what** the audio is and **why** it's playing, allowing more intelligent routing and focus decisions.

```kotlin
// Usage types — what the audio represents to the user
AudioAttributes.USAGE_MEDIA           // music, podcasts, video audio
AudioAttributes.USAGE_GAME            // game audio
AudioAttributes.USAGE_VOICE_COMMUNICATION    // VoIP calls
AudioAttributes.USAGE_ALARM           // alarm
AudioAttributes.USAGE_NOTIFICATION    // notification sound
AudioAttributes.USAGE_ASSISTANCE_NAVIGATION  // turn-by-turn directions
AudioAttributes.USAGE_ASSISTANCE_SONIFICATION // UI sounds

// Content types — what the audio is technically
AudioAttributes.CONTENT_TYPE_MUSIC    // music
AudioAttributes.CONTENT_TYPE_SPEECH   // voice, podcast, audiobook
AudioAttributes.CONTENT_TYPE_MOVIE    // video/movie audio
AudioAttributes.CONTENT_TYPE_SONIFICATION // UI effects, beeps
AudioAttributes.CONTENT_TYPE_UNKNOWN

// Building AudioAttributes
val musicAttributes = AudioAttributes.Builder()
    .setUsage(AudioAttributes.USAGE_MEDIA)
    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
    .build()

val navigationAttributes = AudioAttributes.Builder()
    .setUsage(AudioAttributes.USAGE_ASSISTANCE_NAVIGATION)
    .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
    .build()

val gameAttributes = AudioAttributes.Builder()
    .setUsage(AudioAttributes.USAGE_GAME)
    .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION)
    .build()
```

**Why `AudioAttributes` matter:** The system uses `USAGE` to decide routing (speaker vs earpiece), focus priority, and Do Not Disturb behavior. `USAGE_ALARM` plays even in silent mode. `USAGE_VOICE_COMMUNICATION` is routed to the earpiece automatically. Getting these wrong causes unexpected audio routing.

---

## 4. Audio Focus

Audio focus is Android's cooperative mechanism for mediating between apps that want to produce audio. It's **voluntary** — apps are expected to request and respond to focus, but the system doesn't enforce it (unlike Android 15 AAOS which adds enforcement for automotive).

### 4.1 Focus Types

```kotlin
// Request permanent focus — for long-running playback (music, podcast)
AudioManager.AUDIOFOCUS_GAIN

// Request transient focus — short interruption (navigation prompt, notification)
AudioManager.AUDIOFOCUS_GAIN_TRANSIENT

// Request transient focus, allow other apps to duck (lower volume)
// Use when you want to play a short sound without completely silencing others
AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK

// Request exclusive focus — nothing else can play, nothing can duck
// Use for voice calls, alarms
AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE
```

### 4.2 Focus Change Callbacks

```kotlin
// What your app receives when another app changes focus:

AudioManager.AUDIOFOCUS_GAIN
// You've gained focus (or regained it after a transient loss)
// → Resume playback, restore normal volume

AudioManager.AUDIOFOCUS_LOSS
// Permanent loss — another app took focus indefinitely (user started another media app)
// → Stop playback, abandon focus, release resources

AudioManager.AUDIOFOCUS_LOSS_TRANSIENT
// Temporary loss — phone call started, alarm firing, other short interruption
// → Pause playback (don't release player), resume when focus returns

AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK
// Temporary loss, but you're allowed to keep playing at reduced volume
// → Lower volume to ~20%, restore when focus returns
// API 26+: system handles ducking automatically if you don't set willPauseWhenDucked(true)
```

### 4.3 AudioFocusRequest (API 26+) — Modern API

```kotlin
@Singleton
class AudioFocusManager @Inject constructor(
    @ApplicationContext context: Context
) {
    private val audioManager = context.getSystemService(AudioManager::class.java)
    private var focusRequest: AudioFocusRequest? = null

    // Focus change callback
    private val focusChangeListener = AudioManager.OnAudioFocusChangeListener { focusChange ->
        when (focusChange) {
            AudioManager.AUDIOFOCUS_GAIN -> {
                onFocusGained()
            }
            AudioManager.AUDIOFOCUS_LOSS -> {
                onFocusLostPermanently()
                abandonFocus()  // release focus when permanently lost
            }
            AudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> {
                onFocusLostTransient()
                // Don't abandon — we'll get it back
            }
            AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK -> {
                // API 26+: System handles ducking automatically
                // Only need to handle this if willPauseWhenDucked = true
                onFocusLostTransient()
            }
        }
    }

    fun requestFocus(): Boolean {
        val request = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
            .setAudioAttributes(
                AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .build()
            )
            .setOnAudioFocusChangeListener(focusChangeListener)
            // API 26+ automatic ducking — system lowers our volume automatically
            // when another app requests AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK
            // Set to true if you want to PAUSE instead of duck:
            .setWillPauseWhenDucked(false)   // false = system ducks for us
            // Delayed focus gain: if focus can't be granted now, notify us when it can
            .setAcceptsDelayedFocusGain(true)
            .build()
            .also { focusRequest = it }

        return when (audioManager.requestAudioFocus(request)) {
            AudioManager.AUDIOFOCUS_REQUEST_GRANTED -> true
            AudioManager.AUDIOFOCUS_REQUEST_DELAYED -> {
                // Focus not granted yet but will be — wait for AUDIOFOCUS_GAIN callback
                true  // treat as eventual success
            }
            else -> false
        }
    }

    fun abandonFocus() {
        focusRequest?.let { audioManager.abandonAudioFocusRequest(it) }
        focusRequest = null
    }

    // Override these in your implementation
    open fun onFocusGained() {}
    open fun onFocusLostTransient() {}
    open fun onFocusLostPermanently() {}
}
```

### 4.4 Media3 Handles Focus Automatically ✅

**The most important thing to know:** If you use Media3/ExoPlayer with `MediaSession`, audio focus is handled automatically. You don't need any of the above boilerplate.

```kotlin
// Media3 with MediaSession — audio focus is completely automatic
val player = ExoPlayer.Builder(context).build()
val mediaSession = MediaSession.Builder(context, player).build()
// That's it — Media3 requests AUDIOFOCUS_GAIN when playing,
// pauses on AUDIOFOCUS_LOSS_TRANSIENT, stops on AUDIOFOCUS_LOSS
// Manual focus management only needed if NOT using MediaSession
```

```kotlin
// If you need custom focus behavior with Media3:
val audioAttributes = AudioAttributes.Builder()
    .setUsage(C.USAGE_MEDIA)
    .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
    .build()

val player = ExoPlayer.Builder(context)
    .setAudioAttributes(audioAttributes, /* handleAudioFocus= */ true)
    .build()
// handleAudioFocus = true: ExoPlayer manages focus lifecycle
// handleAudioFocus = false: you manage focus manually
```

---

## 5. Ducking

Ducking = temporarily lowering your volume when another app needs to be heard, without fully pausing.

**Use case:** Navigation GPS prompt while listening to music — the music ducks to ~20%, direction is announced, music returns to normal.

### 5.1 Automatic Ducking (API 26+)

On Android 8.0+, the system handles ducking automatically. When your app has focus with `USAGE_MEDIA` and another app requests `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`:
- System automatically fades your audio down
- Other app plays
- System fades your audio back up
- You receive **no callback** (unless you set `setWillPauseWhenDucked(true)`)

```kotlin
// To receive callbacks for ducking (override auto-ducking):
AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN)
    .setWillPauseWhenDucked(true)     // opt-in to manual ducking callbacks
    .setOnAudioFocusChangeListener(listener)
    .build()

// In your listener:
AudioManager.AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK -> {
    // Manual duck — lower volume
    player.volume = 0.2f   // 20% volume
}
AudioManager.AUDIOFOCUS_GAIN -> {
    // Restore volume
    player.volume = 1.0f
}
```

### 5.2 When to Duck vs Pause

| Situation | Response | Reason |
|---|---|---|
| GPS navigation prompt | **Duck** | Short, speech, user expects music to continue |
| Notification sound | **Nothing** (auto-duck) | System handles automatically |
| Incoming phone call | **Pause** | User is talking, not listening |
| Alarm fires | **Pause** | Alarm must be heard clearly |
| Another music app starts | **Stop + abandon** | User chose different music |
| Short game SFX from another app | **Duck** (or nothing) | Transient, tolerable |
| VoIP call | **Pause** | User is in a call |

---

## 6. Handling Headphone Disconnect

When the user unplugs wired headphones, the audio route switches to the speaker. This is intentional — if music is playing privately through headphones, suddenly blasting it through the speaker is bad UX. You should **pause**.

```kotlin
// Register for headset disconnect broadcast
class BecomingNoisyReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == AudioManager.ACTION_AUDIO_BECOMING_NOISY) {
            // Headphones unplugged OR Bluetooth disconnected
            // PAUSE immediately
            player.pause()
        }
    }
}

// Register/unregister dynamically (not in manifest — only when playing)
private val noisyReceiver = BecomingNoisyReceiver()
private val noisyFilter = IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY)

// When playback starts:
registerReceiver(noisyReceiver, noisyFilter)

// When playback stops or pauses:
unregisterReceiver(noisyReceiver)

// Media3 + MediaSession handles ACTION_AUDIO_BECOMING_NOISY automatically ✅
// Only need manual handling if building a custom player without MediaSession
```

---

## 7. Audio Routing

Audio routing determines which output device plays your audio: speaker, earpiece, wired headset, or Bluetooth.

```kotlin
val audioManager = context.getSystemService(AudioManager::class.java)

// Check available output devices (API 23+)
val availableDevices = audioManager.getDevices(AudioManager.GET_DEVICES_OUTPUTS)
val hasBluetoothHeadset = availableDevices.any { 
    it.type == AudioDeviceInfo.TYPE_BLUETOOTH_A2DP ||
    it.type == AudioDeviceInfo.TYPE_BLUETOOTH_SCO
}
val hasWiredHeadset = availableDevices.any {
    it.type == AudioDeviceInfo.TYPE_WIRED_HEADSET ||
    it.type == AudioDeviceInfo.TYPE_WIRED_HEADPHONES
}

// Prefer a specific output device (API 28+)
player.setPreferredAudioDevice(
    availableDevices.firstOrNull { it.type == AudioDeviceInfo.TYPE_BLUETOOTH_A2DP }
)

// For VoIP — switch to earpiece (privacy)
audioManager.mode = AudioManager.MODE_IN_COMMUNICATION
audioManager.isSpeakerphoneOn = false   // route to earpiece
// Restore after call:
audioManager.mode = AudioManager.MODE_NORMAL
audioManager.isSpeakerphoneOn = false

// Listen for device connections/disconnections
val routingCallback = object : AudioDeviceCallback() {
    override fun onAudioDevicesAdded(addedDevices: Array<out AudioDeviceInfo>) {
        // Bluetooth headset connected? Re-route if needed
    }
    override fun onAudioDevicesRemoved(removedDevices: Array<out AudioDeviceInfo>) {
        // Bluetooth disconnected — pause or reroute to speaker
    }
}
audioManager.registerAudioDeviceCallback(routingCallback, Handler(mainLooper))
```

### 7.1 Bluetooth SCO (for Microphone)

For VoIP or recording through Bluetooth headset mic (not just output):

```kotlin
// Bluetooth SCO (Synchronous Connection-Oriented) — for bidirectional audio (mic + speaker)
// Required for: Voice calls over Bluetooth, Alexa/voice assistants on BT headset

audioManager.startBluetoothSco()
audioManager.isBluetoothScoOn = true   // route mic input + audio output through SCO

// Wait for SCO connection before recording
val scoReceiver = object : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val state = intent.getIntExtra(AudioManager.EXTRA_SCO_AUDIO_STATE, -1)
        when (state) {
            AudioManager.SCO_AUDIO_STATE_CONNECTED -> startRecording()
            AudioManager.SCO_AUDIO_STATE_DISCONNECTED -> handleScoDisconnect()
        }
    }
}
registerReceiver(scoReceiver, IntentFilter(AudioManager.ACTION_SCO_AUDIO_STATE_UPDATED))

// Stop when done
audioManager.stopBluetoothSco()
audioManager.isBluetoothScoOn = false
```

---

## 8. SoundPool — Short Audio Clips

For short sound effects (UI feedback, game sounds, notification pings) that need low latency, use `SoundPool` — NOT `MediaPlayer` or `ExoPlayer`.

```kotlin
// SoundPool: loads sounds into memory, plays with minimal latency
val soundPool = SoundPool.Builder()
    .setMaxStreams(5)   // max simultaneous sounds
    .setAudioAttributes(
        AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_GAME)
            .setContentType(AudioAttributes.CONTENT_TYPE_SONIFICATION)
            .build()
    )
    .build()

// Load sounds at startup (not on the fly — loading takes time)
val clickSoundId = soundPool.load(context, R.raw.click, 1)
val successSoundId = soundPool.load(context, R.raw.success, 1)
val errorSoundId = soundPool.load(context, R.raw.error, 1)

// Play (after sounds are loaded — listen for OnLoadCompleteListener)
soundPool.play(
    clickSoundId,
    leftVolume = 1.0f,
    rightVolume = 1.0f,
    priority = 1,
    loop = 0,       // 0 = no loop, -1 = loop forever
    rate = 1.0f     // playback speed: 0.5 = half speed, 2.0 = double speed
)

// Release when done (e.g., in onDestroy)
soundPool.release()
```

**When to use what:**
| | SoundPool | ExoPlayer | MediaPlayer |
|---|---|---|---|
| Latency | < 5ms | ~100ms | ~100ms |
| Memory | In memory (small files only) | Streamed | Streamed or in memory |
| Max file size | ~1MB recommended | Any | Any |
| Simultaneous | Multiple (configurable) | One per instance | One per instance |
| Use case | UI sounds, game SFX | Music, video, podcasts | Simple audio playback |
| Format support | Limited (WAV, OGG) | Extensive | Limited |

---

## 9. AudioRecord & AudioTrack — Low Level APIs

For apps that need direct PCM access (audio processing, VAD, recording):

```kotlin
// AudioRecord — capture raw PCM from microphone
val sampleRate = 16000  // 16 kHz for voice
val channelConfig = AudioFormat.CHANNEL_IN_MONO
val audioFormat = AudioFormat.ENCODING_PCM_16BIT
val bufferSize = AudioRecord.getMinBufferSize(sampleRate, channelConfig, audioFormat)

val audioRecord = AudioRecord(
    MediaRecorder.AudioSource.VOICE_RECOGNITION,  // best for speech
    // Also: MIC (raw), CAMCORDER (tuned for video), VOICE_COMMUNICATION (VoIP noise cancel)
    sampleRate,
    channelConfig,
    audioFormat,
    bufferSize * 4   // 4x minimum for safety
)

audioRecord.startRecording()

// Read frames on a background thread
val buffer = ShortArray(bufferSize)
while (isRecording) {
    val read = audioRecord.read(buffer, 0, buffer.size)
    if (read > 0) processAudioFrame(buffer.copyOf(read))
}

audioRecord.stop()
audioRecord.release()

// AudioSources comparison:
// MIC              → raw mic input, no processing
// VOICE_RECOGNITION → optimized for speech recognition, noise tuning
// VOICE_COMMUNICATION → AEC (acoustic echo cancellation) + NS (noise suppression) for VoIP
// CAMCORDER        → tuned for on-camera audio
```

```kotlin
// AudioTrack — output raw PCM directly to audio hardware (bypasses MediaPlayer)
// Use for: synthesized audio, processed audio, very low latency output

val audioTrack = AudioTrack.Builder()
    .setAudioAttributes(
        AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_MEDIA)
            .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
            .build()
    )
    .setAudioFormat(
        AudioFormat.Builder()
            .setSampleRate(44100)
            .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
            .setChannelMask(AudioFormat.CHANNEL_OUT_STEREO)
            .build()
    )
    .setTransferMode(AudioTrack.MODE_STREAM)   // streaming (not static for small sounds)
    .setBufferSizeInBytes(
        AudioTrack.getMinBufferSize(44100, AudioFormat.CHANNEL_OUT_STEREO, AudioFormat.ENCODING_PCM_16BIT) * 2
    )
    .build()

audioTrack.play()
audioTrack.write(pcmBuffer, 0, pcmBuffer.size)  // push PCM data
audioTrack.stop()
audioTrack.release()
```

---

## 10. Phone Call Interruptions

Since Android 5.0 (Lollipop), phone call state is integrated into audio focus:
- Incoming call → your app receives `AUDIOFOCUS_LOSS_TRANSIENT`
- Call answered → your app receives `AUDIOFOCUS_LOSS`
- Call ends → your app receives `AUDIOFOCUS_GAIN`

```kotlin
// Modern approach: just handle audio focus callbacks
// No need for PhoneStateListener (deprecated for this use case)

// In your OnAudioFocusChangeListener:
AudioManager.AUDIOFOCUS_LOSS_TRANSIENT -> {
    // Phone ringing — pause playback, keep player ready
    player.pause()
    wasPlayingBeforeInterruption = true
}
AudioManager.AUDIOFOCUS_LOSS -> {
    // Call answered or other app took over permanently
    player.pause()
    // Don't auto-resume — user explicitly answered a call
}
AudioManager.AUDIOFOCUS_GAIN -> {
    // Call ended — resume only if we were playing when interrupted
    if (wasPlayingBeforeInterruption) {
        player.play()
        wasPlayingBeforeInterruption = false
    }
}
```

---

## 11. Do Not Disturb (DND) Integration

DND mode silences most audio. How your audio behaves depends on `AudioAttributes.USAGE`:

```
DND Active (Priority Only mode):
  USAGE_ALARM        → Plays (alarms bypass DND)
  USAGE_MEDIA        → Silenced unless user explicitly allows media in DND
  USAGE_NOTIFICATION → Silenced
  USAGE_RINGTONE     → Depends on DND settings (Calls may be allowed)
  USAGE_VOICE_COMMUNICATION → Usually allowed (calls)

DND Total Silence:
  Everything silenced except device-critical audio
```

**You cannot override DND in your app** — this is intentional. Design for it: gracefully accept silence, don't bypass with `STREAM_ALARM` for non-alarm content (rejected by Play Store policy).

---

## 12. Volume Management

```kotlin
// Adjust volume programmatically (rare — usually let user control)
audioManager.adjustVolume(
    AudioManager.ADJUST_LOWER,     // or ADJUST_RAISE, ADJUST_MUTE
    AudioManager.FLAG_SHOW_UI      // show volume UI to user
)

// Set absolute volume
audioManager.setStreamVolume(
    AudioManager.STREAM_MUSIC,
    targetVolume,      // 0 to maxVolume
    AudioManager.FLAG_SHOW_UI
)

// Get current volume
val currentVolume = audioManager.getStreamVolume(AudioManager.STREAM_MUSIC)
val maxVolume = audioManager.getStreamMaxVolume(AudioManager.STREAM_MUSIC)
val volumePercent = currentVolume.toFloat() / maxVolume.toFloat()

// Check if muted
val isMuted = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    audioManager.isStreamMute(AudioManager.STREAM_MUSIC)
} else {
    audioManager.getStreamVolume(AudioManager.STREAM_MUSIC) == 0
}
```

---

## 13. Complete Audio-Aware Player Implementation

```kotlin
class AudioAwareMediaService : MediaSessionService() {

    private lateinit var player: ExoPlayer
    private lateinit var mediaSession: MediaSession
    private val audioManager by lazy {
        getSystemService(AudioManager::class.java)
    }

    // Noisy intent receiver (headphone disconnect)
    private val noisyReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            if (intent.action == AudioManager.ACTION_AUDIO_BECOMING_NOISY) {
                player.pause()  // pause on headphone disconnect
            }
        }
    }

    override fun onCreate() {
        super.onCreate()

        // Build player — Media3 handles audio focus automatically
        player = ExoPlayer.Builder(this)
            .setAudioAttributes(
                com.google.android.exoplayer2.audio.AudioAttributes.Builder()
                    .setUsage(C.USAGE_MEDIA)
                    .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)
                    .build(),
                true  // handleAudioFocus = true
            )
            .setHandleAudioBecomingNoisy(true)  // Media3 handles headphone disconnect ✅
            .build()

        // MediaSession — connects to system (lock screen, Bluetooth, Assistant)
        mediaSession = MediaSession.Builder(this, player).build()

        // If NOT using handleAudioBecomingNoisy above, register manually:
        // registerReceiver(noisyReceiver, IntentFilter(AudioManager.ACTION_AUDIO_BECOMING_NOISY))
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo) = mediaSession

    override fun onDestroy() {
        // Clean up in reverse order
        // unregisterReceiver(noisyReceiver)  // if registered manually
        mediaSession.release()
        player.release()
        super.onDestroy()
    }
}
```

**Note:** `ExoPlayer.Builder().setHandleAudioBecomingNoisy(true)` and `setAudioAttributes(..., handleAudioFocus = true)` are the two lines that give you full automatic audio management in Media3. Use them.

---

## 14. HLD — Audio for a Music Streaming App

```
Audio Focus Strategy:
  On play: request AUDIOFOCUS_GAIN (USAGE_MEDIA, CONTENT_TYPE_MUSIC)
  On AUDIOFOCUS_LOSS: pause, abandon focus
  On AUDIOFOCUS_LOSS_TRANSIENT: pause, retain focus
  On AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK: system auto-ducks (API 26+)
  On AUDIOFOCUS_GAIN after transient: resume only if user didn't explicitly pause

Interruption Handling:
  Phone call (LOSS_TRANSIENT) → pause, track "interrupted" state
  Call ends (GAIN) → resume if interrupted=true
  GPS directions (CAN_DUCK) → system ducks, resume normal after directions
  Another music app (LOSS) → stop, user chose something else

Headphone Management:
  ACTION_AUDIO_BECOMING_NOISY → pause immediately
  Headphone connected → do NOT auto-resume (user may not expect it)

Background Playback:
  MediaSessionService → survives Activity death
  Foreground notification with controls (Media3 auto-generates from MediaSession)
  Lock screen controls (via MediaSession)
  Bluetooth play/pause button (via MediaSession)

Audio Routing:
  USAGE_MEDIA routes to speaker, headphones, or Bluetooth automatically
  VoIP mode → MODE_IN_COMMUNICATION, earpiece routing for calls
  Bluetooth SCO → only for bidirectional audio (mic + output)
```

---

## 15. React Native Audio

```typescript
// react-native-track-player — best library for audio apps
// Wraps ExoPlayer (Android) + AVPlayer (iOS) with focus management built in
import TrackPlayer, { Capability, Event, State } from 'react-native-track-player'

// Setup (once in app lifecycle)
await TrackPlayer.setupPlayer({
    maxCacheSize: 1024 * 5,  // 5MB cache
})

await TrackPlayer.updateOptions({
    capabilities: [
        Capability.Play,
        Capability.Pause,
        Capability.SkipToNext,
        Capability.SkipToPrevious,
        Capability.SeekTo,
    ],
    // Compact controls (in notification)
    compactCapabilities: [Capability.Play, Capability.Pause, Capability.SkipToNext],
})

// Add tracks
await TrackPlayer.add([{
    id: '1',
    url: 'https://cdn.example.com/track1.mp3',
    title: 'Song Title',
    artist: 'Artist Name',
    artwork: 'https://cdn.example.com/artwork.jpg',
}])

await TrackPlayer.play()

// Audio focus handled automatically by react-native-track-player
// Responds to: interruptions, headphone disconnect, Bluetooth events

// For raw audio recording in RN:
// react-native-audio-record (basic PCM recording)
// @react-native-voice/voice (speech recognition with VAD)
```

---

## 16. Common Misunderstandings & Pitfalls

**❌ Not setting `volumeControlStream` in Activity**
Without this, hardware volume keys control the ringer volume (STREAM_RING) even when your music is playing. Users try to adjust music volume and adjust the ringer instead. Always call `volumeControlStream = AudioManager.STREAM_MUSIC` in `onCreate()`.

**❌ Auto-resuming after `ACTION_AUDIO_BECOMING_NOISY`**
When headphones are disconnected, pause — don't resume when they reconnect. Reconnecting headphones is often accidental (re-inserting to charge). Auto-resuming after reconnect surprises users. Only resume on explicit user action.

**❌ Abandoning audio focus on `AUDIOFOCUS_LOSS_TRANSIENT`**
Transient means temporary — you'll get focus back. If you abandon focus on transient loss, you have to re-request it when focus returns. Worse, you might not get it back immediately. Just pause playback; keep the focus request alive.

**❌ Using `MediaPlayer` for short sound effects**
`MediaPlayer` has ~100ms startup latency and complex lifecycle management. For UI clicks, game sounds, or notification pings, use `SoundPool` — sub-5ms latency, multiple simultaneous sounds, no lifecycle overhead.

**❌ Using `AudioManager.MODE_IN_CALL` instead of `MODE_IN_COMMUNICATION`**
`MODE_IN_CALL` is for the system dialer only. For in-app VoIP, use `MODE_IN_COMMUNICATION` — it enables AEC (acoustic echo cancellation) and routes audio to the earpiece. Using the wrong mode disables noise cancellation and may cause echo.

**❌ Not requesting AUDIOFOCUS before starting playback**
Starting playback without requesting focus is rude to other apps. A podcast might be playing; you should pause it. Always request focus first — if denied, don't start playback.

**❌ Blocking main thread with Bluetooth SCO setup**
`startBluetoothSco()` is asynchronous — it returns before SCO is actually connected. Always wait for `SCO_AUDIO_STATE_CONNECTED` broadcast before starting recording. Attempting to record immediately after `startBluetoothSco()` may use the device mic instead of the Bluetooth mic.

**❌ Manual audio focus with Media3 + MediaSession**
If you use Media3 with `setAudioAttributes(attrs, handleAudioFocus = true)`, Media3 manages focus entirely. Adding your own `AudioFocusRequest` creates a conflict — two focus requests for the same playback. Either use Media3's automatic handling or manage it manually with `handleAudioFocus = false`.

---

## 17. Best Practices

- **Media3 with `handleAudioFocus = true`** — automatic focus management, no boilerplate
- **Media3 with `setHandleAudioBecomingNoisy(true)`** — automatic headphone disconnect handling
- **`setVolumeControlStream(STREAM_MUSIC)`** in `Activity.onCreate()` — hardware buttons control right stream
- **`AudioAttributes` over stream types** — use `USAGE_MEDIA`, `USAGE_GAME`, etc., not `STREAM_MUSIC`
- **Duck vs pause** — duck for transient speech (GPS), pause for calls and alarms
- **`setWillPauseWhenDucked(false)` for music** — let system auto-duck; reserve manual ducking for special cases
- **`setAcceptsDelayedFocusGain(true)`** — handle the case where focus isn't immediately available
- **Never auto-resume on headphone reconnect** — only pause on disconnect, let user resume
- **`MediaSessionService` for background audio** — connects to system, handles all interruptions
- **`SoundPool` for UI sounds and game SFX** — not `MediaPlayer`, not `ExoPlayer`
- **`AudioSource.VOICE_RECOGNITION`** for speech recording, **`VOICE_COMMUNICATION`** for VoIP
- **`MODE_IN_COMMUNICATION`** for in-app calls — enables AEC and earpiece routing
- **Wait for `SCO_AUDIO_STATE_CONNECTED`** before recording from Bluetooth mic
- **Release all audio resources in `onDestroy()`** — player, SoundPool, AudioRecord, AudioTrack

---

## 18. Interview Q&A

**Q1: What happens to your music app when a phone call comes in?**

> The telephony system requests `AUDIOFOCUS_LOSS_TRANSIENT` when the phone starts ringing. My app pauses playback but retains the focus request — I don't abandon it. I track that playback was paused due to an interruption. When the call ends, the system sends `AUDIOFOCUS_GAIN`. In my callback, I check the "interrupted" flag: if true, I resume; if the user explicitly paused before the gain, I don't auto-resume. If the user actually answers the call, the system sends `AUDIOFOCUS_LOSS` (permanent), and I fully stop playback without auto-resuming later — the user clearly chose to take the call. If using Media3 with `handleAudioFocus = true`, this entire flow is handled automatically.

---

**Q2: What's the difference between `AUDIOFOCUS_LOSS` and `AUDIOFOCUS_LOSS_TRANSIENT`?**

> `AUDIOFOCUS_LOSS` is permanent — another app has taken audio focus indefinitely. The canonical example is a user switching from your music app to another music app. You should stop playback and abandon your focus request. Don't auto-resume unless the user explicitly acts. `AUDIOFOCUS_LOSS_TRANSIENT` is temporary — another app needs focus briefly and will return it. A phone call, navigation prompt, or alarm. You should pause but keep your focus request and player ready. When `AUDIOFOCUS_GAIN` arrives, you can resume. There's also `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK` — the system is asking you to keep playing at reduced volume instead of pausing. On API 26+, the system handles ducking automatically; you don't receive a callback unless you opt in with `setWillPauseWhenDucked(true)`.

---

**Q3: How does `AudioAttributes` differ from stream types, and why should you use it?**

> Stream types (`STREAM_MUSIC`, `STREAM_ALARM`, etc.) were the original API for describing audio intent. They've been superseded by `AudioAttributes` since API 21 because they were too coarse — `STREAM_MUSIC` is used for vastly different content types (game music, video, podcasts) but they all need different routing behavior. `AudioAttributes` uses two descriptors: `USAGE` (why you're playing: media, game, navigation, notification) and `CONTENT_TYPE` (what it is: music, speech, sonification). The system uses `USAGE` to decide routing (earpiece vs speaker), DND behavior (alarms bypass DND, notifications don't), and focus priority. `USAGE_ALARM` plays in silent mode; `USAGE_MEDIA` doesn't. `USAGE_VOICE_COMMUNICATION` triggers AEC and routes to earpiece automatically. Getting `AudioAttributes` right means your audio goes to the right place without manual routing code.

---

**Q4: How do you handle headphone disconnect correctly?**

> Register a `BroadcastReceiver` for `ACTION_AUDIO_BECOMING_NOISY` while playback is active. When received, pause immediately — don't lower volume, don't fade, just pause. The reason: when headphones disconnect, audio routes to the speaker. A podcast playing at headphone volume through the speaker in a quiet room or on public transit is embarrassing for the user. The "noisy" naming refers to the audio becoming noise to people around the user. Importantly, do NOT auto-resume when headphones reconnect. Users often re-plug just to check something or by accident. Auto-resume surprises them. If using Media3 with `.setHandleAudioBecomingNoisy(true)`, this is handled automatically.

---

**Q5: When would you use `SoundPool` vs `ExoPlayer`?**

> `SoundPool` is for short, latency-sensitive sound effects — UI button clicks, game hit sounds, notification pings, anything under ~1MB and needing sub-5ms response. It pre-loads sounds into memory at startup and plays them with near-zero latency, supporting multiple simultaneous sounds. `ExoPlayer` is for long-form content — music, podcasts, video — where latency doesn't matter but format support, adaptive streaming, and DRM do. Using `MediaPlayer` or `ExoPlayer` for a button click causes noticeable lag because they need ~100ms to decode and buffer before playing. Using `SoundPool` for a 3-hour podcast wastes memory since it holds everything in RAM. Match the tool to the latency and length requirements.

---

**Q6: How does Media3 handle audio focus automatically, and when would you override it?**

> When you build an `ExoPlayer` with `.setAudioAttributes(attrs, handleAudioFocus = true)` and/or wrap it in a `MediaSession`, Media3 manages the full focus lifecycle: requesting focus on `play()`, releasing on `stop()`, pausing on `AUDIOFOCUS_LOSS_TRANSIENT`, stopping on `AUDIOFOCUS_LOSS`, and handling `ACTION_AUDIO_BECOMING_NOISY` if you also pass `setHandleAudioBecomingNoisy(true)`. I'd override it with `handleAudioFocus = false` in three scenarios: if I need custom ducking behavior (e.g., partially lower one audio stream while another continues), if I'm building a VoIP app where the mode switch to `MODE_IN_COMMUNICATION` and earpiece routing need coordination with focus, or if I have multiple simultaneous players that need coordinated focus management (one player ducks while another takes priority). For standard music or podcast apps, automatic management is correct and significantly simpler.

---

*Previous: [15 — Video Playback & DRM](./15-video-drm.md)*
*Next: [17 — Audio Input Stream Manipulation](./17-audio-input.md)*
