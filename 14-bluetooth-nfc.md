# 14 — Bluetooth & NFC

> **Round type: LLD (primary)**
> **Focus:** Implementation patterns for BLE scanning, GATT operations, NFC tag reading/writing, and Host Card Emulation — the areas interviewers probe when you claim Bluetooth/NFC experience

---

## 1. Why This Matters in Interviews

Bluetooth and NFC are niche but appear frequently at Amazon (Alexa/Echo), Google (Nest, Android), Samsung, fitness/health companies, and fintech. Interviewers expect you to know: permissions model, GATT architecture, connection lifecycle, and common failure modes.

**Common interview angles:**
- "Walk me through how BLE scanning works"
- "What is GATT and how does it structure data?"
- "How do you handle BLE connection drops?"
- "What's the difference between Notify and Indicate in GATT?"
- "How does NFC tag reading work in Android?"
- "What is Host Card Emulation and how does it differ from reading a real NFC card?"
- "How do BLE permissions differ across Android versions?"

---

## 2. Bluetooth Overview

### 2.1 Bluetooth Classic vs BLE

| | Bluetooth Classic (BR/EDR) | Bluetooth Low Energy (BLE) |
|---|---|---|
| Range | ~10m | ~10–100m (depends on power) |
| Bandwidth | High (up to 3 Mbps) | Low (up to 2 Mbps, typically much less) |
| Power | High drain | Very low (designed for years on a coin cell) |
| Connection | Continuous | Connect-transfer-disconnect |
| Use cases | Audio (A2DP), file transfer, HID | Sensors, wearables, beacons, medical |
| Android APIs | `BluetoothDevice`, `BluetoothSocket` | `BluetoothLeScanner`, `BluetoothGatt` |

For modern development, BLE is far more common. Classic Bluetooth is mainly used for audio (headphones, speakers) via profiles like A2DP/HFP — typically handled through system APIs, not custom code.

### 2.2 BLE Key Concepts

```
Central (Phone)               Peripheral (Sensor/Device)
    │                               │
    │─── Scan for Advertisements ──▶│─── Advertises (broadcasts packets)
    │◀── Advertisement Packet ──────│
    │                               │
    │─── Connect Request ──────────▶│
    │◀── Connection Established ────│
    │                               │
    │    GATT Client                │    GATT Server
    │    (reads/writes)             │    (hosts Services/Characteristics)
    │                               │
    │─── Discover Services ────────▶│
    │─── Read Characteristic ──────▶│
    │◀── Characteristic Value ──────│
    │─── Write Characteristic ─────▶│
    │─── Enable Notifications ─────▶│
    │◀── Notification (data) ───────│ (triggered by peripheral when value changes)
```

**GATT hierarchy:**
```
Device
└── Service (e.g., Heart Rate Service - UUID: 0x180D)
    ├── Characteristic (Heart Rate Measurement - UUID: 0x2A37)
    │   ├── Properties: NOTIFY (peripheral pushes data)
    │   └── Client Characteristic Configuration Descriptor (CCCD)
    │       └── Write 0x0001 to enable notifications
    └── Characteristic (Body Sensor Location - UUID: 0x2A38)
        └── Properties: READ
```

**UUID:** Each service and characteristic has a UUID. Standard profiles use short 16-bit UUIDs (0x180D). Custom services use 128-bit UUIDs (e.g., `6E400001-B5A3-F393-E0A9-E50E24DCCA9E`).

---

## 3. BLE Permissions

Permissions changed significantly in Android 12 (API 31). This is a common interview trap.

### 3.1 Manifest Permissions

```xml
<!-- AndroidManifest.xml -->

<!-- Android 12+ (API 31+) — new granular permissions -->
<uses-permission android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation"/>
<!-- neverForLocation: tell Android this BLE scan is NOT for location inference -->
<!-- Without this flag, you also need ACCESS_FINE_LOCATION on API 31+ -->

<uses-permission android:name="android.permission.BLUETOOTH_CONNECT"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE"/>

<!-- Android 11 and below (still needed for older devices) -->
<uses-permission android:name="android.permission.BLUETOOTH"
    android:maxSdkVersion="30"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"
    android:maxSdkVersion="30"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"
    android:maxSdkVersion="30"/>

<!-- Declare BLE requirement (optional: set required="false" for feature degradation) -->
<uses-feature android:name="android.hardware.bluetooth_le" android:required="false"/>
```

### 3.2 Runtime Permission Handling

```kotlin
class BlePermissionHelper(private val activity: Activity) {

    private val requiredPermissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        arrayOf(
            Manifest.permission.BLUETOOTH_SCAN,
            Manifest.permission.BLUETOOTH_CONNECT
        )
    } else {
        arrayOf(
            Manifest.permission.ACCESS_FINE_LOCATION  // required pre-API 31 for BLE scan
        )
    }

    fun hasPermissions(): Boolean =
        requiredPermissions.all {
            ContextCompat.checkSelfPermission(activity, it) == PackageManager.PERMISSION_GRANTED
        }

    fun requestPermissions(launcher: ActivityResultLauncher<Array<String>>) {
        launcher.launch(requiredPermissions)
    }
}

// In Activity/Fragment
private val permissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    val allGranted = permissions.values.all { it }
    if (allGranted) startBleScanning()
    else showPermissionRationale()
}

// Check Bluetooth is enabled
fun isBluetoothEnabled(): Boolean {
    val bluetoothManager = context.getSystemService(BluetoothManager::class.java)
    return bluetoothManager?.adapter?.isEnabled == true
}

// Prompt user to enable Bluetooth
val enableBtIntent = Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE)
startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT)
```

