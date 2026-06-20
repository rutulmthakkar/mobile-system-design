# 28 — Maps Integration

> **Round type: Both (HLD + LLD)**
> **HLD:** Choosing a map provider, designing a real-time location architecture for ride-sharing
> **LLD:** Google Maps Compose setup, markers, polylines, Fused Location Provider, smooth driver animation, geofencing

---

## 1. Why This Matters in Interviews

Maps questions appear at ride-hailing (Uber, Lyft, Ola), delivery (DoorDash, Swiggy), logistics, and fitness companies. They test your understanding of location services, real-time data handling, battery efficiency, and UI smoothness — all in one feature.

**Common interview angles:**
- "Which map SDK would you choose and why?"
- "How does the Fused Location Provider work?"
- "How do you animate a car marker smoothly on the map (Uber-style)?"
- "How would you design the real-time location tracking system for a ride-sharing app?"
- "How do you draw a route between two points?"
- "What is geofencing and how would you use it in a delivery app?"
- "How do you reduce battery drain from continuous GPS tracking?"

---

## 2. Map SDK Comparison

### 2.1 The Main Providers

| | Google Maps | Mapbox | HERE Maps | OpenStreetMap + MapLibre | Azure Maps |
|---|---|---|---|---|---|
| **Data quality** | ⭐⭐⭐⭐⭐ Best global | ⭐⭐⭐⭐ Good (uses OSM) | ⭐⭐⭐⭐⭐ Excellent (logistics) | ⭐⭐⭐⭐ Community-driven | ⭐⭐⭐⭐ Good |
| **Customization** | 🟡 Limited | ✅ Excellent | ✅ Good | ✅ Full control | 🟡 Limited |
| **Offline maps** | ❌ No (except Nav SDK) | ✅ Yes | ✅ Yes | ✅ Yes | 🟡 Partial |
| **Pricing** | Paid (free tier) | Paid (free tier) | Paid (free tier) | **Free / Self-host** | Paid (Azure credit) |
| **Routing API** | Routes API ✅ | Directions API ✅ | Routing API ✅ | OSRM / Valhalla | Route API ✅ |
| **Traffic data** | ✅ Excellent | ✅ Good | ✅ Excellent | ❌ Limited | ✅ Good |
| **Turn-by-turn nav** | Navigation SDK ✅ | ✅ | ✅ | Valhalla | Limited |
| **Fleet/logistics** | Fleet Engine | ✅ | ✅ Best-in-class | ❌ | ✅ |
| **Privacy / EU** | Google-hosted | US-hosted | EU data centers ✅ | Self-host ✅ | Azure region control |
| **Best for** | Consumer apps, global reach | Custom branded maps, rich UI | Logistics, fleet, automotive | Cost-zero, open source | Microsoft ecosystem |

### 2.2 Pricing Reality (2024–2025)

**Google Maps Platform** restructured pricing in March 2025 — removed the universal $200/month credit, added per-SKU free tiers:
- Maps SDK loads: free up to 10,000/month
- Directions API: free up to 10,000/month
- Places Autocomplete: free up to 10,000/month
- At scale (1M requests/month): can reach $1,000–5,000+/month easily

**Mapbox:** ~$0.50 per 1,000 map loads after free tier. Competitive for mid-scale apps.

**HERE Maps:** Enterprise pricing with more predictability; good for B2B/fleet apps.

**OpenStreetMap + MapLibre:** Free tile rendering if self-hosted. Use commercial tile providers (Stadia Maps, MapTiler ~$0.15–0.30 per 1,000 tiles) for managed hosting.

### 2.3 Decision Guide

```
Building a consumer app with global users?
  → Google Maps — best data quality, users trust it, Places Autocomplete is excellent

Need heavy map customization / white-label / brand-consistent design?
  → Mapbox — best styling toolkit, vector tiles, full design control

Logistics / fleet / trucking / automotive?
  → HERE Maps — industry leader for routing with truck restrictions, live traffic

EU data residency required or budget-sensitive?
  → OpenStreetMap + MapLibre — zero licensing cost, self-hostable, GDPR-friendly

Already on Azure / Microsoft ecosystem?
  → Azure Maps — unified billing, good enterprise support

Building a ride-sharing or delivery app?
  → Google Maps (most common), or HERE if fleet data is critical
```

---

## 3. Google Maps SDK Setup (Android)

### 3.1 Dependencies

```kotlin
// gradle
implementation("com.google.maps.android:maps-compose:6.2.1")   // Compose wrapper (official)
implementation("com.google.android.gms:play-services-maps:19.0.0")
implementation("com.google.android.libraries.places:places:4.0.0")  // Places API / autocomplete
implementation("com.google.android.gms:play-services-location:21.3.0")  // Fused Location Provider
implementation("com.google.maps.android:android-maps-utils:3.8.2")  // utilities: PolyUtil, SphericalUtil

// API key (never commit to source control)
// In AndroidManifest.xml:
<meta-data
    android:name="com.google.android.geo.API_KEY"
    android:value="${MAPS_API_KEY}"/>
// Store in local.properties or CI secrets:
// MAPS_API_KEY=your_key_here
```

