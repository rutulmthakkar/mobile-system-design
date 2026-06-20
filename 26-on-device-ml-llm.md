# 26 — On-Device ML & Local LLMs

> **Round type: Both (HLD + LLD)**
> **HLD:** Deciding when to run AI on-device vs cloud, model delivery architecture, feature design
> **LLD:** TFLite, ML Kit, Gemini Nano/AICore, MediaPipe, VAD, audio processing pipeline

---

## 1. Why This Matters in Interviews

On-device ML is a fast-moving area that increasingly appears in interviews — especially at Google, Meta, Amazon, and consumer AI companies. The question isn't just "can you call an API" but "when does it make sense to run inference locally, and how do you design for it?"

**Common interview angles:**
- "How would you implement smart reply suggestions without sending messages to a server?"
- "A user speaks to the app in a noisy environment — how do you filter hesitations before sending audio to ASR?"
- "How does Gemini Nano differ from calling the Gemini API?"
- "What is quantization and why does it matter for on-device models?"
- "How do you handle the case where a model isn't supported on the user's device?"
- "Design a predictive text system that works offline"

---

## 2. On-Device vs Cloud — The Core Decision

```
On-Device ML                          Cloud ML (API call)
─────────────────────────────         ──────────────────────────────
✅ Works offline                       ✅ Unlimited model size/power
✅ Zero latency (no network round trip) ✅ Latest models always available
✅ Privacy — data never leaves device  ✅ No device RAM/CPU constraints
✅ No API cost per inference           ✅ Simpler to update model
✅ Works on locked-down enterprise devices ❌ Network dependency
❌ Limited model size (RAM/storage)    ❌ Latency (100ms–2s per call)
❌ Device capability variance          ❌ Cost at scale
❌ Model updates require app update    ❌ Privacy concerns (data leaves device)
❌ Hardware fragmentation              ❌ Doesn't work offline
```

**Decision framework:**
```
Is the data sensitive (health, private messages, biometrics)?
  → On-device preferred (data never leaves)

Does it need sub-100ms latency (real-time suggestions, audio)?
  → On-device (network round trip too slow)

Is the device offline sometimes?
  → On-device (or on-device with cloud fallback)

Is the model > 2GB or requires GPT-4 level capability?
  → Cloud (on-device can't support it)

Is the feature used by all users equally?
  → Consider device fragmentation — low-end devices may not support it
```

---

## 3. The Android On-Device ML Stack (2024–2025)

```
┌─────────────────────────────────────────────────────────────┐
│                   On-Device ML on Android                    │
├──────────────────────┬──────────────────────┬───────────────┤
│   ML KIT (Firebase)  │  MEDIAPIPE           │  AICORE       │
│  Ready-made tasks:   │  Custom models:      │  (Android 14+)│
│  Smart reply         │  LLM inference       │  Gemini Nano  │
│  Text classification │  Image segmentation  │  System-level │
│  Language ID         │  Face landmarks      │  Zero app size│
│  Translation         │  Audio tasks         │  impact       │
│  Summarization       │  Custom TFLite       │               │
├──────────────────────┴──────────────────────┴───────────────┤
│              TensorFlow Lite / LiteRT (foundation)           │
│        Hardware acceleration: GPU, NNAPI, NPU (Hexagon)      │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Quantization — Why Models Fit on Mobile

A 3B parameter model at FP32 = 12GB RAM. That's more than most phones have total.

**Quantization** reduces numerical precision to shrink model size and memory:

| Precision | Bytes/param | 3B model size | Quality loss |
|---|---|---|---|
| FP32 (full) | 4 bytes | 12 GB | None (baseline) |
| FP16 | 2 bytes | 6 GB | Minimal |
| INT8 | 1 byte | 3 GB | Small |
| INT4 | 0.5 bytes | **1.5 GB** | Moderate |
| INT2/3 | 0.25–0.375 bytes | ~1 GB | Significant |

**INT4 is the sweet spot for mobile LLMs** — Gemini Nano uses INT4 quantization. At ~1.5–2GB, it fits in RAM while leaving room for the OS and other apps.

**Types of quantization:**
- **Post-training quantization (PTQ)** — quantize after training; fastest but most quality loss
- **Quantization-aware training (QAT)** — train with quantization in the loop; better quality
- **GPTQ / AWQ** — advanced INT4 methods for LLMs with minimal perplexity loss

---

## 5. TensorFlow Lite / LiteRT

The foundation for running any custom ML model on Android. TFLite converts TensorFlow or PyTorch models to an optimized `.tflite` format.

### 5.1 Basic TFLite Inference

```kotlin
// gradle
implementation("org.tensorflow:tensorflow-lite:2.14.0")
implementation("org.tensorflow:tensorflow-lite-support:0.4.4")
implementation("org.tensorflow:tensorflow-lite-gpu:2.14.0")  // GPU acceleration

