# 17 — Audio Input Stream Manipulation

> **Round type: LLD (primary)**
> **Focus:** Recording pipeline, real-time audio processing, noise reduction, AEC, AGC, stream chunking, sending processed audio to ASR — the full pipeline from microphone to clean speech

---

## 1. Why This Matters in Interviews

This is a niche but high-signal topic. At Amazon (Alexa), Google (Assistant), Apple (Siri), and any voice-first product, engineers who understand the full audio pipeline from microphone → processing → ASR are rare and valuable. Your Alexa audio streaming work makes this directly relevant.

**Common interview angles:**
- "Walk me through the audio pipeline for a voice assistant"
- "How do you filter out background noise before sending to ASR?"
- "What is AEC and why is it critical for hands-free voice?"
- "How do you handle the trade-off between processing latency and quality?"
- "How do you detect speech vs silence in real time?"
- "What causes echo and how do you cancel it programmatically?"

---

## 2. The Full Audio Input Pipeline

```
Microphone (analog)
        │
        ▼
[ADC: Analog-to-Digital Conversion]
  → PCM (Pulse Code Modulation): raw digital samples
  → 16kHz sample rate for speech (44.1kHz for music)
  → 16-bit PCM (ShortArray) — standard for ASR
        │
        ▼
[Android AudioRecord]
  → Frame-based reading (20–30ms chunks)
  → AudioSource selection (MIC, VOICE_RECOGNITION, VOICE_COMMUNICATION)
        │
        ▼
[Pre-processing Pipeline] ← happens in real-time, per-frame
  ├── AEC (Acoustic Echo Cancellation)  ← remove speaker audio from mic
  ├── NS  (Noise Suppression)           ← reduce background noise
  ├── AGC (Automatic Gain Control)      ← normalize input level
  └── VAD (Voice Activity Detection)   ← detect speech vs silence
        │
        ▼
[Post-processing]
  ├── Hesitation filtering              ← remove "um", "uh", repeated words
  ├── Silence removal                   ← trim excessive pauses
  └── Format conversion                 ← to format required by ASR (Opus, FLAC, PCM)
        │
        ▼
[Output]
  ├── Stream to cloud ASR (Alexa, Google STT, Whisper API)
  └── OR: Local ASR (Vosk, Whisper.cpp via MediaPipe)
```

---

## 3. AudioRecord Setup

```kotlin
class MicrophoneCapture(
    private val scope: CoroutineScope
) {
    // Standard speech processing parameters
    companion object {
        const val SAMPLE_RATE = 16000          // 16kHz — optimal for speech ASR
        const val CHANNEL_CONFIG = AudioFormat.CHANNEL_IN_MONO
        const val AUDIO_FORMAT = AudioFormat.ENCODING_PCM_16BIT
        const val FRAME_DURATION_MS = 20       // 20ms frames (standard for speech)
        val FRAME_SIZE = SAMPLE_RATE * FRAME_DURATION_MS / 1000  // = 320 samples
    }

    private var audioRecord: AudioRecord? = null

    @SuppressLint("MissingPermission")
    fun startCapture(
        audioSource: Int = MediaRecorder.AudioSource.VOICE_RECOGNITION
    ): Flow<ShortArray> = callbackFlow {

        val minBuffer = AudioRecord.getMinBufferSize(SAMPLE_RATE, CHANNEL_CONFIG, AUDIO_FORMAT)
        val bufferSize = maxOf(minBuffer, FRAME_SIZE * 2 * 4)  // at least 4 frames

        audioRecord = AudioRecord(
            audioSource,
            SAMPLE_RATE,
            CHANNEL_CONFIG,
            AUDIO_FORMAT,
            bufferSize
        ).also { record ->
            if (record.state != AudioRecord.STATE_INITIALIZED) {
                throw IllegalStateException("AudioRecord failed to initialize")
            }
        }

        audioRecord!!.startRecording()

        // Attach hardware effects if available (see §4)
        attachHardwareEffects(audioRecord!!.audioSessionId)

        // Read frames continuously
        val frame = ShortArray(FRAME_SIZE)
        while (isActive) {
            val read = audioRecord!!.read(frame, 0, FRAME_SIZE)
            when {
                read > 0 -> trySend(frame.copyOf(read))
                read == AudioRecord.ERROR_INVALID_OPERATION -> {
                    cancel(CancellationException("AudioRecord read error"))
                }
                // read == 0: no data yet, continue
            }
        }

        awaitClose {
            audioRecord?.stop()
            audioRecord?.release()
            audioRecord = null
        }
    }.flowOn(Dispatchers.IO)

    // AudioSource comparison:
    // MIC                   → raw, no processing (for music recording)
    // VOICE_RECOGNITION     → tuned for ASR, some noise filtering, no AEC
    // VOICE_COMMUNICATION   → AEC + NS + AGC enabled (for VoIP)
    // CAMCORDER             → tuned for video recording
    // VOICE_PERFORMANCE     → high quality, minimal processing
}
```

