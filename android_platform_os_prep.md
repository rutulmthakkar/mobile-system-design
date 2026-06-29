# Android Platform & OS Concepts — Interview Prep
**Role:** Software Engineer, Android Platform, Google Beam  
**Focus:** Platform-layer Android, OS internals, hardware integration  
**Format:** Concept → Explanation → Interview Q&A → Key facts to memorize

---

## 1. Android Architecture Stack

### The Layers (top to bottom)
```
┌─────────────────────────────────┐
│         Applications            │  ← Your current expertise
├─────────────────────────────────┤
│      Application Framework      │  ← Activity Manager, Window Manager, etc.
├─────────────────────────────────┤
│    Native Libraries + ART       │  ← libc, OpenGL, WebKit, ART runtime
├─────────────────────────────────┤
│  Hardware Abstraction Layer     │  ← HAL ← Beam lives here
├─────────────────────────────────┤
│        Linux Kernel             │  ← Drivers, power, scheduling
└─────────────────────────────────┘
```

### Key facts
- **ART (Android Runtime)** replaced Dalvik in Android 5.0
- ART uses AOT (Ahead-Of-Time) compilation — compiles to native on install
- Also uses JIT + profile-guided compilation since Android 7.0
- Every app runs in its own ART instance, own process, own UID

### Interview Q&A
**Q: What happens when you launch an app?**  
A: Zygote process (pre-warmed ART instance) forks a new process → loads APK → calls Application.onCreate() → starts the main Activity. Zygote pre-loads common classes so forking is fast.

**Q: What is Zygote?**  
A: A special Android process that pre-initializes ART and common framework classes. All app processes are forked from Zygote — faster than cold-starting a JVM every time.

**Q: What is the difference between ART and Dalvik?**  
A: Dalvik used JIT (compile at runtime, byte-by-byte). ART uses AOT (compile to native machine code at install time) + JIT for hot paths. ART is faster at runtime, slightly slower to install.

---

## 2. Hardware Abstraction Layer (HAL)

### What it is
HAL is the interface between Android's framework and hardware drivers. It lets Android talk to hardware without knowing the hardware details.

```
Android Framework
       ↓  (HIDL/AIDL interface)
      HAL  ← .so shared library, vendor-provided
       ↓  (ioctl / kernel syscalls)
  Linux Kernel Driver
       ↓
    Hardware
```

### Types of HAL
| HAL | Controls | Beam Relevance |
|---|---|---|
| Camera HAL | Camera sensors, capture pipeline | ✅ High — Beam has multi-camera |
| Audio HAL | Microphone, speakers | ✅ High |
| Power HAL | CPU/GPU frequency, sleep states | ✅ High — thermal management |
| Display HAL / HWC | Screen rendering, vsync | ✅ High — 3D display |
| USB HAL | USB device enumeration | ✅ Mentioned in JD |
| Thermal HAL | Temperature sensors, throttling | ✅ Mentioned in JD |

### HIDL vs AIDL
- **HIDL** (HAL Interface Definition Language) — older, C++-based, used for HAL up to Android 10
- **AIDL** (Android Interface Definition Language) — newer, supports Kotlin/Java/C++, preferred from Android 11+
- Both generate Binder-based IPC stubs automatically

### Key facts
- HAL modules are `.so` shared libraries loaded at runtime
- Vendor HALs live in `/vendor/lib/hw/`
- HALs run in their own process (HAL server) — crash-isolated from framework
- **Treble** (Android 8.0+) separated vendor HAL from Android framework — enables faster OS updates

### Interview Q&A
**Q: Why does Android use a HAL instead of talking to drivers directly?**  
A: Abstraction and portability. Hardware vendors write HAL implementations for their chips. Android framework stays the same. Without HAL, every Android version update would require rewriting all hardware drivers.

**Q: What is Project Treble?**  
A: Android 8.0 architecture change that separated the Android OS framework from vendor HAL implementations using a stable HIDL interface. Allows Android OS updates without waiting for vendors to update HAL code.