// Load model from assets
val model = FileUtil.loadMappedFile(context, "text_classifier.tflite")

// Create interpreter with hardware acceleration
val options = Interpreter.Options().apply {
    addDelegate(GpuDelegate())  // use GPU if available
    // or: addDelegate(NnApiDelegate())  // use NNAPI (Android neural networks API)
    numThreads = 4
}
val interpreter = Interpreter(model, options)

// Run inference
val input = Array(1) { FloatArray(128) }  // [batch=1, seq_len=128]
val output = Array(1) { FloatArray(2) }   // [batch=1, num_classes=2]
interpreter.run(input, output)

val probabilities = output[0]  // [positive_prob, negative_prob]
val label = if (probabilities[0] > probabilities[1]) "positive" else "negative"
```

### 5.2 Hardware Acceleration

| Delegate | Hardware | When to use |
|---|---|---|
| **GPU Delegate** | Device GPU | Image, matrix operations; most devices |
| **NNAPI Delegate** | NPU/DSP/GPU | Android 8.1+; uses vendor-optimized neural hardware |
| **Hexagon Delegate** | Qualcomm Hexagon DSP | Snapdragon devices; best power efficiency |
| **CPU (default)** | CPU | Fallback when no accelerator available |

```kotlin
// Try delegates in priority order
val options = Interpreter.Options()
try {
    options.addDelegate(GpuDelegate())
} catch (e: Exception) {
    try {
        options.addDelegate(NnApiDelegate())
    } catch (e2: Exception) {
        options.numThreads = 4  // fall back to multi-threaded CPU
    }
}
```

### 5.3 Model Delivery — Don't Bundle Large Models in APK

```kotlin
// Option 1: Firebase ML — download model over the air
// Users always get the latest model; no APK size impact
val remoteModel = CustomRemoteModel.Builder(FirebaseModelSource.Builder("text_classifier").build()).build()
val downloadConditions = CustomModelDownloadConditions.Builder()
    .requireWifi()  // only download on WiFi
    .build()

FirebaseModelManager.getInstance().download(remoteModel, downloadConditions)
    .addOnSuccessListener { /* model ready */ }

// Option 2: Dynamic Feature Module — download on demand
// Model shipped in a separate module, installed only if user needs the feature

// Option 3: Bundle in assets (small models only — < 5MB)
// Instant access, no download, but increases APK size
```

---

## 6. ML Kit — Ready-Made On-Device Tasks

ML Kit provides pre-built, Google-trained models via a simple API. No ML expertise required.

### 6.1 Text Tasks

```kotlin
// gradle
implementation("com.google.mlkit:smart-reply:17.0.3")
implementation("com.google.mlkit:language-id:17.0.4")
implementation("com.google.mlkit:translate:17.0.2")

// Smart Reply — generates contextual reply suggestions from conversation
val conversation = listOf(
    TextMessage.createForRemoteUser("Hey, are you free tomorrow?", timestamp, "user_a"),
    TextMessage.createForLocalUser("I might be, why?", timestamp2)
)
val smartReply = SmartReply.getClient()
smartReply.suggestReplies(conversation)
    .addOnSuccessListener { result ->
        if (result.status == SmartReplySuggestionResult.STATUS_SUCCESS) {
            val suggestions = result.suggestions
            // e.g. ["Sure!", "What did you have in mind?", "Let me check"]
            showSuggestedReplies(suggestions.map { it.text })
        }
    }

// Language Identification
val languageId = LanguageIdentification.getClient()
languageId.identifyLanguage("Hello, how are you?")
    .addOnSuccessListener { language ->
        // language = "en"
        translateIfNeeded(language)
    }

// On-device Translation
val translator = Translation.getClient(
    TranslatorOptions.Builder()
        .setSourceLanguage(TranslateLanguage.SPANISH)
        .setTargetLanguage(TranslateLanguage.ENGLISH)
        .build()
)
translator.downloadModelIfNeeded()
    .continueWithTask { translator.translate("Hola, ¿cómo estás?") }
    .addOnSuccessListener { translatedText ->
        // "Hello, how are you?"
    }