**Choosing AudioSource:**
| AudioSource | AEC | NS | AGC | Best for |
|---|---|---|---|---|
| `MIC` | ❌ | ❌ | ❌ | Music, raw capture |
| `VOICE_RECOGNITION` | ❌ | 🟡 | ❌ | Wake word, ASR (no speaker present) |
| `VOICE_COMMUNICATION` | ✅ | ✅ | ✅ | VoIP, hands-free (speaker playing) |
| `CAMCORDER` | ❌ | 🟡 | ❌ | Video recording |

---

## 4. Hardware Audio Effects (Android Built-in)

Android provides hardware-accelerated audio effects via `AudioEffect` subclasses. These run in the audio HAL — very low latency, no CPU cost.

```kotlin
fun attachHardwareEffects(audioSessionId: Int) {
    // Acoustic Echo Cancellation — CRITICAL for hands-free scenarios
    // Removes the playback audio (speaker output) from the microphone input
    if (AcousticEchoCanceler.isAvailable()) {
        val aec = AcousticEchoCanceler.create(audioSessionId)
        aec?.enabled = true
        // Note: AEC only works if AudioSource is VOICE_COMMUNICATION
    }

    // Noise Suppressor — reduces background noise (fan, wind, crowd)
    if (NoiseSuppressor.isAvailable()) {
        val ns = NoiseSuppressor.create(audioSessionId)
        ns?.enabled = true
    }

    // Automatic Gain Control — normalizes input level
    // Prevents quiet whisperers being too quiet, loud environments clipping
    if (AutomaticGainControl.isAvailable()) {
        val agc = AutomaticGainControl.create(audioSessionId)
        agc?.enabled = true
    }
}

// Important: these effects are not guaranteed on all devices
// isAvailable() returns false on some OEMs / emulators
// Always have a software fallback
```

**Hardware effects caveat:** Availability and quality varies widely by OEM and chip. Qualcomm Snapdragon chips have excellent hardware AEC. Some MediaTek chips have mediocre implementations. Always test on representative devices.

---

## 5. Acoustic Echo Cancellation (AEC) — Deep Dive

### 5.1 The Problem

```
Hands-free scenario:

[Speaker] ──→ plays TTS response / music ──→ sounds emitted
                                               │
                                               ▼
                                         [Room acoustics]
                                               │ echo / reverb
                                               ▼
[Microphone] ──→ captures user's voice ──→ ALSO captures speaker output
```

Without AEC, your voice assistant hears its own TTS responses as "user speech" → infinite feedback loop.

### 5.2 How AEC Works

```
Reference signal (what the speaker is playing) ──────────────────────┐
                                                                      ▼
Microphone input (voice + echo) ──→ [Adaptive Filter] ──→ error signal
                                         ↑                (voice only, echo removed)
                                    continuously
                                    updated model
                                    of room acoustics
```

The adaptive filter models the acoustic path from speaker to microphone and subtracts a predicted echo from the microphone signal. It continuously adapts to room changes.

### 5.3 WebRTC AEC (Software Implementation)

When hardware AEC is unavailable or insufficient:

```kotlin
// WebRTC AEC via webrtc-android library
// implementation("io.github.webrtc-sdk:android:125.6422.06")

class WebRtcAudioProcessor {
    private var aecHandle: Long = 0  // native handle

    fun initialize(sampleRate: Int, numChannels: Int) {
        // Native WebRTC AEC init
        aecHandle = WebRtcAec3_Create(sampleRate)
        // Set parameters: delay, filter length, etc.
    }

    // Call this with the SPEAKER output BEFORE it plays
    // (the reference signal for the echo canceller)
    fun processReferenceSignal(speakerOutput: FloatArray) {
        WebRtcAec3_BufferFarend(aecHandle, speakerOutput)
    }

    // Call this with the MICROPHONE input
    // Returns: microphone input with echo removed
    fun processMicInput(micInput: FloatArray): FloatArray {
        val output = FloatArray(micInput.size)
        WebRtcAec3_Process(aecHandle, micInput, output)
        return output
    }

    fun release() {
        WebRtcAec3_Free(aecHandle)
    }
}
```

**Critical timing:** The reference signal (speaker output) must be buffered at the known delay ahead of the microphone signal. Getting the delay wrong means AEC doesn't work. Hardware AEC handles this delay automatically; software AEC requires explicit delay specification.