---

## 4. BLE Scanning

### 4.1 Starting a Scan

```kotlin
class BleScanner @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val bluetoothAdapter: BluetoothAdapter by lazy {
        (context.getSystemService(BluetoothManager::class.java)).adapter
    }

    private val scanner: BluetoothLeScanner?
        get() = bluetoothAdapter.bluetoothLeScanner

    private var scanCallback: ScanCallback? = null
    private var scanJob: Job? = null

    fun startScan(
        targetServiceUuid: UUID? = null,
        onDeviceFound: (BluetoothDevice, Int, ScanRecord?) -> Unit,
        onError: (Int) -> Unit
    ) {
        val filters = if (targetServiceUuid != null) {
            listOf(
                ScanFilter.Builder()
                    .setServiceUuid(ParcelUuid(targetServiceUuid))
                    .build()
            )
        } else emptyList()

        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)    // fastest scan, highest battery
            // SCAN_MODE_BALANCED: 50% duty cycle
            // SCAN_MODE_LOW_POWER: scan only periodically (use for background)
            .setCallbackType(ScanSettings.CALLBACK_TYPE_ALL_MATCHES)
            .build()

        scanCallback = object : ScanCallback() {
            override fun onScanResult(callbackType: Int, result: ScanResult) {
                val device = result.device
                val rssi = result.rssi          // signal strength in dBm (-100 to 0; closer to 0 = stronger)
                val scanRecord = result.scanRecord
                onDeviceFound(device, rssi, scanRecord)
            }

            override fun onBatchScanResults(results: List<ScanResult>) {
                results.forEach { onScanResult(ScanSettings.CALLBACK_TYPE_ALL_MATCHES, it) }
            }

            override fun onScanFailed(errorCode: Int) {
                onError(errorCode)
                // Common error codes:
                // SCAN_FAILED_ALREADY_STARTED = 1 (scan already running)
                // SCAN_FAILED_APPLICATION_REGISTRATION_FAILED = 2
                // SCAN_FAILED_OUT_OF_HARDWARE_RESOURCES = 5
            }
        }

        scanner?.startScan(filters, settings, scanCallback!!)

        // Auto-stop after timeout (scanning drains battery)
        scanJob = CoroutineScope(Dispatchers.IO).launch {
            delay(SCAN_TIMEOUT_MS)
            stopScan()
        }
    }

    fun stopScan() {
        scanJob?.cancel()
        scanCallback?.let { scanner?.stopScan(it) }
        scanCallback = null
    }

    companion object {
        const val SCAN_TIMEOUT_MS = 10_000L  // 10 seconds max
    }
}
```

### 4.2 Parsing Advertising Data

```kotlin
fun parseScanRecord(scanRecord: ScanRecord?): Map<String, Any> {
    scanRecord ?: return emptyMap()

    return buildMap {
        // Device name from advertisement
        put("deviceName", scanRecord.deviceName ?: "Unknown")

        // Manufacturer-specific data
        for (i in 0 until scanRecord.manufacturerSpecificData.size()) {
            val companyId = scanRecord.manufacturerSpecificData.keyAt(i)
            val data = scanRecord.manufacturerSpecificData.valueAt(i)
            put("manufacturer_$companyId", data.toHexString())
        }

        // TX Power Level (optional, helps estimate distance)
        val txPower = scanRecord.txPowerLevel
        if (txPower != Int.MIN_VALUE) put("txPower", txPower)

        // Service UUIDs advertised
        put("services", scanRecord.serviceUuids?.map { it.uuid.toString() } ?: emptyList<String>())

        // Service Data
        scanRecord.serviceData.forEach { (uuid, data) ->
            put("serviceData_$uuid", data.toHexString())
        }
    }
}

// Distance estimation from RSSI and TX power
fun estimateDistance(rssi: Int, txPower: Int): Double {
    if (rssi == 0) return -1.0
    val ratio = rssi.toDouble() / txPower.toDouble()
    return if (ratio < 1.0) ratio.pow(10.0)
    else 0.89976 * ratio.pow(7.7095) + 0.111
}
```

---

## 5. GATT Connection & Operations

### 5.1 Connecting

