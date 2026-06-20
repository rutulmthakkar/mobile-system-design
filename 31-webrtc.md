# 31 — WebRTC & Video Calling

> **Round type: Both (HLD + LLD)**
> **HLD:** Video calling topology — P2P vs SFU vs MCU, signaling server design, scaling
> **LLD:** ICE/STUN/TURN, SDP offer/answer, PeerConnection, media tracks, Android implementation

---

## 1. Why This Matters

Video calling is a staple HLD question at Google Meet, Zoom, Meta (Messenger), WhatsApp, and any communication-focused company. Even if you're not building a calling feature, interviewers use it to test: do you understand NAT traversal, peer connection lifecycle, and scaling bottlenecks?

---

## 2. WebRTC Fundamentals

WebRTC (Web Real-Time Communication) is an open standard for peer-to-peer audio, video, and data communication — no plugins, no intermediary server for media (ideally). Built into Android via `webrtc-android` library.

**Three core APIs:**
```
RTCPeerConnection    → establishes and manages the peer-to-peer connection
                       handles ICE, codec negotiation, encryption
MediaStream         → collection of audio/video tracks from camera/mic
RTCDataChannel      → arbitrary data P2P (chat, files, game state)
```

---

## 3. Connection Establishment — The Full Flow

WebRTC needs a **signaling server** to exchange connection metadata (SDP, ICE candidates) before a direct P2P connection can be established. WebRTC doesn't define the signaling protocol — you use WebSocket, HTTP, or any channel.

```
Alice                  Signaling Server            Bob
  │                          │                       │
  │ 1. createOffer()         │                       │
  │    → SDP Offer           │                       │
  │─────────────────────────▶│                       │
  │                          │─── forward offer ────▶│
  │                          │                       │ 2. setRemoteDescription(offer)
  │                          │                       │    createAnswer()
  │                          │                       │    → SDP Answer
  │                          │◀── forward answer ────│
  │ 3. setRemoteDescription  │                       │
  │    (answer)              │                       │
  │                          │                       │
  │ 4. gather ICE candidates │                       │
  │─────────────────────────▶│──── ICE candidate ───▶│
  │◀─────────────────────────│◀─── ICE candidate ────│
  │                          │                       │
  │ 5. ICE connectivity checks (direct or via TURN)  │
  │◀──────────── P2P connection established ─────────│
  │                          │                       │
  │◀═══════════ Media flows (audio/video) ═══════════│
```

### 3.1 SDP (Session Description Protocol)

SDP is a text format that describes what media capabilities each peer has:
```
v=0
o=alice 123456 654321 IN IP4 192.168.1.1
s=Video Call
t=0 0
m=audio 49170 UDP/TLS/RTP/SAVPF 111 103
a=rtpmap:111 opus/48000/2          ← Opus codec for audio
a=rtpmap:103 ISAC/16000
m=video 51372 UDP/TLS/RTP/SAVPF 100 101
a=rtpmap:100 VP8/90000             ← VP8 codec for video
a=rtpmap:101 H264/90000            ← H264 alternative
```

The offer/answer exchange negotiates: codecs, bandwidth, encryption, media directions (send-only, receive-only, send-receive).

---

## 4. NAT Traversal — STUN, TURN, ICE

Most devices are behind NAT (router). Two devices can't directly connect using their private IPs. WebRTC uses the ICE (Interactive Connectivity Establishment) framework to find the best connection path.

### 4.1 ICE Candidate Types

```
Host candidate:        Device's own local IP (192.168.1.x) — works on same LAN
Server-reflexive (srflx): Public IP:port discovered via STUN — works if NAT allows
Relay candidate (relay): Traffic relayed via TURN server — always works
```

### 4.2 STUN (Session Traversal Utilities for NAT)

STUN is a lightweight server that tells a device what its public IP:port is, as seen from the outside internet.

```
Device (192.168.1.5) → STUN server → "Your public address is 203.0.113.5:49152"
Device advertises this as a server-reflexive ICE candidate
```

**STUN succeeds** in ~90% of home network scenarios (Full Cone, Address-Restricted, Port-Restricted NAT).
**STUN fails** for Symmetric NAT (common in corporate/enterprise networks, mobile carrier NAT).

### 4.3 TURN (Traversal Using Relays around NAT)

When P2P fails, both peers send media through a TURN server. The TURN server relays packets between them.

```
Alice → TURN server → Bob
Bob → TURN server → Alice
```