**Q: What happens if a HAL crashes?**  
A: HALs run in isolated processes (HAL servers). If a HAL crashes, only that hardware service is affected. Android's service manager can restart the HAL process. The framework gets a service death notification via Binder death recipients.

**Q: How does Camera HAL work?**  
A: Framework (CameraService) sends capture requests to Camera HAL via HIDL interface. HAL programs the sensor, ISP pipeline, captures frames into shared memory buffers, and notifies the framework. Framework delivers frames to the app via CameraDevice callbacks.

---

## 3. Binder IPC

### What it is
Binder is Android's primary IPC (Inter-Process Communication) mechanism. It's a kernel driver (`/dev/binder`) that enables fast, secure communication between processes.

```
Client Process                    Server Process
    │                                  │
    │──── Binder transaction ────────→ │
    │         (kernel driver)          │
    │←─── Reply ──────────────────────│
```

### How it works
1. Client calls a method on a proxy object (looks like local call)
2. Proxy serializes the call into a `Parcel` (data + file descriptors)
3. Kernel copies Parcel from client's memory to server's memory (one copy only)
4. Server's stub deserializes and executes the method
5. Return value travels back the same way

### Key properties
- **Synchronous** — client blocks until server responds
- **One copy** — kernel uses memory mapping to avoid double-copy
- **1MB limit** — total active Binder transactions per process capped at 1MB
- **Thread pool** — each process has a Binder thread pool (default 15 threads)
- **Security** — kernel enforces caller UID, enables permission checks

### AIDL
AIDL generates the proxy/stub boilerplate for Binder IPC automatically.
```java
// IMyService.aidl
interface IMyService {
    String getData(int id);
}
// Generates: IMyService.Stub (server side) + IMyService.Stub.Proxy (client side)
```

### Binder vs other IPC
| Mechanism | Speed | Use case |
|---|---|---|
| Binder | Fast (1 copy) | Android system services |
| Unix socket | Medium | ADB, network stack |
| Shared memory (ashmem) | Fastest (0 copy) | Large data — camera frames, audio buffers |
| Pipe | Slow | Simple unidirectional data |

### Interview Q&A
**Q: Why does Android use Binder instead of standard Linux IPC?**  
A: Standard Linux IPC (pipes, sockets) requires two copies — kernel→kernel and kernel→userspace. Binder uses memory mapping for one copy. Also Binder provides object-oriented RPC semantics, automatic thread management, and security via UID tracking in the kernel.

**Q: What is the 1MB Binder limit and how do you work around it?**  
A: Each process has ~1MB of Binder transaction space. For large data (camera frames, bitmaps), use shared memory (ashmem/MemoryFile) and pass a file descriptor through Binder instead of the data itself.

**Q: What is a Binder death recipient?**  
A: A callback registered on a Binder proxy. When the remote process dies, the kernel notifies all registered death recipients. Used to handle service crashes gracefully — e.g., Camera client detects CameraService death and cleans up.

**Q: How is Binder related to AIDL and HIDL?**  
A: Both AIDL and HIDL are IDL (Interface Definition Languages) that generate Binder-based IPC code. AIDL generates Java/Kotlin/C++ stubs. HIDL generates C++ stubs for HAL communication. Both use the same underlying Binder kernel driver.

---

## 4. Power Management

### Android Power States
```
Active → Screen On, CPU running at full speed
  ↓ (screen timeout)
Display Off → CPU still running, wakelocks may be held
  ↓ (no wakelocks)
Light Doze → Batched network, deferred jobs
  ↓ (stationary, unplugged)
Deep Doze → Almost no activity, rare maintenance windows
```

### Wakelocks
Prevent the CPU or screen from sleeping.
```java
PowerManager pm = getSystemService(PowerManager.class);
PowerManager.WakeLock wl = pm.newWakeLock(
    PowerManager.PARTIAL_WAKE_LOCK, "MyApp::MyWakelockTag");
wl.acquire(10 * 60 * 1000L); // 10 min timeout
// ... do work
wl.release(); // Always release
```