**API key security:**
- In Google Cloud Console: restrict the key to your app's package name + SHA-256 signing cert
- Even if extracted from APK, the key only works from your signed app

### 3.2 Basic Map in Compose

```kotlin
@Composable
fun BasicMapScreen() {
    val singapore = LatLng(1.35, 103.87)
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(singapore, 12f)
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = MapProperties(
            mapType = MapType.NORMAL,          // NORMAL, SATELLITE, TERRAIN, HYBRID
            isMyLocationEnabled = false,        // shows blue dot — requires permission
            isBuildingEnabled = true,
            isTrafficEnabled = true            // show traffic overlay
        ),
        uiSettings = MapUiSettings(
            zoomControlsEnabled = false,       // hide ugly zoom +/- buttons
            myLocationButtonEnabled = false,   // handle custom location button
            compassEnabled = true,
            mapToolbarEnabled = false          // hide navigation/maps app toolbar
        ),
        onMapClick = { latLng -> /* handle tap */ },
        onMapLongClick = { latLng -> /* handle long press */ }
    ) {
        // Map content as child composables
        Marker(
            state = MarkerState(position = singapore),
            title = "Singapore",
            snippet = "Lion City"
        )
    }
}
```

---

## 4. Fused Location Provider

The Fused Location Provider (FLP) is Google Play Services' smart location API — it combines GPS, WiFi, and cell tower data to provide accurate location with minimal battery drain.

**Never use `LocationManager` directly** in modern apps. FLP is always better.

```kotlin
// gradle
implementation("com.google.android.gms:play-services-location:21.3.0")

class LocationRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val fusedLocationClient = LocationServices.getFusedLocationProviderClient(context)

    // One-time: get current location
    @SuppressLint("MissingPermission")
    suspend fun getCurrentLocation(): LatLng? {
        return suspendCancellableCoroutine { continuation ->
            fusedLocationClient.lastLocation
                .addOnSuccessListener { location ->
                    continuation.resume(location?.let { LatLng(it.latitude, it.longitude) })
                }
                .addOnFailureListener { continuation.resume(null) }
        }
    }

    // Continuous: location updates as Flow
    @SuppressLint("MissingPermission")
    fun getLocationUpdates(intervalMs: Long = 3000L): Flow<LatLng> = callbackFlow {
        val request = LocationRequest.Builder(
            Priority.PRIORITY_HIGH_ACCURACY,   // GPS-level accuracy (battery heavy)
            intervalMs                         // target interval
        )
        .setMinUpdateIntervalMillis(1000)      // fastest possible interval
        .setWaitForAccurateLocation(false)     // don't wait for GPS lock — use best available
        .build()

        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                result.lastLocation?.let { location ->
                    trySend(LatLng(location.latitude, location.longitude))
                }
            }
        }

        fusedLocationClient.requestLocationUpdates(request, callback, Looper.getMainLooper())
        awaitClose { fusedLocationClient.removeLocationUpdates(callback) }
    }
}

// Priority levels:
// PRIORITY_HIGH_ACCURACY   → GPS, highest accuracy, highest battery drain
// PRIORITY_BALANCED_POWER  → WiFi + cell, city-block accuracy, moderate battery
// PRIORITY_LOW_POWER       → Cell-only, city-level accuracy, minimal battery
// PRIORITY_PASSIVE         → Only receives updates from other apps' requests
```

### 4.1 Location Permissions

```kotlin
// AndroidManifest.xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>      // GPS precision
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>    // WiFi/cell only
<!-- Background location — requires separate runtime request on Android 10+ -->
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>

// Runtime permission flow
val locationPermissions = arrayOf(
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.ACCESS_COARSE_LOCATION
)

val permissionLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    val fineGranted = permissions[Manifest.permission.ACCESS_FINE_LOCATION] == true
    val coarseGranted = permissions[Manifest.permission.ACCESS_COARSE_LOCATION] == true
    if (fineGranted || coarseGranted) startTracking()
    else showPermissionRationale()
}

// Background location — must request separately AFTER foreground location granted
// Never request background location without a clear user-visible reason
val backgroundPermLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted ->
    if (!granted) showBackgroundLocationExplanation()
}
// backgroundPermLauncher.launch(Manifest.permission.ACCESS_BACKGROUND_LOCATION)
```

---

## 5. Ride-Sharing App: Full Implementation

### 5.1 Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   Rider App                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │ Map Screen   │    │ RideViewModel│    │  Repository  │  │
│  │ (Compose)    │◀───│              │◀───│              │  │
│  │              │    │ driverLocation│   │ WebSocket    │  │
│  │  GoogleMap   │    │ routePoints  │   │ (real-time)  │  │
│  │  DriverMarker│    │ eta          │   │ Directions   │  │
│  │  RoutePolyline│   │              │   │ API (route)  │  │
│  │  BottomSheet │    └──────────────┘    └──────────────┘  │
│  └──────────────┘                                          │
└────────────────────────────────────────────────────────────┘