---

## 6. Noise Suppression

### 6.1 Hardware (NoiseSuppressor API)

Already shown in §4 — one-liner if available, variable quality by device.

### 6.2 WebRTC Noise Suppressor (Software)

Part of the WebRTC stack, well-tuned for speech:

```kotlin
// WebRTC NS processes 10ms frames at 16kHz (160 samples per frame)
class WebRtcNoiseSuppressor {
    // NS modes: 0=mild, 1=medium, 2=aggressive, 3=very_aggressive
    // Higher mode = more noise removed BUT more speech artifacts
    private val nsMode = 2  // aggressive — good for noisy environments

    fun processFrame(frame: ShortArray): ShortArray {
        val floatFrame = FloatArray(frame.size) { frame[it] / 32768f }
        // native call to WebRTC NS
        val processed = WebRtcNs_Process(floatFrame, nsMode)
        return ShortArray(processed.size) { (processed[it] * 32768).toInt().toShort() }
    }
}
```

### 6.3 RNNoise (ML-based, state of the art)

RNNoise is a recurrent neural network noise suppressor developed by Mozilla. Remarkably small (~100KB), runs on CPU, better quality than WebRTC NS for real-world noise:

```kotlin
// Via JNI — port of RNNoise C library
class RNNoiseProcessor {
    // RNNoise processes fixed 480-sample frames (10ms at 48kHz; 30ms at 16kHz)
    private val FRAME_SIZE = 480
    private var state: Long = rnnoise_create(null)  // native state

    fun processFrame(inputSamples: ShortArray): ShortArray {
        val floatInput = FloatArray(FRAME_SIZE) {
            if (it < inputSamples.size) inputSamples[it].toFloat() else 0f
        }
        val floatOutput = FloatArray(FRAME_SIZE)
        // Returns voice probability (0.0-1.0) — doubles as VAD!
        val voiceProbability = rnnoise_process_frame(state, floatOutput, floatInput)

        return ShortArray(FRAME_SIZE) { floatOutput[it].toInt().toShort() }
    }

    fun release() = rnnoise_destroy(state)
}
```

**NS comparison:**
| | Hardware NS | WebRTC NS | RNNoise |
|---|---|---|---|
| Quality | Variable (OEM) | Good | Best |
| Latency | ~0ms | ~10ms | ~30ms |
| CPU cost | 0 | Low | Low |
| Size | 0 | ~500KB | ~100KB |
| Availability | Device-dependent | Any | Any (JNI needed) |
| VAD built-in | ❌ | ❌ | ✅ |

---

## 7. Automatic Gain Control (AGC)

AGC ensures consistent loudness regardless of microphone distance or environment:

```kotlin
class SoftwareAGC(
    private val targetLevelDbfs: Float = -18f,  // target level (dBFS)
    private val maxGainDb: Float = 30f          // max amplification
) {
    private var currentGain = 1.0f
    private val smoothingFactor = 0.1f  // smoothing prevents abrupt gain changes

    fun processFrame(frame: ShortArray): ShortArray {
        // Calculate RMS (Root Mean Square) level
        val rms = sqrt(frame.map { it.toFloat().pow(2) }.average()).toFloat()
        val dbfs = if (rms > 0) 20f * log10(rms / 32768f) else -120f

        // Calculate desired gain adjustment
        val desiredGainDb = targetLevelDbfs - dbfs
        val clampedGainDb = desiredGainDb.coerceIn(-maxGainDb, maxGainDb)
        val desiredGain = 10f.pow(clampedGainDb / 20f)

        // Smooth gain changes (prevent pumping effect)
        currentGain = currentGain + smoothingFactor * (desiredGain - currentGain)

        // Apply gain
        return ShortArray(frame.size) {
            (frame[it] * currentGain).toInt()
                .coerceIn(Short.MIN_VALUE.toInt(), Short.MAX_VALUE.toInt())
                .toShort()
        }
    }
}
```

---

## 8. VAD (Voice Activity Detection)

> Full VAD implementation covered in **Section 26 — On-Device ML & Local LLMs (§9)**. This section adds the integration context.

### 8.1 VAD in the Pipeline