Cost: ~2x bandwidth through the TURN server (both directions). Latency: TURN adds ~10–30ms vs P2P.

**STUN is free; TURN is expensive** — both bandwidth and infrastructure. Design your system to minimize TURN usage.

### 4.4 ICE Process

```
1. Gather all candidate types: host, srflx (via STUN), relay (via TURN)
2. Exchange candidates via signaling server
3. Run connectivity checks for all candidate pairs
4. Select the best working pair:
   Priority: host > srflx > relay
5. Continuously monitor connection, switch if better path found
```

```kotlin
// ICE server configuration
val iceServers = listOf(
    PeerConnection.IceServer.builder("stun:stun.l.google.com:19302").createIceServer(),
    PeerConnection.IceServer.builder("turn:your-turn-server.com:3478")
        .setUsername("username")
        .setPassword("password")
        .createIceServer()
)
val rtcConfig = PeerConnection.RTCConfiguration(iceServers).apply {
    sdpSemantics = PeerConnection.SdpSemantics.UNIFIED_PLAN  // modern, use this
    iceTransportsType = PeerConnection.IceTransportsType.ALL  // try all paths
}
```

---

## 5. P2P vs SFU vs MCU — Architecture Decision

### 5.1 P2P (Peer-to-Peer)

```
Alice ←──────────────────────▶ Bob

Each peer: 1 upload stream, 1 download stream
N participants: each peer needs N-1 upload + N-1 download streams
```

**Pros:** No media server, lowest latency, cheapest infrastructure
**Cons:** Scales only to ~4-6 participants (upload bandwidth explodes), no recording, no server-side processing

**Use when:** 1:1 calls, small group calls (≤4 people)

### 5.2 SFU (Selective Forwarding Unit) ✅ Standard for group calls

```
             ┌────────────┐
Alice ──────▶│            │──── Alice's stream ────▶ Bob
Bob   ──────▶│  SFU       │──── Bob's stream   ────▶ Alice
Carol ──────▶│  Server    │──── Alice's stream ────▶ Carol
             └────────────┘──── Carol's stream ────▶ Alice
```

Each participant uploads ONE stream to the SFU. SFU forwards streams selectively — each participant downloads N-1 streams. No transcoding — media is forwarded as-is.

**Pros:** Scalable to 100+ participants, lower upload bandwidth per participant, supports simulcast (multiple quality layers), enables recording on server side
**Cons:** Requires server infrastructure, higher latency than P2P (one extra hop)

**Use when:** Group video calls (Zoom, Google Meet style), webinars, live classes

### 5.3 MCU (Multipoint Control Unit)

```
             ┌────────────┐
Alice ──────▶│            │──── Mixed stream ────▶ Alice
Bob   ──────▶│  MCU       │──── Mixed stream ────▶ Bob
Carol ──────▶│  Server    │──── Mixed stream ────▶ Carol
             └────────────┘
```

MCU decodes all streams, composites them into one mixed stream, re-encodes and sends one stream to each participant.

**Pros:** Each participant receives only ONE stream (minimal client bandwidth), legacy device support (older codecs), consistent video layout
**Cons:** Very CPU-intensive server-side (transcode all streams), highest latency, high infrastructure cost

**Use when:** Conference bridges connecting to PSTN/SIP, accessibility requirements, extremely low-bandwidth clients

### 5.4 Architecture Comparison

| | P2P | SFU | MCU |
|---|---|---|---|
| Upload per participant | N-1 streams | 1 stream | 1 stream |
| Download per participant | N-1 streams | N-1 streams | 1 mixed stream |
| Server compute | None | Low (forward only) | Very high (transcode) |
| Server bandwidth | None | High | Medium |
| Max participants | ~4-6 | 100+ | 100+ |
| Latency | Lowest | Low (one hop) | High (transcode delay) |
| Recording | No | Yes (server) | Yes (easy) |
| Used by | 1:1 calls | Zoom, Meet, Teams | Legacy conferencing |

---

## 6. Android Implementation