```kotlin
class GattManager(
    private val context: Context,
    private val scope: CoroutineScope
) {
    private var bluetoothGatt: BluetoothGatt? = null
    private val operationQueue = ArrayDeque<BleOperation>()
    private var pendingOperation: BleOperation? = null

    // GATT operations MUST be serialized — only one at a time!
    // Android BLE stack becomes unreliable if you call multiple operations simultaneously

    private val gattCallback = object : BluetoothGattCallback() {
        override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
            when {
                status == BluetoothGatt.GATT_SUCCESS && newState == BluetoothProfile.STATE_CONNECTED -> {
                    // Connected — must call discoverServices AFTER connection
                    // Small delay needed on some devices (OEM quirk)
                    scope.launch {
                        delay(600)  // give stack time to settle
                        gatt.discoverServices()
                    }
                }
                newState == BluetoothProfile.STATE_DISCONNECTED -> {
                    handleDisconnect(status)
                    gatt.close()  // IMPORTANT: always close to free resources
                    bluetoothGatt = null
                }
            }
        }

        override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                // Services available — can now read/write characteristics
                scope.launch { _connectionState.emit(ConnectionState.Ready(gatt.services)) }
            }
        }

        override fun onCharacteristicRead(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            value: ByteArray,      // API 33+ provides value directly
            status: Int
        ) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                scope.launch {
                    _characteristicValues.emit(characteristic.uuid to value)
                }
            }
            completeOperation()  // signal queue to proceed
        }

        // API 33+ — old callback deprecated
        @Deprecated("Use onCharacteristicRead with value parameter on API 33+")
        override fun onCharacteristicRead(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            status: Int
        ) {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
                onCharacteristicRead(gatt, characteristic, characteristic.value, status)
            }
        }

        override fun onCharacteristicWrite(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            status: Int
        ) {
            completeOperation()
        }

        override fun onCharacteristicChanged(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            value: ByteArray       // API 33+
        ) {
            // Called when peripheral sends a Notification or Indication
            scope.launch {
                _notifications.emit(characteristic.uuid to value)
            }
        }

        override fun onDescriptorWrite(
            gatt: BluetoothGatt,
            descriptor: BluetoothGattDescriptor,
            status: Int
        ) {
            completeOperation()  // CCCD write completed
        }

        override fun onMtuChanged(gatt: BluetoothGatt, mtu: Int, status: Int) {
            // MTU negotiated — adjust data packet size accordingly
            // Default MTU: 23 bytes. Max: 517 bytes.
            // Effective payload: mtu - 3 bytes (ATT header)
            if (status == BluetoothGatt.GATT_SUCCESS) {
                currentMtu = mtu - 3
            }
            completeOperation()
        }
    }

    @SuppressLint("MissingPermission")
    fun connect(device: BluetoothDevice) {
        bluetoothGatt = device.connectGatt(
            context,
            false,       // autoConnect: false = direct connection attempt
                         // true = background connection (slower but more reliable for known devices)
            gattCallback,
            BluetoothDevice.TRANSPORT_LE  // force BLE transport (important for dual-mode devices)
        )
    }
}
```

### 5.2 Reading a Characteristic

```kotlin
fun readCharacteristic(serviceUuid: UUID, charUuid: UUID) {
    val gatt = bluetoothGatt ?: return
    val service = gatt.getService(serviceUuid) ?: return
    val characteristic = service.getCharacteristic(charUuid) ?: return

    if (characteristic.properties and BluetoothGattCharacteristic.PROPERTY_READ == 0) {
        // Characteristic doesn't support read
        return
    }

    enqueueOperation(ReadOperation(characteristic)) {
        gatt.readCharacteristic(characteristic)
    }
}
```

### 5.3 Writing a Characteristic

```kotlin
@SuppressLint("MissingPermission")
fun writeCharacteristic(serviceUuid: UUID, charUuid: UUID, value: ByteArray) {
    val gatt = bluetoothGatt ?: return
    val characteristic = gatt.getService(serviceUuid)?.getCharacteristic(charUuid) ?: return

    val writeType = when {
        characteristic.properties and BluetoothGattCharacteristic.PROPERTY_WRITE != 0 ->
            BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT  // with acknowledgment
        characteristic.properties and BluetoothGattCharacteristic.PROPERTY_WRITE_NO_RESPONSE != 0 ->
            BluetoothGattCharacteristic.WRITE_TYPE_NO_RESPONSE  // faster, no ack
        else -> return
    }

    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        // API 33+ — new API
        gatt.writeCharacteristic(characteristic, value, writeType)
    } else {
        @Suppress("DEPRECATION")
        characteristic.value = value
        characteristic.writeType = writeType
        gatt.writeCharacteristic(characteristic)
    }
}
```

### 5.4 Enabling Notifications

```kotlin
@SuppressLint("MissingPermission")
fun enableNotifications(serviceUuid: UUID, charUuid: UUID) {
    val gatt = bluetoothGatt ?: return
    val characteristic = gatt.getService(serviceUuid)?.getCharacteristic(charUuid) ?: return

    // Step 1: Register local callback
    gatt.setCharacteristicNotification(characteristic, true)

    // Step 2: Write to CCCD descriptor to tell peripheral to start sending
    val cccd = characteristic.getDescriptor(
        UUID.fromString("00002902-0000-1000-8000-00805f9b34fb")  // standard CCCD UUID
    ) ?: return

    val value = if (characteristic.properties and BluetoothGattCharacteristic.PROPERTY_NOTIFY != 0) {
        BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE   // Notify: no acknowledgment
    } else {
        BluetoothGattDescriptor.ENABLE_INDICATION_VALUE     // Indicate: peripheral waits for ack
    }

    // API 33+
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        gatt.writeDescriptor(cccd, value)
    } else {
        @Suppress("DEPRECATION")
        cccd.value = value
        gatt.writeDescriptor(cccd)
    }
}

// Notify vs Indicate:
// NOTIFY:   Peripheral sends data, no ack from central. Fast, data may be lost.
// INDICATE: Peripheral sends, waits for central acknowledgment before next. Reliable.
```