```kotlin
class AudioProcessingPipeline(
    private val vad: VoiceActivityDetector,
    private val noiseSuppressor: WebRtcNoiseSuppressor,
    private val agc: SoftwareAGC
) {
    // Padding: keep N frames before/after speech for natural word boundaries
    private val PRE_SPEECH_PADDING_FRAMES = 10   // 200ms before
    private val POST_SPEECH_PADDING_FRAMES = 15  // 300ms after

    private val frameBuffer = ArrayDeque<ShortArray>()
    private var speechFrameCount = 0
    private var silenceFrameCount = 0
    private var isSpeaking = false

    data class ProcessedAudio(
        val frames: List<ShortArray>,
        val isSpeechSegment: Boolean
    )

    fun processFrame(rawFrame: ShortArray): ProcessedAudio? {
        // Step 1: Apply NS
        val denoised = noiseSuppressor.processFrame(rawFrame)

        // Step 2: Apply AGC
        val normalized = agc.processFrame(denoised)

        // Step 3: VAD
        val isSpeechFrame = vad.isVoiceFrame(normalized)

        // Step 4: Buffer management with padding
        frameBuffer.addLast(normalized)
        if (frameBuffer.size > PRE_SPEECH_PADDING_FRAMES + POST_SPEECH_PADDING_FRAMES + 50) {
            frameBuffer.removeFirst()  // prevent unbounded growth
        }

        return when {
            isSpeechFrame -> {
                silenceFrameCount = 0
                if (!isSpeaking) {
                    isSpeaking = true
                    speechFrameCount = 0
                }
                speechFrameCount++
                null  // accumulating speech
            }
            isSpeaking -> {
                silenceFrameCount++
                if (silenceFrameCount >= POST_SPEECH_PADDING_FRAMES) {
                    // Speech segment ended — emit the buffered speech
                    isSpeaking = false
                    val speechFrames = frameBuffer.toList()
                    frameBuffer.clear()
                    ProcessedAudio(speechFrames, isSpeechSegment = true)
                } else null  // still in post-speech padding
            }
            else -> null  // silence, discard
        }
    }
}
```

---

## 9. Hesitation Filtering

> VAD-based audio silence removal covered in **Section 26 — On-Device ML (§9.3–9.4)**. This section covers the ASR transcript level.

```kotlin
// Two levels of hesitation filtering:
// Level 1: Audio-level — remove silence frames (VAD, covered above)
// Level 2: Transcript-level — remove filler words from ASR output

class TranscriptHesitationFilter(private val locale: Locale) {

    // Filler words by language
    private val fillerWords: Map<String, Set<String>> = mapOf(
        "en" to setOf("um", "uh", "er", "hmm", "like", "you know", "basically", "literally", "right"),
        "de" to setOf("äh", "ähm", "also", "sozusagen"),
        "fr" to setOf("euh", "ben", "voilà"),
        "es" to setOf("eh", "este", "o sea"),
        "hi" to setOf("um", "uh", "matlab", "woh")
    )

    private val currentFillers = fillerWords[locale.language] ?: fillerWords["en"]!!

    data class TranscriptWord(
        val text: String,
        val startTimeMs: Long,
        val endTimeMs: Long,
        val confidence: Float
    )

    fun filter(words: List<TranscriptWord>): List<TranscriptWord> {
        return words
            .filterNot { word ->
                word.text.lowercase().trim() in currentFillers ||
                word.confidence < 0.4f  // remove low-confidence words
            }
            .let { removeRepetitions(it) }
            .let { removeFalseStarts(it) }
    }

    // "I I want to go" → "I want to go"
    private fun removeRepetitions(words: List<TranscriptWord>): List<TranscriptWord> {
        val result = mutableListOf<TranscriptWord>()
        var i = 0
        while (i < words.size) {
            if (i + 1 < words.size &&
                words[i].text.equals(words[i + 1].text, ignoreCase = true)) {
                i++  // skip first occurrence, keep second
            } else {
                result.add(words[i])
                i++
            }
        }
        return result
    }

    // Remove false starts: "I want to— I want to go" → "I want to go"
    // Detected by short words followed by similar content
    private fun removeFalseStarts(words: List<TranscriptWord>): List<TranscriptWord> {
        // Simple heuristic: if a sentence fragment < 3 words is followed by
        // similar content starting with the same word, remove the fragment
        // Full implementation would use NLP similarity scoring
        return words
    }
}
```

---

## 10. Audio Format Conversion for ASR

Different ASR services accept different formats:

```kotlin
class AudioFormatConverter {

    // Convert ShortArray (PCM-16bit) to ByteArray (for streaming)
    fun shortArrayToByteArray(shorts: ShortArray): ByteArray {
        val bytes = ByteArray(shorts.size * 2)
        for (i in shorts.indices) {
            bytes[i * 2] = (shorts[i].toInt() and 0xFF).toByte()       // low byte
            bytes[i * 2 + 1] = (shorts[i].toInt() shr 8 and 0xFF).toByte()  // high byte
        }
        return bytes
    }

    // Write WAV header (for APIs that need a file, not raw PCM)
    fun writeWavHeader(output: OutputStream, sampleRate: Int, bitsPerSample: Int, numChannels: Int) {
        val dataSize = 0  // 0 for streaming (unknown)
        val byteRate = sampleRate * numChannels * bitsPerSample / 8
        val blockAlign = (numChannels * bitsPerSample / 8).toShort()

        output.write("RIFF".toByteArray())
        output.writeInt(dataSize + 36)  // ChunkSize
        output.write("WAVE".toByteArray())
        output.write("fmt ".toByteArray())
        output.writeInt(16)             // Subchunk1Size (PCM = 16)
        output.writeShort(1)            // AudioFormat (PCM = 1)
        output.writeShort(numChannels.toShort())
        output.writeInt(sampleRate)
        output.writeInt(byteRate)
        output.writeShort(blockAlign)
        output.writeShort(bitsPerSample.toShort())
        output.write("data".toByteArray())
        output.writeInt(dataSize)
    }

    // Resample if needed (e.g., 44.1kHz → 16kHz for speech)
    // Simple linear interpolation (production: use SpeexResampler or soxr)
    fun resample(input: ShortArray, inputRate: Int, outputRate: Int): ShortArray {
        val ratio = inputRate.toDouble() / outputRate
        val outputSize = (input.size / ratio).toInt()
        return ShortArray(outputSize) { i ->
            val srcIndex = (i * ratio).toInt().coerceAtMost(input.size - 1)
            input[srcIndex]
        }
    }
}

private fun OutputStream.writeInt(value: Int) {
    write(value and 0xFF)
    write((value shr 8) and 0xFF)
    write((value shr 16) and 0xFF)
    write((value shr 24) and 0xFF)
}
private fun OutputStream.writeShort(value: Short) {
    write(value.toInt() and 0xFF)
    write((value.toInt() shr 8) and 0xFF)
}
```

---

## 11. Streaming to Cloud ASR

### 11.1 Chunked HTTP Streaming

```kotlin
class CloudAsrClient @Inject constructor(
    private val okHttpClient: OkHttpClient
) {
    // Stream audio chunks to Google Speech-to-Text (or similar)
    suspend fun streamRecognize(
        audioFlow: Flow<ShortArray>,
        languageCode: String = "en-US"
    ): Flow<String> = channelFlow {

        val pipe = Pipe(65536)  // 64KB pipe buffer
        val pipeSource = pipe.source().buffer()
        val pipeSink = pipe.sink().buffer()

        // Writer: push audio frames into pipe
        launch(Dispatchers.IO) {
            val converter = AudioFormatConverter()
            audioFlow.collect { frame ->
                val bytes = converter.shortArrayToByteArray(frame)
                pipeSink.write(bytes)
                pipeSink.flush()
            }
            pipeSink.close()
        }

        // Reader: stream pipe to HTTP
        val requestBody = object : RequestBody() {
            override fun contentType() = "audio/l16;rate=16000".toMediaType()
            override fun writeTo(sink: BufferedSink) {
                sink.writeAll(pipeSource)
            }
        }

        val request = Request.Builder()
            .url("https://speech.googleapis.com/v1/speech:recognize")
            .header("Authorization", "Bearer $accessToken")
            .post(requestBody)
            .build()

        okHttpClient.newCall(request).execute().use { response ->
            // Parse streaming response
            response.body?.source()?.let { source ->
                while (!source.exhausted()) {
                    val line = source.readUtf8Line() ?: break
                    if (line.isNotEmpty()) {
                        val transcript = parseTranscript(line)
                        if (transcript != null) send(transcript)
                    }
                }
            }
        }
    }
}
```

### 11.2 WebSocket Streaming (for real-time ASR)

```kotlin
class WebSocketAsrClient(private val wsManager: WebSocketManager) {

    fun streamAudio(audioFlow: Flow<ShortArray>): Flow<AsrResult> = channelFlow {
        // Connect to ASR WebSocket endpoint
        wsManager.connect("wss://asr.example.com/stream")

        // Receive ASR results
        launch {
            wsManager.messages.collect { message ->
                val result = Json.decodeFromString<AsrResult>(message)
                send(result)
            }
        }

        // Send audio frames
        val converter = AudioFormatConverter()
        audioFlow.collect { frame ->
            val bytes = converter.shortArrayToByteArray(frame)
            wsManager.sendBinary(bytes)
        }

        // Signal end of audio
        wsManager.send("""{"type":"end_of_audio"}""")

        awaitClose { wsManager.disconnect() }
    }
}
```

---

## 12. Complete Pipeline Implementation

