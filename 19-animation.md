# 19 — Animation

> **Round type: LLD (primary)**
> **Focus:** Compose animation API landscape, which API for which use case, performance, Shared Element Transitions, gesture-driven animations, React Native Animated API and Reanimated 3

---

## 1. Why This Matters in Interviews

Animation questions test whether you understand Compose's rendering pipeline — specifically which phase (composition, layout, drawing) an animation runs in, and how to keep animations on the RenderThread. A well-placed `graphicsLayer` can make a scroll-driven animation 10x faster.

**Common interview angles:**
- "Which Compose animation API would you use and why?"
- "What's the difference between `animate*AsState` and `Animatable`?"
- "How do you run an animation on the drawing phase to avoid recomposition?"
- "What is a Shared Element Transition and how does it work in Compose?"
- "What's the difference between `spring` and `tween` animation specs?"
- "How does Reanimated 3 differ from the React Native Animated API?"

---

## 2. Compose Animation API — Decision Tree

```
What are you animating?
│
├── Visibility (show/hide a composable)?
│     └── AnimatedVisibility (with enter/exit transitions)
│
├── Switching between different content?
│     └── AnimatedContent (with ContentTransform)
│
├── A single value (color, size, alpha, offset)?
│     └── animate*AsState (animateFloatAsState, animateDpAsState, etc.)
│
├── Multiple coordinated values driven by state?
│     └── updateTransition → transition.animate*()
│
├── Repeating/infinite animation?
│     └── rememberInfiniteTransition → infiniteTransition.animateFloat()
│
├── Gesture-driven or programmatic (not state-driven)?
│     └── Animatable (coroutine-based, manual control)
│
├── Shared element between screens (hero animation)?
│     └── SharedTransitionLayout + SharedTransitionScope
│
└── Simple size change?
      └── Modifier.animateContentSize()
```

---

## 3. Animation Specs

All animations in Compose are configured with an `AnimationSpec`. Choose based on the feel you want:

### 3.1 `tween` — Time-based, eased

```kotlin
// Linear from A to B in specified duration with easing
tween(
    durationMillis = 300,
    delayMillis = 0,
    easing = FastOutSlowInEasing   // Material Design standard
    // Other easings:
    // LinearEasing          — constant speed
    // FastOutLinearInEasing — fast start, constant end (for exits)
    // LinearOutSlowInEasing — constant start, slow end (for entrances)
    // CubicBezierEasing(x1, y1, x2, y2) — custom bezier
)
```

**When:** UI transitions (screen changes, card expansions), anything where precise duration matters.

### 3.2 `spring` — Physics-based, natural feel

```kotlin
// Behaves like a physical spring — may overshoot target (bounciness)
spring(
    dampingRatio = Spring.DampingRatioMediumBouncy,  // 0.5 — some bounce
    // Spring.DampingRatioNoBouncy    = 1.0  — critically damped (no overshoot)
    // Spring.DampingRatioLowBouncy   = 0.2  — very bouncy
    // Spring.DampingRatioHighBouncy  = 0.75 — slight bounce
    stiffness = Spring.StiffnessMedium  // how quickly it reaches target
    // Spring.StiffnessHigh   — snappy
    // Spring.StiffnessLow    — slow, floaty
    // Spring.StiffnessVeryLow— very slow, gentle
)
```

**When:** Interactive elements responding to user touch (FAB, cards), gestures that should feel physical, drag-and-drop. Spring has **no fixed duration** — it responds to velocity.

### 3.3 `keyframes` — Custom intermediate values

```kotlin
// Define specific values at specific time points
keyframes {
    durationMillis = 500
    0.0f at 0       // start at 0
    0.8f at 200     // overshoot to 0.8 at 200ms
    1.0f at 500     // settle at 1.0 at 500ms
    // Custom easing between keyframes:
    0.8f at 200 with FastOutSlowInEasing
}
```

**When:** Complex multi-phase animations (bounce, heartbeat, shake on error).

### 3.4 `repeatable` and `infiniteRepeatable`

```kotlin
// Finite repetitions
repeatable(
    iterations = 3,
    animation = tween(300),
    repeatMode = RepeatMode.Reverse  // or RepeatMode.Restart
)

// Infinite repetitions (for loading spinners, pulsing effects)
infiniteRepeatable(
    animation = tween(1000, easing = LinearEasing),
    repeatMode = RepeatMode.Reverse
)
```