### 5.5 MTU Request (Large Data Transfer)

Default BLE ATT MTU = 23 bytes (20 bytes payload). Negotiate larger MTU for efficient data transfer:

```kotlin
// Request immediately after services discovered
gatt.requestMtu(512)  // request 512 bytes; actual negotiated value in onMtuChanged
// Effective payload = negotiated_mtu - 3

// For data larger than one MTU, split into chunks
fun writeInChunks(characteristic: BluetoothGattCharacteristic, data: ByteArray) {
    val chunks = data.toList().chunked(currentMtu)
    chunks.forEach { chunk ->
        writeCharacteristic(serviceUuid, charUuid, chunk.toByteArray())
        // Each write must wait for onCharacteristicWrite callback before next write
        // Use operation queue to serialize
    }
}
```

---

## 6. BLE Operation Queue (Critical Pattern)

The Android BLE stack is **not thread-safe** and **cannot handle concurrent operations**. Calling `readCharacteristic()` before the previous `readCharacteristic()` callback has returned causes silent failures.

```kotlin
class BleOperationQueue {
    private val queue = ArrayDeque<BleOperation>()
    private var isOperationPending = false

    sealed class BleOperation {
        class Read(val characteristic: BluetoothGattCharacteristic) : BleOperation()
        class Write(val characteristic: BluetoothGattCharacteristic, val value: ByteArray) : BleOperation()
        class EnableNotify(val characteristic: BluetoothGattCharacteristic) : BleOperation()
        class RequestMtu(val mtu: Int) : BleOperation()
    }

    fun enqueue(operation: BleOperation) {
        queue.add(operation)
        if (!isOperationPending) executeNext()
    }

    private fun executeNext() {
        if (queue.isEmpty()) {
            isOperationPending = false
            return
        }
        isOperationPending = true
        val operation = queue.removeFirst()
        execute(operation)
    }

    fun operationCompleted() {
        isOperationPending = false
        executeNext()
    }
    // Call operationCompleted() in every GATT callback
}
```

---

## 7. BLE Connection Stability

### 7.1 Connection Drop Handling

```kotlin
private var reconnectAttempts = 0

private fun handleDisconnect(status: Int) {
    when (status) {
        BluetoothGatt.GATT_SUCCESS -> {
            // Clean disconnect by our request — don't reconnect
        }
        133 -> {
            // Status 133 (0x85) — GATT_ERROR: most common on Android
            // Causes: cache stale, adapter state flip, race condition
            // Fix: close GATT, wait, reconnect with fresh GATT object
            scheduleReconnect()
        }
        19 -> {
            // GATT_CONN_TERMINATE_PEER_USER: peripheral disconnected us
            scheduleReconnect()
        }
        else -> scheduleReconnect()
    }
}

private fun scheduleReconnect() {
    if (reconnectAttempts >= MAX_RECONNECT_ATTEMPTS) {
        _connectionState.value = ConnectionState.Failed
        return
    }
    val delayMs = minOf(1000L * 2.0.pow(reconnectAttempts).toLong(), 30_000L)
    scope.launch {
        delay(delayMs)
        reconnectAttempts++
        connect(lastDevice)
    }
}
```

### 7.2 Status 133 — The Most Common BLE Bug on Android

Status 133 (`GATT_ERROR`) is a notoriously common bug across Android OEMs. It can happen due to:
- Leftover GATT cache from a previous connection
- Bluetooth adapter state issues
- Race conditions in the BLE stack

**Fix sequence:**
```kotlin
// On status 133:
gatt.disconnect()
gatt.close()              // MUST close to free Bluetooth resources
bluetoothGatt = null

delay(1000)               // Wait for adapter to clear state

// Refresh device cache (unofficial but widely used fix)
try {
    val refreshMethod = bluetoothGatt?.javaClass?.getMethod("refresh")
    refreshMethod?.invoke(bluetoothGatt)
} catch (e: Exception) { /* ignore */ }

// Reconnect with a fresh GATT instance
connect(device)
```

---

## 8. NFC

### 8.1 NFC Overview

NFC (Near Field Communication) operates at 13.56 MHz within ~4cm range. Three modes:
- **Reader/Writer:** Device reads or writes NFC tags
- **Card Emulation:** Device acts like an NFC card (contactless payment, access card)
- **Peer-to-peer (P2P):** Device-to-device communication (deprecated; use BLE/WiFi Direct instead)

### 8.2 NFC Tag Types

