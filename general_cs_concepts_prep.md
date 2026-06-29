# General CS Concepts — Interview Prep
**Role:** Software Engineer, Android Platform, Google Beam  
**Focus:** OS fundamentals, networking, data structures, system design  
**Format:** Concept → Explanation → Interview Q&A → Key facts to memorize

---

## 1. Processes vs Threads

### Definitions
- **Process** — independent program in execution; own memory space, own resources, own file descriptors
- **Thread** — unit of execution within a process; shares memory + resources with other threads in same process

### Key differences
| | Process | Thread |
|---|---|---|
| Memory | Isolated (own virtual address space) | Shared (heap, globals) |
| Communication | IPC (pipes, sockets, Binder) | Shared memory directly |
| Creation cost | Expensive (fork + exec) | Cheap |
| Crash impact | Only that process dies | Can crash entire process |
| Context switch | Expensive (full address space switch) | Cheaper (same address space) |

### Context Switching
When OS switches from one thread/process to another:
1. Save current CPU registers + program counter to PCB (Process Control Block)
2. Load next thread/process registers from its PCB
3. Update memory mappings (expensive for process switch)

Cost: ~1-10 microseconds for thread switch, more for process (TLB flush)

### Android context
- Each Android app = one process (own PID, own ART instance)
- Main thread = UI thread — never block it
- Background work → worker threads, Coroutines, ExecutorService
- Camera, audio use dedicated threads with real-time scheduling (`SCHED_FIFO`)