```kotlin
// gradle
implementation("io.github.webrtc-sdk:android:125.6422.06")

class WebRtcClient(
    private val context: Context,
    private val signalingClient: SignalingClient,
    private val scope: CoroutineScope
) {
    private val eglBase = EglBase.create()
    private val peerConnectionFactory: PeerConnectionFactory

    init {
        PeerConnectionFactory.initialize(
            PeerConnectionFactory.InitializationOptions.builder(context)
                .setEnableInternalTracer(false)
                .createInitializationOptions()
        )
        peerConnectionFactory = PeerConnectionFactory.builder()
            .setVideoDecoderFactory(DefaultVideoDecoderFactory(eglBase.eglBaseContext))
            .setVideoEncoderFactory(DefaultVideoEncoderFactory(eglBase.eglBaseContext, true, true))
            .setOptions(PeerConnectionFactory.Options())
            .createPeerConnectionFactory()
    }

    private var peerConnection: PeerConnection? = null
    private var localVideoTrack: VideoTrack? = null
    private var localAudioTrack: AudioTrack? = null

    // Setup local media
    fun startLocalMedia(surfaceView: SurfaceViewRenderer) {
        surfaceView.init(eglBase.eglBaseContext, null)
        surfaceView.setMirror(true)  // front camera mirror

        // Video from camera
        val videoCapturer = createCameraCapturer()
        val videoSource = peerConnectionFactory.createVideoSource(false)
        val surfaceTextureHelper = SurfaceTextureHelper.create("CaptureThread", eglBase.eglBaseContext)
        videoCapturer.initialize(surfaceTextureHelper, context, videoSource.capturerObserver)
        videoCapturer.startCapture(1280, 720, 30)
        localVideoTrack = peerConnectionFactory.createVideoTrack("local_video", videoSource)
        localVideoTrack?.addSink(surfaceView)

        // Audio from microphone
        val audioSource = peerConnectionFactory.createAudioSource(MediaConstraints())
        localAudioTrack = peerConnectionFactory.createAudioTrack("local_audio", audioSource)
    }

    // Create peer connection
    fun createPeerConnection(isInitiator: Boolean) {
        val observer = object : PeerConnection.Observer {
            override fun onIceCandidate(candidate: IceCandidate) {
                // Send ICE candidate to remote peer via signaling
                scope.launch { signalingClient.sendIceCandidate(candidate) }
            }
            override fun onTrack(transceiver: RtpTransceiver) {
                // Remote peer added a track — display it
                val track = transceiver.receiver.track()
                if (track is VideoTrack) {
                    scope.launch(Dispatchers.Main) {
                        track.addSink(remoteVideoRenderer)
                    }
                }
            }
            override fun onConnectionChange(newState: PeerConnection.PeerConnectionState) {
                when (newState) {
                    PeerConnection.PeerConnectionState.CONNECTED -> onCallConnected()
                    PeerConnection.PeerConnectionState.DISCONNECTED -> onCallDisconnected()
                    PeerConnection.PeerConnectionState.FAILED -> onCallFailed()
                    else -> { }
                }
            }
            // Other callbacks...
            override fun onSignalingChange(state: PeerConnection.SignalingState) {}
            override fun onIceConnectionChange(state: PeerConnection.IceConnectionState) {}
            override fun onIceConnectionReceivingChange(receiving: Boolean) {}
            override fun onIceGatheringChange(state: PeerConnection.IceGatheringState) {}
            override fun onIceCandidatesRemoved(candidates: Array<out IceCandidate>) {}
            override fun onAddStream(stream: MediaStream) {}
            override fun onRemoveStream(stream: MediaStream) {}
            override fun onDataChannel(channel: DataChannel) {}
            override fun onRenegotiationNeeded() {}
        }

        peerConnection = peerConnectionFactory.createPeerConnection(rtcConfig, observer)

        // Add local tracks
        localVideoTrack?.let {
            peerConnection?.addTrack(it, listOf("local_stream"))
        }
        localAudioTrack?.let {
            peerConnection?.addTrack(it, listOf("local_stream"))
        }

        if (isInitiator) createOffer()
    }

    private fun createOffer() {
        val constraints = MediaConstraints().apply {
            mandatory.add(MediaConstraints.KeyValuePair("OfferToReceiveAudio", "true"))
            mandatory.add(MediaConstraints.KeyValuePair("OfferToReceiveVideo", "true"))
        }
        peerConnection?.createOffer(object : SdpObserver {
            override fun onCreateSuccess(sdp: SessionDescription) {
                peerConnection?.setLocalDescription(this, sdp)
                scope.launch { signalingClient.sendOffer(sdp) }
            }
            override fun onCreateFailure(error: String) { }
            override fun onSetSuccess() { }
            override fun onSetFailure(error: String) { }
        }, constraints)
    }

    fun handleRemoteOffer(sdp: SessionDescription) {
        peerConnection?.setRemoteDescription(SdpObserverAdapter(), sdp)
        // Create answer
        peerConnection?.createAnswer(object : SdpObserver {
            override fun onCreateSuccess(answer: SessionDescription) {
                peerConnection?.setLocalDescription(SdpObserverAdapter(), answer)
                scope.launch { signalingClient.sendAnswer(answer) }
            }
            override fun onCreateFailure(error: String) {}
            override fun onSetSuccess() {}
            override fun onSetFailure(error: String) {}
        }, MediaConstraints())
    }

    fun addRemoteIceCandidate(candidate: IceCandidate) {
        peerConnection?.addIceCandidate(candidate)
    }

    // Mute / unmute
    fun setAudioEnabled(enabled: Boolean) { localAudioTrack?.setEnabled(enabled) }
    fun setVideoEnabled(enabled: Boolean) { localVideoTrack?.setEnabled(enabled) }

    // Switch camera
    fun switchCamera() {
        (videoCapturer as? CameraVideoCapturer)?.switchCamera(null)
    }

    fun release() {
        peerConnection?.close()
        peerConnectionFactory.dispose()
        eglBase.release()
    }
}
```