| Tag Type | Standard | Common use |
|---|---|---|
| Type 1 (NFC-A / Topaz) | ISO 14443-3A | Simple tags, limited use |
| Type 2 (NFC-A / MIFARE Ultralight) | ISO 14443-3A | Business cards, stickers |
| Type 3 (NFC-F / Sony FeliCa) | JIS X 6319-4 | Transit cards (Japan) |
| Type 4 (NFC-A or NFC-B) | ISO 14443-4 | Smart cards, passports |
| MIFARE Classic | Proprietary (NXP) | Access cards, transit (legacy) |

All modern use cases use **NDEF (NFC Data Exchange Format)** — a standardized message format that wraps the actual content.

### 8.3 NDEF Message Structure

```
NDEF Message
├── NDEF Record
│   ├── TNF (Type Name Format): SHORT_RECORD, WELL_KNOWN, MIME, URI, EXTERNAL
│   ├── Type: identifies the record type
│   ├── ID: optional identifier
│   └── Payload: the actual data
└── NDEF Record 2 (optional — messages can contain multiple records)

Common NDEF record types:
- RTD_TEXT (T): Plain text
- RTD_URI (U): URL/URI (with URI prefix compression)
- MIME type: "text/plain", "application/json"
- External type: "android.com:pkg" (Android Application Record)
```

### 8.4 Reading NFC Tags

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.NFC"/>
<uses-feature android:name="android.hardware.nfc" android:required="false"/>

<activity android:name=".NfcReaderActivity">
    <!-- NDEF_DISCOVERED: highest priority, NDEF-formatted tags -->
    <intent-filter>
        <action android:name="android.nfc.action.NDEF_DISCOVERED"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="application/com.example.myapp"/>
    </intent-filter>

    <!-- TAG_DISCOVERED: fallback for any tag -->
    <intent-filter>
        <action android:name="android.nfc.action.TAG_DISCOVERED"/>
    </intent-filter>
</activity>
```

```kotlin
class NfcReaderActivity : AppCompatActivity() {
    private lateinit var nfcAdapter: NfcAdapter
    private lateinit var pendingIntent: PendingIntent
    private val nfcFilters: Array<IntentFilter> by lazy {
        arrayOf(IntentFilter(NfcAdapter.ACTION_NDEF_DISCOVERED).apply {
            addDataType("*/*")
        })
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        nfcAdapter = NfcAdapter.getDefaultAdapter(this) ?: run {
            showNfcNotSupportedMessage()
            return
        }

        // PendingIntent for foreground dispatch (highest priority when activity is foreground)
        pendingIntent = PendingIntent.getActivity(
            this, 0,
            Intent(this, javaClass).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP),
            PendingIntent.FLAG_MUTABLE
        )

        // Handle tag if app was launched by NFC intent
        handleNfcIntent(intent)
    }

    override fun onResume() {
        super.onResume()
        // Enable foreground dispatch — this activity gets priority over all NFC intents
        nfcAdapter.enableForegroundDispatch(this, pendingIntent, nfcFilters, null)
    }

    override fun onPause() {
        super.onPause()
        nfcAdapter.disableForegroundDispatch(this)
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        handleNfcIntent(intent)
    }

    private fun handleNfcIntent(intent: Intent) {
        if (intent.action != NfcAdapter.ACTION_NDEF_DISCOVERED &&
            intent.action != NfcAdapter.ACTION_TAG_DISCOVERED) return

        val tag = intent.getParcelableExtra<Tag>(NfcAdapter.EXTRA_TAG) ?: return

        // Read NDEF data if present
        val ndef = Ndef.get(tag) ?: run {
            showMessage("Tag is not NDEF formatted")
            return
        }

        lifecycleScope.launch(Dispatchers.IO) {
            try {
                ndef.connect()
                val ndefMessage = ndef.ndefMessage ?: ndef.cachedNdefMessage
                val records = ndefMessage?.records ?: return@launch
                val parsedData = parseNdefRecords(records)
                withContext(Dispatchers.Main) { displayData(parsedData) }
            } finally {
                ndef.close()
            }
        }
    }

    private fun parseNdefRecords(records: Array<NdefRecord>): List<NfcData> {
        return records.mapNotNull { record ->
            when {
                record.tnf == NdefRecord.TNF_WELL_KNOWN &&
                record.type.contentEquals(NdefRecord.RTD_TEXT) -> {
                    // Text record
                    val payload = record.payload
                    val encoding = if (payload[0].toInt() and 0x80 == 0) "UTF-8" else "UTF-16"
                    val langLength = payload[0].toInt() and 0x3F
                    val text = String(payload, langLength + 1, payload.size - langLength - 1, charset(encoding))
                    NfcData.Text(text)
                }
                record.tnf == NdefRecord.TNF_WELL_KNOWN &&
                record.type.contentEquals(NdefRecord.RTD_URI) -> {
                    // URI record (with prefix compression)
                    val uri = record.toUri()?.toString() ?: return@mapNotNull null
                    NfcData.Uri(uri)
                }
                record.tnf == NdefRecord.TNF_MIME_MEDIA -> {
                    val mimeType = String(record.type, Charsets.US_ASCII)
                    NfcData.MimeData(mimeType, record.payload)
                }
                else -> null
            }
        }
    }
}
```

### 8.5 Writing NFC Tags

```kotlin
fun writeNdefTag(tag: Tag, data: String) {
    val ndef = Ndef.get(tag)
    if (ndef != null) {
        // Tag already NDEF formatted — write directly
        writeToNdef(ndef, createNdefMessage(data))
    } else {
        // Tag not yet formatted — format first then write
        val ndefFormatable = NdefFormatable.get(tag)
        if (ndefFormatable != null) {
            formatAndWrite(ndefFormatable, createNdefMessage(data))
        } else {
            throw UnsupportedOperationException("Tag cannot be written")
        }
    }
}