```

### 6.2 Vision Tasks

```kotlin
// Text recognition (OCR)
implementation("com.google.mlkit:text-recognition:16.0.0")
val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)
val inputImage = InputImage.fromBitmap(bitmap, 0)
recognizer.process(inputImage)
    .addOnSuccessListener { result ->
        val fullText = result.text
        // Line-by-line: result.textBlocks[].lines[].text
    }

// Barcode scanning
// Face detection
// Object detection and tracking
// Pose detection (body landmarks)
// Selfie segmentation (background removal)
```

### 6.3 GenAI APIs via ML Kit (Google I/O 2025) — Gemini Nano via ML Kit

```kotlin
// gradle
implementation("com.google.android.gms:play-services-mlkit-genai-summarization:16.0.0")

// Summarization backed by Gemini Nano (system-level, no model download in app)
val summarizerOptions = SummarizerOptions.builder(context)
    .setInputType(InputType.ARTICLE)
    .setOutputType(OutputType.ONE_BULLET)
    .setLanguage(Language.ENGLISH)
    .build()
val summarizer = Summarization.getClient(summarizerOptions)

// Check if feature is available on this device
val featureStatus = summarizer.checkFeatureStatus().await()
when (featureStatus) {
    FeatureStatus.AVAILABLE    -> summarize()
    FeatureStatus.DOWNLOADABLE -> { summarizer.requestFeatureDownload().await(); summarize() }
    FeatureStatus.DOWNLOADING  -> waitForDownload()
    FeatureStatus.UNAVAILABLE  -> fallbackToCloud()
}

// Run summarization
suspend fun summarize() {
    val request = SummarizationRequest.builder(articleText).build()
    val response = summarizer.summarize(request).await()
    showSummary(response.summary)
}
```

Available GenAI tasks via ML Kit (as of 2025): Summarization, Smart Reply, Proofreading, Rewriting.

---

## 7. Gemini Nano & AICore (Android 14+)

### 7.1 What AICore Is

AICore is a **system-level service** introduced in Android 14. It hosts Gemini Nano and makes it available to apps without the app bundling the model. The model is managed by the OS — updated silently, shared across apps, zero download cost to the app.

```
Traditional approach:                AICore approach:
App bundles model (500MB–2GB)        System manages Gemini Nano
Each app downloads separately        One model, shared by all apps
App size increases massively         Zero APK size impact
Model updates = app update           Model updates = OS update
```

**Supported devices (as of 2025):** Pixel 8+ (initial), Pixel 9 series, Samsung Galaxy S24+ series, and growing. Not available on all Android devices.

### 7.2 MediaPipe LLM Inference API (Custom Models)

For when you need more control — run open-weight models (Gemma, Phi, Falcon) locally:

```kotlin
// gradle
implementation("com.google.mediapipe:tasks-genai:0.10.14")

// Download model (e.g., Gemma 2B ~ 1.3GB) — use Dynamic Delivery
// Push model file to device during development:
// adb push gemma-2b-it-cpu-int4.bin /data/local/tmp/

// Initialize
val options = LlmInference.LlmInferenceOptions.builder()
    .setModelPath("/data/local/tmp/gemma-2b-it-cpu-int4.bin")
    .setMaxTokens(1024)
    .setTopK(40)
    .setTemperature(0.8f)
    .setRandomSeed(42)
    .build()

val llmInference = LlmInference.createFromOptions(context, options)

// Synchronous inference
val response = llmInference.generateResponse("Summarize this in one sentence: $article")

// Streaming inference (token-by-token output)
llmInference.generateResponseAsync("Write a reply to: $message") { partialResult, done ->
    // partialResult: each token as it's generated
    appendToUI(partialResult)
    if (done) showCompleteResponse()
}
```

**Model size impact:**
- Gemma 2B INT4: ~1.3GB disk, ~1.3GB RAM
- Gemma 7B INT4: ~4GB disk, ~4GB RAM (requires high-end device)
- Phi-2 (2.7B): ~1.5GB disk

---

## 8. Use Case: Auto-Suggestions & Smart Reply

### 8.1 Architecture for Real-Time Suggestions

```
User types/speaks
        │
        ▼
[Input Buffer] ← accumulate text/tokens
        │
        ▼ (debounce: wait 300ms after last keystroke)