Driver App:
  FusedLocationProvider → sends location every 3s → WebSocket → Server
  Server → broadcasts to rider's WebSocket → Rider App updates marker
```

### 5.2 State Model

```kotlin
data class RideUiState(
    val riderLocation: LatLng? = null,
    val driverLocation: LatLng? = null,
    val driverBearing: Float = 0f,      // direction driver is heading
    val pickupLocation: LatLng? = null,
    val destinationLocation: LatLng? = null,
    val routePoints: List<LatLng> = emptyList(),  // decoded polyline from Directions API
    val eta: String = "",               // "5 min away"
    val rideStatus: RideStatus = RideStatus.SEARCHING,
    val cameraTarget: LatLng? = null    // what camera should focus on
)

enum class RideStatus {
    SEARCHING,     // looking for driver
    DRIVER_FOUND,  // driver assigned, heading to pickup
    DRIVER_NEARBY, // driver < 500m from pickup
    IN_RIDE,       // rider in car, heading to destination
    COMPLETED
}
```

### 5.3 Fetching Route from Directions API

```kotlin
class DirectionsRepository @Inject constructor(
    private val api: DirectionsApi   // Retrofit interface
) {
    // Google Routes API (replacement for older Directions API)
    suspend fun getRoute(origin: LatLng, destination: LatLng): RouteResult {
        val response = api.computeRoutes(
            ComputeRoutesRequest(
                origin = Waypoint(LatLng(origin.latitude, origin.longitude)),
                destination = Waypoint(LatLng(destination.latitude, destination.longitude)),
                travelMode = "DRIVE",
                routingPreference = "TRAFFIC_AWARE",
                computeAlternativeRoutes = false,
                languageCode = "en-US",
                units = "METRIC"
            ),
            fieldMask = "routes.duration,routes.distanceMeters,routes.polyline,routes.legs"
        )

        val route = response.routes.firstOrNull() ?: throw Exception("No route found")

        // Decode the encoded polyline string into LatLng list
        val points = PolyUtil.decode(route.polyline.encodedPolyline)

        return RouteResult(
            points = points,
            distanceMeters = route.distanceMeters,
            durationSeconds = route.duration.removeSuffix("s").toInt(),
            etaText = formatEta(route.duration)
        )
    }
}

// OR: Use the legacy Directions API directly
interface DirectionsApi {
    @GET("maps/api/directions/json")
    suspend fun getDirections(
        @Query("origin") origin: String,         // "lat,lng"
        @Query("destination") destination: String,
        @Query("mode") mode: String = "driving",
        @Query("traffic_model") trafficModel: String = "best_guess",
        @Query("departure_time") departureTime: String = "now",
        @Query("key") apiKey: String = BuildConfig.MAPS_API_KEY
    ): DirectionsResponse
}
```

### 5.4 Drawing the Route on Map

```kotlin
@Composable
fun RideMapScreen(viewModel: RideViewModel = hiltViewModel()) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    val cameraPositionState = rememberCameraPositionState()

    // Move camera when target changes
    LaunchedEffect(state.cameraTarget) {
        state.cameraTarget?.let { target ->
            cameraPositionState.animate(
                CameraUpdateFactory.newLatLngZoom(target, 15f),
                durationMs = 500
            )
        }
    }

    // Fit route in camera when route is available
    LaunchedEffect(state.routePoints) {
        if (state.routePoints.size >= 2) {
            val bounds = state.routePoints.fold(LatLngBounds.Builder()) { builder, point ->
                builder.include(point)
            }.build()
            cameraPositionState.animate(
                CameraUpdateFactory.newLatLngBounds(bounds, 100 /* padding dp */),
                durationMs = 800
            )
        }
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        // Route polyline
        if (state.routePoints.isNotEmpty()) {
            Polyline(
                points = state.routePoints,
                color = Color(0xFF1976D2),   // Material Blue
                width = 12f,
                jointType = JointType.ROUND,
                startCap = RoundCap(),
                endCap = RoundCap()
            )
        }

        // Pickup marker
        state.pickupLocation?.let { pickup ->
            Marker(
                state = MarkerState(position = pickup),
                icon = BitmapDescriptorFactory.fromResource(R.drawable.ic_pickup),
                title = "Pickup"
            )
        }

        // Destination marker
        state.destinationLocation?.let { dest ->
            Marker(
                state = MarkerState(position = dest),
                icon = BitmapDescriptorFactory.fromResource(R.drawable.ic_destination),
                title = "Destination"
            )
        }

        // Driver marker (animated — see §5.5)
        state.driverLocation?.let { driverPos ->
            AnimatedDriverMarker(
                position = driverPos,
                bearing = state.driverBearing
            )
        }

        // Rider's location
        state.riderLocation?.let { riderPos ->
            Circle(
                center = riderPos,
                radius = 30.0,           // 30 meters
                fillColor = Color(0x221976D2),
                strokeColor = Color(0xFF1976D2),
                strokeWidth = 2f
            )
        }
    }
}
```

### 5.5 Smooth Driver Marker Animation (Uber-Style)

The key to smooth car movement: interpolate between GPS updates using `ValueAnimator`.

```kotlin
@Composable
fun AnimatedDriverMarker(position: LatLng, bearing: Float) {
    // Previous position for animation
    var previousPosition by remember { mutableStateOf(position) }
    val animatedPosition = remember { Animatable(position, LatLngTwoWayConverter) }
    val animatedBearing = remember { Animatable(bearing) }

    LaunchedEffect(position) {
        // Animate smoothly from previous to new GPS position
        launch {
            animatedPosition.animateTo(
                position,
                animationSpec = tween(
                    durationMillis = 2000,    // 2s animation = between 3s GPS updates
                    easing = LinearEasing
                )
            )
        }

        // Rotate marker toward new bearing
        launch {
            // Handle wrap-around (e.g., 350° → 10° should go through 0°, not 340°)
            val currentBearing = animatedBearing.value
            val delta = ((bearing - currentBearing + 540) % 360) - 180
            animatedBearing.animateTo(
                currentBearing + delta,
                animationSpec = tween(durationMillis = 500)
            )
        }
        previousPosition = position
    }

    // Car icon that rotates with bearing
    val currentPosition = animatedPosition.value
    val currentBearing = animatedBearing.value

    Marker(
        state = MarkerState(position = currentPosition),
        icon = createRotatedCarBitmap(currentBearing),
        flat = true,       // lies flat on map (rotates with bearing)
        anchor = Offset(0.5f, 0.5f),  // center of icon
        zIndex = 1f        // draw above other markers
    )
}

