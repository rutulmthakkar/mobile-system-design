# 10 — Security

> **Round type: Both (HLD + LLD)**
> **HLD:** Designing a security architecture — trust model, data classification, threat model
> **LLD:** Keystore, cert pinning, biometrics, encryption at rest, obfuscation, root detection

---

## 1. Why This Matters in Interviews

Security questions reveal whether you think adversarially — not just "what does this code do" but "what happens when an attacker has the device, sniffs the network, or decompiles the APK." Interviewers at financial, healthcare, and consumer tech companies probe this hard.

**Common interview angles:**
- "Where do you store auth tokens on Android?"
- "How do you prevent a man-in-the-middle attack?"
- "What's certificate pinning and when would you use it?"
- "How do you protect an API key in an Android app?"
- "A user's device is rooted. What do you do?"
- "How do you implement biometric authentication?"
- "What data should never be stored locally?"

---

## 2. Threat Model — Think Like an Attacker

Before writing security code, define what you're protecting against:

```
Attack surfaces for a mobile app:

1. NETWORK     → MITM, traffic sniffing, API abuse
2. LOCAL       → rooted device, stolen device, malware reading local files
3. BINARY      → APK decompilation, reverse engineering, repackaging
4. RUNTIME     → Frida hooking, debugger attachment, memory inspection
5. BACKEND     → credential stuffing, replay attacks, token theft
```

**Data classification — not everything needs the same protection:**
```
CRITICAL:  auth tokens, private keys, biometric data, payment info
HIGH:      PII (email, phone), health data, location history
MEDIUM:    user preferences, cached content, analytics IDs
LOW:       public content, non-sensitive settings
```

Apply security controls proportional to data sensitivity. Over-engineering low-sensitivity data wastes effort. Under-protecting critical data creates liability.

---

## 3. Secure Token Storage

### 3.1 What NOT to Do

```kotlin
// ❌ Plain SharedPreferences — plaintext XML file, readable on rooted devices
prefs.edit().putString("auth_token", token).apply()

// ❌ Internal storage without encryption — still accessible on rooted devices
File(filesDir, "token.txt").writeText(token)

// ❌ Hardcoded in code
private const val API_KEY = "sk-abc123xyz"  // extractable via JADX in minutes

// ❌ BuildConfig
buildConfigField("String", "API_KEY", "\"${System.getenv("API_KEY")}\"")
// Still ends up in the APK, decompilable
```

### 3.2 EncryptedSharedPreferences — ⚠️ DEPRECATED (April 2025)

`EncryptedSharedPreferences` and `EncryptedFile` from `androidx.security:security-crypto` were **officially deprecated at version `1.1.0-alpha07` (April 2025)**. Do not use in new code.

**Why deprecated:**
- Inconsistent behavior across OEM devices (keyset corruption crashes)
- Fragile AES-GCM support on certain manufacturers
- No clean path to upgrade encryption schemes without data loss
- Synchronous I/O on the calling thread — StrictMode violations
- Tightly coupled to Android Keystore in ways that caused device-specific bugs

### 3.3 Google Tink ✅ Modern replacement