---

## 4. `animate*AsState` — Single Value Animation

The simplest API. Animate a single value whenever its target changes.

```kotlin
// Color animation
var selected by remember { mutableStateOf(false) }
val backgroundColor by animateColorAsState(
    targetValue = if (selected) MaterialTheme.colorScheme.primary else MaterialTheme.colorScheme.surface,
    animationSpec = tween(300),
    label = "card background"  // label for Android Studio animation inspector
)

// Size animation
val size by animateDpAsState(
    targetValue = if (selected) 200.dp else 100.dp,
    animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
    label = "card size"
)

// Float (alpha, scale)
val alpha by animateFloatAsState(
    targetValue = if (visible) 1f else 0f,
    animationSpec = tween(200),
    label = "visibility alpha"
)

// Offset (position)
val offsetX by animateIntOffsetAsState(
    targetValue = if (moved) IntOffset(100, 0) else IntOffset(0, 0),
    label = "position"
)

// Usage
Box(modifier = Modifier
    .size(size)
    .background(backgroundColor)
    .alpha(alpha)
    .offset { offsetX }
)
```

**`animate*AsState` variants:** `animateFloatAsState`, `animateDpAsState`, `animateIntAsState`, `animateIntOffsetAsState`, `animateSizeAsState`, `animateColorAsState`, `animateRectAsState`, `animateOffsetAsState`.

**Performance note:** `animate*AsState` drives recomposition — the composable re-runs on every animation frame. For scroll-driven or frequent animations, use `graphicsLayer` instead (§9).

---

## 5. `AnimatedVisibility` — Show/Hide Composables

```kotlin
var visible by remember { mutableStateOf(true) }

AnimatedVisibility(
    visible = visible,
    enter = fadeIn(animationSpec = tween(300)) +
            slideInVertically(initialOffsetY = { -it }),  // slides from top
    exit = fadeOut(animationSpec = tween(300)) +
           slideOutVertically(targetOffsetY = { -it })
) {
    Card { Text("I animate in and out!") }
}

// Enter/exit combinations:
// fadeIn() / fadeOut()
// slideInHorizontally { fullWidth } / slideOutHorizontally { -fullWidth }
// slideInVertically { -fullHeight } / slideOutVertically { fullHeight }
// expandVertically() / shrinkVertically()
// expandHorizontally() / shrinkHorizontally()
// expandIn() / shrinkOut()
// scaleIn() / scaleOut()
```