Types:
- `PARTIAL_WAKE_LOCK` — CPU on, screen can turn off
- `SCREEN_BRIGHT_WAKE_LOCK` — CPU + screen on (deprecated)
- `PROXIMITY_SCREEN_OFF_WAKE_LOCK` — screen off when near face

### Thermal Management
Critical for Beam — sustained 3D video processing generates heat.

```
Temperature sensors → Thermal HAL → ThermalManager (framework)
                                          ↓
                               Throttling decisions:
                               - Reduce CPU/GPU frequency
                               - Drop video bitrate
                               - Alert app to reduce workload
```

```java
// Listen for thermal status changes
ThermalManager tm = getSystemService(ThermalManager.class);
tm.addThermalStatusListener(executor, status -> {
    switch (status) {
        case THERMAL_STATUS_MODERATE: // reduce quality
        case THERMAL_STATUS_SEVERE:   // aggressively reduce
        case THERMAL_STATUS_CRITICAL: // emergency shutdown
    }
});
```

### Battery Optimization
- **Doze mode** — batches background work when idle/stationary
- **App Standby** — buckets apps by usage: Active, Working Set, Frequent, Rare, Restricted
- **WorkManager** — respects Doze, defers non-urgent work

### Interview Q&A
**Q: A 3D video streaming device like Beam runs hot. How do you manage thermal?**  
A: Register a ThermalStatusListener. On MODERATE, reduce video encode bitrate. On SEVERE, drop to lower resolution. On CRITICAL, pause non-essential processing. Also design the processing pipeline to offload to dedicated hardware blocks (GPU, DSP, video encoder ASIC) instead of CPU — hardware blocks are more power-efficient.

**Q: What is Doze mode and how does it affect background work?**  
A: Doze kicks in when the device is stationary, unplugged, and screen off. It blocks network access, defers JobScheduler jobs, and prevents wakelocks except during maintenance windows. WorkManager uses JobScheduler under the hood and respects Doze automatically.

**Q: What is a wakelock leak and how do you detect it?**  
A: A wakelock acquired but never released — keeps CPU awake, drains battery. Detect with `adb shell dumpsys power | grep "Wake Locks"`. Always use try/finally to guarantee release, or set a timeout on acquire().

---

## 5. Camera System Architecture

### Camera Stack
```
App (Camera2 API)
      ↓
CameraManager (Framework)
      ↓
CameraService (system server)
      ↓
Camera HAL (vendor .so)
      ↓
Camera Driver (kernel)
      ↓
Image Sensor Hardware
```

### Camera2 API Key Concepts
```kotlin
// Core objects
CameraManager        // discover/open cameras
CameraDevice         // represents one camera
CameraCaptureSession // configured pipeline for captures
CaptureRequest       // one frame's parameters (exposure, focus, etc.)
ImageReader          // consumer that receives captured frames
```

### Buffer pipeline
```
Camera Sensor → ISP (Image Signal Processor) → BufferQueue
                                                     ↓
                                    ImageReader / SurfaceView / MediaRecorder
```

- **BufferQueue** — producer-consumer queue for graphics buffers
- **ANativeWindow** — C interface to BufferQueue surface
- **GraphicBuffer** — represents one frame buffer, backed by gralloc (GPU-accessible)

### Interview Q&A
**Q: How does Camera2 differ from the old Camera API?**  
A: Camera2 (API 21+) gives full manual control — per-frame exposure, ISO, focus, white balance. Async callback-based pipeline. Supports RAW capture, burst mode, reprocessing. Old Camera API was synchronous and limited. Camera2 maps directly to the HAL3 interface.

**Q: What is an ImageReader?**  
A: A consumer of camera frames that gives app access to raw pixel data. You specify format (JPEG, YUV_420_888, RAW_SENSOR) and maxImages. Call `acquireLatestImage()` to get a frame, **must close() it** when done or the queue stalls.