// Create rotated bitmap for car icon
fun createRotatedCarBitmap(bearing: Float): BitmapDescriptor {
    val bitmap = BitmapFactory.decodeResource(resources, R.drawable.ic_car)
    val matrix = Matrix().apply { postRotate(bearing) }
    val rotated = Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
    return BitmapDescriptorFactory.fromBitmap(rotated)
}

// Custom LatLng Animatable converter
val LatLngTwoWayConverter = TwoWayConverter<LatLng, AnimationVector2D>(
    convertToVector = { AnimationVector2D(it.latitude.toFloat(), it.longitude.toFloat()) },
    convertFromVector = { LatLng(it.v1.toDouble(), it.v2.toDouble()) }
)
```

### 5.6 Real-Time Driver Location via WebSocket

```kotlin
class RideViewModel @Inject constructor(
    private val locationRepo: LocationRepository,
    private val directionsRepo: DirectionsRepository,
    private val wsManager: WebSocketManager
) : ViewModel() {

    private val _uiState = MutableStateFlow(RideUiState())
    val uiState = _uiState.asStateFlow()

    fun startRide(rideId: String) {
        // Subscribe to driver location updates via WebSocket
        viewModelScope.launch {
            wsManager.messages
                .filter { it.type == "driver_location" }
                .collect { message ->
                    val newDriverPos = LatLng(message.lat, message.lng)
                    val previousPos = _uiState.value.driverLocation

                    // Calculate bearing from previous to new position
                    val bearing = if (previousPos != null) {
                        SphericalUtil.computeHeading(previousPos, newDriverPos).toFloat()
                    } else 0f

                    _uiState.update { state ->
                        state.copy(
                            driverLocation = newDriverPos,
                            driverBearing = bearing,
                            eta = message.etaText
                        )
                    }

                    // Auto-pan camera to driver when status is DRIVER_FOUND
                    if (_uiState.value.rideStatus == RideStatus.DRIVER_FOUND) {
                        _uiState.update { it.copy(cameraTarget = newDriverPos) }
                    }
                }
        }
    }
}
```

### 5.7 Driver App: Sending Location

```kotlin
// Driver's foreground service — keeps GPS active while driving
class DriverLocationService : Service() {
    private val locationRepo: LocationRepository by inject()
    private val wsManager: WebSocketManager by inject()

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(NOTIF_ID, buildNotification())  // required to prevent OS kill

        // Send location every 3 seconds
        serviceScope.launch {
            locationRepo.getLocationUpdates(intervalMs = 3000)
                .collect { location ->
                    wsManager.send(DriverLocationMessage(
                        lat = location.latitude,
                        lng = location.longitude,
                        timestamp = System.currentTimeMillis()
                    ))
                }
        }
        return START_STICKY
    }
}
```

---

## 6. Places API — Location Search & Autocomplete

```kotlin
// Initialize Places SDK
Places.initialize(applicationContext, BuildConfig.MAPS_API_KEY)
val placesClient = Places.createClient(context)

// Autocomplete as user types
suspend fun searchPlaces(query: String): List<AutocompletePrediction> {
    val request = FindAutocompletePredictionsRequest.builder()
        .setQuery(query)
        .setCountries("US", "CA")    // restrict to countries
        .setTypesFilter(listOf("geocode", "establishment"))
        .build()

    return suspendCancellableCoroutine { continuation ->
        placesClient.findAutocompletePredictions(request)
            .addOnSuccessListener { response ->
                continuation.resume(response.autocompletePredictions)
            }
            .addOnFailureListener { continuation.resume(emptyList()) }
    }
}