private fun writeToNdef(ndef: Ndef, message: NdefMessage) {
    ndef.connect()
    try {
        if (!ndef.isWritable) throw IOException("Tag is read-only")
        if (ndef.maxSize < message.byteArrayLength) throw IOException("Tag too small")
        ndef.writeNdefMessage(message)
    } finally {
        ndef.close()
    }
}

private fun createNdefMessage(text: String): NdefMessage {
    val textRecord = NdefRecord.createTextRecord("en", text)
    // Or create URI record:
    // val uriRecord = NdefRecord.createUri("https://example.com")
    // Or create MIME record:
    // val mimeRecord = NdefRecord.createMime("application/json", jsonBytes)
    return NdefMessage(arrayOf(textRecord))
}

// Android Application Record — opens your app when tag is tapped
fun addAndroidApplicationRecord(message: NdefMessage): NdefMessage {
    val aar = NdefRecord.createApplicationRecord("com.example.myapp")
    return NdefMessage(arrayOf(*message.records, aar))
}
```

---

## 9. Host Card Emulation (HCE)

HCE lets your Android app behave like an NFC card — no physical secure element needed. Used for contactless payments, loyalty cards, access control.

### 9.1 How HCE Works

```
NFC Reader (payment terminal, access reader)
    │
    │─── SELECT AID (Application Identifier) ──▶ Android OS
    │                                              │ routes to your HostApduService
    │                                              ▼
    │◀── Response (SW1 SW2 = 90 00 = OK) ─────── HostApduService
    │
    │─── Custom APDU commands ─────────────────▶ HostApduService
    │◀── APDU responses ───────────────────────── HostApduService
```

### 9.2 HCE Implementation

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.NFC"/>

<service
    android:name=".MyHceService"
    android:exported="true"
    android:permission="android.permission.BIND_NFC_SERVICE">
    <intent-filter>
        <action android:name="android.nfc.cardemulation.action.HOST_APDU_SERVICE"/>
    </intent-filter>
    <meta-data
        android:name="android.nfc.cardemulation.host_apdu_service"
        android:resource="@xml/apduservice"/>
</service>

<!-- res/xml/apduservice.xml -->
<host-apdu-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:description="@string/service_description"
    android:requireDeviceUnlock="false">
    <aid-group android:description="@string/aid_description" android:category="other">
        <!-- AID = Application Identifier that NFC reader uses to select your service -->
        <aid-filter android:name="F0394148148100"/>  <!-- custom AID -->
    </aid-group>
</host-apdu-service>
```

```kotlin
class MyHceService : HostApduService() {

    override fun processCommandApdu(commandApdu: ByteArray, extras: Bundle?): ByteArray {
        // Parse the incoming APDU command
        val commandHex = commandApdu.toHexString()
        Log.d("HCE", "Received APDU: $commandHex")

        return when {
            commandApdu.startsWith(SELECT_AID_COMMAND) -> {
                // Reader selected our AID — respond with OK
                SUCCESS_RESPONSE
            }
            commandApdu.startsWith(GET_DATA_COMMAND) -> {
                // Reader is requesting data — send our payload
                buildResponse(getPayloadData())
            }
            else -> {
                UNKNOWN_COMMAND_RESPONSE
            }
        }
    }

    override fun onDeactivated(reason: Int) {
        // NFC connection ended
        // reason: DEACTIVATION_LINK_LOSS (RF field gone) or DEACTIVATION_DESELECTED (reader selected another AID)
        Log.d("HCE", "Deactivated: $reason")
    }

    private fun buildResponse(data: ByteArray): ByteArray {
        // Append SW1 SW2 = 0x9000 (success) to data
        return data + SUCCESS_RESPONSE
    }

    companion object {
        val SELECT_AID_COMMAND = byteArrayOf(0x00.toByte(), 0xA4.toByte(), 0x04.toByte(), 0x00.toByte())
        val SUCCESS_RESPONSE = byteArrayOf(0x90.toByte(), 0x00.toByte())
        val UNKNOWN_COMMAND_RESPONSE = byteArrayOf(0x00.toByte(), 0x00.toByte())
        val GET_DATA_COMMAND = byteArrayOf(0x00.toByte(), 0xB0.toByte())
    }
}
```