```kotlin
class VoiceInputManager @Inject constructor(
    @ApplicationContext private val context: Context,
    private val asrClient: CloudAsrClient,
    private val scope: CoroutineScope
) {
    private val capture = MicrophoneCapture(scope)
    private val noiseSuppressor = WebRtcNoiseSuppressor()
    private val agc = SoftwareAGC(targetLevelDbfs = -18f)
    private val vad = NeuralVAD(context)
    private val pipeline = AudioProcessingPipeline(vad, noiseSuppressor, agc)
    private val transcriptFilter = TranscriptHesitationFilter(Locale.getDefault())

    // Full pipeline: mic → process → VAD → ASR → filter
    fun startListening(): Flow<String> = capture
        .startCapture(MediaRecorder.AudioSource.VOICE_RECOGNITION)
        .map { rawFrame ->
            // Apply processing pipeline per frame
            pipeline.processFrame(rawFrame)
        }
        .filterNotNull()  // only emit when speech segment is complete
        .filter { it.isSpeechSegment }
        .flatMapConcat { speechSegment ->
            // Stream speech segment to ASR
            asrClient.streamRecognize(
                audioFlow = speechSegment.frames.asFlow(),
                languageCode = Locale.getDefault().toLanguageTag()
            )
        }
        .map { rawTranscript ->
            // Filter hesitations from transcript
            val words = parseWordsWithTimestamps(rawTranscript)
            val filtered = transcriptFilter.filter(words)
            filtered.joinToString(" ") { it.text }
        }
        .filter { it.isNotBlank() }
        .flowOn(Dispatchers.Default)

    // Use in ViewModel
    fun startVoiceInput() {
        viewModelScope.launch {
            startListening().collect { transcript ->
                _uiState.update { it.copy(transcript = transcript) }
            }
        }
    }
}
```

---

## 13. Latency Considerations

```
Total pipeline latency budget for a voice assistant:
  AudioRecord buffer:     20ms  (1 frame)
  AEC processing:         0ms   (hardware) / 10ms (software)
  NS processing:          0ms   (hardware) / 10–30ms (software)
  VAD:                    20–30ms
  Post-speech padding:    300ms  (must wait to confirm speech ended)
  Network round trip:     100–500ms
  ASR processing:         100–500ms (streaming, partial results faster)
  ─────────────────────────────────────────
  Total (hardware effects): ~550–900ms
  Total (software effects): ~600–1000ms

For wake-word detection (Alexa "Hey Alexa"):
  Must respond in < 500ms to feel responsive
  Solution: wake-word runs locally in parallel with cloud ASR
  Local wake-word (Snowboy, PocketSphinx): < 10ms

For dictation (where users type slowly):
  Post-speech padding can be 500ms–1s
  No latency pressure — optimize for accuracy

For voice commands (user expects fast response):
  Reduce post-speech padding: 200–300ms
  Use streaming ASR (partial results) to start processing before silence
```

---

## 14. HLD — Voice Input Architecture

### "Design the audio input pipeline for Alexa"

```
Wake Word Detection (always on, local):
  Tiny ML model (< 1MB) running on DSP/NPU
  Processes 20ms frames continuously
  On trigger: wake CPU, start recording

Full Recording Pipeline (on wake):
  AudioRecord (VOICE_RECOGNITION, 16kHz, 16-bit mono)
  Hardware AEC (if available) — critical for Echo devices with speaker
  Software AEC fallback (WebRTC AEC3) for mobile devices
  NS: hardware first, RNNoise fallback
  AGC: hardware first, software fallback
  VAD: marks speech frames vs silence

Streaming to Cloud:
  Speech frames streamed in real-time via WebSocket (not batch upload)
  Format: Opus (compressed, ~32kbps) or raw PCM-16
  Partial results returned as ASR processes each frame
  Final result returned when ASR detects end-of-utterance

Post-processing (cloud):
  Intent recognition (NLU on transcript)
  Response generation
  TTS response streamed back

Privacy:
  Only audio after wake word is sent to cloud
  Local VAD + end-of-utterance detection minimizes unnecessary data
  No audio logging without explicit user consent
  On-device option (Whisper.cpp) for fully private mode
```

---

## 15. React Native Audio Input

```typescript
// react-native-audio-record — PCM audio from microphone
import AudioRecord from 'react-native-audio-record'

AudioRecord.init({
    sampleRate: 16000,
    channels: 1,
    bitsPerSample: 16,
    audioSource: 6,  // VOICE_RECOGNITION on Android
    wavFile: 'audio.wav'
})

AudioRecord.on('data', (data) => {
    // Base64 encoded PCM data
    const pcmBytes = Buffer.from(data, 'base64')
    processPcmFrame(pcmBytes)
})

AudioRecord.start()
const file = await AudioRecord.stop()  // returns WAV file path

// For real-time processing in RN:
// Process in native module (JNI/C++) for performance
// JS thread is too slow for real-time audio processing (use native bridge)

// @react-native-voice/voice — high-level speech recognition
import Voice from '@react-native-voice/voice'

Voice.onSpeechResults = (e) => setTranscript(e.value[0])
Voice.onSpeechPartialResults = (e) => setPartialTranscript(e.value[0])
await Voice.start('en-US')  // uses on-device ASR (SpeechRecognizer API)
```