// Get place details from a prediction
suspend fun getPlaceDetails(placeId: String): Place {
    val fields = listOf(
        Place.Field.ID, Place.Field.NAME,
        Place.Field.LAT_LNG, Place.Field.ADDRESS
    )
    val request = FetchPlaceRequest.newInstance(placeId, fields)
    return suspendCancellableCoroutine { continuation ->
        placesClient.fetchPlace(request)
            .addOnSuccessListener { continuation.resume(it.place) }
            .addOnFailureListener { continuation.resumeWithException(it) }
    }
}

// In Compose — autocomplete search field
@Composable
fun PlaceSearchField(onPlaceSelected: (Place) -> Unit) {
    var query by remember { mutableStateOf("") }
    var predictions by remember { mutableStateOf<List<AutocompletePrediction>>(emptyList()) }
    val scope = rememberCoroutineScope()

    LaunchedEffect(query) {
        if (query.length >= 3) {
            delay(300)  // debounce
            predictions = placesRepo.searchPlaces(query)
        }
    }

    Column {
        OutlinedTextField(
            value = query,
            onValueChange = { query = it },
            placeholder = { Text("Where to?") },
            leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) }
        )
        predictions.forEach { prediction ->
            ListItem(
                headlineContent = { Text(prediction.getPrimaryText(null).toString()) },
                supportingContent = { Text(prediction.getSecondaryText(null).toString()) },
                modifier = Modifier.clickable {
                    scope.launch {
                        val place = placesRepo.getPlaceDetails(prediction.placeId ?: return@launch)
                        onPlaceSelected(place)
                        predictions = emptyList()
                    }
                }
            )
        }
    }
}
```

---

## 7. Geofencing

Geofencing triggers an event when the device enters or exits a geographic boundary — used to detect "driver arrived at pickup", "driver arrived at destination", "delivery at customer's door".

```kotlin
class GeofencingRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val geofencingClient = LocationServices.getGeofencingClient(context)

    @SuppressLint("MissingPermission")
    fun addGeofence(
        id: String,
        center: LatLng,
        radiusMeters: Float = 100f,
        onEnter: Boolean = true,
        onExit: Boolean = false
    ) {
        val geofence = Geofence.Builder()
            .setRequestId(id)
            .setCircularRegion(center.latitude, center.longitude, radiusMeters)
            .setExpirationDuration(Geofence.NEVER_EXPIRE)
            .setTransitionTypes(
                (if (onEnter) Geofence.GEOFENCE_TRANSITION_ENTER else 0) or
                (if (onExit) Geofence.GEOFENCE_TRANSITION_EXIT else 0)
            )
            .setLoiteringDelay(30_000)  // must be inside for 30s before DWELL triggers
            .build()

        val request = GeofencingRequest.Builder()
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_ENTER)
            .addGeofence(geofence)
            .build()

        val pendingIntent = PendingIntent.getBroadcast(
            context, 0,
            Intent(context, GeofenceBroadcastReceiver::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_MUTABLE
        )

        geofencingClient.addGeofences(request, pendingIntent)
    }

    fun removeGeofence(id: String) {
        geofencingClient.removeGeofences(listOf(id))
    }
}

// Receive geofence events
class GeofenceBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val event = GeofencingEvent.fromIntent(intent) ?: return
        if (event.hasError()) return

        val transition = event.geofenceTransition
        val triggeredGeofences = event.triggeringGeofences ?: return

        triggeredGeofences.forEach { geofence ->
            when (transition) {
                Geofence.GEOFENCE_TRANSITION_ENTER -> {
                    // Driver entered pickup/delivery zone
                    notifyDriverArrived(context, geofence.requestId)
                }
                Geofence.GEOFENCE_TRANSITION_EXIT -> {
                    // Driver left the zone
                }
            }
        }
    }
}

// Ride-sharing use case:
// 1. When driver is assigned → add geofence at pickup location (100m radius)
// 2. When geofence ENTER event → notify rider "Driver has arrived"
// 3. Show "Meet your driver" UI
// 4. Remove geofence after pickup confirmation
```

---

## 8. HERE Maps SDK (Alternative)

```kotlin
// gradle
implementation("com.here.sdk:mapview-android-lite:4.19.1")

// Setup (in Application.onCreate)
SDKNativeEngine.makeSharedInstance(
    SDKOptions.withAccessKeySecret(
        accessKeyId = BuildConfig.HERE_ACCESS_KEY_ID,
        accessKeySecret = BuildConfig.HERE_ACCESS_KEY_SECRET
    )
)

// In Compose — HERE doesn't have native Compose support; use AndroidView
@Composable
fun HereMapView() {
    val mapView = remember { MapView(context) }
    AndroidView(factory = { mapView }) {
        it.onCreate(Bundle())
    }
    DisposableEffect(Unit) {
        mapView.onResume()
        onDispose { mapView.onPause(); mapView.onDestroy() }
    }
}

