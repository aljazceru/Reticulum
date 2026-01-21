# Reticulum ESP32 Port - Comprehensive Implementation Plan

**Author:** Claude
**Date:** 2026-01-21
**Target Platform:** ESP32 (and other resource-constrained microcontrollers)
**Source:** Reticulum Python implementation (RNS)
**Objective:** Create a production-ready, memory-efficient implementation of Reticulum for embedded systems

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Language Selection: C vs C++ vs Rust (no_std)](#language-selection)
3. [Architecture Overview](#architecture-overview)
4. [Implementation Phases](#implementation-phases)
5. [Memory Management Strategy](#memory-management-strategy)
6. [Cryptography Library Selection](#cryptography-library-selection)
7. [File Structure & Module Organization](#file-structure--module-organization)
8. [Platform Abstraction Layer](#platform-abstraction-layer)
9. [Testing Strategy](#testing-strategy)
10. [Integration & Deployment](#integration--deployment)
11. [Risk Analysis & Mitigation](#risk-analysis--mitigation)

---

## Executive Summary

**Project Goal:** Port Reticulum networking stack to ESP32-class microcontrollers

**Recommended Language:** **C** (with selective C++ features if using ESP-IDF)

**Implementation Strategy:** Phased approach starting with minimal viable protocol

**Estimated Codebase Size:** 15,000-25,000 lines of C code

**Timeline Phases:**
- Phase 0 (Foundation): Cryptography + Packet structures
- Phase 1 (Core): Identity, Link, Basic Transport
- Phase 2 (Routing): Multi-hop, Announces, Path discovery
- Phase 3 (Features): Resources, Multiple interfaces, Advanced features

**Critical Success Factors:**
1. Memory-efficient data structures (< 100KB RAM for core functionality)
2. Reliable cryptography implementation (security-critical)
3. Robust packet handling (error recovery, retransmission)
4. Compatibility with existing Python Reticulum network

---

## Language Selection

### Evaluation Criteria

| Criterion | Weight | C | C++ | Rust (no_std) |
|-----------|--------|---|-----|---------------|
| **Memory footprint** | 25% | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **ESP32 toolchain maturity** | 20% | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Cryptography libraries** | 20% | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Developer productivity** | 15% | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Safety guarantees** | 10% | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Portability** | 10% | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Community/ecosystem** | 5% | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **TOTAL** | 100% | **4.65** | **4.25** | **3.85** |

### Detailed Analysis

#### **Option 1: C (RECOMMENDED)**

**Pros:**
- ✅ **Smallest memory footprint** - Critical for ESP32's 520KB RAM
- ✅ **Maximum portability** - Works on AVR, ARM Cortex-M, RISC-V, ESP32, etc.
- ✅ **Mature ESP-IDF support** - ESP32's official framework is C-based
- ✅ **Best cryptography library availability** - mbedTLS, libsodium, micro-ecc all in C
- ✅ **Predictable behavior** - No hidden allocations, vtables, or runtime overhead
- ✅ **Existing RNode firmware is C** - Integration path is clear
- ✅ **Direct hardware access** - Essential for radio interfaces
- ✅ **Deterministic memory usage** - Can calculate exact RAM requirements

**Cons:**
- ❌ Manual memory management (pointers, malloc/free)
- ❌ No built-in data structures (need to implement lists, hash tables)
- ❌ More verbose code
- ❌ Easy to introduce buffer overflows if not careful
- ❌ No RAII or automatic cleanup

**Best Use Cases:**
- Hard real-time requirements
- Extreme memory constraints (< 100KB available RAM)
- Maximum portability across MCU families
- Integration with existing C codebases (like RNode)

---

#### **Option 2: C++**

**Pros:**
- ✅ **ESP-IDF supports C++** - Can mix C and C++ freely
- ✅ **RAII** - Automatic resource cleanup (destructors)
- ✅ **Templates** - Type-safe generic programming
- ✅ **Classes/namespaces** - Better code organization
- ✅ **Optional features** - Can use only what you need
- ✅ **mbedTLS C++ wrappers** available
- ✅ **Better string handling** - std::string_view (C++17, no allocation)

**Cons:**
- ❌ **Larger binary size** - Exception tables, RTTI (can disable)
- ❌ **More complex build system** - Linker issues with C++
- ❌ **STL not ideal** - Even with newlib, allocations problematic
- ❌ **Compiler support varies** - GCC is good, but older toolchains struggle
- ❌ **Hidden costs** - Virtual functions, constructors add overhead

**Recommended C++ Subset for ESP32:**
```cpp
// YES - Use these:
- Classes (without virtuals)
- Namespaces
- References
- const/constexpr
- Templates (sparingly)
- inline functions
- RAII for resources

// NO - Avoid these:
- Exceptions (use -fno-exceptions)
- RTTI (use -fno-rtti)
- STL containers (use custom or etl::)
- Virtual inheritance
- operator new/delete (override carefully)
- std::string (use string_view or char*)
```

**Verdict:** Good choice if team has C++ expertise and wants safer abstractions, but adds complexity.

---

#### **Option 3: Rust (no_std)**

**Pros:**
- ✅ **Memory safety** - No buffer overflows, use-after-free at compile time
- ✅ **Fearless concurrency** - Borrow checker prevents data races
- ✅ **Zero-cost abstractions** - Iterators, enums compile to efficient code
- ✅ **Excellent type system** - Option<T>, Result<T,E> for error handling
- ✅ **Cargo package manager** - Easy dependency management
- ✅ **Modern tooling** - Great formatter, linter, documentation
- ✅ **Embedded Rust community** - embassy, RTIC frameworks
- ✅ **Cryptography crates** - `crypto-bigint`, `ed25519-dalek`, `x25519-dalek`

**Cons:**
- ❌ **ESP32 toolchain is immature** - esp-rs is improving but not production-ready
- ❌ **Limited ESP-IDF integration** - Can call C via FFI but awkward
- ❌ **Larger learning curve** - Borrow checker takes time to master
- ❌ **Smaller embedded ecosystem** - Fewer battle-tested libraries
- ❌ **Binary size** - Larger than C (panic handlers, unwinding)
- ❌ **Compile times** - Slower than C/C++
- ❌ **Debugging** - LLDB support varies on embedded
- ❌ **Async runtime overhead** - embassy/RTIC add complexity

**ESP32 Rust Toolchain Status (2026):**
```
esp-rs/esp-idf-hal     - Bindings to ESP-IDF (usable but incomplete)
esp-rs/esp-hal         - Pure Rust HAL (beta, missing features)
espressif/rust         - Official support (improving)
embedded-hal           - Standard traits (good)
```

**Cryptography Crates (no_std compatible):**
- `x25519-dalek` - Curve25519 ECDH ✅
- `ed25519-dalek` - Ed25519 signatures ✅
- `aes` - AES block cipher ✅
- `sha2` - SHA-256/512 ✅
- `hkdf` - HKDF key derivation ✅
- **All available, well-maintained**

**Verdict:** Ideal for memory safety and correctness, but ESP32 ecosystem not mature enough for production use in 2026. Reconsider in 2027-2028.

---

### **FINAL RECOMMENDATION: C with Selective C++ Features**

**Decision:** Use **C (C11)** as the primary language with optional **C++ features** where beneficial.

**Rationale:**

1. **Memory constraints are paramount** - ESP32 has only 520KB RAM, must support multiple radios, packet buffers, routing tables
2. **Portability required** - Should work on ESP32, STM32, nRF52, AVR (future)
3. **Integration with existing ecosystem** - RNode firmware is C, ESP-IDF is C
4. **Cryptography library maturity** - mbedTLS is production-hardened C code
5. **Predictable resource usage** - Need deterministic memory for embedded

**Implementation Approach:**
```c
// Core implementation: Pure C11
rns_packet.c
rns_identity.c
rns_transport.c
rns_link.c

// Optional C++ wrappers (if team prefers):
RNS::Packet wrapper class (RAII)
RNS::Identity wrapper class
```

**Compiler Flags:**
```makefile
CFLAGS = -std=c11 -Wall -Wextra -Werror -O2 -g
CFLAGS += -fno-strict-aliasing
CFLAGS += -fstack-protector-strong  # Security

# If using C++:
CXXFLAGS = -std=c++17 -fno-exceptions -fno-rtti
```

---

## Architecture Overview

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Layer                        │
│  (User code: sensors, actuators, mesh chat, file transfer)  │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                    RNS API Layer                             │
│  rns_destination_*()  rns_link_*()  rns_packet_*()          │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                  Transport Layer                             │
│  - Packet routing (path_table, link_table)                  │
│  - Announce handling & path discovery                        │
│  - Duplicate detection (packet_hashlist)                     │
│  - Multi-hop forwarding (hop count, TTL)                     │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│               Link & Identity Layer                          │
│  - Link establishment (4-way handshake)                      │
│  - Per-link encryption (session keys)                        │
│  - Identity management (keypairs, hashing)                   │
│  - Destination addressing (16-byte hashes)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                 Cryptography Layer                           │
│  - X25519 (ECDH key exchange)                               │
│  - Ed25519 (signatures)                                      │
│  - AES-256-CBC (encryption)                                  │
│  - SHA-256/512 (hashing)                                     │
│  - HKDF (key derivation)                                     │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│                 Interface Layer                              │
│  - Serial/UART (HDLC framing)                               │
│  - LoRa (RNode protocol / direct SX127x)                    │
│  - WiFi (ESP-NOW, UDP)                                       │
│  - Bluetooth (future)                                        │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│              Platform Abstraction Layer (PAL)                │
│  - Threading (FreeRTOS tasks)                               │
│  - Timers (esp_timer)                                        │
│  - Storage (NVS, SPIFFS)                                     │
│  - Logging (ESP_LOG)                                         │
└─────────────────────────────────────────────────────────────┘
```

### Core Data Structures

```c
// Packet structure (in-memory representation)
typedef struct rns_packet {
    uint8_t header_type;           // HEADER_1 or HEADER_2
    uint8_t flags;                 // Packet type, destination type, etc.
    uint8_t hops;                  // Hop count
    uint8_t destination_hash[16];  // Truncated hash
    uint8_t transport_id[16];      // Optional (HEADER_2)
    uint8_t context;               // Optional context flag
    uint8_t *data;                 // Payload (dynamically allocated or pool)
    uint16_t data_len;
    uint32_t sent_at;              // Timestamp
    struct rns_packet *next;       // For linked lists
} rns_packet_t;

// Identity structure
typedef struct rns_identity {
    uint8_t x25519_private[32];    // ECDH private key
    uint8_t x25519_public[32];     // ECDH public key
    uint8_t ed25519_private[64];   // Signature private key
    uint8_t ed25519_public[32];    // Signature public key
    uint8_t hash[32];              // SHA-256 of public keys
    uint8_t hash_truncated[16];    // 16-byte truncated hash
} rns_identity_t;

// Link structure
typedef struct rns_link {
    uint8_t link_id[16];           // Link identifier
    uint8_t destination_hash[16];  // Remote destination
    rns_identity_t *local_identity;
    uint8_t shared_secret[32];     // ECDH result
    uint8_t session_key[32];       // Derived AES key
    uint8_t remote_public_key[64]; // Remote ECPUBSIZE
    uint32_t last_rx;              // Last received timestamp
    uint32_t last_tx;              // Last transmitted timestamp
    enum link_state state;         // PENDING, ACTIVE, STALE, CLOSED
    struct rns_link *next;
} rns_link_t;

// Transport path table entry
typedef struct rns_path {
    uint8_t destination_hash[16];
    uint8_t next_hop[16];          // Interface + next hop hash
    uint8_t hops;                  // Distance
    uint32_t expires_at;           // Path expiration (timestamp)
    uint32_t last_updated;
    struct rns_path *next;
} rns_path_t;

// Interface structure
typedef struct rns_interface {
    char name[32];
    bool in;                       // Can receive
    bool out;                      // Can transmit
    bool fwd;                      // Can forward
    bool rpt;                      // Can repeat
    uint16_t mtu;
    void *hw_context;              // Pointer to hardware (UART, SPI, etc.)
    int (*send)(struct rns_interface*, uint8_t*, uint16_t);  // TX callback
    void (*start)(struct rns_interface*);
    void (*stop)(struct rns_interface*);
    struct rns_interface *next;
} rns_interface_t;
```

### Memory Budget (ESP32 - 520KB Total RAM)

| Component | Size (bytes) | Quantity | Total | Notes |
|-----------|--------------|----------|-------|-------|
| **Packet buffers** | 512 | 8 | 4 KB | MTU=500, circular pool |
| **Path table** | 48 | 64 | 3 KB | Max cached paths |
| **Link table** | 256 | 8 | 2 KB | Active links |
| **Announce queue** | 64 | 16 | 1 KB | Pending announces |
| **Packet hash set** | 20 | 256 | 5 KB | Duplicate detection |
| **Identity cache** | 128 | 32 | 4 KB | Known identities |
| **Crypto buffers** | 512 | 4 | 2 KB | Encryption/decryption work |
| **Interface buffers** | 1024 | 2 | 2 KB | RX/TX per interface |
| **Stack (tasks)** | 4096 | 3 | 12 KB | FreeRTOS tasks |
| **Heap overhead** | - | - | 5 KB | Fragmentation, metadata |
| **TOTAL (Core)** | | | **40 KB** | |
| **FreeRTOS kernel** | | | 15 KB | OS overhead |
| **ESP-IDF drivers** | | | 30 KB | WiFi/BT/UART |
| **Application** | | | 50 KB | User code |
| **TOTAL USAGE** | | | **135 KB** | **26% of available** |
| **Available** | | | **385 KB** | For buffers, features |

**Conclusion:** Feasible with room for growth.

---

## Implementation Phases

### Phase 0: Foundation (Weeks 1-4)

**Goal:** Establish cryptographic primitives and basic packet handling

**Deliverables:**
1. ✅ Cryptography library integration
   - mbedTLS configuration for ESP32
   - X25519 key exchange (test vectors)
   - Ed25519 signing/verification (test vectors)
   - AES-256-CBC encryption (test vectors)
   - SHA-256/512 hashing
   - HKDF key derivation

2. ✅ Packet structure implementation
   - `rns_packet_t` struct
   - `rns_packet_pack()` - serialize to wire format
   - `rns_packet_unpack()` - deserialize from bytes
   - Flag bit manipulation
   - Header type support (HEADER_1, HEADER_2)

3. ✅ Memory management framework
   - Packet pool allocator (circular buffer)
   - Hash table implementation (for path/link tables)
   - Linked list primitives

4. ✅ Unit test framework
   - Unity test framework integration
   - Test vectors from Python implementation
   - Interoperability tests (pack/unpack compatibility)

**Success Criteria:**
- All crypto primitives pass test vectors
- Packets pack/unpack identically to Python RNS
- Zero memory leaks detected (valgrind on host)

**Files Created:**
```
rns_crypto/
├── rns_x25519.c/h
├── rns_ed25519.c/h
├── rns_aes.c/h
├── rns_sha.c/h
└── rns_hkdf.c/h

rns_core/
├── rns_packet.c/h
├── rns_mem.c/h      # Memory pools
└── rns_utils.c/h    # Hash table, lists

tests/
├── test_crypto.c
├── test_packet.c
└── test_vectors/
```

---

### Phase 1: Core Protocol (Weeks 5-10)

**Goal:** Identity, Link establishment, basic Transport

**Deliverables:**
1. ✅ Identity management
   - `rns_identity_create()`
   - `rns_identity_from_file()` / `rns_identity_to_file()`
   - Hash calculation (full & truncated)
   - Key serialization

2. ✅ Destination system
   - `rns_destination_create()`
   - Destination types (SINGLE, GROUP, PLAIN, LINK)
   - Hash calculation with app_name + aspects
   - Packet receive callbacks

3. ✅ Link establishment
   - 4-way handshake (LINKREQUEST → LINKPROOF)
   - Session key derivation
   - Link state machine
   - Keepalive mechanism
   - Link teardown

4. ✅ Basic Transport (single-hop only)
   - `rns_transport_outbound()` - send packet
   - `rns_transport_inbound()` - receive packet
   - Interface registration
   - Packet encryption/decryption
   - Duplicate detection

5. ✅ Serial interface (first hardware interface)
   - UART driver integration (ESP-IDF)
   - HDLC framing (FLAG 0x7E, ESC 0x7D)
   - RX/TX queues

**Success Criteria:**
- Two ESP32 nodes establish link over serial
- Encrypted packets transmitted successfully
- Link survives keepalive timeout
- Packet loss handled gracefully

**Files Created:**
```
rns_core/
├── rns_identity.c/h
├── rns_destination.c/h
├── rns_link.c/h
├── rns_transport.c/h

rns_interfaces/
├── rns_interface.c/h        # Base interface
└── rns_serial_interface.c/h

examples/
└── 01_link_test/
```

---

### Phase 2: Multi-Hop Routing (Weeks 11-16)

**Goal:** Path discovery, announces, multi-hop forwarding

**Deliverables:**
1. ✅ Announce system
   - `rns_destination_announce()`
   - Announce packet structure
   - Signature verification
   - Announce caching

2. ✅ Path discovery
   - Path table management
   - Path requests (when destination unknown)
   - Path responses
   - Path aging & expiration

3. ✅ Multi-hop forwarding
   - Hop count decrement
   - Next-hop lookup
   - Proof routing (reverse path)
   - Loop prevention

4. ✅ Interface diversity
   - Multiple interface support
   - Selective forwarding (per-interface)
   - Interface priorities

**Success Criteria:**
- Three nodes form linear chain (A ↔ B ↔ C)
- Node A discovers path to Node C via B
- Packets route correctly through intermediate hops
- Announces propagate through network

**Files Created:**
```
rns_core/
├── rns_announce.c/h
├── rns_path.c/h

examples/
└── 02_multihop_test/
```

---

### Phase 3: Advanced Features (Weeks 17-24)

**Goal:** Resource transfer, additional interfaces, optimization

**Deliverables:**
1. ✅ Resource transfer
   - Large file splitting (chunks)
   - Windowed transmission (WINDOW=4)
   - Compression (optional, if RAM allows)
   - Reassembly
   - Checksumming

2. ✅ LoRa interface
   - SX127x driver (direct SPI)
   - OR RNode protocol (KISS over serial)
   - Frequency, bandwidth, SF configuration

3. ✅ WiFi interface (ESP-NOW or UDP)
   - ESP-NOW for local mesh
   - UDP for IP networks
   - Broadcast handling

4. ✅ Storage & configuration
   - NVS for identity persistence
   - Configuration file parsing
   - Known destinations cache

5. ✅ Optimization
   - Profiling (CPU, RAM)
   - Packet pool tuning
   - Hash table optimization
   - Crypto acceleration (ESP32 hardware AES)

**Success Criteria:**
- Transfer 1MB file over LoRa (long-range test)
- Mixed interface network (Serial + LoRa + WiFi)
- Identity persists across reboots
- < 50KB RAM usage for core protocol

**Files Created:**
```
rns_core/
├── rns_resource.c/h
├── rns_config.c/h
├── rns_storage.c/h

rns_interfaces/
├── rns_lora_interface.c/h
├── rns_wifi_interface.c/h

examples/
└── 03_file_transfer/
```

---

### Phase 4: Production Hardening (Weeks 25-30)

**Goal:** Stability, security, documentation

**Deliverables:**
1. ✅ Security audit
   - Buffer overflow prevention
   - Integer overflow checks
   - Constant-time crypto (timing attacks)
   - Secure random number generation
   - Memory wiping for secrets

2. ✅ Error handling
   - Graceful degradation
   - Error codes (enum)
   - Logging infrastructure
   - Watchdog timer integration

3. ✅ Power management
   - Sleep mode support
   - Wake-on-packet
   - Low-power LoRa modes

4. ✅ Documentation
   - API reference (Doxygen)
   - Integration guide
   - Example projects
   - Porting guide for other MCUs

5. ✅ Interoperability testing
   - Test with Python RNS
   - Test with RNode firmware
   - Protocol conformance tests

**Success Criteria:**
- Zero crashes in 7-day stress test
- Interoperates with Python RNS network
- API documentation complete
- Security review passes

**Files Created:**
```
docs/
├── API.md
├── PORTING.md
├── SECURITY.md
└── Doxyfile

tests/
└── interop/
    ├── test_with_python.py
    └── protocol_conformance.c
```

---

## Memory Management Strategy

### Challenge

ESP32 has **520KB SRAM** shared between:
- FreeRTOS kernel
- WiFi/BT drivers (if enabled)
- Application code
- Packet buffers
- Routing tables

Fragmentation is a serious risk with frequent malloc/free.

### Solution: Hybrid Approach

#### 1. **Static Allocation** (Preferred)

Pre-allocate fixed-size pools at startup:

```c
// rns_mem.h
#define RNS_PACKET_POOL_SIZE  8
#define RNS_PATH_TABLE_SIZE   64
#define RNS_LINK_TABLE_SIZE   8

typedef struct {
    rns_packet_t packets[RNS_PACKET_POOL_SIZE];
    uint8_t packet_data[RNS_PACKET_POOL_SIZE][RNS_MTU];
    uint32_t allocated_bitmap;  // 1 = in use, 0 = free

    rns_path_t paths[RNS_PATH_TABLE_SIZE];
    uint64_t path_bitmap;

    rns_link_t links[RNS_LINK_TABLE_SIZE];
    uint8_t link_bitmap;
} rns_memory_pool_t;

extern rns_memory_pool_t rns_pool;

// Allocation
rns_packet_t* rns_packet_alloc(void) {
    for (int i = 0; i < RNS_PACKET_POOL_SIZE; i++) {
        if (!(rns_pool.allocated_bitmap & (1 << i))) {
            rns_pool.allocated_bitmap |= (1 << i);
            return &rns_pool.packets[i];
        }
    }
    return NULL;  // Pool exhausted
}

// Deallocation
void rns_packet_free(rns_packet_t *pkt) {
    int index = pkt - rns_pool.packets;
    rns_pool.allocated_bitmap &= ~(1 << index);
}
```

**Advantages:**
- Zero fragmentation
- Predictable memory usage
- Fast allocation (no syscalls)
- Easy to audit max memory

**Disadvantages:**
- Wastes memory if pool oversized
- Pool exhaustion if undersized

#### 2. **Dynamic Allocation** (Careful Use)

For variable-size data (e.g., application payloads):

```c
// Use FreeRTOS heap (pvPortMalloc/vPortFree)
void* rns_malloc(size_t size) {
    void *ptr = pvPortMalloc(size);
    if (ptr == NULL) {
        ESP_LOGE(TAG, "Allocation failed: %zu bytes", size);
        // Trigger garbage collection or pool cleanup
    }
    return ptr;
}

void rns_free(void *ptr) {
    vPortFree(ptr);
}
```

**Guidelines:**
- Only for data that can't be pooled
- Always check return value
- Free ASAP (don't hold long-term)
- Monitor heap watermark (`xPortGetFreeHeapSize()`)

#### 3. **Stack Allocation** (For Temporary Buffers)

```c
// Encrypt a packet (stack buffer for intermediate steps)
int rns_packet_encrypt(rns_packet_t *pkt) {
    uint8_t plaintext[RNS_MTU];  // Stack allocation
    uint8_t iv[16];

    // ... encryption logic ...

    return 0;  // Stack freed automatically
}
```

**Guidelines:**
- Small buffers only (< 1KB)
- ESP32 task stacks are 4KB by default
- Watch for stack overflow (FreeRTOS checks)

---

### Garbage Collection Strategy

Periodically reclaim stale resources:

```c
void rns_gc_run(void) {
    uint32_t now = rns_time_ms();

    // Expire old paths
    for (int i = 0; i < RNS_PATH_TABLE_SIZE; i++) {
        if (rns_pool.path_bitmap & (1ULL << i)) {
            rns_path_t *path = &rns_pool.paths[i];
            if (now > path->expires_at) {
                rns_path_free(path);
            }
        }
    }

    // Close stale links
    for (int i = 0; i < RNS_LINK_TABLE_SIZE; i++) {
        if (rns_pool.link_bitmap & (1 << i)) {
            rns_link_t *link = &rns_pool.links[i];
            if (now - link->last_rx > RNS_LINK_TIMEOUT) {
                rns_link_close(link);
            }
        }
    }

    // Evict old packet hashes
    rns_packet_hashlist_prune();
}
```

Run from timer (every 60 seconds):

```c
esp_timer_handle_t gc_timer;
esp_timer_create_args_t gc_args = {
    .callback = rns_gc_run,
    .name = "rns_gc"
};
esp_timer_create(&gc_args, &gc_timer);
esp_timer_start_periodic(gc_timer, 60 * 1000000);  // 60s
```

---

## Cryptography Library Selection

### Requirements

1. **Algorithms needed:**
   - X25519 (ECDH on Curve25519)
   - Ed25519 (EdDSA signatures)
   - AES-256-CBC
   - SHA-256, SHA-512
   - HMAC-SHA256
   - HKDF (RFC 5869)

2. **Constraints:**
   - Small code size (< 100KB)
   - Low RAM usage (< 10KB for operations)
   - ESP32 hardware acceleration support
   - Battle-tested (production use)
   - Permissive license

### Option Comparison

| Library | Size | Algorithms | HW Accel | License | ESP32 Support |
|---------|------|------------|----------|---------|---------------|
| **mbedTLS** | ~150KB | ✅ All | ✅ AES, SHA | Apache 2.0 | ✅ Excellent |
| **libsodium** | ~180KB | ✅ All | ❌ No | ISC | ⚠️ Manual build |
| **micro-ecc** | ~15KB | ❌ No Ed25519 | ❌ No | BSD | ⚠️ Incomplete |
| **BearSSL** | ~80KB | ❌ No X25519 | ❌ No | MIT | ⚠️ Manual port |
| **TweetNaCl** | ~10KB | ✅ All | ❌ No | Public | ❌ Slow |
| **wolfSSL** | ~200KB | ✅ All | ✅ Yes | GPLv2/Comm | ✅ Good |

### **RECOMMENDATION: mbedTLS**

**Rationale:**

1. ✅ **Official ESP-IDF integration** - Already included, tested, maintained
2. ✅ **Hardware acceleration** - Uses ESP32 AES and SHA engines automatically
3. ✅ **All algorithms** - Complete coverage (X25519, Ed25519, AES, SHA, HMAC)
4. ✅ **Small footprint** - Can disable unused features (TLS, X.509, etc.)
5. ✅ **Battle-tested** - Used in billions of IoT devices
6. ✅ **Permissive license** - Apache 2.0 (compatible with Reticulum)

**Configuration:**

```c
// sdkconfig (ESP-IDF menuconfig)
CONFIG_MBEDTLS_HARDWARE_AES=y           // Use HW AES
CONFIG_MBEDTLS_HARDWARE_SHA=y           // Use HW SHA
CONFIG_MBEDTLS_ECDH_C=y                 // X25519 support
CONFIG_MBEDTLS_ECDSA_C=y                // Ed25519 support
CONFIG_MBEDTLS_AES_C=y
CONFIG_MBEDTLS_SHA256_C=y
CONFIG_MBEDTLS_SHA512_C=y
CONFIG_MBEDTLS_HKDF_C=y

// Disable unused features
CONFIG_MBEDTLS_TLS_ENABLED=n            // No TLS
CONFIG_MBEDTLS_X509_USE_C=n             // No certificates
CONFIG_MBEDTLS_SSL_CLI_C=n              // No SSL client
```

**API Wrappers:**

```c
// rns_crypto/rns_x25519.c
#include <mbedtls/ecdh.h>

int rns_x25519_keypair(uint8_t *private_key, uint8_t *public_key) {
    mbedtls_ecdh_context ctx;
    mbedtls_ecdh_init(&ctx);

    // Generate keypair
    int ret = mbedtls_ecdh_gen_public(
        &ctx.grp, &ctx.d, &ctx.Q,
        mbedtls_ctr_drbg_random, &ctr_drbg
    );

    // Export keys
    mbedtls_mpi_write_binary(&ctx.d, private_key, 32);
    mbedtls_ecp_point_write_binary(&ctx.grp, &ctx.Q,
        MBEDTLS_ECP_PF_COMPRESSED, &olen, public_key, 32);

    mbedtls_ecdh_free(&ctx);
    return ret;
}

int rns_x25519_shared_secret(
    const uint8_t *our_private,
    const uint8_t *their_public,
    uint8_t *shared_secret
) {
    // ... ECDH computation ...
}
```

**Alternative if mbedTLS too large:** Combine **micro-ecc** (X25519) + **TweetNaCl** (Ed25519) + **mbedTLS AES** (total ~50KB).

---

## File Structure & Module Organization

### Directory Layout

```
reticulum_esp32/
│
├── CMakeLists.txt                 # ESP-IDF build config
├── sdkconfig                      # ESP32 configuration
├── Kconfig.projbuild              # menuconfig options
├── README.md
├── LICENSE
│
├── components/
│   └── rns/                       # Main RNS component
│       ├── CMakeLists.txt
│       ├── include/
│       │   └── rns/               # Public API headers
│       │       ├── rns.h          # Main include
│       │       ├── rns_packet.h
│       │       ├── rns_identity.h
│       │       ├── rns_destination.h
│       │       ├── rns_link.h
│       │       ├── rns_transport.h
│       │       ├── rns_interface.h
│       │       └── rns_types.h
│       │
│       ├── src/
│       │   ├── core/
│       │   │   ├── rns_packet.c
│       │   │   ├── rns_identity.c
│       │   │   ├── rns_destination.c
│       │   │   ├── rns_link.c
│       │   │   ├── rns_transport.c
│       │   │   ├── rns_announce.c
│       │   │   ├── rns_path.c
│       │   │   └── rns_resource.c
│       │   │
│       │   ├── crypto/
│       │   │   ├── rns_x25519.c
│       │   │   ├── rns_ed25519.c
│       │   │   ├── rns_aes.c
│       │   │   ├── rns_sha.c
│       │   │   ├── rns_hkdf.c
│       │   │   └── rns_token.c
│       │   │
│       │   ├── interfaces/
│       │   │   ├── rns_interface.c
│       │   │   ├── rns_serial_interface.c
│       │   │   ├── rns_lora_interface.c
│       │   │   └── rns_wifi_interface.c
│       │   │
│       │   ├── utils/
│       │   │   ├── rns_mem.c       # Memory pools
│       │   │   ├── rns_list.c      # Linked lists
│       │   │   ├── rns_hashtable.c # Hash tables
│       │   │   ├── rns_time.c      # Timing utilities
│       │   │   └── rns_log.c       # Logging
│       │   │
│       │   └── platform/
│       │       ├── rns_pal_esp32.c # ESP32-specific (timers, NVS, etc.)
│       │       └── rns_pal.h       # Platform abstraction interface
│       │
│       └── test/
│           ├── test_crypto.c
│           ├── test_packet.c
│           ├── test_identity.c
│           └── test_vectors/
│
├── examples/
│   ├── 01_hello_world/            # Basic packet send/receive
│   ├── 02_link_example/           # Link establishment
│   ├── 03_multihop/               # Multi-hop routing
│   ├── 04_file_transfer/          # Resource transfer
│   └── 05_lora_mesh/              # LoRa mesh network
│
├── docs/
│   ├── API_REFERENCE.md
│   ├── PORTING_GUIDE.md           # Port to other MCUs
│   ├── PROTOCOL.md                # Wire protocol spec
│   ├── MEMORY_OPTIMIZATION.md
│   └── SECURITY.md
│
└── tools/
    ├── interop_test.py            # Test with Python RNS
    └── packet_inspector.py        # Debug tool
```

### Module Dependencies

```
┌─────────────────┐
│   Application   │
└────────┬────────┘
         │
┌────────▼────────────────────────────────────┐
│  rns.h (Public API)                         │
│  - rns_init()                               │
│  - rns_packet_send()                        │
│  - rns_destination_create()                 │
│  - rns_link_establish()                     │
└────────┬────────────────────────────────────┘
         │
    ┌────┴────┬────────┬──────────┐
    │         │        │          │
┌───▼───┐ ┌──▼──┐ ┌───▼────┐ ┌───▼────────┐
│Packet │ │Link │ │Destin- │ │Transport   │
│       │ │     │ │ation   │ │            │
└───┬───┘ └──┬──┘ └───┬────┘ └───┬────────┘
    │        │        │          │
    └────────┴────────┴──────────┘
             │
        ┌────▼─────┐
        │ Identity │
        │          │
        └────┬─────┘
             │
        ┌────▼─────┐
        │  Crypto  │
        └────┬─────┘
             │
        ┌────▼─────┐
        │ mbedTLS  │
        └──────────┘
```

**Dependency Rules:**
- **No circular dependencies** - Enforce with build checks
- **Layered architecture** - Higher layers can call lower, not reverse
- **Interface abstraction** - Platform-specific code in `rns_pal_*`

---

## Platform Abstraction Layer

### Purpose

Enable porting to non-ESP32 platforms (STM32, nRF52, Linux, etc.)

### PAL Interface (rns_pal.h)

```c
// Time functions
uint32_t rns_pal_time_ms(void);        // Milliseconds since boot
uint64_t rns_pal_time_us(void);        // Microseconds
void rns_pal_delay_ms(uint32_t ms);

// Threading (optional - can be single-threaded)
typedef void* rns_task_handle_t;
rns_task_handle_t rns_pal_task_create(
    void (*task_func)(void*),
    const char *name,
    uint32_t stack_size,
    void *param
);
void rns_pal_task_delete(rns_task_handle_t task);

// Mutex (for multi-threaded builds)
typedef void* rns_mutex_t;
rns_mutex_t rns_pal_mutex_create(void);
void rns_pal_mutex_lock(rns_mutex_t mutex);
void rns_pal_mutex_unlock(rns_mutex_t mutex);
void rns_pal_mutex_delete(rns_mutex_t mutex);

// Storage (identity persistence)
int rns_pal_storage_read(const char *key, uint8_t *buf, size_t len);
int rns_pal_storage_write(const char *key, const uint8_t *data, size_t len);

// Logging
void rns_pal_log(int level, const char *tag, const char *format, ...);

// Random number generation (cryptographically secure)
int rns_pal_random_bytes(uint8_t *buf, size_t len);

// Hardware interfaces (optional - can be NULL if not used)
void* rns_pal_uart_init(int port, int baudrate);
int rns_pal_uart_read(void *handle, uint8_t *buf, size_t len, uint32_t timeout);
int rns_pal_uart_write(void *handle, const uint8_t *data, size_t len);
```

### ESP32 Implementation (rns_pal_esp32.c)

```c
#include "rns_pal.h"
#include "freertos/FreeRTOS.h"
#include "esp_timer.h"
#include "nvs_flash.h"
#include "esp_random.h"

uint32_t rns_pal_time_ms(void) {
    return (uint32_t)(esp_timer_get_time() / 1000);
}

uint64_t rns_pal_time_us(void) {
    return esp_timer_get_time();
}

void rns_pal_delay_ms(uint32_t ms) {
    vTaskDelay(ms / portTICK_PERIOD_MS);
}

rns_task_handle_t rns_pal_task_create(
    void (*task_func)(void*),
    const char *name,
    uint32_t stack_size,
    void *param
) {
    TaskHandle_t handle;
    xTaskCreate(task_func, name, stack_size, param, 5, &handle);
    return (rns_task_handle_t)handle;
}

int rns_pal_storage_read(const char *key, uint8_t *buf, size_t len) {
    nvs_handle_t nvs;
    esp_err_t err = nvs_open("rns", NVS_READONLY, &nvs);
    if (err != ESP_OK) return -1;

    err = nvs_get_blob(nvs, key, buf, &len);
    nvs_close(nvs);
    return (err == ESP_OK) ? len : -1;
}

int rns_pal_random_bytes(uint8_t *buf, size_t len) {
    esp_fill_random(buf, len);
    return 0;
}

// ... etc
```

### Linux/POSIX Implementation (for development/testing)

```c
#include <sys/time.h>
#include <pthread.h>
#include <unistd.h>

uint32_t rns_pal_time_ms(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000 + tv.tv_usec / 1000;
}

rns_task_handle_t rns_pal_task_create(...) {
    pthread_t *thread = malloc(sizeof(pthread_t));
    pthread_create(thread, NULL, (void*(*)(void*))task_func, param);
    return (rns_task_handle_t)thread;
}

// ... etc
```

**Benefit:** Core RNS code compiles unchanged on Linux for rapid testing.

---

## Testing Strategy

### Multi-Level Testing

```
┌──────────────────────────────────────────────┐
│ Level 4: Interoperability Tests              │
│  - Test with Python RNS network              │
│  - Protocol conformance                      │
│  - Cross-platform compatibility              │
└──────────────────────────────────────────────┘
                    ▲
┌──────────────────────────────────────────────┐
│ Level 3: Integration Tests                   │
│  - Multi-hop routing                         │
│  - Link establishment end-to-end             │
│  - Resource transfer                         │
└──────────────────────────────────────────────┘
                    ▲
┌──────────────────────────────────────────────┐
│ Level 2: Component Tests                     │
│  - Packet pack/unpack                        │
│  - Path table operations                     │
│  - Interface TX/RX                           │
└──────────────────────────────────────────────┘
                    ▲
┌──────────────────────────────────────────────┐
│ Level 1: Unit Tests                          │
│  - Crypto primitives (test vectors)          │
│  - Hash table operations                     │
│  - Memory pool allocation                    │
└──────────────────────────────────────────────┘
```

### Unit Test Framework: Unity

```c
// test/test_crypto.c
#include "unity.h"
#include "rns_x25519.h"

void test_x25519_test_vector_1(void) {
    // RFC 7748 test vector
    uint8_t scalar[32] = {
        0xa5, 0x46, 0xe3, 0x6b, 0xf0, 0x52, 0x7c, 0x9d,
        // ...
    };
    uint8_t basepoint[32] = {
        0xe6, 0xdb, 0x68, 0x67, 0x58, 0x30, 0x30, 0xdb,
        // ...
    };
    uint8_t expected[32] = {
        0xc3, 0xda, 0x55, 0x37, 0x9d, 0xe9, 0xc6, 0x90,
        // ...
    };

    uint8_t result[32];
    rns_x25519_scalarmult(result, scalar, basepoint);

    TEST_ASSERT_EQUAL_HEX8_ARRAY(expected, result, 32);
}

void test_x25519_keypair_generation(void) {
    uint8_t private_key[32];
    uint8_t public_key[32];

    int ret = rns_x25519_keypair(private_key, public_key);
    TEST_ASSERT_EQUAL(0, ret);

    // Public key should be non-zero
    bool all_zero = true;
    for (int i = 0; i < 32; i++) {
        if (public_key[i] != 0) all_zero = false;
    }
    TEST_ASSERT_FALSE(all_zero);
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_x25519_test_vector_1);
    RUN_TEST(test_x25519_keypair_generation);
    // ... more tests
    return UNITY_END();
}
```

### Interoperability Testing

```python
# tools/interop_test.py
import RNS
import serial
import struct

# Connect to ESP32 over serial
esp32 = serial.Serial("/dev/ttyUSB0", 115200)

# Initialize Python RNS
reticulum = RNS.Reticulum()
identity = RNS.Identity()
destination = RNS.Destination(
    identity,
    RNS.Destination.IN,
    RNS.Destination.SINGLE,
    "test_app",
    "test_aspect"
)

def packet_received(data, packet):
    print(f"Received from ESP32: {data}")
    # Verify packet structure matches
    assert len(packet.destination_hash) == 16
    assert packet.packet_type == RNS.Packet.DATA

destination.set_packet_callback(packet_received)

# Send packet to ESP32
packet = RNS.Packet(destination, b"Hello ESP32")
packet.send()

# Wait for ESP32 response
while True:
    if esp32.in_waiting > 0:
        data = esp32.read(esp32.in_waiting)
        # Verify ESP32 sent valid RNS packet
        verify_packet(data)
```

### Hardware-in-Loop Testing

```c
// ESP32 test fixture
void test_lora_link(void) {
    // Two ESP32 boards with LoRa modules
    // Automated test rig

    // Board A: Transmitter
    rns_packet_t *pkt = rns_packet_create();
    rns_packet_set_data(pkt, "test", 4);
    rns_packet_send(pkt);

    // Board B: Receiver (GPIO connected to test PC)
    // PC verifies reception via serial
}
```

### Continuous Integration

```yaml
# .github/workflows/ci.yml
name: RNS ESP32 CI

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install ESP-IDF
        run: |
          git clone --depth 1 -b v5.2 https://github.com/espressif/esp-idf.git
          cd esp-idf && ./install.sh
      - name: Build tests
        run: |
          . esp-idf/export.sh
          cd tests && idf.py build
      - name: Run host tests (Linux PAL)
        run: |
          cd tests && make host-test
          ./test_runner

  interop-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Install Python RNS
        run: pip install rns
      - name: Run interop tests
        run: python tools/interop_test.py
```

---

## Integration & Deployment

### ESP-IDF Project Structure

```c
// main/main.c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "rns/rns.h"

void app_main(void) {
    // Initialize RNS
    rns_config_t config = {
        .storage_path = "/spiffs/rns",
        .log_level = RNS_LOG_INFO,
    };
    rns_init(&config);

    // Create identity (or load from storage)
    rns_identity_t *identity = rns_identity_load("my_identity");
    if (!identity) {
        identity = rns_identity_create();
        rns_identity_save(identity, "my_identity");
    }

    // Create destination
    rns_destination_t *dest = rns_destination_create(
        identity,
        RNS_DEST_IN,
        RNS_DEST_SINGLE,
        "example_app",
        "node"
    );

    // Set packet callback
    rns_destination_set_callback(dest, on_packet_received, NULL);

    // Add LoRa interface
    rns_interface_t *lora = rns_lora_interface_create(
        "LoRa",
        433000000,  // 433 MHz
        125000,     // 125 kHz bandwidth
        7,          // SF7
        5           // CR 4/5
    );
    rns_add_interface(lora);

    // Announce presence
    rns_destination_announce(dest, NULL, 0);

    printf("RNS node started\n");
    printf("Destination hash: ");
    for (int i = 0; i < 16; i++) {
        printf("%02x", dest->hash[i]);
    }
    printf("\n");

    // Main loop
    while (1) {
        vTaskDelay(1000 / portTICK_PERIOD_MS);
    }
}

void on_packet_received(uint8_t *data, size_t len, void *context) {
    printf("Received %zu bytes: %.*s\n", len, (int)len, data);
}
```

### Build & Flash

```bash
# Set up ESP-IDF environment
. ~/esp/esp-idf/export.sh

# Configure project
idf.py menuconfig
# Select:
#  - Component config → mbedTLS → Enable ECDH, Ed25519, AES, SHA
#  - Component config → RNS → Set packet pool size, path table size

# Build
idf.py build

# Flash to ESP32
idf.py -p /dev/ttyUSB0 flash

# Monitor output
idf.py -p /dev/ttyUSB0 monitor
```

### Integration with RNode Firmware

RNode firmware already runs on ESP32. Options:

1. **RNS-ESP32 as standalone** - Direct LoRa control
2. **RNS-ESP32 + RNode** - Use RNode as modem via KISS protocol
3. **Merge into RNode** - Add RNS protocol to RNode firmware

**Recommended:** Start with option 1 (standalone) for maximum control, provide option 2 for compatibility.

---

## Risk Analysis & Mitigation

### Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **Memory exhaustion** | Medium | High | Static pools, tunable sizes, GC, profiling |
| **Crypto implementation bugs** | Low | Critical | Use mbedTLS, test vectors, audit |
| **Protocol incompatibility** | Medium | High | Interop tests with Python RNS, conformance tests |
| **Radio driver issues** | Medium | Medium | Test with RNode first, then direct SPI |
| **Real-time deadline misses** | Low | Medium | Profiling, priority tuning, watchdog |
| **Flash wear (NVS)** | Low | Low | Limit writes, use wear leveling |
| **Fragmentation** | Medium | Medium | Static allocation, pool-based memory |

### Security Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **Buffer overflow** | Medium | Critical | Bounds checking, static analysis, fuzzing |
| **Timing attacks on crypto** | Low | High | Use constant-time mbedTLS functions |
| **Weak RNG** | Low | Critical | Use ESP32 hardware RNG (`esp_random`) |
| **Key extraction** | Low | Medium | Secure boot, flash encryption, memory wiping |
| **Replay attacks** | Medium | Medium | Nonce/counter in packets, timestamp validation |

### Operational Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| **ESP-IDF version changes** | High | Low | Pin to specific version (v5.2), test upgrades |
| **Hardware unavailability** | Low | Medium | Support multiple ESP32 variants (S2, S3, C3) |
| **Community support** | Low | Low | Comprehensive documentation, examples |
| **Scope creep** | High | Medium | Phased implementation, MVP focus |

---

## Appendix A: Constants & Configuration

```c
// rns_config.h
#ifndef RNS_CONFIG_H
#define RNS_CONFIG_H

// Protocol constants (from Reticulum spec)
#define RNS_MTU                     500     // Maximum packet size
#define RNS_MTU_MIN                 219     // Absolute minimum
#define RNS_HEADER_MIN_SIZE         4       // flags + hops + (reserved)
#define RNS_HEADER_MAX_SIZE         19      // All fields
#define RNS_TRUNCATED_HASHLENGTH    16      // Destination hash size
#define RNS_MAX_HOPS                128     // Maximum hop count
#define RNS_TIMEOUT_PER_HOP         6       // Seconds per hop

// Link constants
#define RNS_ECPUBSIZE               64      // X25519 + Ed25519 public keys
#define RNS_KEYSIZE                 32      // Symmetric key size
#define RNS_MDU                     383     // Max data unit per link
#define RNS_LINK_TIMEOUT            (60*12) // 12 minutes
#define RNS_KEEPALIVE_TIMEOUT       360     // 6 minutes

// Resource transfer
#define RNS_RESOURCE_WINDOW         4       // Transfer window size
#define RNS_RESOURCE_MAX_RETRIES    3

// Memory pool sizes (tunable per platform)
#define RNS_PACKET_POOL_SIZE        8       // Simultaneous packets
#define RNS_PATH_TABLE_SIZE         64      // Cached paths
#define RNS_LINK_TABLE_SIZE         8       // Active links
#define RNS_ANNOUNCE_QUEUE_SIZE     16      // Pending announces
#define RNS_PACKET_HASHLIST_SIZE    256     // Duplicate detection
#define RNS_IDENTITY_CACHE_SIZE     32      // Known identities

// Timing
#define RNS_ANNOUNCE_CAP            2       // Max 2% bandwidth for announces
#define RNS_PATH_EXPIRE_TIME        (60*60*24*7)  // 1 week
#define RNS_PATH_REQUEST_TIMEOUT    15      // Seconds
#define RNS_GC_INTERVAL             60      // Garbage collection interval

// Packet types
#define RNS_PACKET_DATA             0x00
#define RNS_PACKET_ANNOUNCE         0x01
#define RNS_PACKET_LINKREQUEST      0x02
#define RNS_PACKET_PROOF            0x03

// Destination types
#define RNS_DEST_SINGLE             0x00
#define RNS_DEST_GROUP              0x01
#define RNS_DEST_PLAIN              0x02
#define RNS_DEST_LINK               0x03

// Destination directions
#define RNS_DEST_IN                 0x01    // Can receive
#define RNS_DEST_OUT                0x02    // Can send

#endif // RNS_CONFIG_H
```

---

## Appendix B: API Example

```c
// Complete example: Echo server

#include "rns/rns.h"
#include <stdio.h>

void on_packet(uint8_t *data, size_t len, rns_packet_t *pkt, void *ctx) {
    printf("Received: %.*s\n", (int)len, data);

    // Echo back
    rns_destination_t *dest = (rns_destination_t*)ctx;
    rns_packet_t *response = rns_packet_create(pkt->destination_hash);
    rns_packet_set_data(response, data, len);
    rns_packet_send(response);
}

void app_main(void) {
    // Initialize
    rns_init(NULL);

    // Load or create identity
    rns_identity_t *id = rns_identity_load("echo_server");
    if (!id) {
        id = rns_identity_create();
        rns_identity_save(id, "echo_server");
    }

    // Create destination
    rns_destination_t *dest = rns_destination_create(
        id, RNS_DEST_IN, RNS_DEST_SINGLE, "echo", "server"
    );
    rns_destination_set_callback(dest, on_packet, dest);

    // Add serial interface
    rns_interface_t *serial = rns_serial_interface_create(
        "Serial", UART_NUM_1, 115200
    );
    rns_add_interface(serial);

    // Announce
    rns_destination_announce(dest, NULL, 0);

    printf("Echo server started\n");
    printf("Hash: ");
    rns_hexdump(dest->hash, 16);

    // Event loop
    while (1) {
        rns_update();  // Process packets
        vTaskDelay(10 / portTICK_PERIOD_MS);
    }
}
```

---

## Conclusion

This plan provides a comprehensive roadmap for porting Reticulum to ESP32 in C. Key decisions:

1. **Language:** C for maximum portability and efficiency
2. **Crypto:** mbedTLS for production-grade security
3. **Memory:** Static pools with hybrid allocation
4. **Phases:** 0-3 over ~30 weeks
5. **Testing:** Multi-level with interoperability focus

**Next Steps:**
1. Set up ESP-IDF development environment
2. Create repository structure
3. Begin Phase 0: Crypto integration
4. Establish CI/CD pipeline
5. Start documentation

**Success Metrics:**
- < 100KB RAM usage
- Full protocol compatibility with Python RNS
- Production-ready by Phase 4

---

**Document Version:** 1.0
**Last Updated:** 2026-01-21