### Interview Q&A
**Q: When would you use multiple processes vs multiple threads?**  
A: Multiple threads when tasks share data frequently and need low overhead communication. Multiple processes when isolation is critical (crash in one shouldn't affect other), security boundary needed (different permissions), or running untrusted code. Android uses processes for app isolation — one app crashing doesn't affect others.

**Q: What is a context switch and why is it expensive?**  
A: Saving/restoring CPU state (registers, program counter, stack pointer). Process switches also flush TLB (Translation Lookaside Buffer — CPU cache for virtual→physical memory mappings), which causes cache misses on next memory accesses. Thread switches within same process skip TLB flush — cheaper.

**Q: What is a race condition?**  
A: Two threads access shared data concurrently and the outcome depends on scheduling order. Classic example: two threads both read a counter (value=5), both increment (value=6), both write — expected 7, got 6. Fix with synchronization.

---

## 2. Synchronization & Concurrency

### Mutex (Mutual Exclusion Lock)
Only one thread can hold the mutex at a time. Others block until released.
```kotlin
val mutex = Mutex()
mutex.withLock {
    // critical section — only one thread here at a time
    sharedCounter++
}
```

### Semaphore
Allows N threads concurrently. Binary semaphore (N=1) = mutex.
```kotlin
val semaphore = Semaphore(3) // max 3 concurrent
semaphore.acquire()
try { /* work */ } finally { semaphore.release() }
```

### Deadlock
Two or more threads waiting on each other forever.
```
Thread A holds Lock 1, waiting for Lock 2
Thread B holds Lock 2, waiting for Lock 1
→ Neither can proceed
```

**Prevention strategies:**
- Always acquire locks in the same order
- Use tryLock() with timeout
- Lock ordering (assign global order to all locks)
- Avoid holding lock while calling external code

### Livelock
Threads keep changing state in response to each other but make no progress. Like two people stepping aside for each other in a hallway — both keep moving but never pass.

### Starvation
A thread never gets CPU time because higher-priority threads always preempt it.

### Monitor / Condition Variables
```kotlin
// Java/Kotlin: synchronized + wait/notify
@Synchronized fun produce(item: Item) {
    while (queue.isFull()) wait()
    queue.add(item)
    notifyAll()
}
```

### Lock-free / Wait-free
- **Lock-free** — at least one thread makes progress (uses CAS — Compare-And-Swap)
- **Wait-free** — every thread makes progress in bounded steps
- Android uses AtomicInteger, AtomicReference for lock-free counters

### Interview Q&A
**Q: What are the four conditions for deadlock (Coffman conditions)?**  
A: (1) Mutual exclusion — resource held by only one thread. (2) Hold and wait — thread holds one resource while waiting for another. (3) No preemption — resources can't be forcibly taken. (4) Circular wait — circular chain of threads each waiting on the next. Break any one condition to prevent deadlock.

**Q: What is the difference between a mutex and a semaphore?**  
A: Mutex has ownership — only the thread that acquired it can release it. Semaphore has no ownership — any thread can signal it. Use mutex for protecting critical sections. Use semaphore for signaling between threads (producer-consumer).

**Q: What is CAS (Compare-And-Swap)?**  
A: Atomic CPU instruction: if memory[addr] == expected, set memory[addr] = newValue, return true. Else return false. Foundation of lock-free algorithms. AtomicInteger.compareAndSet() uses CAS. Avoids mutex overhead for simple counters.

**Q: What is the happens-before relationship?**  
A: JMM (Java Memory Model) guarantee. If action A happens-before action B, A's effects are visible to B. synchronized, volatile, thread start/join establish happens-before. Without it, threads may see stale cached values.

---

## 3. Memory Management

### Stack vs Heap
| | Stack | Heap |
|---|---|---|
| What's stored | Local variables, function call frames | Objects, dynamic allocations |
| Size | Small (~1-8MB per thread) | Large (limited by RAM) |
| Allocation | Automatic (push/pop) | Manual (new/malloc) or GC |
| Speed | Very fast | Slower (allocation + GC) |
| Thread safety | Per-thread (no sharing) | Shared (needs sync) |

### Virtual Memory
Each process gets its own virtual address space (64-bit on modern Android = 256TB addressable).
- **MMU (Memory Management Unit)** — CPU hardware translates virtual → physical addresses
- **Page Table** — maps virtual pages to physical frames
- **TLB** — CPU cache for recent virtual→physical translations
- **Page fault** — access to unmapped page → OS maps it → transparent to process

### Garbage Collection (JVM/ART)
ART uses **generational GC**:
```
Young generation (Eden + Survivor) → frequent, fast GC
           ↓ (survived N collections)
Old generation (Tenured) → infrequent, slow GC (stop-the-world)
```

- **Minor GC** — cleans young generation; fast (~1ms)
- **Major GC** — cleans old generation; slow, can cause jank
- **Stop-the-world** — all threads pause during GC; visible as frame drops

**ART improvements:**
- Concurrent GC (most work done without stopping threads)
- Compaction — moves objects to eliminate fragmentation
- Background GC — runs during idle time

### Memory Leaks
Object referenced longer than needed — GC can't collect it.
Common Android causes:
- Static reference to Context/Activity
- Inner class holding outer class reference
- Registered listener never unregistered
- Bitmap not recycled (pre-ART)

### Interview Q&A
**Q: What is a memory leak and how do you find one in Android?**  
A: GC-rooted reference to object that should have been collected. In Android: Activity held by static field, anonymous listener holding Activity reference. Find with: LeakCanary (runtime detection), Android Studio Memory Profiler (heap dump + reference path), `adb shell dumpsys meminfo`.

**Q: What is the difference between stack overflow and heap out of memory?**  
A: Stack overflow = too many nested function calls (infinite recursion), stack grows beyond its limit → StackOverflowError. Heap OOM = allocated more objects than available heap → OutOfMemoryError. Fix stack: add base case to recursion. Fix heap: reduce allocations, increase heap size, find leaks.

**Q: What is garbage collection pause and how does it affect Android?**  
A: Stop-the-world GC pauses all threads while collecting. If pause > 16ms, you drop a frame (jank). Minimize by: avoiding allocations in draw/animation loops, using object pools, preferring primitive arrays over boxed types, pre-allocating in onStart/onCreate.

---

## 4. CPU Scheduling

### Scheduling Algorithms
- **FIFO** — first in, first out; simple, no starvation
- **Round Robin** — each thread gets a time quantum; fair
- **Priority-based** — higher priority runs first; can starve low priority
- **CFS (Completely Fair Scheduler)** — Linux's algorithm; each thread gets proportional CPU time based on weight (nice value)

### Linux / Android Scheduling
- Android runs Linux kernel → uses CFS for most threads
- Real-time threads: `SCHED_FIFO` or `SCHED_RR` — run until blocked/preempted by higher priority
- Audio thread, camera thread → real-time priority (`SCHED_FIFO`) to prevent jitter
- Nice value: -20 (highest priority) to +19 (lowest)

### Priority Inversion
```
High priority task H waiting for resource held by Low priority task L
Medium priority task M preempts L
→ M effectively blocks H (priority inversion)
```

**Solution:** Priority inheritance — temporarily boost L's priority to H's while L holds the resource.  
Used in Android's Binder — inherits caller's priority.

### Interview Q&A
**Q: What is CFS and how does it differ from round-robin?**  
A: CFS tracks virtual runtime (vruntime) for each thread. Always picks thread with lowest vruntime. Threads with lower nice value accumulate vruntime slower (get more CPU). Unlike round-robin, CFS adapts to thread weight, not just position in queue.

**Q: Why do audio and camera use SCHED_FIFO in Android?**  
A: Audio and camera have hard deadlines — audio glitches and frame drops are immediately perceptible. SCHED_FIFO threads run until they block or a higher-priority thread preempts — no time-slicing preemption. This guarantees bounded latency for real-time work.

**Q: What is priority inversion and when did it famously occur?**  
A: Mars Pathfinder (1997) — high-priority meteorological thread blocked by low-priority data collection thread holding a mutex, while medium-priority communication thread preempted the data thread. Caused system resets. Fixed by enabling priority inheritance in VxWorks.

---

## 5. Networking Fundamentals

### OSI Model (memorize this)
```
Layer 7 — Application   HTTP, WebRTC, DNS, SMTP
Layer 6 — Presentation  TLS/SSL, encoding
Layer 5 — Session       Session management
Layer 4 — Transport     TCP, UDP (ports)
Layer 3 — Network       IP (addresses, routing)
Layer 2 — Data Link     Ethernet, WiFi (MAC addresses)
Layer 1 — Physical      Cables, radio waves
```

**Mnemonic:** All People Seem To Need Data Processing (top to bottom)

### TCP vs UDP
| | TCP | UDP |
|---|---|---|
| Connection | Yes (3-way handshake) | No |
| Reliability | Guaranteed delivery, ordered | Best effort, no ordering |
| Error recovery | Retransmission on loss | No retransmission |
| Speed | Slower (ACKs, flow control) | Faster |
| Header size | 20 bytes | 8 bytes |
| Use case | HTTP, file transfer, email | Video streaming, gaming, DNS |

**TCP 3-way handshake:**
```
Client → SYN → Server
Client ← SYN-ACK ← Server
Client → ACK → Server
[Connection established]
```

**TCP 4-way close:**
```
Client → FIN → Server
Client ← ACK ← Server
Client ← FIN ← Server
Client → ACK → Server
```

### Why Beam uses UDP
Real-time 3D video cannot afford TCP retransmission latency.
- Lost packet → request retransmission → wait → stutter (visible to user)
- With UDP: lost packet → skip it → minor artifact → much less perceptible than stutter
- Use FEC (Forward Error Correction) to recover some packet loss without retransmission

### IP Addressing
- **IPv4** — 32-bit, 4.3B addresses, e.g., 192.168.1.1
- **IPv6** — 128-bit, e.g., 2001:0db8::1
- **NAT** — maps multiple private IPs to one public IP
- **Subnet mask** — divides IP into network + host portions
- **CIDR** — /24 means 24 bits for network, 8 for host (256 addresses)

### DNS
```
User types google.com
    ↓
Check local cache (browser/OS)
    ↓
Query local resolver (ISP's DNS)
    ↓
Query root nameserver → .com nameserver → google.com nameserver
    ↓
Returns IP address → cached with TTL
```

- **TTL** — time to cache the DNS response; short TTL = fresh but more queries
- **DNS over HTTPS (DoH)** — encrypted DNS; Android 9+ supports Private DNS

### HTTP/HTTPS
- **HTTP/1.1** — one request per connection (or pipelining, rarely used); text headers
- **HTTP/2** — multiplexed streams over one connection; binary; header compression (HPACK); server push
- **HTTP/3** — runs over QUIC (UDP-based); no head-of-line blocking; faster handshake
- **TLS** — Transport Layer Security; encrypts HTTP → HTTPS

### TLS Handshake (simplified)
```
Client → ClientHello (supported cipher suites)
Server → ServerHello + Certificate
Client → verifies cert → generates session key
Client → encrypted with server's public key
Both derive symmetric session key
[Encrypted communication begins]
```

### WebSocket
- Persistent bidirectional connection over HTTP upgrade
- Client sends `Upgrade: websocket` header
- Server responds 101 Switching Protocols
- Use for: real-time data, live updates, signaling (WebRTC)

### WebRTC (critical for Beam)
Google's real-time communication framework. Beam likely uses it.
```
Signaling Server (WebSocket)
     ↓ (exchange SDP offers, ICE candidates)
Peer A ←──── DTLS/SRTP (UDP) ────→ Peer B
             (direct P2P if possible)
             (TURN relay if NAT blocks)
```

Key components:
- **SDP (Session Description Protocol)** — describes media capabilities (codecs, bitrates)
- **ICE (Interactive Connectivity Establishment)** — finds best network path
- **STUN** — tells peer its public IP/port
- **TURN** — relay server when direct P2P blocked by NAT/firewall
- **SRTP** — encrypted RTP; media payload
- **DTLS** — encrypted data channel

### Latency vs Bandwidth vs Throughput
- **Latency** — time for one packet to travel source → destination (ms)
- **Bandwidth** — maximum capacity of a link (Gbps)
- **Throughput** — actual data transferred per second (always ≤ bandwidth)
- **Jitter** — variance in packet arrival times; killer for real-time video
- **RTT (Round Trip Time)** — latency × 2; ping measures this

### Interview Q&A
**Q: TCP guarantees delivery. Why would you ever use UDP?**  
A: When latency matters more than reliability. TCP retransmission can add 100ms+ of latency (wait for ACK, detect loss, retransmit). For live video, a slightly corrupted frame is better than a frozen screen. Use FEC or just skip lost packets.

**Q: What is head-of-line blocking?**  
A: In HTTP/1.1, requests are processed in order. If one request is slow, all later requests wait (blocked behind it). HTTP/2 multiplexes streams but still has TCP HOL blocking at transport layer. HTTP/3/QUIC moves to UDP, eliminating TCP HOL blocking entirely.

**Q: What happens when you type google.com in a browser?**  
A: DNS resolution → TCP connection (or QUIC) → TLS handshake → HTTP GET request → server processes → HTTP response → browser parses HTML → fetches sub-resources → renders page. Each step can be optimized (DNS prefetch, connection pooling, CDN, HTTP/2 push).

**Q: What is a jitter buffer?**  
A: Buffer that holds received packets briefly to smooth out arrival time variance. If packets arrive at 0ms, 15ms, 3ms, 20ms (jittery), jitter buffer holds them and delivers at 20ms, 35ms, 40ms, 55ms (smooth). Trade-off: adds latency. Beam needs minimal jitter buffer for real-time feel.

**Q: Explain NAT traversal for WebRTC.**  
A: Most devices are behind NAT (router). Two peers can't directly connect without knowing each other's public IP:port. STUN server tells each peer its public address. ICE tries: direct host connection → STUN reflexive → TURN relay. TURN is last resort (adds latency, costs bandwidth).

---

## 6. Data Structures for Platform Engineering

### Ring Buffer (Circular Buffer)
Critical for audio/video pipelines. Fixed-size buffer where write pointer wraps around.
```
[  0  |  1  |  2  |  3  |  4  ]
        ^read                ^write
```
- O(1) read and write
- No memory allocation after initialization
- Lock-free implementation possible with atomic pointers
- Used in: audio HAL (PCM samples), camera frame queue, network packet buffer

```kotlin
class RingBuffer<T>(private val capacity: Int) {
    private val buffer = arrayOfNulls<Any>(capacity)
    private var readIdx = AtomicInteger(0)
    private var writeIdx = AtomicInteger(0)

    fun write(item: T): Boolean {
        val w = writeIdx.get()
        val next = (w + 1) % capacity
        if (next == readIdx.get()) return false // full
        buffer[w] = item
        writeIdx.set(next)
        return true
    }
}
```

### Priority Queue / Min-Heap
Used for: scheduling, Dijkstra, top-K problems, event queues.
```
         1
       /   \
      3     5
     / \   /
    7   9 8
```
- Parent ≤ children (min-heap)
- Insert: add at end, bubble up — O(log n)
- Extract min: remove root, move last to root, bubble down — O(log n)
- Build heap from array — O(n)

### HashMap Internals
```
Key → hashCode() → index in array → linked list/tree at that index
```
- **Load factor** — ratio of entries to buckets (default 0.75)
- **Rehashing** — when load factor exceeded, new array 2x size, re-insert all — O(n) amortized O(1)
- **Collision** — two keys same hash → same bucket → linked list (Java 8+: tree if >8 entries)
- **hashCode() contract** — equal objects must have same hashCode; hashCode must be consistent

### Graph Representations
```
Adjacency List (space-efficient for sparse):     Adjacency Matrix (fast lookup):
0 → [1, 2]                                       [0, 1, 1, 0]
1 → [3]                                          [0, 0, 0, 1]
2 → [3, 4]                                       [0, 0, 0, 1]
```

### Interview Q&A
**Q: Why use a ring buffer for audio instead of a regular queue?**  
A: Audio processing is real-time — must not allocate memory during processing (GC pause = audio glitch). Ring buffer is pre-allocated, fixed size, O(1) operations, can be lock-free. Regular queue may need memory allocation when growing.

**Q: What is amortized O(1) for HashMap?**  
A: Single insert is O(1) usually, O(n) when rehashing. But rehashing doubles capacity, so next n inserts are all O(1). Total cost for n inserts = O(n). Amortized per insert = O(1).

**Q: When would you use a graph vs a tree?**  
A: Tree when hierarchy + single parent (file system, XML/HTML DOM, organization chart). Graph when arbitrary relationships + possible cycles (network topology, social connections, dependency graphs). Tree is a special case of graph (connected, acyclic, directed).

---

## 7. System Design Concepts

### Key principles
- **Single responsibility** — each component does one thing well
- **Loose coupling** — components don't depend on each other's internals
- **High cohesion** — related things together
- **Scalability** — horizontal (more machines) vs vertical (bigger machine)
- **Availability** — % time system is operational (99.9% = 8.7 hours downtime/year)
- **Consistency** — all nodes see same data at same time
- **Partition tolerance** — system works despite network failures

### CAP Theorem
In a distributed system, you can only guarantee 2 of 3:
- **C**onsistency
- **A**vailability
- **P**artition tolerance

In practice: network partitions happen, so choose CA or CP.

### Backpressure
When consumer is slower than producer — queue fills up.
```
Camera (30fps) → Frame Queue → Encoder (can only do 20fps)
                    ↑
              Queue fills up → drop oldest frames / block producer
```
Solutions: drop frames, reduce producer rate, increase consumer capacity, bounded queue with drop policy.

### Producer-Consumer Pattern
```kotlin
val channel = Channel<Frame>(capacity = 10) // bounded

// Producer (camera thread)
launch { camera.frames.collect { frame -> channel.send(frame) } }

// Consumer (encoder thread)
launch { for (frame in channel) { encoder.encode(frame) } }
```

### Graceful Degradation
System reduces quality under load rather than failing completely.
Beam example:
```
Normal → 4K 3D 60fps
Thermal moderate → 1080p 3D 60fps
Thermal severe → 1080p 3D 30fps
Thermal critical → 720p 2D 30fps
```

### Observability (Metrics, Logging, Tracing)
- **Metrics** — aggregated numbers (frame rate, latency percentiles, error count)
- **Logging** — events with timestamp and context
- **Distributed tracing** — follows request across multiple services (spans + traces)
- **Alerting** — trigger on metric threshold

Your 32 breadcrumb metrics system is a strong observability story. Frame it as: zero-visibility problem → instrumented critical path → attribution of failures across layers.

### Interview Q&A
**Q: Design a low-latency 3D video streaming pipeline for Beam.**  
A: Capture → Camera HAL → YUV frame → Video encoder (hardware H.265/AV1) → RTP packetizer → UDP socket → Network → RTP depacketizer → Video decoder (hardware) → 3D compositor (GPU, Vulkan) → Display. Key decisions: hardware encode/decode (not software — too slow), UDP (not TCP), minimal jitter buffer, adaptive bitrate based on bandwidth estimate, thermal-aware bitrate reduction.

**Q: How do you handle a 3x spike in data throughput?**  
A: Depends on system. For Beam: adaptive bitrate — reduce video quality to match available bandwidth. For servers: horizontal scaling, load balancer, auto-scaling groups. For queues: add consumers, increase partition count. Always: measure first, identify bottleneck (CPU/network/memory/IO), scale that specific resource.

**Q: What is the difference between latency and throughput optimization?**  
A: Latency: minimize time for single operation — reduce hops, cache data closer to user, avoid blocking calls, use async I/O. Throughput: maximize operations per second — batch work, use pipelines, parallelize, increase queue depth. Often trade-off: larger batches increase throughput but increase latency.

---

## 8. Linux Fundamentals (Android runs on Linux)

### Key concepts
- **Everything is a file** — devices, pipes, sockets are all file descriptors
- **File descriptor** — integer handle to open file/socket/device
- **System calls** — how userspace talks to kernel (read, write, open, ioctl, mmap, fork, exec)
- **Signals** — async notifications to processes (SIGKILL, SIGTERM, SIGSEGV)
- **ioctl** — generic device control system call (camera, GPU, USB use this heavily)

### Processes
```bash
fork()   # create child process (copy of parent)
exec()   # replace current process image with new program
wait()   # parent waits for child to finish
exit()   # terminate process
```

### File I/O
```bash
open()   # get file descriptor
read()   # read bytes
write()  # write bytes
close()  # release file descriptor
mmap()   # map file/device into process memory (zero copy)
ioctl()  # device-specific control
```

### Useful commands
```bash
ps aux              # list all processes
top / htop          # real-time process monitor
strace -p PID       # trace system calls of a process
lsof -p PID         # list open file descriptors
/proc/PID/maps      # memory map of process
dmesg               # kernel ring buffer (driver messages)
```

### Interview Q&A
**Q: What is mmap and why is it used for camera frames?**  
A: mmap maps a file or device memory directly into process virtual address space. Access memory like a regular array — kernel handles I/O transparently. For camera: GPU/camera hardware writes frames to physical memory, mmap gives CPU access without copying. Zero copy — crucial for high-bandwidth video.

**Q: What is ioctl?**  
A: System call for device-specific operations not covered by read/write. Camera driver uses ioctl to set exposure, start/stop streaming. GPU driver uses ioctl for command submission. Argument is an integer code + data struct. Android's camera HAL wraps ioctl calls in C++ API.

**Q: What happens when a process receives SIGSEGV?**  
A: Segmentation fault — process accessed invalid memory (null pointer, buffer overflow, use-after-free). Kernel delivers SIGSEGV → default handler terminates process + generates core dump. In Android, ART catches SIGSEGV, generates a tombstone file (crash dump) in /data/tombstones/.

---

## Quick Reference — One-liners

| Concept | One-liner |
|---|---|
| Process vs Thread | Process = isolated memory; Thread = shared memory within process |
| Deadlock | Circular wait for resources; prevent by consistent lock ordering |
| Race condition | Outcome depends on thread timing; fix with synchronization |
| CAS | Atomic compare-and-swap; foundation of lock-free algorithms |
| Virtual memory | Each process has own address space; MMU translates to physical |
| GC pause | Stop-the-world collection; >16ms causes Android frame drop |
| TCP | Reliable, ordered, connection-based; slower |
| UDP | Unreliable, unordered, fast; use for real-time video/audio |
| TLS | Encrypts TCP; asymmetric handshake → symmetric session key |
| WebRTC | P2P real-time comms; SDP + ICE + SRTP + DTLS |
| Ring buffer | Fixed circular buffer; O(1), no allocation; audio/video pipelines |
| Backpressure | Consumer slower than producer; drop/block/slow down producer |
| Jitter buffer | Smooths packet arrival variance; trades latency for smoothness |
| CAP theorem | Can only guarantee 2 of: Consistency, Availability, Partition tolerance |
| Priority inversion | Low priority blocks high priority; fix with priority inheritance |
| CFS | Linux scheduler; picks thread with lowest virtual runtime |
| mmap | Maps file/device to virtual memory; zero-copy I/O |
| ioctl | Generic device control syscall; camera/GPU control |