**Key behavior:** `AnimatedVisibility` keeps the composable in the composition during the exit animation — it only removes it when the exit animation completes. This means:
- Exit animations actually play (unlike `alpha = 0f` which hides but doesn't remove)
- The composable occupies layout space until fully gone

### 5.1 Per-Child Enter/Exit with `animateEnterExit`

```kotlin
AnimatedVisibility(visible = visible) {
    Row {
        Icon(
            Icons.Default.Check,
            contentDescription = null,
            modifier = Modifier.animateEnterExit(  // child has its OWN animation
                enter = scaleIn(),
                exit = scaleOut()
            )
        )
        Text("Saved!")  // uses parent AnimatedVisibility enter/exit
    }
}
```

---

## 6. `AnimatedContent` — Animate Between Different Content

```kotlin
var count by remember { mutableStateOf(0) }

AnimatedContent(
    targetState = count,
    transitionSpec = {
        // Determine direction based on count change
        if (targetState > initialState) {
            // Count increasing → slide up new, slide out old
            slideInVertically { it } + fadeIn() togetherWith
            slideOutVertically { -it } + fadeOut()
        } else {
            slideInVertically { -it } + fadeIn() togetherWith
            slideOutVertically { it } + fadeOut()
        }.using(SizeTransform(clip = false))
    },
    label = "count animation"
) { targetCount ->
    Text("Count: $targetCount")
}

// Crossfade — simple version of AnimatedContent
Crossfade(
    targetState = currentScreen,
    animationSpec = tween(300),
    label = "screen crossfade"
) { screen ->
    when (screen) {
        Screen.Home -> HomeScreen()
        Screen.Profile -> ProfileScreen()
    }
}
```

---

## 7. `Modifier.animateContentSize()` — Animate Size Changes

```kotlin
// Automatically animates when content size changes
var expanded by remember { mutableStateOf(false) }

Card(
    modifier = Modifier
        .animateContentSize(animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy))
        .clickable { expanded = !expanded }
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Title", style = MaterialTheme.typography.titleMedium)
        if (expanded) {
            Text("This is the expanded content that animates in smoothly when the card expands.")
        }
    }
}
```

**`animateContentSize` placement matters:** It must come BEFORE any size-related modifiers. Placing it after `padding` but before `fillMaxWidth` may cause unexpected behavior.

---

## 8. `updateTransition` — Multiple Coordinated Values

When multiple properties animate together in response to the same state change:

```kotlin
enum class CardState { Collapsed, Expanded }

@Composable
fun AnimatedCard() {
    var state by remember { mutableStateOf(CardState.Collapsed) }
    val transition = updateTransition(targetState = state, label = "card state")

    // All these values animate simultaneously, driven by the same state
    val cardHeight by transition.animateDp(label = "height") { cardState ->
        when (cardState) {
            CardState.Collapsed -> 56.dp
            CardState.Expanded -> 200.dp
        }
    }
    val cornerRadius by transition.animateDp(label = "corner radius") { cardState ->
        when (cardState) {
            CardState.Collapsed -> 28.dp  // pill shape
            CardState.Expanded -> 12.dp   // card shape
        }
    }
    val contentAlpha by transition.animateFloat(label = "content alpha") { cardState ->
        when (cardState) {
            CardState.Collapsed -> 0f
            CardState.Expanded -> 1f
        }
    }
    val backgroundColor by transition.animateColor(label = "background") { cardState ->
        when (cardState) {
            CardState.Collapsed -> MaterialTheme.colorScheme.primaryContainer
            CardState.Expanded -> MaterialTheme.colorScheme.surface
        }
    }

    Card(
        modifier = Modifier
            .height(cardHeight)
            .clickable { state = if (state == CardState.Collapsed) CardState.Expanded else CardState.Collapsed },
        shape = RoundedCornerShape(cornerRadius),
        colors = CardDefaults.cardColors(containerColor = backgroundColor)
    ) {
        // Content with animated alpha
        Box(modifier = Modifier.alpha(contentAlpha)) {
            Text("Expanded content")
        }
    }

    // Transition state inspection (for debugging)
    // transition.isRunning — true while animating
    // transition.currentState — current state
    // transition.targetState — target state
}
```

### 8.1 `AnimatedVisibility` as Transition extension

```kotlin
// AnimatedVisibility can be a child of updateTransition
// — ensures the visibility animation is coordinated with other transition animations
val transition = updateTransition(expanded, label = "content")

transition.AnimatedVisibility(
    visible = { it },
    enter = expandVertically(),
    exit = shrinkVertically()
) {
    Text("This is coordinated with the parent transition")
}
```

---

## 9. `rememberInfiniteTransition` — Repeating Animations

```kotlin
// Pulsing loading indicator
@Composable
fun PulsingDot() {
    val infiniteTransition = rememberInfiniteTransition(label = "pulsing")
    val scale by infiniteTransition.animateFloat(
        initialValue = 0.8f,
        targetValue = 1.2f,
        animationSpec = infiniteRepeatable(
            animation = tween(600, easing = FastOutSlowInEasing),
            repeatMode = RepeatMode.Reverse
        ),
        label = "scale"
    )
    val alpha by infiniteTransition.animateFloat(
        initialValue = 0.4f,
        targetValue = 1.0f,
        animationSpec = infiniteRepeatable(
            animation = tween(600),
            repeatMode = RepeatMode.Reverse
        ),
        label = "alpha"
    )

    Box(
        modifier = Modifier
            .size(20.dp)
            .graphicsLayer(scaleX = scale, scaleY = scale, alpha = alpha)
            .background(MaterialTheme.colorScheme.primary, CircleShape)
    )
}

// Rotating loading spinner
@Composable
fun LoadingSpinner() {
    val infiniteTransition = rememberInfiniteTransition(label = "spinner")
    val rotation by infiniteTransition.animateFloat(
        initialValue = 0f,
        targetValue = 360f,
        animationSpec = infiniteRepeatable(
            animation = tween(1000, easing = LinearEasing),
            repeatMode = RepeatMode.Restart
        ),
        label = "rotation"
    )

    CircularProgressIndicator(
        modifier = Modifier.graphicsLayer { rotationZ = rotation }
    )
}
```

---

## 10. `Animatable` — Coroutine-Based Manual Control

Use when animation is driven by gestures, programmatic control, or needs to be interrupted and redirected:

```kotlin
@Composable
fun DraggableCard() {
    val offsetX = remember { Animatable(0f) }
    val coroutineScope = rememberCoroutineScope()

    Box(
        modifier = Modifier
            .offset { IntOffset(offsetX.value.roundToInt(), 0) }
            .pointerInput(Unit) {
                detectHorizontalDragGestures(
                    onDragEnd = {
                        coroutineScope.launch {
                            // Snap back if not dragged far enough
                            if (abs(offsetX.value) < 200f) {
                                offsetX.animateTo(
                                    targetValue = 0f,
                                    animationSpec = spring(
                                        dampingRatio = Spring.DampingRatioMediumBouncy,
                                        stiffness = Spring.StiffnessMedium
                                    )
                                )
                            } else {
                                // Fling off screen
                                offsetX.animateTo(
                                    targetValue = if (offsetX.value > 0) 1000f else -1000f,
                                    animationSpec = tween(300)
                                )
                                // Card dismissed — handle dismiss event
                            }
                        }
                    },
                    onHorizontalDrag = { _, dragAmount ->
                        coroutineScope.launch {
                            offsetX.snapTo(offsetX.value + dragAmount)  // instant (no animation)
                        }
                    }
                )
            }
    ) {
        Card { Text("Swipe me!") }
    }
}

// Key Animatable APIs:
// animateTo(target, animationSpec) — animate to target value
// snapTo(target) — instantly jump to target (no animation)
// animateDecay(velocity, decaySpec) — decelerate from velocity (fling)
// stop() — stop current animation, stay at current value
// value — current animated value (State<T>)
// velocity — current velocity of the animation
```

---

## 11. Shared Element Transitions

Compose 1.7+ introduced `SharedTransitionLayout` for hero animations between screens or composables.

```kotlin
// Shared element between a list item and detail screen
@Composable
fun SharedElementDemo() {
    SharedTransitionLayout {  // provides SharedTransitionScope
        var showDetail by remember { mutableStateOf(false) }

        AnimatedContent(targetState = showDetail, label = "screen") { isDetail ->
            if (!isDetail) {
                // List view
                ListItem(
                    leadingContent = {
                        AsyncImage(
                            model = "https://example.com/image.jpg",
                            contentDescription = null,
                            modifier = Modifier
                                .size(56.dp)
                                .sharedElement(          // marks this as a shared element
                                    rememberSharedContentState(key = "hero_image"),
                                    animatedVisibilityScope = this@AnimatedContent
                                )
                        )
                    },
                    headlineContent = { Text("Title") },
                    modifier = Modifier.clickable { showDetail = true }
                )
            } else {
                // Detail view — same key = shared animation
                Column {
                    AsyncImage(
                        model = "https://example.com/image.jpg",
                        contentDescription = null,
                        modifier = Modifier
                            .fillMaxWidth()
                            .height(300.dp)
                            .sharedElement(
                                rememberSharedContentState(key = "hero_image"),
                                animatedVisibilityScope = this@AnimatedContent
                            )
                    )
                    Text("Detail content")
                    Button(onClick = { showDetail = false }) { Text("Back") }
                }
            }
        }
    }
}

// With Jetpack Navigation (compose-navigation-animation)
// SharedTransitionLayout wraps NavHost
// Each destination receives AnimatedVisibilityScope from the transition
```

---

## 12. `graphicsLayer` — Performance-Critical Animations

`graphicsLayer` runs in the **drawing phase** — not in composition or layout. This means:
- No recomposition triggered on every frame
- Animation runs on the RenderThread (hardware accelerated)
- No layout recalculation

```kotlin
// ❌ Bad — triggers recomposition on every frame (runs on UI thread)
val alpha by animateFloatAsState(if (visible) 1f else 0f, label = "")
Box(modifier = Modifier.alpha(alpha))  // alpha read during composition → recomposes each frame

// ✅ Better — graphicsLayer reads alpha in drawing phase, no recomposition
val alpha = remember { Animatable(1f) }
Box(modifier = Modifier.graphicsLayer { this.alpha = alpha.value })

// ✅ Best for scroll-driven — pure drawing phase, zero composition cost
@Composable
fun ParallaxHeader(scrollState: ScrollState) {
    Box(modifier = Modifier
        .fillMaxWidth()
        .height(300.dp)
        .graphicsLayer {
            // This lambda runs in the drawing phase on every scroll
            // NOT during composition — zero recomposition
            translationY = scrollState.value * 0.5f  // parallax
            alpha = 1f - (scrollState.value / 600f).coerceIn(0f, 1f)  // fade on scroll
        }
    ) {
        AsyncImage(/* header image */)
    }
}
```

**`graphicsLayer` properties:**
```kotlin
Modifier.graphicsLayer {
    alpha = 0.5f              // transparency
    scaleX = 1.2f             // horizontal scale
    scaleY = 1.2f             // vertical scale
    rotationX = 15f           // 3D tilt around X axis
    rotationY = 15f           // 3D tilt around Y axis
    rotationZ = 45f           // 2D rotation
    translationX = 100f       // horizontal offset (pixels)
    translationY = 50f        // vertical offset (pixels)
    shadowElevation = 8f      // drop shadow
    shape = CircleShape        // clip shape
    clip = true                // clip to shape
    transformOrigin = TransformOrigin(0.5f, 0f)  // pivot point
    cameraDistance = 8f * density  // 3D perspective
}
```

---

## 13. Gesture-Driven Animation

```kotlin
// Swipe to dismiss card stack (Tinder-style)
@Composable
fun SwipeableCard(onDismiss: (direction: DismissDirection) -> Unit) {
    val offsetX = remember { Animatable(0f) }
    val offsetY = remember { Animatable(0f) }
    val rotation = remember { Animatable(0f) }
    val scope = rememberCoroutineScope()

    Box(
        modifier = Modifier
            .graphicsLayer {
                translationX = offsetX.value
                translationY = offsetY.value
                rotationZ = rotation.value
            }
            .pointerInput(Unit) {
                detectDragGestures(
                    onDrag = { _, dragAmount ->
                        scope.launch {
                            offsetX.snapTo(offsetX.value + dragAmount.x)
                            offsetY.snapTo(offsetY.value + dragAmount.y)
                            // Rotate card as it moves sideways (natural feel)
                            rotation.snapTo(offsetX.value / 30f)
                        }
                    },
                    onDragEnd = {
                        val threshold = 300f
                        scope.launch {
                            when {
                                offsetX.value > threshold -> {
                                    // Fling right
                                    launch { offsetX.animateTo(1000f, tween(300)) }
                                    launch { rotation.animateTo(30f, tween(300)) }
                                    awaitAll()
                                    onDismiss(DismissDirection.Right)
                                }
                                offsetX.value < -threshold -> {
                                    launch { offsetX.animateTo(-1000f, tween(300)) }
                                    launch { rotation.animateTo(-30f, tween(300)) }
                                    awaitAll()
                                    onDismiss(DismissDirection.Left)
                                }
                                else -> {
                                    // Spring back
                                    launch { offsetX.animateTo(0f, spring(Spring.DampingRatioMediumBouncy)) }
                                    launch { offsetY.animateTo(0f, spring(Spring.DampingRatioMediumBouncy)) }
                                    launch { rotation.animateTo(0f, spring(Spring.DampingRatioMediumBouncy)) }
                                }
                            }
                        }
                    }
                )
            }
    ) {
        Card(modifier = Modifier.fillMaxSize()) { /* card content */ }
    }
}
```

---

## 14. React Native — Animated API vs Reanimated 3

### 14.1 React Native `Animated` API (built-in)

```typescript
import { Animated, PanResponder } from 'react-native'

// Value-based animation
const fadeAnim = useRef(new Animated.Value(0)).current

// Fade in
Animated.timing(fadeAnim, {
    toValue: 1,
    duration: 500,
    easing: Easing.out(Easing.quad),
    useNativeDriver: true,   // ← CRITICAL: runs on native UI thread
}).start()

// Spring
Animated.spring(scaleAnim, {
    toValue: 1,
    friction: 7,          // damping
    tension: 40,          // stiffness
    useNativeDriver: true
}).start()

// Parallel (simultaneous)
Animated.parallel([
    Animated.timing(fadeAnim, { toValue: 1, duration: 300, useNativeDriver: true }),
    Animated.spring(scaleAnim, { toValue: 1, useNativeDriver: true })
]).start()

// Sequence
Animated.sequence([
    Animated.timing(anim1, { toValue: 1, duration: 200, useNativeDriver: true }),
    Animated.timing(anim2, { toValue: 1, duration: 200, useNativeDriver: true })
]).start()

// Apply to view
<Animated.View style={{ opacity: fadeAnim, transform: [{ scale: scaleAnim }] }}>
    <Text>Animated!</Text>
</Animated.View>
```

**`useNativeDriver: true`** is critical. Without it, animation runs on the JS thread — every frame requires a JS→native bridge crossing, causing jank especially when the JS thread is busy (JSON parsing, navigation). With it, the animation runs entirely on the native UI thread.

**Limitation:** `useNativeDriver: true` only works for: `opacity`, `transform` (translate, scale, rotate). NOT for: `width`, `height`, `backgroundColor`, `padding`, `margin`. For layout properties, you must use `useNativeDriver: false` — which means JS thread.

### 14.2 Reanimated 3 ✅ Recommended for complex animations

```typescript
import Animated, {
    useAnimatedStyle,
    useSharedValue,
    withTiming,
    withSpring,
    withRepeat,
    withSequence,
    withDelay,
    runOnJS,
    useAnimatedGestureHandler,
    interpolate,
    Extrapolation
} from 'react-native-reanimated'
import { PanGestureHandler } from 'react-native-gesture-handler'

// Shared values — live on the UI thread, not JS thread
const scale = useSharedValue(1)
const translateX = useSharedValue(0)
const opacity = useSharedValue(1)

// Animated style — runs on UI thread via Reanimated worklet
const animatedStyle = useAnimatedStyle(() => ({
    transform: [
        { scale: scale.value },
        { translateX: translateX.value }
    ],
    opacity: opacity.value
}))

// Animate — all on UI thread
scale.value = withSpring(1.2, { damping: 10, stiffness: 100 })
opacity.value = withTiming(0.5, { duration: 300 })

// Complex sequences
scale.value = withSequence(
    withTiming(1.2, { duration: 100 }),
    withSpring(1, { damping: 8 })
)

// Infinite
translateX.value = withRepeat(
    withTiming(20, { duration: 500 }),
    -1,    // -1 = infinite
    true   // reverse = true (pingpong)
)

// Gesture-driven (Swipe to dismiss)
const panGesture = useAnimatedGestureHandler({
    onStart: (_, ctx) => {
        ctx.startX = translateX.value
    },
    onActive: (event, ctx) => {
        translateX.value = ctx.startX + event.translationX  // direct assignment = instant
    },
    onEnd: (event) => {
        if (Math.abs(event.translationX) > 150) {
            translateX.value = withTiming(event.translationX > 0 ? 500 : -500, {}, () => {
                runOnJS(onDismiss)()  // callback to JS thread when done
            })
        } else {
            translateX.value = withSpring(0)  // snap back
        }
    }
})

// Interpolation — map one range to another
const scrollY = useSharedValue(0)
const headerStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
        scrollY.value,
        [0, 200],          // input range
        [1, 0],            // output range
        Extrapolation.CLAMP // clamp outside range
    )
}))

// Usage
<PanGestureHandler onGestureEvent={panGesture}>
    <Animated.View style={[styles.card, animatedStyle]}>
        <Text>Swipe me!</Text>
    </Animated.View>
</PanGestureHandler>
```

### 14.3 Animated API vs Reanimated 3

| | React Native `Animated` | Reanimated 3 |
|---|---|---|
| Thread | JS → native bridge per frame (unless useNativeDriver) | Fully on UI thread (worklets) |
| `useNativeDriver` limit | opacity + transform only | ANY style property |
| Setup | Built-in | npm install |
| Gesture integration | PanResponder (clunky) | react-native-gesture-handler (smooth) |
| Interpolation | `interpolate()` | `interpolate()` (more powerful) |
| Layout animations | Limited | `Layout.springify()`, `FadeIn`, etc. |
| Performance | Good (with useNativeDriver) | Best |
| Best for | Simple opacity/transform | Complex gestures, physics, layout animations |

### 14.4 Reanimated Layout Animations

```typescript
import Animated, { FadeIn, FadeOut, SlideInLeft, Layout } from 'react-native-reanimated'

// Animate item entering/leaving a list
{items.map(item => (
    <Animated.View
        key={item.id}
        entering={FadeIn.duration(300)}
        exiting={FadeOut.duration(200)}
        layout={Layout.springify()}  // animate when other items shift
    >
        <ItemRow item={item} />
    </Animated.View>
))}

// Built-in entering/exiting animations
FadeIn, FadeOut
SlideInLeft, SlideInRight, SlideInUp, SlideInDown
SlideOutLeft, SlideOutRight, SlideOutUp, SlideOutDown
ZoomIn, ZoomOut
BounceIn, BounceOut
FlipInXUp, FlipOutXDown
// All are configurable: FadeIn.duration(500).delay(200).springify()
```

---

## 15. Common Misunderstandings & Pitfalls

**❌ `alpha = 0f` vs `AnimatedVisibility`**
Setting `alpha` to 0 hides the composable visually but keeps it in the composition — screen readers still see it, it still receives touch events, it still occupies layout space. `AnimatedVisibility` removes the composable from composition after the exit animation. Use `AnimatedVisibility` for things that are truly gone; use `alpha` only for transient visual effects.

**❌ `animate*AsState` for scroll-driven animations**
`animate*AsState` drives recomposition — every animation frame causes the composable to re-run. For scroll-driven effects (parallax, fade on scroll), use `graphicsLayer { }` which reads scroll state in the drawing phase with zero recomposition cost.

**❌ Missing `remember` around `derivedStateOf` in animations**
Already covered in section 18, but especially common in animation code. `val x by derivedStateOf { }` without `remember` creates a new derived state on every recomposition, defeating the optimization. Always `val x by remember { derivedStateOf { } }`.

**❌ `rememberInfiniteTransition` animation running when composable is off-screen**
Infinite transitions continue running even when the composable is not visible. They only stop when the composable leaves the composition. For battery efficiency, conditionally compose the animated element or use `isActive` from a lifecycle effect to pause.

**❌ `useNativeDriver: false` for layout animations in RN `Animated`**
Layout property animations (`width`, `height`, `backgroundColor`) require `useNativeDriver: false` in the RN Animated API — they run on the JS thread with bridge calls every frame. Reanimated 3 worklets avoid this because they run on the UI thread regardless of which property is animated.

**❌ Forgetting `label` parameter in Compose animations**
Every `animate*AsState`, `updateTransition`, and related API has a `label` parameter. Without it, Android Studio's animation inspection tools can't identify which animation is which. Always provide descriptive labels.

**❌ `Animatable` in a regular function instead of `remember`**
`val anim = Animatable(0f)` in a composable body creates a new `Animatable` on every recomposition, losing the current animation value. Always `val anim = remember { Animatable(0f) }`.

---

## 16. Best Practices

- **Choose the right API** — use the decision tree: `animate*AsState` for simple values, `updateTransition` for coordinated multi-value, `AnimatedVisibility` for show/hide, `Animatable` for gesture-driven
- **`graphicsLayer` for scroll-driven animations** — runs in drawing phase, zero recomposition cost
- **`spring` for interactive/gesture animations** — physics feel, responds to velocity
- **`tween` for UI transitions** — predictable duration, Material easing
- **Always `label` your animations** — enables Android Studio's animation inspector
- **`remember { Animatable(...) }`** — never bare `Animatable()` in composable body
- **`SharedTransitionLayout` for hero animations** — use `sharedElement()` modifier on both source and destination
- **Reanimated 3 for RN complex animations** — fully on UI thread, gesture-handler integration
- **`useNativeDriver: true` always for RN Animated** — if the property supports it; opacity + transform
- **`AnimatedContent` for content switching** — use `ContentTransform` for directional animations
- **Prefer `withSpring` in Reanimated** — physics-based animations feel natural on mobile
- **`runOnJS()` for callbacks from UI thread** — state updates from Reanimated gestures must go through `runOnJS`

---

## 17. Interview Q&A

**Q1: Which Compose animation API would you use for a card that expands when tapped?**

> For a card expanding to show more content, I'd use `animateContentSize()` modifier for simple height-based expansion — it automatically animates size changes without any state tracking. For a richer animation (coordinated color, elevation, and corner radius changes), I'd use `updateTransition` to animate all properties simultaneously from the same state. I'd use `spring` for the animation spec to make it feel physical — the card springs open like a real physical object. If the expanded state reveals entirely different content (not just more content), `AnimatedContent` with a size transform would handle the content crossfade smoothly.

---

**Q2: What's the performance difference between `Modifier.alpha()` and `graphicsLayer { alpha = ... }`?**

> `Modifier.alpha(value)` reads the alpha value during the composition phase — Compose must re-run the composable function on every animation frame to apply the updated alpha. This means full recomposition every 16ms for a 60fps animation. `graphicsLayer { alpha = animatedValue.value }` reads the value during the drawing phase — the composable doesn't recompose at all. The drawing happens on the RenderThread, which is hardware-accelerated and runs in parallel with the main thread. For scroll-driven animations or infinite animations, `graphicsLayer` can be 10x fewer operations — it's the difference between 60 recompositions per second vs 0. The rule: read animation state inside `graphicsLayer { }` whenever possible, especially for continuous or scroll-triggered animations.

---

**Q3: When would you use `Animatable` instead of `animate*AsState`?**

> `animate*AsState` is reactive — it animates whenever its target state changes, with a single animation that runs to completion. It's the right choice when state drives the animation: `selected = true` → animate color, `visible = false` → animate opacity. `Animatable` is imperative — you control it via coroutines: `anim.animateTo(target)`, `anim.snapTo(target)`, `anim.animateDecay(velocity)`. I'd use `Animatable` when: (1) the animation is gesture-driven — I need to update the value continuously as the user drags, then spring back or fling based on release velocity; (2) I need to interrupt a running animation and redirect it — `Animatable.animateTo()` on a new target cancels the current animation and starts fresh; (3) I need the current velocity to inform the next animation, like a fling that continues from the last drag velocity.

---

**Q4: How do Shared Element Transitions work in Compose?**

> `SharedTransitionLayout` provides a `SharedTransitionScope` that tracks composables marked with the `sharedElement()` modifier using a shared key. When content transitions (via `AnimatedContent` or navigation transitions) and both the source and destination have a composable with the same key, Compose animates from the source position/size to the destination position/size — the image or element appears to fly from the list item to the detail screen. The `rememberSharedContentState(key)` creates the link between the two. During the transition, the shared element is rendered at an intermediate position calculated by the layout system. You also get `Modifier.sharedBounds()` for animating the bounding box of a container, and `Modifier.renderInSharedTransitionScopeOverlay()` to keep content visible above other composables during the transition.

---

**Q5: What's the difference between `spring` and `tween` animation specs?**

> `tween` is duration-based with easing — it runs for exactly the specified milliseconds, using an easing curve to control acceleration. It's predictable: a 300ms tween always takes 300ms. `spring` is physics-based — it simulates a spring between the current value and the target. The `dampingRatio` controls bounciness (1.0 = no bounce, 0.2 = very bouncy) and `stiffness` controls how quickly it reaches the target. Spring animations have no fixed duration — they play until the value is close enough to the target with near-zero velocity. Spring animations can also carry velocity from a previous animation, so a gesture-release can continue with natural momentum. Use `tween` for deliberate UI transitions (screen changes, dialogs) and `spring` for anything the user interacts with directly (draggable items, interactive cards, expandable panels) because spring feels physically responsive.

---

**Q6: How does Reanimated 3 differ from React Native's Animated API?**

> The fundamental difference is thread execution. RN's Animated API, even with `useNativeDriver: true`, still requires setting up the animation from JavaScript — only the interpolation runs natively. Without `useNativeDriver`, every frame causes a JS-bridge crossing. Also, `useNativeDriver` only works for transform and opacity, not layout properties. Reanimated 3 uses worklets — JavaScript functions annotated with `'worklet'` that are copied to and executed directly on the UI thread. This means animated gesture handlers respond at 120fps even when the JS thread is busy doing other work (parsing data, handling navigation). The `useAnimatedStyle` hook also runs on the UI thread. Additionally, Reanimated supports layout animations (items entering/exiting lists with smooth neighboring item shifts) which Animated doesn't have. The trade-off is setup complexity and an additional dependency, but for any animation that involves user gestures or must be smooth under JS load, Reanimated is the right choice.

---

*Previous: [18 — State Management](./18-state-management.md)*
*Next: [20 — Caching Deep Dive](./20-caching.md)*