**Q: How do you stream camera to display with minimal latency?**  
A: Use a `SurfaceTexture` as the camera target. SurfaceTexture puts frames into a GL texture. Render with OpenGL/Vulkan directly to a SurfaceView. This path goes: Camera HAL → BufferQueue → GPU → Display. Avoids CPU copies entirely.

---

## 6. Display System (SurfaceFlinger)

### How Android renders to screen
```
App draws to Surface (Canvas/OpenGL/Vulkan)
         ↓
    BufferQueue (app is producer)
         ↓
  SurfaceFlinger (compositor — consumes all app surfaces)
         ↓
  Hardware Composer (HWC HAL) — composites layers in hardware
         ↓
      Display
```

### Key concepts
- **SurfaceFlinger** — system service that composites all app windows into final frame
- **HWC (Hardware Composer)** — hardware-accelerated compositor in GPU/display chip
- **VSYNC** — vertical sync signal from display (~60Hz/90Hz/120Hz)
- **Choreographer** — Android class that syncs animations/drawing to VSYNC
- **Triple buffering** — 3 buffers rotating: one displaying, one being composited, one being rendered — reduces jitter

### Frame pipeline timing (16ms at 60fps)
```
VSYNC → App renders frame (≤8ms) → SurfaceFlinger composites (≤4ms) → Display
```

### Interview Q&A
**Q: What causes jank (dropped frames)?**  
A: Frame not ready before next VSYNC. Causes: too much work on main thread (>16ms), GC pause, lock contention, slow layouts. Detect with Systrace/Perfetto.

**Q: What is the difference between a Surface and a SurfaceView?**  
A: Surface is the raw buffer producer (gralloc buffer + BufferQueue). SurfaceView is a View that holds a Surface — it has its own layer in SurfaceFlinger, rendered independently of the View hierarchy. Useful for video/camera — bypasses View invalidation overhead.

**Q: What is Vulkan and when would you use it over OpenGL ES?**  
A: Vulkan is a lower-level GPU API — explicit memory management, multi-threaded command recording, less driver overhead. Use for performance-critical rendering (3D video, AR/VR) where OpenGL's implicit state machine overhead is too high. Required for Beam's 3D rendering pipeline.

---

## 7. USB & System Interfaces

### USB in Android
- Android supports USB Host (controls other devices) and USB Accessory (controlled by other device)
- **USB HAL** enumerates connected devices, handles enumeration protocol
- `UsbManager` — framework API for USB device/accessory access
- USB classes: HID (keyboard/mouse), Mass Storage, Audio, Video (UVC)

### Key for Beam
Beam JD mentions USB interfaces — likely for:
- External display/accessory connectivity
- Firmware/software updates via USB
- High-bandwidth data transfer (USB 3.x for video?)

### Interview Q&A
**Q: How does Android handle USB device attachment?**  
A: Kernel detects USB device → sends uevent → udevd updates device nodes → Android's USB service receives broadcast → queries device VID/PID → checks for matching app or prompts user.

---

## 8. CI/CD for Hardware Platforms

### Your existing strength — frame it for hardware
Your Kiro agent + CloudWatch + automated fault reporting maps directly. Add:

### Hardware-specific CI/CD concepts
- **Flashing** — deploying OS image to physical device (`fastboot flash`)
- **OTA (Over-The-Air) updates** — A/B partition updates, rollback support
- **Android Virtual Device (AVD)** — emulator for CI when hardware is scarce
- **CTS (Compatibility Test Suite)** — Google's test suite for Android certification
- **VTS (Vendor Test Suite)** — tests HAL implementations

### A/B OTA System
```
Partition A (current, booted)    Partition B (new, being updated)
                                        ↓ (download + verify in background)
                                   Reboot → boot from B
                                        ↓ (if boot succeeds)
                                   Mark B as current, A as backup
```

### Interview Q&A
**Q: How do you do CI/CD for a hardware device like Beam?**  
A: Build pipeline compiles Android image → runs unit tests on host → flashes to device farm (physical devices in rack) → runs instrumented tests + CTS subset → if pass, generates OTA package → staged rollout to small % of devices → monitor metrics → full rollout. Rollback via A/B partition if error rate spikes.

