# Mediasoup SFU Architecture

## Introduction
**mediasoup** is a powerful, low-level WebRTC Selective Forwarding Unit (SFU) library designed for building high-performance real-time media applications. Unlike many other SFUs, mediasoup is unopinionated and doesn't provide a signaling protocol or application logic—it focuses purely on efficient media routing. It is built with a Node.js API that controls a high-performance C++ media engine.

## Core Architectural Components

### 1. Worker (C++ Subprocess)
The **Worker** is the core process of mediasoup. Each Worker is a single-threaded C++ subprocess that runs on a single CPU core. 
- **Efficiency:** By spawning one Worker per CPU core, mediasoup achieves massive parallelism.
- **Isolation:** If one Worker crashes, others remain unaffected.

### 2. Router (The "Media Room")
A **Router** is a logical entity within a Worker that acts as a "Virtual Room." It manages the exchange of media packets between participants.
- **Routing Logic:** It ensures that media from a Producer is sent only to interested Consumers.
- **Capabilities:** It negotiates RTP capabilities (codecs, extensions) to ensure compatibility.

### 3. Transport (The Network Path)
A **Transport** represents the connection between the Router and the outside world.
- **WebRtcTransport:** Used for browser/mobile clients (handles ICE, DTLS, SRTP).
- **PlainTransport:** Used for connecting simple RTP streams (e.g., FFmpeg, GStreamer).
- **PipeTransport:** Used for core-to-core or server-to-server media "piping."

### 4. Producer & Consumer
- **Producer:** Represents a media stream (audio or video) being sent *into* the Router.
- **Consumer:** Represents a media stream being sent *out* of the Router to a specific participant.

```mermaid
flowchart TD
    subgraph Server_Node
        subgraph Worker1[Worker - CPU Core 1]
            R1[Router A]
            R1 --> P1[WebRtcTransport]
            P1 --> Prod1((Producer))
            R1 --> C1[WebRtcTransport]
            C1 --> Cons1((Consumer))
        end
        
        subgraph Worker2[Worker - CPU Core 2]
            R2[Router B]
            R2 --> PipeT[PipeTransport]
        end
    end

    R1 <-->|Pipe Path| PipeT
```

## Scaling Strategy

### Vertical Scaling (Multi-Core)
Since a single Worker runs on one core, it can handle a finite number of participants (typically ~500 consumers). To scale, applications spawn multiple Workers and create Routers across them.

### Horizontal Scaling (Multi-Server)
To scale beyond a single machine, we use **PipeTransport**.
1. **Pipe Layer:** When a participant on `Server A` needs to see a producer on `Server B`, the system creates a PipeTransport between the two servers.
2. **Media Relaying:** The media is "piped" from `Server B` to `Server A`, making the producer available on the local router of `Server A`.

## Handling Advanced WebRTC Features

### Simulcast and SVC
Mediasoup excels at handling **Simulcast** (Vp8/H264) and **SVC** (Vp9/L1T3). 
- The SFU does not re-encode video (no transcoding).
- Instead, it dynamically switches between different spatial and temporal layers provided by the client's encoder based on the receiver's bandwidth.

## Real-World Use Cases

### 1. Video Conferencing
Handling many-to-many meetings where each participant produces their own audio/video and consumes everyone else's.

### 2. Large Scale Broadcasting (1 to N)
A single high-quality producer is piped across dozens of servers to reach thousands of concurrent viewers with sub-second latency.

## Pros and Cons

### Pros
- **Extremely Low Latency:** Pure SFU approach with no transcoding overhead.
- **Performance:** Highly optimized C++ core with Node.js control plane.
- **Flexibility:** Can be integrated into any signaling system (WebSockets, gRPC, etc.).
- **Client Control:** Full support for modern WebRTC features like Simulcast and SVC.

### Cons
- **Complexity:** Higher learning curve compared to "plug-and-play" solutions like Jitsi.
- **Infrastructure Management:** Requires explicit management of multi-worker/multi-server logic (piping).
- **No Built-in Signaling:** Developers must build their own Room/Peer management logic.