---

## 16. Common Misunderstandings & Pitfalls

**❌ Using `AudioSource.MIC` when AEC is needed**
Hardware AEC only activates when `AudioSource.VOICE_COMMUNICATION` is used. Using `AudioSource.MIC` or `VOICE_RECOGNITION` disables hardware AEC even if you enable `AcousticEchoCanceler`. For any scenario with a speaker playing simultaneously (voice assistant, VoIP), use `VOICE_COMMUNICATION`.

**❌ Reading audio on the main thread**
`audioRecord.read()` is a blocking call. Always run it in a `Dispatchers.IO` coroutine or a dedicated thread. Blocking the main thread for audio = immediate ANR.

**❌ Applying effects before AudioRecord starts**
Hardware effects (`AcousticEchoCanceler`, `NoiseSuppressor`) must be created and enabled AFTER `AudioRecord` is created and started. Creating them before `startRecording()` may cause the session ID to be invalid.

**❌ Not releasing AudioRecord on error**
If `AudioRecord.STATE_ERROR` is returned during initialization, the AudioRecord object is in an invalid state. Release it immediately and throw an exception. Keeping an invalid `AudioRecord` and calling `startRecording()` on it causes undefined behavior.

**❌ Software AEC without correct reference signal timing**
Software AEC requires the reference signal (speaker output) to be delivered at the correct delay relative to the microphone signal — the acoustic delay from speaker to microphone. If this delay is wrong, AEC performs worse than no AEC. Hardware AEC handles delay automatically.

**❌ Not handling audio focus before recording**
Starting recording without requesting audio focus means you'll capture audio from other apps' output through the microphone (for voice assistants). Always request `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE` before starting voice recording to ensure the environment is as clean as possible.

**❌ Incorrect VAD padding leading to clipped words**
Too short a post-speech padding (< 200ms) clips the ends of words — "record" becomes "recor". Users with speech impediments or slow speakers need more padding. Always add at least 200ms before and 300ms after detected speech.

**❌ Processing audio on the same thread as UI**
Audio processing (NS, AGC, VAD) involves CPU-heavy DSP operations. Run on `Dispatchers.Default`, never on Main. Use `flowOn(Dispatchers.Default)` in the pipeline flow.

---

## 17. Best Practices

- **`VOICE_RECOGNITION` for ASR, `VOICE_COMMUNICATION` for VoIP** — source affects which hardware effects activate
- **Hardware effects first, software fallback** — always check `isAvailable()` before `create()`
- **Processing on `Dispatchers.Default`** — never on Main thread
- **Frame size: 20ms at 16kHz** — 320 samples; matches WebRTC, VAD, and most ASR expectations
- **VAD padding** — 200ms pre-speech, 300ms post-speech minimum
- **Stream to ASR via WebSocket** — not batch upload; streaming gives partial results and lower perceived latency
- **Request audio focus before recording** — `AUDIOFOCUS_GAIN_TRANSIENT_EXCLUSIVE`
- **Release AudioRecord in `finally`** — never leak; always stop + release in try/finally
- **Level meter UI** — show the user that the mic is active; display RMS level as a visual indicator
- **Localize filler word lists** — "um/uh" in English, "äh/ähm" in German; don't hardcode English
- **Smooth AGC** — use smoothing factor to prevent pumping (abrupt gain jumps)
- **Monitor pipeline latency** — measure time from first VAD speech frame to ASR response; alert if > 1s
- **Privacy by default** — buffer locally until wake word, only stream confirmed speech to cloud

---

## 18. Interview Q&A

**Q1: Walk me through the audio pipeline for a voice assistant like Alexa.**

> The pipeline has three phases. Pre-activation: a tiny ML model (< 1MB) runs continuously on the device's DSP or NPU, processing 20ms audio frames to detect the wake word ("Alexa") locally, using no cloud resources. On wake-word detection: the full recording pipeline starts — `AudioRecord` with `VOICE_COMMUNICATION` source activates hardware AEC (critical because Alexa's speaker output would otherwise feed back into the mic), then hardware NS, then AGC to normalize volume. VAD detects when the user starts and stops speaking, with pre/post-speech padding to avoid clipping. Active utterance: each 20ms frame is processed through the pipeline and the speech frames are streamed in real-time via WebSocket to the cloud ASR in compressed Opus format. Partial results come back as ASR processes each chunk, so intent recognition can start before the user finishes speaking. End-of-utterance is detected both locally (VAD silence) and by the ASR service.