[On-Device Inference]
  ├── ML Kit Smart Reply (conversation context)
  ├── Gemini Nano (via AICore) for richer suggestions
  └── TFLite custom model (domain-specific — e.g., coding suggestions)
        │
        ▼
[Ranked Suggestions] ← sort by confidence
        │
        ▼
[UI: Show top 3 chips]
```

```kotlin
// Debounced suggestion pipeline
class SuggestionEngine(
    private val context: Context,
    private val scope: CoroutineScope
) {
    private val inputFlow = MutableStateFlow("")
    val suggestions = inputFlow
        .debounce(300)                          // wait 300ms after last keystroke
        .filter { it.length > 3 }              // don't suggest for very short input
        .mapLatest { input ->                  // cancel previous inference if new input arrives
            withContext(Dispatchers.Default) {
                generateSuggestions(input)
            }
        }
        .stateIn(scope, SharingStarted.Lazily, emptyList())

    fun onInputChanged(text: String) { inputFlow.value = text }

    private suspend fun generateSuggestions(input: String): List<String> {
        return try {
            val suggestions = llmInference.generateResponse(
                "Complete this sentence with 3 short variations. Input: '$input'. Output JSON array:"
            )
            parseJsonSuggestions(suggestions)
        } catch (e: Exception) {
            emptyList()  // graceful degradation — no suggestions if inference fails
        }
    }
}
```

---

## 9. Use Case: Audio Hesitation Filtering

A common Alexa/voice assistant feature — strip "um", "uh", silent pauses, and repeated words from audio input before sending to ASR (Automatic Speech Recognition). Done on-device for privacy and low latency.

### 9.1 The Pipeline

```
Microphone (PCM audio stream)
        │
        ▼
[Voice Activity Detection (VAD)]
  → Detect speech frames vs silence/noise frames
  → Mark each 20-30ms frame: SPEECH or NON-SPEECH
        │
        ▼
[Hesitation Detection]
  → Identify filled pauses: "um", "uh", "er", "like" (ASR transcript labels)
  → Identify repeated words: "I I want" → "I want"
  → Identify long silences within speech (> 500ms)
        │
        ▼
[Audio Editing]
  → Remove or shorten silence frames
  → Remove marked hesitation tokens
  → Smooth splice points (crossfade 10ms to avoid clicks)
        │
        ▼
[Cleaned Audio] → send to ASR / STT API
```

### 9.2 Voice Activity Detection (VAD) on Android

**Option 1: WebRTC VAD (widely used, lightweight)**
WebRTC includes a VAD implementation used in billions of devices (Chrome, Android telephony). Available via JNI or the `webrtc-android` library.

```kotlin
// Using Android's built-in WebRTC VAD via JNI (or community wrapper)
// Available as: implementation("io.github.webrtc-sdk:android:104.5112.01")

class VoiceActivityDetector {
    // Frame size: 10ms, 20ms, or 30ms at 8/16/32 kHz
    private val SAMPLE_RATE = 16000
    private val FRAME_SIZE_MS = 20  // 20ms frames
    private val FRAME_SAMPLES = SAMPLE_RATE * FRAME_SIZE_MS / 1000  // 320 samples

    // WebRTC VAD modes: 0 (quality) → 3 (very aggressive)
    // Higher = more aggressive noise filtering but may clip speech
    private val VAD_MODE = 2

    fun isVoiceFrame(audioBuffer: ShortArray): Boolean {
        // Returns true if this 20ms frame contains speech
        return nativeVadProcess(audioBuffer, FRAME_SAMPLES, SAMPLE_RATE, VAD_MODE)
    }

    // Filter audio: only keep speech frames
    fun filterSilence(audioFrames: List<ShortArray>): List<ShortArray> {
        return audioFrames.filter { frame -> isVoiceFrame(frame) }
    }
}
```

**Option 2: TFLite VAD model**
For higher accuracy in noisy environments, use a lightweight neural network VAD model (< 1MB):

```kotlin
class NeuralVAD(context: Context) {
    private val interpreter: Interpreter

    init {
        val model = FileUtil.loadMappedFile(context, "vad_model.tflite")
        interpreter = Interpreter(model, Interpreter.Options().apply {
            numThreads = 2
        })
    }