### 6.1 Simulcast

Simulcast lets the sender encode at multiple resolutions/bitrates simultaneously. The SFU forwards the appropriate layer to each receiver based on their bandwidth.

```kotlin
// Enable simulcast on the video sender
peerConnection?.senders?.find { it.track()?.kind() == "video" }?.let { sender ->
    val params = sender.parameters
    // Encoding layers: low (180p@200kbps), medium (360p@500kbps), high (720p@1.5Mbps)
    params.encodings = listOf(
        RtpParameters.Encoding("low", true, 0.25),    // 25% scale, ~200kbps
        RtpParameters.Encoding("med", true, 0.5),     // 50% scale, ~500kbps
        RtpParameters.Encoding("high", true, 1.0)     // full scale, ~1.5Mbps
    )
    sender.parameters = params
}
```

---

## 7. HLD — Video Calling Architecture (Zoom-style)

```
Components:

1. Signaling Server (WebSocket)
   → Exchanges SDP offer/answer between peers
   → Relays ICE candidates
   → Manages room state (who's in the call, mute state)
   → Lightweight — only metadata, no media

2. STUN Server
   → Helps peers discover their public IP
   → Free, low traffic (just the initial handshake)
   → Use Google's public STUN or host your own: coturn

3. TURN Server (coturn or commercial: Twilio, Agora)
   → Relay media when P2P fails (Symmetric NAT, corporate firewalls)
   → ~30% of connections need TURN (P2P succeeds ~70% of time)
   → Most expensive component — bandwidth cost per relay minute

4. SFU (for group calls > 2)
   → LiveKit, mediasoup, Janus, Ion-SFU
   → Receives one stream per participant
   → Forwards selected streams to each participant
   → Supports simulcast, recording, server-side AI processing

5. Recording Server
   → SFU forwards stream to recording service
   → Store as MP4 in S3/GCS
   → Transcode async to different resolutions

Mobile Client:
  → RTCPeerConnection to SFU (not to each peer directly in group call)
  → Publish one stream (camera/mic)
  → Subscribe to N-1 remote streams
  → Adaptive: reduce quality if bandwidth drops

Scaling:
  STUN: stateless, trivially scalable
  Signaling: stateful (WebSocket), scale with sticky sessions + Redis pub/sub
  TURN: stateless relay, scale horizontally behind load balancer
  SFU: stateful (media state), scale with room-based routing
       (all participants in a room connect to same SFU instance)
       horizontal scaling: route rooms to different SFU instances
```

---

## 8. React Native

```typescript
// react-native-webrtc (most popular)
import {
    RTCPeerConnection,
    RTCIceCandidate,
    RTCSessionDescription,
    mediaDevices,
    RTCView
} from 'react-native-webrtc'

// Get local stream
const stream = await mediaDevices.getUserMedia({
    audio: true,
    video: { facingMode: 'user', width: 1280, height: 720 }
})

// Create peer connection
const pc = new RTCPeerConnection(configuration)
stream.getTracks().forEach(track => pc.addTrack(track, stream))

// Handle remote stream
pc.addEventListener('track', event => {
    setRemoteStream(event.streams[0])
})

// Render
<RTCView streamURL={localStream.toURL()} style={styles.localVideo} mirror={true}/>
<RTCView streamURL={remoteStream?.toURL()} style={styles.remoteVideo}/>

// For production, use LiveKit or Daily.co SDKs
// which handle STUN/TURN/SFU complexity for you
import { useRoom, useLocalParticipant } from '@livekit/react-native'
```