// HERE Routing
val routingEngine = RoutingEngine()
routingEngine.calculateRoute(
    listOf(Waypoint(GeoCoordinates(origin.latitude, origin.longitude)),
           Waypoint(GeoCoordinates(dest.latitude, dest.longitude))),
    CarOptions()
) { routingError, routes ->
    routes?.firstOrNull()?.let { route ->
        val polyline = route.geometry  // GeoPolyline
        val etaSecs = route.duration.toSeconds()
        val distanceMeters = route.lengthInMeters
    }
}
```

**Why HERE over Google Maps for logistics:**
- Truck routing with height/weight/hazmat restrictions
- ISO country-level routing restrictions
- Superior live traffic for fleet management
- Predictable enterprise pricing
- EU-hosted data centers

---

## 9. Mapbox SDK

```kotlin
// gradle
implementation("com.mapbox.maps:android:11.7.0")

// In Compose
@Composable
fun MapboxMapScreen() {
    val mapInitOptions = MapInitOptions(
        context = LocalContext.current,
        styleUri = Style.MAPBOX_STREETS
    )

    AndroidView(
        factory = { context ->
            MapView(context, mapInitOptions).also { mapView ->
                mapView.mapboxMap.loadStyle(Style.MAPBOX_STREETS) { style ->
                    // Add custom layers, sources
                }
            }
        }
    )
}

// Mapbox Directions API
val directionsClient = MapboxDirections.builder()
    .origin(Point.fromLngLat(origin.longitude, origin.latitude))
    .destination(Point.fromLngLat(dest.longitude, dest.latitude))
    .profile(DirectionsCriteria.PROFILE_DRIVING_TRAFFIC)  // uses live traffic
    .accessToken(BuildConfig.MAPBOX_ACCESS_TOKEN)
    .build()

directionsClient.enqueueCall(object : Callback<DirectionsResponse> {
    override fun onResponse(call: Call<DirectionsResponse>, response: Response<DirectionsResponse>) {
        val route = response.body()?.routes()?.firstOrNull() ?: return
        val geometry = route.geometry()  // encoded polyline or LineString
        val distance = route.distance()
        val duration = route.duration()
    }
    override fun onFailure(call: Call<DirectionsResponse>, t: Throwable) { }
})
```

**Mapbox strengths:** Highly customizable map styles (Studio), smooth animations, Navigation SDK with voice guidance, offline map downloads.

---

## 10. OpenStreetMap + MapLibre (Free Option)

```kotlin
// gradle
implementation("org.maplibre.gl:android-sdk:11.8.3")

// Simple map
@Composable
fun OpenStreetMapView() {
    AndroidView(
        factory = { context ->
            MapView(context).apply {
                getMapAsync { map ->
                    map.setStyle(Style.Builder().fromUri("https://demotiles.maplibre.org/style.json"))
                    map.cameraPosition = CameraPosition.Builder()
                        .target(LatLng(48.8566, 2.3522))  // Paris
                        .zoom(12.0)
                        .build()
                }
            }
        }
    )
}
// Tile providers: MapTiler, Stadia Maps, OpenFreeMap — all support MapLibre style format
```

---

## 11. Battery Efficiency for Location Tracking

```kotlin
// Adaptive location strategy based on context
class AdaptiveLocationStrategy {

    fun getRequest(scenario: Scenario): LocationRequest {
        return when (scenario) {
            // Driver app: high accuracy, frequent updates (user is navigating)
            Scenario.ACTIVE_RIDE -> LocationRequest.Builder(
                Priority.PRIORITY_HIGH_ACCURACY, 3000L
            ).setMinUpdateIntervalMillis(1000).build()

            // Rider watching driver approach: balanced accuracy
            Scenario.WATCHING_DRIVER -> LocationRequest.Builder(
                Priority.PRIORITY_BALANCED_POWER, 5000L
            ).build()

            // Background: minimal updates (not in ride)
            Scenario.BACKGROUND -> LocationRequest.Builder(
                Priority.PRIORITY_LOW_POWER, 60_000L
            ).build()
        }
    }
}

// Tips for battery efficiency:
// 1. Use PRIORITY_BALANCED_POWER for rider watching driver — cell/WiFi is enough
// 2. Remove location updates when app is backgrounded (unless driver foreground service)
// 3. Use geofencing instead of polling for "driver arrived" detection
// 4. Increase update interval as driver gets closer (10s → 5s → 3s when < 500m)
// 5. Stop requesting location when ride is completed
```

---

## 12. HLD — Ride-Sharing Real-Time Location Architecture

```
Rider App:
  Google Maps Compose → shows driver marker + route
  WebSocket (receives driver location updates from server)
  Smooth marker animation between GPS updates

Driver App:
  FusedLocationProvider (3s interval, HIGH_ACCURACY, foreground service)
  WebSocket → sends DriverLocationMessage to server
  Geofence at pickup → triggers "Ready to pick up" state

Server (not mobile, but relevant for interview):
  WebSocket gateway → broadcasts driver location to relevant riders
  Fleet tracking DB: Redis GEOADD + GEORADIUS for nearby driver queries
  Routes API proxy → server calls Google Routes API, caches results
  ETA calculation: cached route + current position = remaining time