    // Input: mel-spectrogram features from 30ms audio frame
    // Output: speech probability [0.0, 1.0]
    fun getSpeechProbability(audioFrame: FloatArray): Float {
        val melFeatures = extractMelFeatures(audioFrame)  // STFT → mel-filterbank
        val input = arrayOf(melFeatures)
        val output = Array(1) { FloatArray(1) }
        interpreter.run(input, output)
        return output[0][0]
    }

    fun isVoiceFrame(audioFrame: FloatArray, threshold: Float = 0.5f): Boolean {
        return getSpeechProbability(audioFrame) > threshold
    }
}
```

**Option 3: Android's built-in SoundTrigger / AudioRecord**
For wake-word detection and simple energy-based VAD built into Android.

### 9.3 Hesitation Detection

```kotlin
class HesitationFilter {
    // Hesitation words — can be per-language
    private val hesitationWords = setOf("um", "uh", "er", "hmm", "like", "you know", "basically")

    // Applied to ASR transcript words with timestamps
    data class Word(val text: String, val startMs: Long, val endMs: Long)

    fun filterHesitations(words: List<Word>): List<Word> {
        return words
            .filterNot { word ->
                word.text.lowercase().trim() in hesitationWords  // remove filler words
            }
            .let { removeRepetitions(it) }
    }

    // "I I want to" → "I want to"
    private fun removeRepetitions(words: List<Word>): List<Word> {
        val result = mutableListOf<Word>()
        var i = 0
        while (i < words.size) {
            if (i + 1 < words.size &&
                words[i].text.equals(words[i + 1].text, ignoreCase = true)) {
                // Skip duplicate, keep second occurrence (has later timestamp)
                i++
            } else {
                result.add(words[i])
                i++
            }
        }
        return result
    }
}
```

### 9.4 Silence Removal from Audio Buffer

```kotlin
class SilenceRemover(private val vad: NeuralVAD) {
    private val FRAME_SIZE_SAMPLES = 480  // 30ms at 16kHz
    private val MIN_SILENCE_MS = 300L     // don't remove silences < 300ms (natural pauses)
    private val PADDING_MS = 200L         // keep 200ms before/after speech (avoid clipping)

    fun removeExcessSilence(pcmAudio: ShortArray, sampleRate: Int): ShortArray {
        val frames = pcmAudio.toList().chunked(FRAME_SIZE_SAMPLES) { it.toShortArray() }

        // Mark each frame as speech or silence
        val frameLabels = frames.map { frame ->
            val floatFrame = FloatArray(frame.size) { frame[it] / 32768f }
            vad.isVoiceFrame(floatFrame)
        }

        // Apply padding — keep frames before/after speech
        val paddingFrames = (PADDING_MS / 30).toInt()
        val paddedLabels = frameLabels.toMutableList()
        for (i in frameLabels.indices) {
            if (frameLabels[i]) {
                // Extend keepAlive window around speech
                for (j in maxOf(0, i - paddingFrames)..minOf(frameLabels.size - 1, i + paddingFrames)) {
                    paddedLabels[j] = true
                }
            }
        }

        // Reconstruct audio keeping only speech frames
        return frames.zip(paddedLabels)
            .filter { (_, isSpeech) -> isSpeech }
            .flatMap { (frame, _) -> frame.toList() }
            .toShortArray()
    }
}
```

### 9.5 Full Pipeline Implementation

```kotlin
class AudioInputPipeline(
    private val context: Context,
    private val scope: CoroutineScope
) {
    private val vad = NeuralVAD(context)
    private val hesitationFilter = HesitationFilter()

    // Audio capture → VAD → send clean audio to ASR
    fun startRecording(): Flow<CleanAudioChunk> = channelFlow {
        val recorder = AudioRecord(
            MediaRecorder.AudioSource.VOICE_COMMUNICATION,  // noise suppression built-in
            SAMPLE_RATE,
            AudioFormat.CHANNEL_IN_MONO,
            AudioFormat.ENCODING_PCM_16BIT,
            BUFFER_SIZE
        )
        recorder.startRecording()

        val buffer = ShortArray(FRAME_SIZE_SAMPLES)
        while (isActive) {
            val read = recorder.read(buffer, 0, buffer.size)
            if (read > 0) {
                val floatBuffer = FloatArray(read) { buffer[it] / 32768f }
                val isSpeech = vad.isVoiceFrame(floatBuffer)
                if (isSpeech) {
                    send(CleanAudioChunk(buffer.copyOf(read)))
                }
                // Silence frames are silently dropped
            }
        }
        recorder.stop()
        recorder.release()
    }
}
```

---

## 10. Use Case: Predictive Text

### 10.1 Approaches

| Approach | Size | Quality | Offline |
|---|---|---|---|
| N-gram language model | < 1MB | Basic | ✅ |
| TFLite LSTM model | 10–50MB | Good | ✅ |
| Gemma 2B (full LLM) | ~1.3GB | Excellent | ✅ |
| Gemini Nano (AICore) | 0 (system) | Excellent | ✅ |
| Cloud API (Gemini API) | 0 | Best | ❌ |

**For a keyboard/messaging app:**
- Gboard uses a custom on-device model (privacy-critical — can't send keystrokes to server)
- Smart compose in Gmail uses a combination of on-device lightweight model + optional cloud for better suggestions

### 10.2 TFLite Sequence Model for Next Word Prediction

```kotlin
class PredictiveTextEngine(context: Context) {
    private val interpreter: Interpreter
    private val tokenizer: BertTokenizer  // or custom tokenizer