---

## 9. Common Misunderstandings & Pitfalls

**❌ Assuming P2P always works**
~30% of real-world connections (corporate firewalls, mobile carrier NAT) fail P2P and require TURN. Never ship without a TURN server. Always test on corporate WiFi and mobile data, not just home WiFi.

**❌ Using public STUN servers for production**
Google's public STUN is rate-limited and has no SLA. Host your own STUN or use a managed service. STUN is stateless and cheap to host.

**❌ Forgetting to release PeerConnection**
`RTCPeerConnection` holds camera, mic, codec resources. Always call `peerConnection.close()` when the call ends and `peerConnectionFactory.dispose()` when done. Missing this causes camera stays on, codec leak, OOM.

**❌ P2P for group calls (3+ participants)**
With P2P mesh topology, each participant uploads N-1 streams. For 5 participants: 4 upload streams simultaneously. This saturates mobile upload bandwidth (typically 5–10 Mbps) and drains battery. Always use SFU for 3+ participants.

**❌ SDP manipulation without understanding codec order**
Reordering codecs in SDP to prefer VP9 or H264 is a common optimization. Doing it wrong silently breaks the call. Use the WebRTC APIs to set codec preferences rather than manual SDP parsing.

---

## 10. Best Practices

- **P2P for 1:1, SFU for groups** — clear rule, follow it
- **Always provide TURN servers** — test on corporate WiFi and mobile data
- **Simulcast for group calls** — let SFU choose quality layer per recipient
- **Signaling via WebSocket** — low latency, bidirectional, ideal for ICE candidate trickle
- **Trickle ICE** — send ICE candidates as they're gathered, don't wait for all candidates
- **Monitor connection quality** — `getStats()` for RTT, packet loss, jitter; degrade quality before dropping
- **TURN authentication** — use time-limited HMAC credentials, never static username/password in APK
- **Use managed services for TURN/SFU at scale** — Twilio, LiveKit, Daily, Agora rather than self-hosting
- **Mute by disabling tracks** — `track.setEnabled(false)` not `track.dispose()` (can't re-enable)
- **Camera release** — always `videoCapturer.stopCapture()` and `dispose()` when call ends

---

## 11. Interview Q&A

**Q1: What's the role of a signaling server in WebRTC?**

> WebRTC establishes peer-to-peer connections, but two peers need to exchange their SDP (session description — what codecs and media they support) and ICE candidates (how they can be reached on the network) before they can connect. This exchange can't happen directly because they don't know each other's addresses yet. The signaling server is a rendezvous point — both peers connect to it via WebSocket, and it relays the SDP offer/answer and ICE candidates between them. Once the peers have exchanged this metadata, they try to connect directly (or via TURN if necessary), and all media flows peer-to-peer — the signaling server is no longer involved. This is why the signaling server is lightweight: it only carries metadata, never media.

---

**Q2: What are STUN and TURN and when is each needed?**

> STUN (Session Traversal Utilities for NAT) is a server that tells a device its public IP and port as seen from the internet — the address it needs to advertise to peers. Most home routers use NAT types that STUN can traverse — P2P connections work in about 70% of real-world scenarios with just STUN. TURN (Traversal Using Relays around NAT) is a relay server for the cases where direct P2P fails — typically Symmetric NAT (corporate firewalls, mobile carrier NAT). Instead of connecting directly, both peers send media through the TURN server which relays it. TURN always works but adds latency and costs bandwidth proportional to call duration. You always need both: STUN first (cheap, fast) with TURN as fallback (expensive but reliable). Never ship without TURN or about 30% of your users won't be able to connect.

---

**Q3: When would you choose SFU over P2P for a video call feature?**

> The rule is: P2P for 1-to-1 calls, SFU for 3 or more. In a P2P mesh with N participants, each device uploads N-1 streams simultaneously. For 5 participants, that's 4 simultaneous uploads — easily saturating mobile upload bandwidth and draining battery. An SFU has each participant upload ONE stream to the server, which then forwards streams to each participant. The bandwidth tradeoff is clear: the client's upload is constant regardless of the number of participants. Additionally, SFU enables simulcast (multiple quality layers, SFU picks the right one per recipient), server-side recording, and mixing for accessibility features. MCU takes this further by transcoding all streams into one mixed stream, but at very high server CPU cost — only justified for legacy device support or very low-bandwidth clients.

---

*Previous: [30 — In-App Purchases](./30-in-app-purchases.md)*