[Tink](https://developers.google.com/tink) is Google's production cryptography library — used internally at Google and now the recommended approach for Android encryption. It decouples encryption (Tink) from storage (DataStore/SharedPreferences) and adds key rotation support.

```kotlin
// gradle
implementation("com.google.crypto.tink:tink-android:1.14.1")
// For DataStore integration (official artifact, added 2025)
implementation("androidx.datastore:datastore-tink:1.1.0-alpha10")
```

**Option A: Tink + DataStore (recommended for new code)**

```kotlin
// Step 1: Create a Tink keyset handle backed by Android Keystore
AeadConfig.register()

val keysetHandle = AndroidKeysetManager.Builder()
    .withKeyTemplate(KeyTemplates.get("AES256_GCM"))
    .withSharedPref(
        context,
        "my_keyset",                          // keyset name in prefs
        "my_keyset_prefs"                     // prefs file holding keyset
    )
    .withMasterKeyUri("android-keystore://tink_master_key")  // Keystore-backed
    .build()
    .keysetHandle

// Step 2: Get the AEAD primitive
val aead = keysetHandle.getPrimitive(RegistryConfiguration.get(), Aead::class.java)

// Step 3: Use AeadSerializer to encrypt DataStore data transparently
val aeadSerializer = AeadSerializer(
    aead = aead,
    wrappedSerializer = MySettingsSerializer,     // your existing DataStore serializer
    associatedData = "my_settings.pb".encodeToByteArray()  // binds ciphertext to file
)

// Step 4: Create encrypted DataStore
val encryptedDataStore = DataStoreFactory.create(
    serializer = aeadSerializer,
    produceFile = { context.dataStoreFile("my_settings.pb") }
)

// Usage — same as regular DataStore
encryptedDataStore.updateData { current -> current.toBuilder().setToken(token).build() }
val token = encryptedDataStore.data.first().token
```

**Option B: Tink directly for arbitrary data**

```kotlin
// Encrypt
val encryptedBytes = aead.encrypt(
    "my_secret_token".toByteArray(),   // plaintext
    "auth_context".toByteArray()        // associated data (prevents ciphertext swapping)
)
val stored = Base64.encodeToString(encryptedBytes, Base64.DEFAULT)

// Store in plain DataStore (already encrypted) or plain SharedPreferences
prefs.edit().putString("token", stored).apply()

// Decrypt
val decoded = Base64.decode(prefs.getString("token", null)!!, Base64.DEFAULT)
val plaintext = String(aead.decrypt(decoded, "auth_context".toByteArray()))
```

**How Tink + Keystore works:**
- `AndroidKeysetManager` generates a Tink keyset (collection of cryptographic keys) stored in SharedPreferences
- The keyset itself is encrypted with a master key stored in the **Android Keystore** (hardware-backed)
- Tink's `Aead.encrypt()` / `decrypt()` run in-process using the keyset — much faster than routing every operation through the Keystore directly
- **Key rotation** is built in — add a new key to the keyset, rotate, old key still decrypts old ciphertext

**Why Tink over the old approach:**
| | `EncryptedSharedPreferences` | Tink + DataStore |
|---|---|---|
| Status | ⚠️ **Deprecated** April 2025 | ✅ Active, Google-maintained |
| OEM compatibility | Brittle on some devices | Stable across devices |
| Key rotation | ❌ Not supported | ✅ Built-in |
| Async I/O | ❌ Synchronous (StrictMode) | ✅ Coroutine-based DataStore |
| Encryption strength | AES-256-GCM | AES-256-GCM (same) |
| Flexibility | SharedPreferences only | Any storage backend |
| Upgrade path | Data loss on scheme change | Smooth key rotation |

### 3.4 Migration: EncryptedSharedPreferences → Tink + DataStore

If you have existing users with data in `EncryptedSharedPreferences`:

```kotlin
// One-time migration on app upgrade
suspend fun migrateFromEncryptedSharedPreferences(context: Context) {
    val migrationDone = plainPrefs.getBoolean("tink_migration_done", false)
    if (migrationDone) return

    // Read from old EncryptedSharedPreferences (still works even if deprecated)
    val oldPrefs = EncryptedSharedPreferences.create(
        context, "secure_prefs", legacyMasterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
    val token = oldPrefs.getString("auth_token", null)

    // Write to new Tink-encrypted DataStore
    if (token != null) {
        encryptedDataStore.updateData { it.toBuilder().setAuthToken(token).build() }
        oldPrefs.edit().remove("auth_token").apply()  // clean up
    }

    plainPrefs.edit().putBoolean("tink_migration_done", true).apply()
}
```

### 3.5 Android Keystore Directly — For Generated Keys — For Generated Keys

```kotlin
// Generate an AES key in the Keystore
val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
keyGenerator.init(
    KeyGenParameterSpec.Builder(
        "my_key_alias",
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
    )
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setUserAuthenticationRequired(true)          // requires biometric/PIN for each use
    .setUserAuthenticationValidityDurationSeconds(30)  // valid for 30s after auth
    .setInvalidatedByBiometricEnrollment(true)    // invalidated if new biometric enrolled
    .build()
)
val secretKey = keyGenerator.generateKey()

// Encrypt
fun encrypt(data: ByteArray): Pair<ByteArray, ByteArray> {
    val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    val secretKey = keyStore.getKey("my_key_alias", null) as SecretKey
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.ENCRYPT_MODE, secretKey)
    return Pair(cipher.doFinal(data), cipher.iv)  // return encrypted data + IV
}

// Decrypt
fun decrypt(encryptedData: ByteArray, iv: ByteArray): ByteArray {
    val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    val secretKey = keyStore.getKey("my_key_alias", null) as SecretKey
    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, secretKey, GCMParameterSpec(128, iv))
    return cipher.doFinal(encryptedData)
}
```

### 3.6 Storage Decision by Data Type

| Data | Storage | Why |
|---|---|---|
| Auth tokens | Tink + DataStore (`AeadSerializer`) | Modern, non-deprecated, key rotation |
| Refresh tokens | Tink + DataStore with `setUserAuthenticationRequired(true)` on master key | Extra protection for long-lived tokens |
| Private keys | `AndroidKeyStore` directly | Hardware-backed, never extracted |
| Sensitive DB (health, finance) | `SQLCipher` | Full DB encryption |
| Sensitive files | `EncryptedFile` (Jetpack Security) | AES-256-GCM per file |
| Non-sensitive settings | `DataStore` | No encryption needed |
| Secrets that must be on server | **Server-side only** | Never in APK |

---

## 4. Certificate Pinning

### 4.1 What It Is and Why

Standard TLS trusts any certificate signed by a trusted CA in the device's trust store. If an attacker gets a fraudulent cert from a compromised CA (or installs a root CA on the device), they can intercept your HTTPS traffic — MITM attack.

Certificate pinning tells the app: "only trust THIS specific certificate or public key, regardless of what the device's trust store says."

```
Without pinning:
App → TLS → Attacker's proxy (fraudulent cert from any CA) → Your server
         ↑ attacker reads/modifies traffic

With pinning:
App → TLS → Attacker's proxy → App REJECTS (cert doesn't match pin) ✅
```

### 4.2 OkHttp CertificatePinner ✅ Recommended approach

```kotlin
// Step 1: Get the pin hash for your domain
// Run: openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout | 
//      openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64

val certificatePinner = CertificatePinner.Builder()
    .add("api.example.com", "sha256/primary_pin_base64==")       // current cert
    .add("api.example.com", "sha256/backup_pin_base64==")        // backup cert (ALWAYS include)
    .add("api.example.com", "sha256/root_ca_pin_base64==")       // root CA pin (most stable)
    .build()

val okHttpClient = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()
// OkHttp throws SSLPeerUnverifiedException if pin doesn't match
```

**Always include a backup pin.** If your primary cert expires and you haven't included a backup pin, you must ship an emergency app update to restore connectivity. Backup pins are typically the CA's public key (more stable across certificate rotations).

### 4.3 Network Security Configuration (Android 7.0+ / API 24+)

XML-based pinning without code — easier to update via OTA config:

```xml
<!-- res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Block all cleartext (HTTP) traffic — enforce HTTPS app-wide -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system"/>
        </trust-anchors>
    </base-config>

    <!-- Pin for specific domain -->
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-01-01">  <!-- expiration forces pin rotation review -->
            <pin digest="SHA-256">primary_pin_base64==</pin>
            <pin digest="SHA-256">backup_pin_base64==</pin>
        </pin-set>
    </domain-config>

    <!-- Debug only: trust user-added CAs (never in release) -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="user"/>  <!-- allows Burp Suite / Charles Proxy -->
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

```xml
<!-- AndroidManifest.xml -->
<application
    android:networkSecurityConfig="@xml/network_security_config">
```

### 4.4 Leaf vs CA vs SPKI Pinning

| Type | Pin to | Pros | Cons |
|---|---|---|---|
| **Leaf cert** | Specific server certificate | Most precise | Breaks when cert renews (typically 1 year) |
| **Intermediate CA** | Intermediate CA cert | Survives cert rotation | Breaks if CA changes |
| **Root CA** | Root CA public key | Most stable | Weakest protection |
| **SPKI** ✅ | Server's public key (subject public key info) | Survives cert renewal (same key) | Still breaks if key rotated |

**Best practice: pin the SPKI (public key).** You can renew the certificate with the same key pair — pin remains valid. OkHttp's `sha256/` prefix pins the SPKI.

### 4.5 When NOT to Pin

Certificate pinning is not free — it creates operational risk:
- Certificate rotation becomes a crisis if not planned
- MITM testing in QA requires bypassing pins
- Third-party CDNs/APIs may rotate certs without notice

**Pin when:** You handle financial data, health records, or authentication — high-sensitivity apps where MITM is a real threat model.

**Don't pin when:** Low-sensitivity app, using third-party APIs you don't control (their cert rotation breaks your app), small team without cert rotation runbook.

---

## 5. Biometric Authentication

### 5.1 BiometricPrompt (Modern API)

```kotlin
// gradle
implementation("androidx.biometric:biometric:1.2.0-alpha05")

class BiometricHelper(private val activity: FragmentActivity) {

    fun authenticate(
        onSuccess: (BiometricPrompt.AuthenticationResult) -> Unit,
        onError: (Int, CharSequence) -> Unit,
        onFailed: () -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)

        val biometricPrompt = BiometricPrompt(activity, executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    onSuccess(result)
                }
                override fun onAuthenticationError(code: Int, message: CharSequence) {
                    onError(code, message)
                }
                override fun onAuthenticationFailed() {
                    onFailed()  // valid biometric, but not enrolled one (e.g., wrong finger)
                }
            })

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Verify your identity")
            .setSubtitle("Use biometric to unlock")
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG or  // fingerprint, face, iris
                BiometricManager.Authenticators.DEVICE_CREDENTIAL     // PIN/password fallback
            )
            .build()

        biometricPrompt.authenticate(promptInfo)
    }
}
```

### 5.2 Biometric + Keystore (Crypto-bound authentication)

The most secure pattern: biometric authentication **unlocks a Keystore key** which then decrypts data. The key is unusable without biometric success.

```kotlin
// Create a key that requires biometric per use
val keyGenSpec = KeyGenParameterSpec.Builder("biometric_key",
    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
    .setUserAuthenticationRequired(true)               // requires biometric
    .setUserAuthenticationValidityDurationSeconds(-1)  // -1 = require auth for every use
    .setInvalidatedByBiometricEnrollment(true)         // invalidate if new biometric added
    .build()

// Authenticate WITH a cipher — the cipher is unlocked only on biometric success
fun authenticateWithCipher(onCipherReady: (Cipher) -> Unit) {
    val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    val secretKey = keyStore.getKey("biometric_key", null) as SecretKey

    val cipher = Cipher.getInstance("AES/GCM/NoPadding")
    cipher.init(Cipher.DECRYPT_MODE, secretKey, GCMParameterSpec(128, storedIv))

    val cryptoObject = BiometricPrompt.CryptoObject(cipher)

    // Authenticate — cipher is only usable after successful biometric
    biometricPrompt.authenticate(promptInfo, cryptoObject)
    // In onAuthenticationSucceeded: result.cryptoObject?.cipher!!.doFinal(encryptedData)
}
```

**Why this matters:** Biometric authentication without `CryptoObject` can be bypassed on rooted devices (hooks override the result). With `CryptoObject`, the Keystore key is cryptographically locked to biometric success in hardware — no software bypass possible.

### 5.3 Check Biometric Availability

```kotlin
fun checkBiometricSupport(context: Context): BiometricStatus {
    val biometricManager = BiometricManager.from(context)
    return when (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG)) {
        BiometricManager.BIOMETRIC_SUCCESS          -> BiometricStatus.AVAILABLE
        BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> BiometricStatus.NO_HARDWARE
        BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> BiometricStatus.TEMPORARILY_UNAVAILABLE
        BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED  -> BiometricStatus.NOT_ENROLLED
        else -> BiometricStatus.UNAVAILABLE
    }
}
```

---

## 6. Encryption at Rest

### 6.1 EncryptedFile

```kotlin
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