    init {
        val model = FileUtil.loadMappedFile(context, "next_word_predictor.tflite")
        interpreter = Interpreter(model, Interpreter.Options().apply {
            addDelegate(GpuDelegate())
        })
        tokenizer = BertTokenizer.fromFile(context, "vocab.txt")
    }

    // Given last N words, predict next word probabilities
    fun predictNextWords(context: String, topK: Int = 5): List<String> {
        val tokens = tokenizer.encode(context.takeLast(50))  // last 50 chars context
        val inputIds = Array(1) { IntArray(128) }  // padded to 128 tokens
        tokens.forEachIndexed { i, id -> if (i < 128) inputIds[0][i] = id }

        val outputLogits = Array(1) { FloatArray(VOCAB_SIZE) }
        interpreter.run(inputIds, outputLogits)

        // Apply softmax and return top-K
        val probabilities = softmax(outputLogits[0])
        return probabilities.indices
            .sortedByDescending { probabilities[it] }
            .take(topK)
            .map { tokenizer.decode(it) }
    }
}
```

---

## 11. Device Capability & Graceful Degradation

On-device ML has a hard requirement: the device must have enough RAM, storage, and compute.

```kotlin
class OnDeviceMLCapabilityChecker(private val context: Context) {

    // Check if device can run a specific model
    fun canRunLLM(requiredRamMb: Int = 2048): Boolean {
        val activityManager = context.getSystemService(ActivityManager::class.java)
        val memInfo = ActivityManager.MemoryInfo()
        activityManager.getMemoryInfo(memInfo)
        val availableRamMb = memInfo.availMem / (1024 * 1024)
        return availableRamMb >= requiredRamMb
    }

    fun canRunGeminiNano(): Boolean {
        // AICore availability check
        return try {
            val genAiStatus = GenerativeAIChecker.checkAvailability(context)
            genAiStatus == FeatureStatus.AVAILABLE
        } catch (e: Exception) {
            false
        }
    }

    // Feature decision
    fun getSmartReplyCapability(): SmartReplyCapability {
        return when {
            canRunGeminiNano() -> SmartReplyCapability.GEMINI_NANO     // best quality
            hasMLKitSmartReply() -> SmartReplyCapability.ML_KIT        // good quality
            else -> SmartReplyCapability.SIMPLE_TEMPLATE               // fallback
        }
    }
}

// Always have a fallback
sealed class SmartReplyCapability {
    object GeminiNano : SmartReplyCapability()
    object MlKit : SmartReplyCapability()
    object SimpleTemplate : SmartReplyCapability()  // "Sounds good", "Got it"
}
```

---

## 12. HLD — On-Device AI Feature Architecture

### "Design auto-suggestions for a messaging app without sending messages to the server"

```
1. Capability detection at app start
   - Check AICore/Gemini Nano availability
   - Check available RAM for TFLite model
   - Determine which tier of capability to offer

2. Model delivery
   - Tier 1 (Gemini Nano): zero download, system provides
   - Tier 2 (ML Kit Smart Reply): small model downloaded via Play services
   - Tier 3 (TFLite custom): download via Firebase ML on WiFi, store in app-specific dir
   - Tier 4 (fallback): hardcoded template replies

3. Inference pipeline (on Dispatchers.Default)
   - Conversation context → tokenization → model inference → post-process
   - Debounced: only run when user stops typing for 300ms
   - Cancelled: cancel in-flight inference when new message arrives (mapLatest)
   - Timeout: cancel if inference > 1 second (too slow for UX)