---

## 9. Memory Management at Platform Level

### Linux Memory Concepts (Android runs on Linux)
- **Virtual memory** — each process has its own 64-bit address space
- **Physical memory** — actual RAM, shared across all processes
- **Page** — 4KB unit of memory mapping
- **Page fault** — process accesses unmapped page → kernel maps it → continue
- **OOM Killer** — Linux kills processes when RAM runs out

### Android-specific
- **LMK (Low Memory Killer)** — Android's OOM killer, kills by oom_adj score
- **ashmem (Anonymous Shared Memory)** — Android's shared memory implementation, file descriptor based
- **ION allocator** — hardware-contiguous memory allocator for camera/GPU buffers (pre-Android 12)
- **DMA-BUF** — modern replacement for ION, standard Linux kernel mechanism

### Interview Q&A
**Q: How does Android decide which process to kill when memory is low?**  
A: LMK (Low Memory Killer) ranks processes by oom_adj score. Higher score = more likely to be killed. Order: cached processes → background → services → foreground → persistent → system. Foreground app gets lowest oom_adj (hardest to kill).

**Q: What is the difference between ashmem and regular heap memory?**  
A: Heap memory is private to the process. ashmem creates a shared memory region backed by a file descriptor that can be passed via Binder to another process — zero copy. Used for camera frames, audio buffers, large bitmaps shared between processes.

---

## 10. Android Boot Process

### Sequence
```
Power On
   ↓
Bootloader (fastboot mode possible here)
   ↓
Linux Kernel loads → mounts partitions
   ↓
init process (PID 1) → reads init.rc
   ↓
Zygote starts → preloads ART + framework classes
   ↓
System Server starts → launches all system services:
   ActivityManagerService, WindowManagerService,
   PackageManagerService, CameraService, AudioFlinger...
   ↓
Launcher app starts → Home screen visible
```

### Interview Q&A
**Q: What is init.rc?**  
A: Android's init configuration language. Defines what services to start at boot, what file permissions to set, what environment variables to export. HAL servers are typically started via init.rc.

**Q: How long does Android take to boot and how do you optimize it?**  
A: Cold boot ~15-30 seconds. Optimize by: lazy-starting non-critical services, reducing init.rc work, enabling A/B boot with snapshot (faster OS updates), profiling with Systrace bootchart.

---

## Quick Reference — Key Commands

```bash
# Check running processes and their Binder threads
adb shell dumpsys activity processes

# Check power/wakelock state  
adb shell dumpsys power

# Check thermal status
adb shell dumpsys thermalservice

# Check camera state
adb shell dumpsys media.camera

# Check SurfaceFlinger / display
adb shell dumpsys SurfaceFlinger

# Trace system performance
adb shell perfetto ...

# Flash device partition
fastboot flash boot boot.img

# Check USB devices
adb shell lsusb
```

---

## Interview Cheat Sheet — One-liners

| Concept | One-liner |
|---|---|
| HAL | Interface between Android framework and hardware drivers; vendor-provided .so libraries |
| Binder | Android's IPC kernel driver; one-copy, synchronous, 1MB limit |
| Zygote | Pre-warmed ART process; all apps fork from it |
| SurfaceFlinger | System compositor; combines all app surfaces into display frame |
| VSYNC | Display sync signal; Choreographer aligns drawing to it |
| BufferQueue | Producer-consumer queue for graphics frames |
| ashmem | Shared memory via file descriptor; zero-copy between processes |
| LMK | Kills background processes by oom_adj score when RAM is low |
| Treble | Android 8.0 separation of OS from vendor HAL; enables faster updates |
| ART | Android Runtime; AOT + JIT compilation; replaced Dalvik in Android 5 |
| Doze | Power-saving mode; batches network/jobs when screen off + stationary |
| OTA A/B | Dual partition update; new OS downloaded to inactive partition |