// Write
val encryptedFile = EncryptedFile.Builder(
    context,
    File(context.filesDir, "sensitive_data.bin"),
    masterKey,
    EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
).build()

encryptedFile.openFileOutput().use { output ->
    output.write(sensitiveData.toByteArray())
}

// Read
encryptedFile.openFileInput().use { input ->
    val data = input.readBytes()
}
```

### 6.2 SQLCipher — Encrypted Database

```kotlin
// gradle
implementation("net.zetetic:android-database-sqlcipher:4.5.4")
implementation("androidx.sqlite:sqlite:2.3.1")

// Room integration
val db = Room.databaseBuilder(context, AppDatabase::class.java, "encrypted.db")
    .openHelperFactory(SupportFactory(SQLiteDatabase.getBytes("passphrase".toCharArray())))
    .build()
```

**Passphrase management:** Don't hardcode the passphrase. Generate it at first run and store in `EncryptedSharedPreferences` or derive it from user password + salt via PBKDF2.

### 6.3 What to Encrypt vs What to Leave

| | Encrypt locally | Why |
|---|---|---|
| Auth tokens, refresh tokens | ✅ Always | High-value target for account takeover |
| PII (name, email, phone) | ✅ For healthcare/finance | GDPR/HIPAA compliance |
| Health/financial records | ✅ Always | Regulatory requirement |
| Cached feed posts | ❌ Usually not | Low sensitivity, encryption overhead |
| User preferences | ❌ Usually not | Not sensitive |
| Images/media | 🟡 Context-dependent | Depends on content |

---

## 7. Code Protection

### 7.1 R8 / ProGuard Obfuscation

```kotlin
// app/build.gradle.kts
android {
    buildTypes {
        release {
            isMinifyEnabled = true      // enables R8 shrinking + obfuscation
            isShrinkResources = true    // removes unused resources
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

R8 renames classes from `UserRepository` → `a`, methods from `fetchUser()` → `b`. This makes decompiled APK harder to read.

**Limitations:**
- Logic flow remains intact — experienced reversers still understand what code does
- Reflection-based code can't be fully obfuscated
- String constants (API endpoints, error messages) are NOT encrypted by default
- R8 obfuscation is easily reversed if the attacker has the mapping file

**For stronger protection:** DexGuard (commercial, by Guardsquare) adds string encryption, class encryption, control flow obfuscation, and anti-tampering on top of R8.

### 7.2 Prevent Debug Builds in Production

```kotlin
// Check if debuggable (debug builds bypass many protections)
if (0 != applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE) {
    // Running in debug mode
    if (!BuildConfig.DEBUG) {
        // Debuggable in release build = repackaged/tampered
        // Log, report, or restrict functionality
    }
}

// Check app signature at runtime (detect repackaging)
val signature = packageManager.getPackageInfo(
    packageName, PackageManager.GET_SIGNING_CERTIFICATES
).signingInfo.apkContentsSigners[0]

val expectedSignatureHash = "sha256_of_your_release_signature"
val actualHash = MessageDigest.getInstance("SHA-256")
    .digest(signature.toByteArray())
    .let { Base64.encodeToString(it, Base64.DEFAULT) }

if (actualHash != expectedSignatureHash) {
    // App has been repackaged with a different signing key
    // Attacker likely removed certificate pinning or added malicious code
}
```

---

## 8. Root & Integrity Detection

### 8.1 Play Integrity API (Replaces SafetyNet, deprecated June 2024)

```kotlin
// gradle
implementation("com.google.android.play:integrity:1.3.0")

// Request an integrity token
val integrityManager = IntegrityManagerFactory.create(context)

integrityManager.requestIntegrityToken(
    IntegrityTokenRequest.builder()
        .setNonce(generateNonce())  // server-generated nonce, prevents replay
        .build()
).addOnSuccessListener { response ->
    val token = response.token()
    // Send token to YOUR server for verification
    // Server calls Play Integrity API to decode and verify
}

// Server receives three verdicts:
// MEETS_BASIC_INTEGRITY     - basic checks pass (may be custom ROM)
// MEETS_DEVICE_INTEGRITY    - certified Android, locked bootloader (standard requirement)
// MEETS_STRONG_INTEGRITY    - hardware-backed attestation (highest assurance)
```

**Important:** Never verify the integrity token on the device — send it to your server. Client-side verification is easily bypassed.

### 8.2 Manual Root Detection (Supplementary)

```kotlin
fun isDeviceRooted(): Boolean {
    // Check for common su binary locations
    val suPaths = listOf(
        "/system/bin/su", "/system/xbin/su", "/sbin/su",
        "/system/su", "/system/app/Superuser.apk"
    )
    if (suPaths.any { File(it).exists() }) return true

    // Try to execute su
    return try {
        Runtime.getRuntime().exec(arrayOf("/system/xbin/su", "-c", "id"))
        true
    } catch (e: IOException) {
        false
    }
}
```

**Limitations:** Root detection is an arms race — Magisk Hide, Shamiko, and similar tools bypass most checks. Treat root detection as a speed bump, not a guarantee. Use Play Integrity API for serious integrity requirements.

**Response to rooted devices:**
```kotlin
when {
    isRooted && appHandlesSensitiveData -> {
        // Show warning, restrict sensitive features (hide payment info, logout)
        showRootedDeviceWarning()
        disableSensitiveFeatures()
    }
    isRooted && !appHandlesSensitiveData -> {
        // Log for analytics, continue normally
        analytics.trackEvent("rooted_device_detected")
    }
}
// Never crash or force-close — it looks malicious to the user
```

---

## 9. Network Security

### 9.1 Prevent Cleartext Traffic

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false"/>
    <!-- Android 9+ already defaults to false, but explicit is better -->
</network-security-config>
```

### 9.2 TLS Configuration

```kotlin
// Minimum TLS 1.2 (OkHttp default since v3.13)
// Prefer TLS 1.3 for new connections
val tlsSpecs = listOf(
    ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)
        .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2)
        .cipherSuites(
            CipherSuite.TLS_AES_128_GCM_SHA256,       // TLS 1.3
            CipherSuite.TLS_AES_256_GCM_SHA384,       // TLS 1.3
            CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384  // TLS 1.2 fallback
        )
        .build()
)

val client = OkHttpClient.Builder()
    .connectionSpecs(tlsSpecs)
    .build()
```

### 9.3 API Key Protection

**Never put API keys in the APK.** Any key in `BuildConfig`, `strings.xml`, or source code is extractable via JADX in minutes.

```kotlin
// ❌ Any of these are extractable
val apiKey = "sk-abc123"
val apiKey = BuildConfig.MAPS_API_KEY
val apiKey = context.getString(R.string.api_key)

// ✅ For server-called APIs: keep key on server, app calls your backend
// App → Your server (authenticated) → Third-party API (with key)

// ✅ For client-side APIs (Maps, etc.): use app restrictions
// In Google Cloud Console: restrict Maps API key to your app's package name + SHA-256 cert
// Even if extracted, the key only works from your signed app

// ✅ For dynamic keys: fetch at runtime from your authenticated backend
val mapsTileUrl = authApi.getSignedMapsTileUrl()  // server signs the URL
```

---

## 10. Sensitive Data Lifecycle

```kotlin
// Clear sensitive data from memory after use
var password: CharArray? = null
try {
    password = getPasswordFromInput()
    authenticateUser(password)
} finally {
    password?.fill('\u0000')  // overwrite with zeros before GC
    password = null
}

// Clear on logout
fun logout() {
    encryptedPrefs.edit()
        .remove("auth_token")
        .remove("refresh_token")
        .apply()
    dao.clearUserData()      // clear sensitive cached data
    fileCache.clearSensitiveFiles()
}

// Prevent backup of sensitive files
// AndroidManifest.xml
<application android:allowBackup="false"  <!-- or use fullBackupContent to exclude -->
             android:dataExtractionRules="@xml/backup_rules">
```

```xml
<!-- res/xml/backup_rules.xml — exclude sensitive files from backup -->
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="sharedpref" path="secure_prefs.xml"/>
        <exclude domain="database" path="encrypted.db"/>
    </cloud-backup>
</data-extraction-rules>
```

---

## 11. React Native Security

```typescript
// ❌ AsyncStorage — plaintext, readable on rooted devices
await AsyncStorage.setItem('token', authToken)

// ✅ react-native-keychain — uses iOS Keychain + Android Keystore
import * as Keychain from 'react-native-keychain'

await Keychain.setGenericPassword('username', authToken, {
    accessible: Keychain.ACCESSIBLE.WHEN_UNLOCKED,
    accessControl: Keychain.ACCESS_CONTROL.BIOMETRY_ANY  // optional: require biometric
})

const credentials = await Keychain.getGenericPassword()
const token = credentials ? credentials.password : null

// ✅ expo-secure-store (for Expo projects)
import * as SecureStore from 'expo-secure-store'
await SecureStore.setItemAsync('token', authToken)
const token = await SecureStore.getItemAsync('token')

// Certificate pinning in RN
// Use react-native-ssl-pinning or OkHttp custom build via native module
import { fetch } from 'react-native-ssl-pinning'
const response = await fetch('https://api.example.com/data', {
    method: 'GET',
    sslPinning: {
        certs: ['sha256/primary_pin==', 'sha256/backup_pin==']
    }
})

// API key protection — same rule as Android
// Never in JS source — extractable from bundle with `npx react-native bundle`
// Use react-native-dotenv carefully — still ends up in bundle
// Better: fetch from your authenticated server
```

---

## 12. HLD — Security Architecture

### "Design the security architecture for a mobile banking app"

```
1. Authentication layer
   - OAuth 2.0 + PKCE via system browser (no in-app WebView for auth)
   - Short-lived access tokens (15 min), long-lived refresh tokens
   - Refresh tokens stored in EncryptedSharedPreferences
   - Biometric unlock (crypto-bound to Keystore) for session resume

2. Network layer
   - TLS 1.3 mandatory
   - Certificate pinning (SPKI + backup pin) for all endpoints
   - No cleartext traffic (network security config)
   - Request signing with HMAC for sensitive operations

3. Local storage
   - Auth tokens → EncryptedSharedPreferences
   - Account data → SQLCipher encrypted database
   - No PII in logs, analytics, or crash reports

4. Runtime protection
   - Play Integrity API check on launch (server-side verification)
   - Root detection → restrict sensitive features, show warning
   - Debug build detection → refuse to run in release mode if debuggable
   - Screenshot prevention for sensitive screens

5. Code hardening
   - R8 minification + obfuscation
   - DexGuard for string encryption (commercial option)
   - App signature verification at runtime

6. Data lifecycle
   - Wipe tokens on logout
   - Wipe on 5 failed PIN attempts
   - Auto-logout on 5 min inactivity
   - Exclude from Android backup
```

---

## 13. Common Misunderstandings & Pitfalls

**❌ Biometric without CryptoObject is bypassable on rooted devices**
A Frida hook can override `onAuthenticationSucceeded` and return success without actual biometric. When you pass a `CryptoObject` (backed by Keystore), the key only becomes usable after real hardware-level biometric verification — not software bypassable.

**❌ Certificate pinning without a backup pin**
The most common cert pinning mistake. When you rotate your certificate and didn't include a backup pin, users on the old app version can't connect — the only fix is a force-update. Always include at least one backup pin (usually the CA key).

**❌ Putting API keys in `BuildConfig` or `strings.xml`**
Both end up in the compiled APK and are extractable with JADX in under a minute. Keys that must exist on the device should be fetched from your authenticated backend or restricted to your app's package + signing cert in the API provider's console.

**❌ Still using `EncryptedSharedPreferences` in new code**
`EncryptedSharedPreferences` was deprecated in April 2025. It has known OEM compatibility issues causing "keyset corruption" exceptions in Crashlytics, no key rotation support, and synchronous I/O that violates StrictMode. New code should use Tink + DataStore. Existing apps should migrate by reading old ESP values and writing them to the new Tink-encrypted DataStore in a one-time migration on app upgrade.

**❌ Storing tokens in plain `SharedPreferences` "because we use HTTPS"**
HTTPS protects data in transit. Tokens in SharedPreferences are at risk if the device is rooted, lost, or targeted by malware. Defense in depth: HTTPS + encrypted local storage + short token lifetime.

**❌ Logging tokens or PII in debug builds**
Debug logs often end up in crash reports if `Timber.plant(DebugTree())` is called unconditionally. Always gate debug logging to `BuildConfig.DEBUG` and use a production-safe Timber tree that strips sensitive values.

**❌ Not clearing sensitive data on logout**
After logout, auth tokens must be deleted from `EncryptedSharedPreferences`, sensitive DB tables cleared, and file cache cleaned. Users who lend a device expect the next person can't access their account.

**❌ `android:allowBackup="true"` (default)**
Android backs up app data including SharedPreferences to Google Drive by default. Tokens in unencrypted SharedPreferences get backed up. Disable backup or use `dataExtractionRules` to exclude sensitive files.

---

## 14. Best Practices

- **`EncryptedSharedPreferences` for tokens** — never plain SharedPreferences
- **Biometric + `CryptoObject`** — cryptographically binds authentication to Keystore
- **Certificate pinning with backup pin** — SPKI pinning, always a fallback, have a rotation runbook
- **Play Integrity API for device integrity** — server-side verification, replaces deprecated SafetyNet
- **No API keys in APK** — server-side for server APIs; app restrictions in provider console for client APIs
- **`setInvalidatedByBiometricEnrollment(true)`** — invalidate Keystore keys when new biometric enrolled (prevents attacker adding their biometric)
- **Short-lived access tokens** — 15 min expiry limits damage if stolen
- **TLS 1.3 + no cleartext** — `cleartextTrafficPermitted="false"` in network security config
- **`allowBackup="false"` or explicit exclusion rules** — prevent sensitive data in cloud backups
- **Screenshot prevention for sensitive screens** — `window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)`
- **Clear sensitive data on logout** — tokens, cache, DB rows
- **Fail securely** — if integrity check fails, restrict features, don't crash; crashes look malicious
- **Never validate Play Integrity token on device** — always send to server for verification

---

## 15. Interview Q&A

**Q1: Where do you store auth tokens on Android?**

> `EncryptedSharedPreferences` was the previous answer but it was **deprecated in April 2025** due to OEM compatibility issues, keyset corruption crashes, and no key rotation support. The modern answer is **Google Tink + Jetpack DataStore**. Tink's `AndroidKeysetManager` generates an AES-256-GCM keyset stored in SharedPreferences, encrypted by a master key in the Android Keystore — hardware-backed on supported devices. Tink's `AeadSerializer` wraps a DataStore serializer so all reads/writes are transparently encrypted. This approach is more reliable across OEMs, supports key rotation, and is coroutine-safe (no StrictMode violations). For highest sensitivity like a banking app, I'd additionally set `setUserAuthenticationRequired(true)` on the Keystore master key so it requires biometric or PIN each time it unlocks the Tink keyset. What I never do: plain SharedPreferences (plaintext XML), plain DataStore without encryption, or anything backed by a hardcoded key.

---

**Q2: What is certificate pinning and when would you use it?**

> Certificate pinning tells the app to only trust a specific certificate or public key rather than any cert signed by a trusted CA in the device's trust store. Without pinning, an attacker who can install a root CA (corporate MDM, rooted device) can perform MITM on HTTPS traffic. I'd use it for apps handling financial data, health records, or authentication — high-sensitivity scenarios where MITM is a real threat model. I always pin the SPKI (public key hash) rather than the leaf certificate, because you can renew a certificate with the same key pair and the pin stays valid. I always include a backup pin. For lower-sensitivity apps, proper TLS with no cleartext is sufficient — pinning adds operational risk and isn't always worth the complexity.

---

**Q3: An attacker decompiles your APK with JADX. What can they see and what can they not see?**

> With R8 obfuscation enabled: they see obfuscated class and method names (a.b.c() instead of UserRepository.fetchUser()), but the logic flow is still visible — an experienced reverse engineer can understand the code. String constants — including API endpoint URLs — are visible unless you use additional string encryption (DexGuard). What they can't see: Keystore private keys (never leave hardware), encrypted data in EncryptedSharedPreferences, or auth tokens that are fetched at runtime and never embedded. The defense strategy is: don't put secrets in the APK, use R8 as a deterrent not a security control, fetch sensitive configuration from your authenticated backend, and use the Keystore for any keys that must exist on device.

---

**Q4: How do you implement biometric authentication securely?**

> Two approaches. The basic approach: `BiometricPrompt` with a `PromptInfo` — shows the system biometric dialog and calls `onAuthenticationSucceeded`. This is fine for UX but bypassable on rooted devices with Frida. The secure approach: pass a `CryptoObject` containing a Keystore-backed `Cipher`. The cipher is only usable after real hardware-level biometric verification — the Keystore enforces this in the TEE, not software. In `onAuthenticationSucceeded`, I get the `result.cryptoObject?.cipher` and use it to decrypt the auth token. An attacker who hooks `onAuthenticationSucceeded` still can't decrypt the token without the hardware-backed cipher. I always set `setInvalidatedByBiometricEnrollment(true)` so keys are invalidated if someone adds their biometric to the device.

---

**Q5: How do you protect an API key that must be used in an Android app?**

> The honest answer is: if a key must be on the device, you can't fully protect it — you can only make extraction harder. For keys that call server-to-server APIs (payment processor, SMS provider), the key should never be in the app — the app calls your backend which calls the API. For client-side APIs (Maps, analytics), use the API provider's app restrictions: restrict the key to your package name + SHA-256 signing certificate fingerprint. Even if someone extracts the key, it only works from your signed app binary. For any key that must be fetched dynamically, fetch it from your authenticated backend at runtime, store it in `EncryptedSharedPreferences`, and give it a short TTL so it expires. Never embed in `BuildConfig`, `strings.xml`, or `.env` files — all end up in the APK.

---

**Q6: A user's device is rooted. How does your app respond?**

> The response depends on what the app handles. First, I detect root using Play Integrity API (server-side verdict) as the primary signal and manual checks (su binary, Magisk presence) as supplementary signals. For a banking or healthcare app: show a clear warning explaining that sensitive features are disabled on rooted devices, and restrict access to account data, payments, and sensitive screens. Don't crash or force-close — that's a bad user experience and users who legitimately root their devices for development shouldn't be blacklisted without recourse. Log the rooted device event to the observability stack for security analytics. For a lower-sensitivity app: log it and continue normally — most root detection is easily bypassed anyway and the cost of false positives (legitimate power users) isn't worth it.

---

*Previous: [09 — Observability](./09-observability.md)*
*Next: [11 — Feature Flags, A/B Testing & Remote Config](./11-feature-flags.md)*
