# 🚀 Deep Packet Inspection Engine (C++17)

A high-performance Deep Packet Inspection (DPI) engine I built from scratch in C++17 to understand how real network traffic classification systems work.

It parses raw PCAP files, tracks connections using 5-tuples, extracts TLS SNI from HTTPS traffic, classifies applications (YouTube, Facebook, Netflix, etc.), and applies flow-based blocking rules — all without using any external DPI libraries.

![C++17](https://img.shields.io/badge/C++-17-blue)
![Threads](https://img.shields.io/badge/MT-Yes-green)
![PCAP](https://img.shields.io/badge/PCAP-Parsing-orange)
---

## Why I Built This

I wanted to answer some real questions:

- How do firewalls identify websites even when traffic is encrypted?
- What actually happens inside a packet from Layer 2 to Layer 7?
- How do you process hundreds of thousands of packets per second?
- How do you design multi-threaded systems without race conditions?

---

## What It Does

### **Input**
- PCAP file (Wireshark capture)

### **Output**
- Filtered PCAP
- Application classification report
- Traffic statistics

### **Core Features**

- Manual Ethernet / IPv4 / TCP / UDP parsing
- Five-tuple based flow tracking
- TLS Client Hello parsing
- SNI extraction (Extension type `0x0000`)
- HTTP Host header extraction
- 15+ application classifications
- Blocking by IP, app, or domain
- Multi-threaded packet processing
- Per-thread flow tables (no global locks)

No external DPI libraries (nDPI, libprotoident, etc.)  
No frameworks.  
Just C++17 and the standard library.

---

## Architecture

I built two versions.

---

### 1️⃣ Single-Threaded Version

I started with a simple pipeline to get the core logic correct:

```cpp
while (reader.readNextPacket(raw)) {
    PacketParser::parse(raw, parsed);

    FiveTuple tuple = extractFiveTuple(parsed);
    Flow& flow = flows[tuple];

    if (parsed.dest_port == 443) {
        auto sni = SNIExtractor::extract(payload, length);
        if (sni) {
            flow.sni = *sni;
            flow.app_type = classifyApp(*sni);
        }
    }

    if (shouldBlock(flow)) {
        dropped++;
    } else {
        output.write(raw);
        forwarded++;
    }
}
```

This version worked but peaked around ~50K packets/sec.

That’s when I redesigned it.

---

### 2️⃣ Multi-Threaded Version

The challenge: packets from the same connection must go to the same thread.

Processing pipeline:

```
Reader Thread
      ↓
Load Balancer Threads
      ↓
Fast Path Threads (DPI Processing)
      ↓
Output Writer Thread
```

#### Key Design Decisions

**Consistent hashing of the 5-tuple**  
Same connection → same thread.

**Per-thread flow tables**  
Eliminates locking between worker threads.

**Thread-safe bounded queues**  
Built using `std::mutex` and `std::condition_variable`.

**Minimal shared state**  
Reduced synchronization overhead.

---

## The Hard Parts

### Endianness

Network data is big-endian.  
My machine is little-endian.

Spent too long debugging incorrect port numbers before remembering `ntohs()`.

---

### Pointer Arithmetic

Parsing raw byte buffers means manually advancing offsets through variable-length fields.

One off-by-one error and the whole TLS structure becomes garbage.

Added assertions everywhere to catch boundary mistakes early.

---

### Race Conditions

My first multi-threaded version had corrupted flow tables and random packet drops.

The fix:

- Remove shared flow tables
- Use per-thread state
- Avoid global locks
- Redesign ownership model

---

### Performance Bottlenecks

Initial scaling wasn’t linear.

Problems I ran into:

- False sharing between threads
- Too much memory allocation
- Hash collisions in flow map
- Reader thread becoming a bottleneck

Fixes:

- Cache-line alignment
- Buffer reuse
- Improved hash function
- Two-stage load balancing

---

## Flow-Based Blocking Logic

Blocking happens at the connection level.

Example:

```
Packet 1 (SYN)          → No SNI yet → Forward
Packet 2 (SYN-ACK)      → No SNI yet → Forward
Packet 3 (ACK)          → No SNI yet → Forward
Packet 4 (Client Hello) → SNI detected → Classified
                        → Rule match → Block flow
Packet 5+               → Flow marked blocked → Drop
```

Once classified, the entire connection is dropped.

---

## Performance Results

| Threads | Packets/sec | Speedup |
|----------|--------------|----------|
| 1        | 50,000       | 1x       |
| 2        | 95,000       | 1.9x     |
| 4        | 180,000      | 3.6x     |
| 8        | 320,000      | 6.4x     |
| 16       | 500,000      | 10x      |

Memory usage: ~10MB for ~10,000 concurrent flows.

---

## Build

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp \
    src/load_balancer.cpp \
    src/fast_path.cpp \
    src/connection_tracker.cpp \
    src/rule_manager.cpp \
    src/thread_safe_queue.cpp
```

---

## Run

```bash
./dpi_engine input.pcap output.pcap --block-app YouTube
```

Optional thread configuration:

```bash
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

---

## Sample Output

```
Total Packets: 142
Forwarded: 128
Dropped: 14

Application Breakdown:
  HTTPS     72
  YouTube   14 (BLOCKED)
  Google    12
  Facebook   8
  DNS        4
```

---

## What's Next
- QUIC/HTTP3 support (SNI in UDP)
- Live capture mode (not just files)
- Simple web dashboard

## What I Learned

- Ethernet / IP / TCP internals
- TLS handshake structure
- Raw binary parsing
- Flow tracking at scale
- Thread ownership models
- Avoiding false sharing
- Designing high-throughput systems
- Debugging with GDB, Valgrind, AddressSanitizer