Key design decisions:
  Location update frequency: 3s for active ride, 10s for searching
  Smoothing: marker interpolation (2s animation between 3s updates)
  Accuracy vs battery: HIGH_ACCURACY during ride, LOW_POWER between rides
  Geofencing: server-side + client-side for pickup/dropoff detection
  Map provider: Google Maps (data quality), or HERE (fleet/enterprise)
```

---

## 13. React Native

```typescript
// react-native-maps — the standard RN map library
// Supports Google Maps (Android) and Apple Maps + Google Maps (iOS)
import MapView, { Marker, Polyline, PROVIDER_GOOGLE } from 'react-native-maps'

// Basic map with driver and route
function RideMap({ driverLocation, routePoints, pickup, destination }) {
    return (
        <MapView
            provider={PROVIDER_GOOGLE}    // use Google Maps on both platforms
            style={{ flex: 1 }}
            initialRegion={{
                latitude: pickup.lat,
                longitude: pickup.lng,
                latitudeDelta: 0.05,
                longitudeDelta: 0.05,
            }}
            showsTraffic={true}
            showsUserLocation={true}
        >
            {/* Driver marker */}
            {driverLocation && (
                <Marker
                    coordinate={driverLocation}
                    image={require('./assets/car.png')}
                    rotation={driverLocation.bearing}
                    anchor={{ x: 0.5, y: 0.5 }}
                />
            )}

            {/* Route polyline */}
            <Polyline
                coordinates={routePoints}
                strokeColor="#1976D2"
                strokeWidth={4}
            />

            {/* Pickup and destination */}
            <Marker coordinate={pickup} pinColor="green" title="Pickup"/>
            <Marker coordinate={destination} pinColor="red" title="Destination"/>
        </MapView>
    )
}

// Smooth animation in React Native
import { Animated } from 'react-native'
// Use react-native-maps AnimatedRegion for smooth marker movement
const driverCoordinate = useRef(new Animated.Region({
    latitude: initialLat,
    longitude: initialLng,
    latitudeDelta: 0,
    longitudeDelta: 0,
}))