4. UI
   - Show up to 3 suggestion chips
   - Animate in with 150ms delay (don't distract while typing)
   - Hide if no suggestions in 1s

5. Privacy
   - No message content leaves the device
   - No analytics logged with message content
   - Model inference happens on Dispatchers.Default (off main thread)
```

---

## 13. React Native Equivalent

```typescript
// React Native AI / LLM options:

// 1. react-native-fast-tflite — run TFLite models in RN
import { loadTensorflowModel } from 'react-native-fast-tflite'
const model = await loadTensorflowModel(require('./assets/model.tflite'))
const output = await model.run([inputTensor])

// 2. Expo AI (experimental) — Gemini Nano on supported devices
// import { ExpoAI } from 'expo-ai'

// 3. Google AI Edge for JS/Web (via WebAssembly — not native performance)
// Best for React Native Web

// 4. Call native module wrapping Android LlmInference
// Write a native module that exposes LlmInference to JS

// For VAD in React Native:
// react-native-audio-record → capture audio
// Process with WebRTC VAD via native module
// Or: send to Whisper via server (not on-device)

// Whisper (OpenAI) running locally via llama.cpp:
// Available via community ports — heavy (39MB–1.5GB depending on model)
```

---

## 14. Common Misunderstandings & Pitfalls

**❌ Running model inference on the main thread**
LLM inference can take 500ms–5 seconds. Always run on `Dispatchers.Default`. Use `mapLatest` in Flow to cancel previous inference when new input arrives.

**❌ Not handling devices that don't support AICore/Gemini Nano**
As of 2025, Gemini Nano via AICore is only on select high-end devices. Always check `FeatureStatus` and provide a fallback (ML Kit, cloud API, or template responses).

**❌ Bundling large models in the APK**
A 1.3GB Gemma model in your APK adds 1.3GB to install size — instant uninstall for users on limited storage. Use Firebase ML, Dynamic Feature Modules, or background download on WiFi.

**❌ Not debouncing inference calls**
Every keystroke triggering a model inference call = continuous GPU usage = battery drain + janky UI. Debounce by 200–500ms and use `mapLatest` to cancel superseded calls.

**❌ Ignoring the RAM ceiling**
A Gemma 2B model (~1.3GB RAM) on a device with 2GB total RAM = OOM. Always check available memory before loading a model and gracefully fall back to a lighter model or cloud API.

**❌ VAD threshold too aggressive**
Setting VAD aggressiveness too high (mode 3 in WebRTC VAD) clips the beginnings and endings of words. Always add 200ms padding before and after detected speech segments.

**❌ Hardcoding hesitation words without localization**
Hesitation words vary by language and dialect. "Um/uh" in English, "euh" in French, "äh" in German. Load hesitation word lists from a localized resource, not hardcoded strings.

**❌ MediaPipe LLM Inference in production**
As of the MediaPipe documentation: "The MediaPipe LLM Inference API is intended for experimental and research use only." Use Gemini Nano via AICore (ML Kit GenAI APIs) for production.

---

## 15. Best Practices

- **Capability check first** — always check `FeatureStatus` / RAM before loading a model; never assume capability
- **Always have a fallback** — on-device → ML Kit → cloud API → template; no dead ends
- **Inference on `Dispatchers.Default`** — never block main thread; cancel with `mapLatest` for real-time input
- **Debounce 200–500ms** — don't run inference on every character; wait for user to pause
- **Dynamic model delivery** — Firebase ML or Dynamic Feature Modules; never bundle large models in APK
- **Quantized models only** — INT4/INT8 for mobile; FP32 won't fit
- **VAD before ASR** — filter silence first; reduces bytes sent to ASR by 50–80% and saves cost + latency
- **VAD padding** — always add 200ms before/after speech segments to avoid word clipping
- **WebRTC VAD for simplicity** — battle-tested, zero ML expertise needed; use TFLite VAD for noisy environments
- **Test on low-end devices** — AI features behave very differently on 2GB RAM vs 12GB RAM flagship
- **Stream output** — use `generateResponseAsync` for token-by-token streaming; don't wait for full response
- **Monitor inference latency** — track P95 inference time; set a 2s timeout and fall back to cloud if exceeded

---

## 16. Interview Q&A

**Q1: How would you implement smart reply suggestions without sending message content to a server?**

> The key requirement is on-device inference for privacy. I'd build a tiered system: First check if Gemini Nano is available via Android AICore (Android 14+, select devices) — it lives at the OS level so there's zero model download cost. If not available, use ML Kit's Smart Reply API which runs a smaller pre-downloaded model locally. Both are completely on-device. The inference pipeline: conversation context feeds into the model with a 300ms debounce after the last message, `mapLatest` cancels previous in-flight inference when new context arrives, and the output is ranked suggestions shown as chips. Privacy guarantee: nothing leaves the device — no message text in analytics, no cloud API call.

---

**Q2: What is quantization and why does it matter for mobile LLMs?**

> Quantization reduces the numerical precision of model weights — from 32-bit floats (FP32) to 8-bit integers (INT8) or 4-bit integers (INT4). A 3B parameter model at FP32 requires 12GB of RAM — more than any phone has. At INT4, the same model needs ~1.5GB, which fits on a modern Android device alongside the OS. The quality tradeoff is real but acceptable for many tasks: INT4 models lose some nuance versus FP32 but still perform well for summarization, smart reply, and proofreading. Gemini Nano uses INT4 quantization. For mobile, INT4 is the sweet spot — anything larger doesn't fit, anything higher precision doesn't run fast enough.

---

**Q3: How do you filter hesitations from audio input on-device?**

> Three stages. First, Voice Activity Detection: classify each 20–30ms audio frame as speech or silence using WebRTC VAD (lightweight, battle-tested) or a neural network TFLite model for noisy environments. Silence frames are discarded, but I keep 200ms of audio before and after speech segments as padding to avoid clipping word boundaries. Second, ASR runs on the clean audio. Third, post-process the transcript to remove filler words ("um", "uh", "like") using a language-specific list, and remove consecutive repeated words. The output is a cleaned transcript and optionally a cleaned audio file. All this happens on `Dispatchers.Default` in a streaming coroutine pipeline fed by `AudioRecord` reading PCM frames.

---

**Q4: When would you use AICore/Gemini Nano vs MediaPipe LLM Inference vs ML Kit?**

> Three different scenarios. AICore/Gemini Nano via ML Kit GenAI APIs: for production apps that need LLM-quality output (summarization, smart reply, proofreading) on supported devices. Zero model download cost, system-managed updates. The catch: only available on Android 14+ high-end devices, so need fallback. MediaPipe LLM Inference: for research, experimentation, or when you need to run a specific open-weight model (Gemma, Phi, Falcon) with full control over prompting and fine-tuning. Not production-ready per Google's own docs. ML Kit vision/text tasks: for well-defined tasks (OCR, barcode, pose detection, language ID) where you want a pre-trained ready-made model with no ML expertise required. Choose based on task specificity, device support requirements, and whether you need production reliability.

---

**Q5: How do you handle the case where a user's device doesn't support on-device AI?**

> Design with graceful degradation from the start, not as an afterthought. The architecture has tiers: check AICore → check ML Kit availability → check device RAM for TFLite → fall back to cloud API → fall back to template responses. Each fallback is slightly lower quality but always functional. The UI adapts: on Gemini Nano devices, show rich contextual suggestions; on unsupported devices, show simpler "Sounds good" / "Thanks" templates. I never show an error or disable the feature entirely — users shouldn't know which tier they're on. I also track which tier users fall into with analytics to understand the real-world distribution and decide when it's worth improving the fallback experience.

---

**Q6: What's the battery impact of running on-device LLM inference continuously?**

> High. A 2B parameter model running continuous inference consumes significant GPU/CPU power — comparable to playing a video. Mitigations: First, only run inference when necessary — debounce aggressively (300–500ms), cancel in-flight inference when context changes (`mapLatest`), skip inference when the keyboard is closed. Second, use the most efficient hardware path — NNAPI Delegate routes to the NPU on supported devices which is far more power-efficient than CPU. Third, set a token limit — don't generate a paragraph when 3 words are needed. Fourth, consider batching — collect multiple suggestion requests and run one inference pass. Fifth, monitor energy impact with Android Studio's Energy Profiler during dev and set a budget: if on-device inference uses > X% of battery/hour for an average session, fall back to lighter model or cloud.

---

*Previous: [10 — Security](./10-security.md)*
*Next: [11 — Feature Flags, A/B Testing & Remote Config](./11-feature-flags.md)*