**HCE constraints:**
- Only works while screen is on and device is unlocked (unless `requireDeviceUnlock="false"`)
- Cannot emulate MIFARE Classic tags (proprietary protocol, NFC chip handles it)
- Cannot emulate NFC-F (FeliCa) — only ISO-DEP / NFC-A Type 4 cards
- For payments: must use Android's Payment Tokenization Service (Google Pay) or a licensed secure element

---

## 10. React Native

```typescript
// react-native-ble-plx — most popular BLE library for RN
import { BleManager, Device, State } from 'react-native-ble-plx'
const manager = new BleManager()

// Check BLE state
manager.onStateChange((state) => {
    if (state === State.PoweredOn) startScan()
}, true)

// Scan
const subscription = manager.startDeviceScan(
    [SERVICE_UUID],   // filter by service UUID (null = all devices)
    null,
    (error, device) => {
        if (error) { console.error(error); return }
        if (device?.name === 'MyDevice') {
            manager.stopDeviceScan()
            connectAndInteract(device)
        }
    }
)

// Connect, discover, read
async function connectAndInteract(device: Device) {
    const connected = await device.connect()
    const discovered = await connected.discoverAllServicesAndCharacteristics()
    const value = await discovered.readCharacteristicForService(
        SERVICE_UUID,
        CHARACTERISTIC_UUID
    )
    const decoded = atob(value.value ?? '')  // BLE values are base64 encoded
    console.log('Read:', decoded)
}

// NFC for React Native
// react-native-nfc-manager
import NfcManager, { NfcTech, Ndef } from 'react-native-nfc-manager'
NfcManager.start()

async function readNfcTag() {
    await NfcManager.requestTechnology(NfcTech.Ndef)
    const tag = await NfcManager.getTag()
    const ndefMessage = tag?.ndefMessage ?? []
    ndefMessage.forEach(record => {
        const text = Ndef.text.decodePayload(record.payload)
        console.log('NFC text:', text)
    })
    NfcManager.cancelTechnologyRequest()
}
```

---

## 11. Common Misunderstandings & Pitfalls

**❌ Not serializing GATT operations**
Calling `readCharacteristic()` before the previous callback has returned causes silent failures — the second read simply doesn't happen and no error is reported. Always use an operation queue; Android's BLE stack processes only one GATT operation at a time.

**❌ Not calling `gatt.close()` on disconnect**
`bluetoothGatt.disconnect()` closes the connection but doesn't free native resources. You MUST call `gatt.close()` afterward. Failure to do so leaks native Bluetooth handles and eventually causes new connections to fail.

**❌ Forgetting the 600ms delay after connection before discovering services**
On many Android devices (Samsung in particular), calling `gatt.discoverServices()` immediately after `onConnectionStateChange(STATE_CONNECTED)` fails with status 133. A small delay (600ms) after connection before calling `discoverServices()` fixes this on most devices.

**❌ Status 133 without proper recovery**
Status 133 (`GATT_ERROR`) is the most common BLE failure on Android. Recovery requires: `gatt.disconnect()` → `gatt.close()` → `delay(1000)` → fresh `device.connectGatt()`. Trying to reconnect without closing the old GATT object won't work.