---

**Q2: What is AEC and why is it essential for hands-free voice?**

> AEC — Acoustic Echo Cancellation — removes the device's own speaker output from the microphone signal. Without it, when Alexa speaks a response, the microphone captures that audio and sends it back to the ASR cloud, which tries to interpret Alexa's own voice as a user command — creating an infinite feedback loop. AEC works by maintaining a reference signal of what the speaker is playing, running an adaptive filter that models the acoustic path from speaker to microphone (including room reflections), and subtracting the predicted echo from the microphone input. The filter continuously adapts as the room acoustics change. On Android, hardware AEC is available via `AcousticEchoCanceler` and only activates with `AudioSource.VOICE_COMMUNICATION`. For devices with poor hardware AEC, WebRTC AEC3 provides a software implementation that's used in Chrome and hundreds of millions of devices.

---

**Q3: What is the difference between `AudioSource.VOICE_RECOGNITION` and `AudioSource.VOICE_COMMUNICATION`?**

> Both are ASR-oriented, but they activate different hardware processing chains. `VOICE_RECOGNITION` is optimized for speech input without a concurrent speaker — think a user speaking in a quiet room. It may apply light noise shaping but does NOT activate AEC. `VOICE_COMMUNICATION` activates the full voice processing chain: AEC, noise suppression, and AGC. It's designed for scenarios where the device speaker is also playing audio (VoIP calls, voice assistants). For a dictation app where no speaker is playing, `VOICE_RECOGNITION` is correct. For a voice assistant app or VoIP app where the speaker plays TTS or the other party's voice, `VOICE_COMMUNICATION` is essential — otherwise the microphone captures the speaker output and ASR hears it as speech.

---

**Q4: How do you handle the trade-off between processing latency and quality in a real-time audio pipeline?**

> The key decision is hardware vs software processing. Hardware effects (AEC, NS, AGC via Android AudioEffect APIs) add near-zero latency and cost zero CPU but have variable quality across OEMs. Software processing (WebRTC NS, RNNoise, software AGC) adds 10–30ms per stage but gives consistent quality across devices. For a production voice assistant: use hardware effects as the first pass and layer software processing on top for additional quality. For VAD, the 20ms frame size is the minimum latency unit — I accept this as the baseline. Post-speech padding adds 200–300ms but is necessary to avoid clipping. For the overall pipeline, I keep a latency budget: the user should get a response within 1 second of finishing speaking. If cloud ASR adds 400ms, I have 600ms for the local pipeline — more than enough. For truly latency-critical scenarios (sub-500ms response), run a local ASR model in parallel with cloud and use whichever responds first.

---

**Q5: How do you detect when the user has finished speaking?**

> Two signals combined. First: VAD-based silence detection. The VAD classifies each 20ms frame as speech or silence. When silence frames accumulate beyond a threshold (typically 300–500ms of continuous silence after speech), the utterance is considered complete. The threshold is tuned: too short clips slow speakers or natural pauses mid-sentence; too long makes the assistant feel unresponsive. Second: ASR-side end-of-utterance. Streaming ASR services (Google STT, Alexa ASR) also detect end of utterance server-side based on acoustic and language model signals. The first to detect it wins. I also expose an explicit "stop" button in the UI for users who prefer to control submission manually — important for users with speech differences who naturally pause longer.

---

**Q6: How do you minimize unnecessary data sent to the cloud ASR?**

> Three techniques. First, local VAD: only stream frames classified as speech — silence frames (background noise, pauses) are discarded before transmission. This typically reduces transmitted audio by 50–80% compared to sending everything. Second, audio compression: raw PCM at 16kHz, 16-bit is 256Kbps. Encoding to Opus at 32Kbps gives 8:1 compression with minimal speech quality loss. Third, pre-speech buffering: buffer the last 200ms of audio locally before VAD triggers (pre-speech padding). Send this buffer when VAD activates so the ASR has the natural word onset, but we haven't been streaming silence before that. Together: VAD reduces frames × 8x compression = typical 16x bandwidth reduction vs naively streaming everything. Privacy benefit: the VAD gatekeeping means nothing is sent to the cloud during silence, which also reduces accidental capture of private conversations.

---

*Previous: [16 — Audio Management & Focus](./16-audio-management.md)*
*Next: [18 — State Management](./18-state-management.md)*