function updateDriverPosition(newLat, newLng) {
    Animated.timing(driverCoordinate.current, {
        toValue: { latitude: newLat, longitude: newLng },
        duration: 2000,
        useNativeDriver: false,   // AnimatedRegion doesn't support native driver
    }).start()
}
```

---

## 14. Common Misunderstandings & Pitfalls

**❌ Using `LocationManager` directly instead of Fused Location Provider**
`LocationManager` requires manual sensor management and has no battery optimization. Fused Location Provider automatically chooses the best source (GPS/WiFi/cell) for the required accuracy and reduces battery drain significantly.

**❌ Requesting `ACCESS_FINE_LOCATION` when `ACCESS_COARSE_LOCATION` is sufficient**
Rider watching a driver approach doesn't need GPS-level accuracy — city-block precision from WiFi/cell is fine. Only request fine location for driver navigation and user's own pickup pin placement.

**❌ Jumping markers between GPS updates instead of animating**
Raw GPS updates every 3 seconds make the car marker teleport. Users expect smooth movement. Animate between positions over the update interval using `ValueAnimator` or `Animatable`.

**❌ Not handling the wrap-around case in bearing animation**
Rotating from 350° to 10° should go through 0° (10° turn), not through 180° (350° turn). Always calculate delta as `((newBearing - oldBearing + 540) % 360) - 180`.

**❌ Hardcoding the API key in source code**
API keys in `strings.xml` or source files are extractable from APK. Store in `local.properties` (not committed), inject via CI/CD for release builds. Restrict key to your package + signing cert in Google Cloud Console.

**❌ Not removing location updates when not needed**
Forgetting to call `fusedLocationClient.removeLocationUpdates(callback)` leaks the callback and keeps GPS running after the feature is done. Always remove in `onPause` or when collecting from a Flow with `awaitClose`.

**❌ Geofencing for real-time "driver arrived" without server-side validation**
Client-side geofencing can be spoofed (fake GPS apps). For payment triggers or critical events, validate geofence events server-side using the driver's GPS coordinates from your backend.

**❌ Using `lastLocation` instead of a fresh location request**
`fusedLocationClient.lastLocation` may return a cached location minutes old or null if the device hasn't recently requested location. For pickup scenarios where precision matters, use `getCurrentLocation()` with a timeout.

---

## 15. Best Practices

- **Fused Location Provider only** — never `LocationManager` in new code
- **Match priority to use case** — driver navigation: `HIGH_ACCURACY`; rider watching: `BALANCED_POWER`
- **Animate between GPS updates** — 2s animation between 3s updates = smooth car movement
- **Request only needed permissions** — `COARSE` for watching, `FINE` for placing pin or driver navigation
- **Restrict API key** — package name + SHA-256 cert restriction in Google Cloud Console
- **Geofencing for arrival detection** — more battery-efficient than polling location every second
- **Server-side route caching** — call Directions API from server, cache result; don't call from every client
- **Adaptive update intervals** — increase frequency as driver approaches: 10s (far) → 5s (nearby) → 3s (arriving)
- **Stop updates on completion** — remove location callbacks when ride ends
- **Fit camera to route** — `LatLngBounds` with padding shows full route on screen
- **Custom marker bitmap** — rotate car icon to match bearing; use `flat = true` for ground-level markers
- **Test on real device** — GPS, location permissions, and map rendering behave differently than emulator

---

## 16. Interview Q&A

**Q1: Which map SDK would you choose for a ride-sharing app and why?**

> Google Maps Platform for most consumer ride-sharing apps — it has the best global map data quality, the most accurate traffic data, excellent Places Autocomplete for destination search, and users are familiar with the UI. The Fleet Engine product specifically targets ride-hailing with real-time vehicle tracking built in. The main concern is cost at scale: with 1M+ map loads per month, the bill becomes significant, so I'd build a caching layer for route data and use conditional tile loading. For a B2B logistics or fleet management product, I'd consider HERE Maps — better truck routing with vehicle restrictions, EU data residency, and more predictable enterprise pricing. For a cost-sensitive startup or one with EU GDPR requirements needing full data control, OpenStreetMap with MapLibre and a commercial tile provider like MapTiler is worth evaluating.

---

**Q2: How do you animate a car marker smoothly on the map (Uber-style)?**

> GPS updates arrive every 3 seconds. Without animation, the marker jumps every 3 seconds — jarring UX. The solution: interpolate position between updates using animation. When a new GPS coordinate arrives, start a `ValueAnimator` (or `Animatable` in Compose) that runs over the same interval as GPS updates (3 seconds), smoothly moving the marker from the previous position to the new one. By the time the animation completes, the next GPS update arrives and starts a new animation — creating the illusion of continuous smooth movement. For bearing (car direction), calculate `SphericalUtil.computeHeading(previousPos, newPos)` and animate the rotation separately, handling the 350°→10° wrap-around by always rotating through the shortest arc.

---

**Q3: How does the Fused Location Provider differ from using GPS directly?**

> The Fused Location Provider (FLP) is a Google Play Services API that intelligently selects the best location source based on the accuracy requirements and power budget. When you request `PRIORITY_HIGH_ACCURACY`, FLP uses GPS when available and falls back to WiFi/cell if GPS signal is weak. When you request `PRIORITY_BALANCED_POWER`, it uses WiFi and cell towers (city-block accuracy) without activating the GPS chip at all — dramatically reducing battery drain. It also batches location updates from multiple apps, so if another app is already requesting GPS, FLP can piggyback on that signal for free. Using `LocationManager` directly means manually managing these trade-offs. FLP handles them optimally and is the Android-recommended approach for all location use cases.

---

**Q4: How would you design geofencing for "driver arrived at pickup"?**

> Two layers: client-side and server-side. Client-side: when the driver is assigned, the rider app adds a geofence at the pickup location (100m radius) using the Android Geofencing API. When the driver's device crosses into the geofence, a `BroadcastReceiver` fires. The rider app shows "Your driver has arrived" notification. Server-side: the driver app sends location every 3 seconds. The server calculates distance between driver's current GPS and pickup location using the Haversine formula. When distance < 100m, the server sends a "driver_arrived" message via WebSocket to the rider. Server-side validation is important for payment triggers — geofencing on the client can be spoofed with fake GPS apps, so any financial event (trip start, trip end) should be validated server-side.

---

**Q5: How do you draw a route between pickup and destination?**

> Call the Google Directions API (or newer Routes API) from the server — not from the client directly. The server calls the API, caches the response (route doesn't change unless there's a major traffic event), and sends the encoded polyline to the client. The client decodes the polyline string into a list of `LatLng` points using `PolyUtil.decode()` from the Maps Android Utils library, then renders it with a `Polyline` composable in `GoogleMap`. I always adjust the camera to fit the route using `LatLngBounds.Builder().include(point)` for all route points, then `CameraUpdateFactory.newLatLngBounds(bounds, padding)` to show the full route with padding. The route is updated if the driver deviates significantly — detected server-side by comparing driver's GPS to nearest point on route.

---

**Q6: How do you handle battery drain from continuous GPS tracking in a driver app?**

> Four strategies. First, use the right priority: `PRIORITY_HIGH_ACCURACY` only while actively navigating a trip; switch to `PRIORITY_BALANCED_POWER` (WiFi/cell) while waiting for a ride request — city-block accuracy is enough for showing driver availability. Second, adaptive interval: 10s while far from destination, 5s when nearby, 3s when arriving — no need for frequent updates when nothing is changing. Third, foreground service: the driver app must use a foreground service with a persistent notification to prevent the OS from killing location updates; background location without foreground service is heavily throttled on Android 10+. Fourth, geofencing for events: instead of polling "has driver reached pickup?" every second, add a geofence at pickup — triggered automatically by the OS when the device enters, with no polling cost.

---

*Previous: [13 — Cross-Cutting Concerns](./13-cross-cutting.md)*
*Standalone section added to reference guide.*