**❌ Wrong permissions for Android version**
On Android 12+ (API 31+), `BLUETOOTH` and `BLUETOOTH_ADMIN` are replaced by `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, and `BLUETOOTH_ADVERTISE`. Apps that only declare old permissions silently fail to scan on API 31+.

**❌ Forgetting `android:usesPermissionFlags="neverForLocation"` on `BLUETOOTH_SCAN`**
Without this flag on API 31+, your app also needs `ACCESS_FINE_LOCATION` even if you're not doing location-based scanning. The `neverForLocation` flag signals to Android (and users) that BLE is not being used for location.

**❌ Not closing NDEF connection after reading/writing**
`Ndef.connect()` must always be paired with `Ndef.close()` in a `finally` block. Forgetting to close the connection prevents other apps from reading the tag.

**❌ Testing HCE on an emulator**
HCE requires real NFC hardware. It cannot be tested on Android emulators. You need two physical Android devices — one as the HCE host, another as the reader.

---

## 12. Best Practices

**BLE:**
- Always serialize GATT operations — one at a time, use an operation queue
- Call `gatt.close()` after every `gatt.disconnect()` — no exceptions
- Add 600ms delay between `STATE_CONNECTED` and `discoverServices()`
- Declare `neverForLocation` on `BLUETOOTH_SCAN` if not doing location
- Filter by service UUID in scan — don't scan for everything
- Stop scanning after 10s maximum — continuous scanning drains battery
- Use `TRANSPORT_LE` in `connectGatt()` for dual-mode devices
- Negotiate MTU (request 512) after connecting for faster large data transfer
- Handle status 133 with close → delay → fresh connect sequence
- Use `WRITE_TYPE_NO_RESPONSE` for frequent small writes (e.g., streaming sensor config)

**NFC:**
- Enable foreground dispatch in `onResume`, disable in `onPause`
- Always wrap NDEF operations in try-finally to ensure `close()` is called
- Use `ACTION_NDEF_DISCOVERED` with MIME filter for the most specific intent dispatch
- Add Android Application Record (AAR) to tags to ensure your app is launched
- Run NFC tag read/write on a background coroutine (IO operations)
- Test with NFC-enabled physical devices only (no emulator support)

---

## 13. Interview Q&A

**Q1: Walk me through how BLE communication works from scan to data transfer.**

> BLE has two roles: the central (phone) and peripheral (sensor). The peripheral broadcasts advertising packets every 20–10,000ms containing its name, services, and manufacturer data. The central scans for these packets using `BluetoothLeScanner`. Once the user selects a device, the central calls `device.connectGatt()` to establish a connection. After `onConnectionStateChange(STATE_CONNECTED)`, I call `discoverServices()` (after a small delay for stability). `onServicesDiscovered()` returns all GATT services and their characteristics. A service groups related characteristics — for example, the Heart Rate Service contains a Heart Rate Measurement characteristic. I then read, write, or enable notifications on characteristics as needed. Critical: all GATT operations must be serialized — only one at a time.

---

**Q2: What's the difference between Notify and Indicate in GATT?**

> Both are mechanisms where the peripheral pushes data to the central when a characteristic value changes — the central doesn't need to poll. The difference is in acknowledgment. Notify: peripheral sends the data, no acknowledgment from central required — faster, but data can be lost if the central's buffer is full. Indicate: peripheral sends the data and waits for an acknowledgment from the central before sending the next indication — slower but reliable, no data loss. For sensor streaming (heart rate, accelerometer) where occasional loss is acceptable, use Notify. For critical data (commands, configuration) that must be delivered, use Indicate. To enable either, write to the Client Characteristic Configuration Descriptor (CCCD): `0x0001` for notifications, `0x0002` for indications.

---

**Q3: How do you handle status 133 in BLE?**

> Status 133 (GATT_ERROR) is the most common BLE failure on Android and notoriously poorly documented. It can result from stale GATT cache, Bluetooth adapter state inconsistency, or race conditions in the BLE stack. The recovery sequence is: call `gatt.disconnect()`, then `gatt.close()` (mandatory to free native resources), wait about 1 second for the adapter to clear state, then call `device.connectGatt()` with a fresh GATT object. Never try to reconnect without closing the old GATT — it won't work. Some codebases also call the undocumented `gatt.refresh()` via reflection to clear the GATT cache, which helps with some OEM-specific variants of this error. After 3–5 failed reconnect attempts with exponential backoff, surface a user-visible error rather than retrying indefinitely.

---

**Q4: How does NFC tag reading work in Android?**

> Android uses an intent-based dispatch system. When a tag is detected, Android creates an intent (`ACTION_NDEF_DISCOVERED`, `ACTION_TECH_DISCOVERED`, or `ACTION_TAG_DISCOVERED`) and routes it to the app most specifically matching the tag — a more specific intent filter wins over a general one. The activity receives the tag as a `Tag` extra. It wraps the tag with `Ndef.get(tag)` to access NDEF functionality, connects (`ndef.connect()`), reads the NDEF message (`ndef.ndefMessage`), parses each record by its TNF and type, then closes the connection. `enableForegroundDispatch()` in `onResume` gives the foreground activity highest priority for tag dispatch — important so your app intercepts tags instead of a competing app. Always run NFC I/O on a background thread since `connect()` is a blocking call.

---

**Q5: What is HCE and when would you use it?**

> Host Card Emulation allows an Android app to emulate an NFC smart card without a hardware secure element. When an NFC reader taps the device, Android routes the APDU commands to your `HostApduService` based on the Application Identifier (AID) the reader sends. Your service responds with APDU responses that mimic what a real card would send. Use cases: corporate access badges (your app serves the door controller the same APDUs a physical card would), loyalty card aggregation (app stores multiple loyalty cards and presents the right one based on AID), custom data exchange between two Android devices. Limitations: can only emulate ISO-DEP (Type 4) cards — cannot emulate MIFARE Classic (proprietary) or FeliCa. For payments, HCE must integrate with Android's payment infrastructure (Google Pay) rather than handling card data directly.

---

**Q6: Why does BLE scanning require location permission on older Android versions?**

> On Android 11 and below (API 30-), BLE scanning requires `ACCESS_FINE_LOCATION` because BLE beacons can be used for indoor positioning and geofencing — the OS treats BLE device detection as a potential location inference mechanism. If users disable location services, BLE scanning results are also blocked. Android 12 (API 31) introduced granular Bluetooth permissions (`BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`) that separate Bluetooth access from location access. If your app is genuinely not using BLE for location purposes, you can declare `android:usesPermissionFlags="neverForLocation"` on `BLUETOOTH_SCAN` to exempt your app from the location permission requirement even on API 31+. This also means your scanning still works when the user has location services turned off, which matters for apps that scan BLE devices for non-location purposes (IoT device setup, fitness equipment pairing).

---

*Previous: [13 — Cross-Cutting Concerns](./13-cross-cutting.md)*
*Next: [15 — Video Playback & DRM](./15-video-drm.md)*
