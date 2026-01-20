# Reticulum Rust no_std Port - Comprehensive Implementation Plan

## Executive Summary

This document outlines a detailed plan for porting **Reticulum** (a cryptography-based networking stack) from Python to Rust with **no_std** constraints. Reticulum is a ~22,763 line Python codebase that provides decentralized mesh networking with strong cryptographic guarantees. The port targets embedded systems, microcontrollers, and resource-constrained environments where the standard library is unavailable.

**Target Features:**
- Full protocol compatibility with existing Reticulum network
- Embedded-first design (no_std, no heap allocator required for core)
- Support for MCUs with ≥64KB RAM and ≥256KB flash
- Minimal external dependencies
- Security-first implementation with constant-time cryptography

---

## 1. Current State Analysis

### 1.1 Reticulum Architecture Overview

**Core Components (~22,763 lines of Python):**
- **Reticulum.py** (78KB): Main instance, initialization, configuration
- **Identity.py** (35KB): Cryptographic identity management (X25519/Ed25519)
- **Link.py** (72KB): Verified encrypted link establishment and management
- **Transport.py** (186KB): Routing engine, path discovery, announce propagation
- **Destination.py** (31KB): Network endpoint management
- **Packet.py** (25KB): Packet creation, parsing, encryption
- **Resource.py** (59KB): Large file transfer with windowing/compression
- **Channel.py** (26KB): High-level message passing interface
- **Buffer.py** (13KB): Stream I/O interface
- **Discovery.py** (40KB): Interface discovery and bootstrapping

**Interface Layer:**
- 15+ interface types: Serial, TCP/UDP, LoRa (RNode), KISS, AX.25, I2P, AutoInterface
- Hardware abstraction for heterogeneous physical media

**Cryptography Layer:**
- X25519 key exchange (256-bit ECDH)
- Ed25519 signatures (256-bit EdDSA)
- AES-256-CBC with PKCS7 padding
- HMAC-SHA256 for authentication
- HKDF for key derivation
- Token system (Fernet-like)
- Pure Python fallback implementations available

### 1.2 Key Protocol Specifications

**Packet Format:**
```
[Flags:1][Hops:1][Destination Hash:16][Context:1][Data:0-465]
```

- **MTU**: 500 bytes (configurable, absolute minimum 219 bytes)
- **Header Size**: 2-34 bytes (min: 2+1+16, max: 2+1+16+16)
- **IFAC Salt**: 32 bytes fixed
- **Truncated Hash**: 128 bits (16 bytes)
- **MDU**: 465 bytes (500 - 34 - 1)
- **Encrypted MDU**: 383 bytes
- **Max Hops**: 128

**Transport Types:**
- `BROADCAST` (0x00): Single-hop broadcast
- `TRANSPORT` (0x01): Multi-hop routed
- `RELAY` (0x02): Relay mode
- `TUNNEL` (0x03): Tunnel mode

**Destination Types:**
- `SINGLE` (asymmetric, per-packet ephemeral keys)
- `GROUP` (symmetric, pre-shared keys)
- `PLAIN` (unencrypted)
- `LINK` (forward secrecy over established links)

**Timing Constants:**
- Per-hop timeout: 6 seconds
- Path expiry: 7 days
- Link timeout: STALE_TIME × 1.25
- Announce cap: 2% of interface bandwidth
- Minimum bitrate: 5 bits/second

### 1.3 Python Dependencies Analysis

**External Dependencies:**
- `cryptography>=3.4.7` (PyCA/OpenSSL) - **PRIMARY BLOCKER**
- `pyserial>=3.5` - Serial port communication

**Vendored Pure-Python Components:**
- `umsgpack` - MessagePack serialization
- `configobj` - INI-style config parsing
- `i2plib` - I2P network support
- Pure Python crypto fallbacks (X25519, Ed25519, SHA, AES)

**Standard Library Usage:**
- Threading, multiprocessing, signals
- File I/O, sockets, serial ports
- Time, struct, array, hashlib
- OS-specific platform detection

---

## 2. no_std Rust Requirements & Constraints

### 2.1 no_std Environment Characteristics

**Available:**
- Core language features (no heap required)
- `core` library (no allocator needed)
- `alloc` crate (requires global allocator)
- Embedded HAL abstractions
- Const evaluation, panic handlers
- Static memory allocation

**NOT Available:**
- `std::fs`, `std::io`, `std::net` (file/network I/O)
- Threading primitives from std
- Dynamic allocation (without explicit allocator)
- OS-specific APIs
- `std::collections` (use `alloc::collections` instead)

### 2.2 Target Hardware Profiles

**Tier 1 - Minimal (LoRa Node):**
- MCU: ARM Cortex-M4 (STM32, nRF52, ESP32)
- RAM: 64-128KB
- Flash: 256-512KB
- Features: Core protocol + single interface (Serial/LoRa)

**Tier 2 - Standard (Gateway):**
- MCU: ARM Cortex-M7 / RISC-V
- RAM: 512KB-1MB
- Flash: 1-2MB
- Features: Full routing, multiple interfaces, persistence

**Tier 3 - Advanced (Embedded Linux):**
- Platform: OpenWrt routers, RPi Zero
- RAM: 32-128MB
- Flash: 8-32MB
- Features: All protocol features, std fallback

### 2.3 Memory Constraints

**Static Allocation Budget (Tier 1):**
- Packet buffers: 2-4 × 500 bytes = 1-2KB
- Path table: 32 entries × 45 bytes = 1.44KB
- Link table: 8 entries × 48 bytes = 384 bytes
- Announce queue: 16 entries × 500 bytes = 8KB
- Crypto state: ~4KB (key material, ciphers)
- Interface buffers: 2-4KB
- **Total**: ~20KB minimum, ~32KB comfortable

**Stack Usage:**
- Packet processing: ~2KB max
- Crypto operations: ~4KB max
- Interface I/O: ~1KB
- **Total**: ~8KB minimum stack

---

## 3. Rust no_std Architecture Design

### 3.1 Crate Structure

```
reticulum-core/          (no_std, no allocator required for basic ops)
├── src/
│   ├── lib.rs           (Feature gates, no_std setup)
│   ├── protocol/        (Protocol constants and types)
│   ├── crypto/          (Cryptographic primitives)
│   ├── packet/          (Packet encoding/decoding)
│   ├── identity/        (Identity management)
│   ├── destination/     (Destination logic)
│   └── utils/           (Helpers, buffer management)
│
reticulum-transport/     (requires alloc)
├── src/
│   ├── lib.rs
│   ├── routing.rs       (Path table, announce handling)
│   ├── link.rs          (Link establishment)
│   ├── resource.rs      (File transfer)
│   └── tables.rs        (Transport tables)
│
reticulum-interfaces/    (Hardware abstraction)
├── src/
│   ├── lib.rs
│   ├── traits.rs        (Interface trait definitions)
│   ├── serial.rs        (Serial interface - embedded-hal)
│   ├── lora.rs          (LoRa/RNode interface)
│   └── tcp_udp.rs       (Network interfaces - std only)
│
reticulum-utils/         (Optional features)
├── src/
│   ├── config.rs        (Config parsing - alloc)
│   ├── persistence.rs   (Storage layer - std)
│   └── channel.rs       (High-level API)
│
reticulum/               (Umbrella crate with feature flags)
└── src/
    └── lib.rs           (Re-exports based on features)
```

### 3.2 Feature Flag Strategy

```toml
[features]
default = ["alloc"]

# Core features
alloc = []                    # Enable heap allocation
std = ["alloc"]              # Enable std library

# Protocol features
routing = ["alloc"]          # Full routing/transport
links = ["alloc"]            # Link establishment
resources = ["alloc"]        # Resource transfers
channels = ["alloc"]         # High-level channels

# Cryptography backends
crypto-dalek = []            # curve25519-dalek (no_std)
crypto-ring = ["std"]        # ring (faster, needs std)
crypto-software = []         # Pure Rust fallback

# Interface types
interface-serial = ["embedded-hal"]
interface-lora = []
interface-net = ["std"]
interface-i2p = ["std", "alloc"]

# Storage and config
persistence = ["std"]
config = ["alloc"]

# Profile presets
profile-minimal = ["crypto-dalek", "interface-serial"]
profile-gateway = ["alloc", "routing", "links", "crypto-dalek"]
profile-full = ["std", "routing", "links", "resources", "channels", "persistence"]
```

### 3.3 Core Type Definitions

```rust
// Core protocol types (no_std compatible)
#[repr(u8)]
pub enum PacketType {
    Data = 0x00,
    Announce = 0x01,
    LinkRequest = 0x02,
    Proof = 0x03,
}

#[repr(u8)]
pub enum TransportType {
    Broadcast = 0x00,
    Transport = 0x01,
    Relay = 0x02,
    Tunnel = 0x03,
}

pub struct TruncatedHash([u8; 16]);  // 128-bit hash
pub struct FullHash([u8; 32]);       // 256-bit hash

// Zero-copy packet structure
pub struct Packet<'a> {
    flags: u8,
    hops: u8,
    destination: TruncatedHash,
    transport_id: Option<TruncatedHash>,
    context: u8,
    data: &'a [u8],
}

// Const-generic buffer for different sizes
pub struct PacketBuffer<const N: usize> {
    data: [u8; N],
    len: usize,
}

// Identity with owned keys (no heap)
pub struct Identity {
    encryption_key: [u8; 32],
    signing_key: [u8; 32],
    public_key: [u8; 64],
}
```

### 3.4 Memory Management Strategy

**Static Allocation Pattern:**
```rust
// Global packet pool (no allocator needed)
static PACKET_POOL: Mutex<PacketPool<4, 500>> = Mutex::new(PacketPool::new());

pub struct PacketPool<const N: usize, const SIZE: usize> {
    buffers: [PacketBuffer<SIZE>; N],
    free_bitmap: u32,
}

// Per-interface ringbuffers
pub struct InterfaceBuffer<const SIZE: usize> {
    buffer: [u8; SIZE],
    read_pos: AtomicUsize,
    write_pos: AtomicUsize,
}
```

**Allocation Strategy (with `alloc`):**
- Use `BTreeMap` for path/announce tables (no HashMap hasher)
- `Vec` for dynamic packet queues with capacity limits
- `Arc<Mutex<T>>` for shared state (async-friendly)
- Pre-allocate with `with_capacity()` to avoid fragmentation

---

## 4. Phase-by-Phase Implementation Plan

### Phase 1: Foundation & Cryptography (Weeks 1-3)

**Goal:** Establish no_std-compatible cryptographic primitives and protocol types.

**Tasks:**
1. **Project Setup**
   - Create workspace with crate structure
   - Configure no_std in `Cargo.toml`
   - Set up panic handler and allocator stubs
   - Configure CI for cross-compilation (ARM targets)

2. **Protocol Constants & Types** (`reticulum-core/protocol`)
   - Define all packet types, flags, constants
   - Implement `TruncatedHash`, `FullHash` types
   - Create const-generic `PacketBuffer<N>`
   - Zero-copy packet parser/serializer

3. **Cryptography Layer** (`reticulum-core/crypto`)
   - Integrate `curve25519-dalek` (no_std compatible)
   - Implement X25519 key exchange wrapper
   - Implement Ed25519 signature wrapper
   - Add `aes` crate (pure Rust) for AES-256-CBC
   - Implement HMAC-SHA256 using `sha2` crate
   - Implement HKDF key derivation
   - Create `Token` type (Fernet-like)
   - **Constant-time operations audit**

4. **Identity Management** (`reticulum-core/identity`)
   - `Identity` struct with key generation
   - `load_public_key()`, `get_public_key()`
   - `sign()`, `verify()` operations
   - `encrypt()`, `decrypt()` for SINGLE destinations
   - Hash derivation functions
   - Ratchet key support (optional)

**Deliverables:**
- `reticulum-core` crate compiles for `thumbv7em-none-eabihf`
- Unit tests for all crypto operations
- Benchmark suite for crypto performance
- Memory usage report (<16KB static)

**Success Criteria:**
- All crypto operations constant-time
- Compatible hashes/signatures with Python version
- Zero heap allocations in core crypto

---

### Phase 2: Packet Layer (Weeks 4-5)

**Goal:** Complete packet encoding/decoding with encryption support.

**Tasks:**
1. **Packet Structure** (`reticulum-core/packet`)
   - Implement `Packet::pack()` with zero-copy
   - Implement `Packet::unpack()` from raw bytes
   - Header flag packing/unpacking
   - MTU validation and truncation
   - Context type handling

2. **Destination Logic** (`reticulum-core/destination`)
   - `Destination` type with hash derivation
   - Support SINGLE, GROUP, PLAIN, LINK types
   - Per-destination encryption key derivation
   - Aspect naming and hashing

3. **Packet Encryption**
   - SINGLE destination: ephemeral key per packet
   - GROUP destination: symmetric pre-shared key
   - LINK destination: session key support
   - IFAC (Interface Authentication Code) calculation
   - Proof generation and verification

4. **Packet Receipt**
   - `PacketReceipt` structure
   - Implicit/explicit proof handling
   - Timeout calculation (per-hop × hops)
   - Receipt callbacks (trait-based)

**Deliverables:**
- Full packet encode/decode pipeline
- Interoperability tests with Python Reticulum
- Fuzzing harness for packet parser
- Zero-copy packet handling benchmark

**Success Criteria:**
- Packets generated by Rust accepted by Python
- Packets from Python correctly parsed
- MDU calculations match exactly (465 bytes)
- Encrypted MDU = 383 bytes

---

### Phase 3: Interface Abstraction (Weeks 6-7)

**Goal:** Define interface traits and implement core interfaces for embedded.

**Tasks:**
1. **Interface Traits** (`reticulum-interfaces/traits`)
   ```rust
   pub trait Interface {
       fn send(&mut self, data: &[u8]) -> Result<(), InterfaceError>;
       fn receive(&mut self, buf: &mut [u8]) -> Result<usize, InterfaceError>;
       fn mtu(&self) -> usize;
       fn bitrate(&self) -> u32;
       fn is_online(&self) -> bool;
   }

   pub trait InterfaceRx {
       fn poll(&mut self) -> Option<&[u8]>;
   }
   ```

2. **Serial Interface** (`interface-serial`)
   - Use `embedded-hal` traits
   - Ring buffer for RX/TX
   - COBS/KISS framing support
   - Baudrate configuration
   - Non-blocking I/O

3. **LoRa/RNode Interface** (`interface-lora`)
   - RNode protocol implementation
   - LoRa parameter configuration (SF, BW, CR)
   - RSSI/SNR monitoring
   - TX power control

4. **Local Interface** (for testing)
   - In-memory packet queue
   - Loopback support
   - Simulated latency/packet loss

**Deliverables:**
- `Interface` trait with default implementations
- Serial interface working on STM32F4/nRF52
- LoRa interface (RNode compatible)
- Hardware-in-the-loop tests

**Success Criteria:**
- Interface can send/receive packets ≥5 bps
- MTU auto-detection working
- Bandwidth limiting (announce cap)
- RSSI/SNR metadata capture

---

### Phase 4: Transport Layer - Basic Routing (Weeks 8-10)

**Goal:** Implement core transport with routing, path discovery, and announce handling.

**Tasks:**
1. **Transport Core** (`reticulum-transport/routing`)
   - `Transport` struct with bounded tables
   - Packet forwarding logic
   - Hop count increment
   - Duplicate detection (bounded hashlist)
   - Transport type handling

2. **Path Table** (`reticulum-transport/tables`)
   ```rust
   struct PathTableEntry {
       timestamp: u64,
       next_hop: TruncatedHash,
       hops: u8,
       expires_at: u64,
       received_via: InterfaceId,
   }

   // Bounded map, oldest entries evicted
   struct PathTable<const MAX: usize> {
       entries: BTreeMap<TruncatedHash, PathTableEntry>,
   }
   ```

3. **Announce Handling**
   - Announce packet validation
   - Identity extraction from announces
   - Announce propagation with rate limiting
   - Announce queue with priority (fewer hops first)
   - Bandwidth cap enforcement (2% default)

4. **Path Discovery**
   - Path request generation
   - Path response handling
   - Grace periods (0.4s + 1.5s roaming)
   - Request timeout (15s default)

**Deliverables:**
- Working single-hop and multi-hop packet delivery
- Announce propagation across network
- Path table with expiry
- Transport-level tests with 3+ node topology

**Success Criteria:**
- Packets routed up to 128 hops
- Announce rate limiting prevents floods
- Path expiry working (7 days)
- Compatible with Python transport

---

### Phase 5: Link Establishment (Weeks 11-13)

**Goal:** Implement encrypted link establishment with forward secrecy.

**Tasks:**
1. **Link Request** (`reticulum-transport/link`)
   - 3-way handshake implementation
   - Ephemeral key generation
   - Link ID derivation
   - Proof validation

2. **Link State Machine**
   ```rust
   enum LinkState {
       Pending,
       Handshake,
       Active,
       Stale,
       Closed,
   }

   struct Link {
       state: LinkState,
       link_id: TruncatedHash,
       session_key: [u8; 32],
       tx_key: [u8; 32],
       rx_key: [u8; 32],
       created_at: u64,
       last_activity: u64,
   }
   ```

3. **Link Encryption**
   - Per-link session key derivation
   - Separate TX/RX keys
   - Ratchet support (optional)
   - Link proof verification

4. **Link Management**
   - Keepalive packets
   - Link timeout detection
   - Link close handshake
   - Stale link cleanup

**Deliverables:**
- Working link establishment
- Encrypted communication over link
- Link timeout and cleanup
- Link state persistence (optional)

**Success Criteria:**
- Link establishment: 3 packets, 297 bytes total
- Forward secrecy maintained
- Links survive interface changes
- Compatible link protocol with Python

---

### Phase 6: Resource Transfer (Weeks 14-15)

**Goal:** Implement windowed resource transfer for large data.

**Tasks:**
1. **Resource Advertisement** (`reticulum-transport/resource`)
   - Resource metadata packet
   - Hash map advertisement
   - Compression support (optional)
   - Size and part count calculation

2. **Resource Transfer Protocol**
   - Sliding window implementation
   - Part request/response
   - Out-of-order handling
   - Retry logic with exponential backoff

3. **Resource Reception**
   - Hash map validation
   - Part assembly
   - Progress tracking
   - Completion callbacks

4. **Flow Control**
   - Window size adjustment
   - Congestion detection
   - Fast path optimization

**Deliverables:**
- Large file transfer working (>10KB)
- Resource windowing and retries
- Progress callbacks
- Benchmark: throughput vs Python version

**Success Criteria:**
- Can transfer 1MB file over slow link
- Hash validation ensures integrity
- Window size adapts to link quality
- Memory usage bounded (no full buffering)

---

### Phase 7: Configuration & Persistence (Weeks 16-17)

**Goal:** Add configuration parsing and state persistence (std/alloc only).

**Tasks:**
1. **Configuration** (`reticulum-utils/config`)
   - Config file parser (TOML/JSON)
   - Interface definitions
   - Transport parameters
   - Identity storage paths

2. **Persistence Layer** (`reticulum-utils/persistence`)
   - Identity save/load
   - Known destinations table
   - Path table persistence
   - Announce cache

3. **Storage Abstraction**
   ```rust
   pub trait Storage {
       fn read(&self, key: &str) -> Result<Vec<u8>, StorageError>;
       fn write(&self, key: &str, data: &[u8]) -> Result<(), StorageError>;
       fn delete(&self, key: &str) -> Result<(), StorageError>;
   }
   ```

**Deliverables:**
- Config parsing compatible with Python
- Identity persistence
- Transport table save/restore
- NOR flash storage backend (embedded)

**Success Criteria:**
- Config files from Python work
- Identity persisted correctly
- Transport state survives reboot
- Minimal flash wear

---

### Phase 8: High-Level APIs (Weeks 18-19)

**Goal:** Implement Channel, Buffer, and convenience APIs.

**Tasks:**
1. **Channel API** (`reticulum-utils/channel`)
   - Message-based interface over links
   - Ordered delivery
   - Message types and system messages
   - Async-friendly API

2. **Buffer API** (`reticulum-utils/buffer`)
   - Stream-like read/write
   - RawChannelReader/Writer
   - Buffering and flow control

3. **Request/Response**
   - Request packet handling
   - Response routing
   - Timeout management

4. **Discovery API**
   - Simplified announce interface
   - Automatic path requests
   - Bootstrap helpers

**Deliverables:**
- High-level API examples
- Async/await support (with feature)
- Documentation and tutorials

**Success Criteria:**
- API ergonomics match Python
- Works with `async-std`/`tokio` (std)
- Works with `embassy` (no_std)

---

### Phase 9: Advanced Interfaces (Weeks 20-21)

**Goal:** Implement remaining interface types and auto-discovery.

**Tasks:**
1. **TCP/UDP Interfaces** (std only)
   - TCP server/client
   - UDP broadcast/multicast
   - Interface auto-detection

2. **AutoInterface** (std only)
   - Network interface enumeration
   - Multicast discovery
   - Peer detection

3. **I2P Interface** (std only, optional)
   - I2P router integration
   - Tunnel management
   - Privacy-preserving routing

4. **Bluetooth Interface** (embedded)
   - BLE GATT service
   - Nordic UART Service (NUS)
   - Mesh support

**Deliverables:**
- All interface types implemented
- Interface auto-configuration
- Bluetooth examples (nRF52)

**Success Criteria:**
- Interfaces interoperate with Python
- Auto-discovery finds peers
- Bluetooth range comparable to WiFi

---

### Phase 10: Testing, Optimization & Documentation (Weeks 22-24)

**Goal:** Comprehensive testing, performance optimization, and documentation.

**Tasks:**
1. **Interoperability Testing**
   - Cross-version tests (Rust ↔ Python)
   - Multi-hop network scenarios
   - Link establishment stress tests
   - Resource transfer reliability tests

2. **Performance Optimization**
   - Crypto operation optimization
   - Zero-copy packet handling
   - Memory pool tuning
   - Flash/RAM usage reduction

3. **Security Audit**
   - Constant-time crypto verification
   - Side-channel analysis
   - Fuzzing (AFL, cargo-fuzz)
   - Penetration testing

4. **Documentation**
   - API documentation (rustdoc)
   - Embedded examples (embassy)
   - Migration guide from Python
   - Protocol specification
   - Hardware porting guide

5. **Benchmarking**
   - Throughput tests (vs Python)
   - Latency measurements
   - Memory usage profiling
   - Power consumption (embedded)

**Deliverables:**
- Full test coverage (>80%)
- Benchmarking suite
- Security audit report
- Complete documentation
- Example applications

**Success Criteria:**
- 100% protocol compatibility
- Performance within 80% of Python
- Memory usage <50KB for full stack
- Documentation complete

---

## 5. Critical Challenges & Solutions

### 5.1 Challenge: Cryptography in no_std

**Problem:**
- Most crypto libraries assume std
- Constant-time requirements
- Hardware RNG access

**Solution:**
- Use `curve25519-dalek` (no_std compatible, pure Rust)
- Use `aes` crate (pure Rust, no_std)
- Use `sha2` crate (no_std)
- Integrate with `rand_core` for RNG abstraction
- Hardware RNG via HAL on embedded (TRNG)
- Fallback to deterministic RNG for testing

**Implementation:**
```rust
#[cfg(not(feature = "std"))]
use rand_core::{RngCore, CryptoRng};

pub fn generate_keypair<R: RngCore + CryptoRng>(rng: &mut R) -> Identity {
    let secret = x25519_dalek::StaticSecret::new(rng);
    // ...
}
```

### 5.2 Challenge: Dynamic Data Structures

**Problem:**
- Path/announce tables need dynamic sizing
- Python uses dicts extensively
- HashMap not available in core

**Solution:**
- Use `BTreeMap` from `alloc` (no hasher needed)
- Implement bounded collections with LRU eviction
- Static allocation for minimal profile
- Const generics for compile-time sizing

**Implementation:**
```rust
pub struct BoundedPathTable<const MAX: usize> {
    map: BTreeMap<TruncatedHash, PathEntry>,
}

impl<const MAX: usize> BoundedPathTable<MAX> {
    pub fn insert(&mut self, key: TruncatedHash, value: PathEntry) {
        if self.map.len() >= MAX {
            // Evict oldest entry
            let oldest = self.map.iter()
                .min_by_key(|(_, v)| v.timestamp)
                .map(|(k, _)| *k);
            if let Some(key) = oldest {
                self.map.remove(&key);
            }
        }
        self.map.insert(key, value);
    }
}
```

### 5.3 Challenge: Time and Timers

**Problem:**
- no_std has no `std::time::Instant`
- Embedded needs monotonic clock
- Timer interrupts for timeouts

**Solution:**
- Abstract time via trait:
```rust
pub trait Clock {
    fn now(&self) -> u64;  // milliseconds since boot
}

// Embedded implementation
#[cfg(not(feature = "std"))]
struct MonotonicClock;

impl Clock for MonotonicClock {
    fn now(&self) -> u64 {
        // Platform-specific: cortex_m::peripheral::DWT
        unsafe { DWT::cycle_count() / CYCLES_PER_MS }
    }
}

// std implementation
#[cfg(feature = "std")]
impl Clock for std::time::Instant {
    fn now(&self) -> u64 {
        self.elapsed().as_millis() as u64
    }
}
```

### 5.4 Challenge: Threading and Concurrency

**Problem:**
- Python uses threading extensively
- no_std has no threads
- Interrupt-based communication

**Solution:**
- Use critical sections and atomics:
```rust
use core::sync::atomic::{AtomicU32, Ordering};
use cortex_m::interrupt::{free, Mutex};

static PACKET_QUEUE: Mutex<RefCell<Option<PacketQueue>>> = Mutex::new(RefCell::new(None));

// In interrupt handler
free(|cs| {
    if let Some(queue) = PACKET_QUEUE.borrow(cs).borrow_mut().as_mut() {
        queue.push(packet);
    }
});
```

- For std: use `std::sync::Mutex`
- For async: use `embassy` executor (no_std)

### 5.5 Challenge: I/O and Blocking

**Problem:**
- Python uses blocking I/O
- Embedded needs non-blocking
- Async/await support

**Solution:**
- Define non-blocking interface trait
- Provide async wrappers:
```rust
// Sync API (blocking or nb::Error::WouldBlock)
pub trait InterfaceSync {
    fn try_send(&mut self, data: &[u8]) -> nb::Result<(), InterfaceError>;
    fn try_recv(&mut self, buf: &mut [u8]) -> nb::Result<usize, InterfaceError>;
}

// Async API (for embassy/tokio)
#[cfg(feature = "async")]
pub trait InterfaceAsync {
    async fn send(&mut self, data: &[u8]) -> Result<(), InterfaceError>;
    async fn recv(&mut self, buf: &mut [u8]) -> Result<usize, InterfaceError>;
}
```

### 5.6 Challenge: MessagePack Serialization

**Problem:**
- Python uses umsgpack for config/announces
- Need no_std compatible serialization

**Solution:**
- Use `rmp` (RMP = Rust MessagePack) with no_std
- Or use `serde` with `serde-json-core` for JSON
- Or implement minimal msgpack parser:
```rust
// Minimal msgpack for announces (no_std)
pub struct MsgPackWriter<'a> {
    buf: &'a mut [u8],
    pos: usize,
}

impl<'a> MsgPackWriter<'a> {
    pub fn write_map(&mut self, len: usize) -> Result<(), Error> {
        if len <= 15 {
            self.write_byte(0x80 | len as u8)
        } else {
            // ...
        }
    }
}
```

### 5.7 Challenge: Float/Time Calculations

**Problem:**
- Bandwidth calculations use floats
- Embedded may not have FPU
- Python's arbitrary precision

**Solution:**
- Use fixed-point arithmetic:
```rust
// Bandwidth in millibits per second (avoid float)
pub struct Bandwidth(u64);  // mbps * 1000

impl Bandwidth {
    pub fn from_bps(bps: u32) -> Self {
        Self(bps as u64 * 1000)
    }

    pub fn percent(&self, pct: u32) -> u64 {
        (self.0 * pct as u64) / 100
    }
}
```

- Use `libm` for required math functions (no_std)

---

## 6. Dependencies & Crate Selection

### 6.1 Cryptography Crates

| Crate | Purpose | no_std | Size | Notes |
|-------|---------|--------|------|-------|
| `curve25519-dalek` | X25519/Ed25519 | ✅ | ~100KB | Pure Rust, audited |
| `x25519-dalek` | X25519 wrapper | ✅ | ~50KB | Used by Signal |
| `ed25519-dalek` | Ed25519 wrapper | ✅ | ~50KB | Used by Signal |
| `aes` | AES-256-CBC | ✅ | ~20KB | Pure Rust, audited |
| `cbc` | CBC mode | ✅ | ~5KB | Block cipher mode |
| `sha2` | SHA-256/512 | ✅ | ~15KB | Pure Rust |
| `hmac` | HMAC | ✅ | ~2KB | Generic over hash |
| `hkdf` | HKDF | ✅ | ~2KB | Key derivation |
| `rand_core` | RNG trait | ✅ | ~5KB | Abstraction only |
| `getrandom` | OS RNG | ⚠️ | ~10KB | Needs platform impl |

**Total Crypto Stack**: ~150KB flash (with optimizations)

### 6.2 Core Utilities

| Crate | Purpose | no_std | Size | Notes |
|-------|---------|--------|------|-------|
| `heapless` | Fixed-size collections | ✅ | ~10KB | Vec, String, Map |
| `arrayvec` | Array-backed Vec | ✅ | ~5KB | Alternative to heapless |
| `tinyvec` | Small vec optimization | ✅ | ~3KB | Enum Vec/Array |
| `embedded-hal` | Hardware abstraction | ✅ | ~2KB | Traits only |
| `nb` | Non-blocking API | ✅ | <1KB | Error type |
| `serde` (no_std) | Serialization | ✅ | ~20KB | With derive |
| `rmp-serde` | MessagePack | ✅ | ~15KB | Serde-based |

### 6.3 Embedded-Specific

| Crate | Purpose | no_std | Notes |
|-------|---------|--------|-------|
| `cortex-m` | ARM Cortex-M | ✅ | Interrupts, peripherals |
| `cortex-m-rt` | Runtime | ✅ | Startup code |
| `panic-halt` | Panic handler | ✅ | Minimal panic |
| `defmt` | Logging | ✅ | Efficient logging |
| `embassy` | Async runtime | ✅ | Modern async for embedded |
| `embedded-alloc` | Allocator | ✅ | Simple bump allocator |

### 6.4 Interface-Specific

| Crate | Purpose | no_std | Notes |
|-------|---------|--------|-------|
| `embedded-hal-async` | Async HAL | ✅ | For async interfaces |
| `usart-serial` | Serial | ✅ | USART abstraction |
| `sx127x` | LoRa SX127x | ✅ | Semtech LoRa chips |
| `sx126x` | LoRa SX126x | ✅ | Newer LoRa chips |
| `socket2` | Sockets | ❌ | std only |
| `tokio` | Async runtime | ❌ | std only |

### 6.5 Dependency Management Strategy

**Minimize Dependencies:**
- Only include crates for enabled features
- Prefer `no_std` crates even for std builds
- Vendor critical code if crate is unmaintained
- Audit all dependencies for security

**Feature-Gated Dependencies:**
```toml
[dependencies]
# Core (always)
curve25519-dalek = { version = "4", default-features = false }
sha2 = { version = "0.10", default-features = false }
aes = { version = "0.8", default-features = false }
cbc = { version = "0.1", default-features = false }
rand_core = { version = "0.6", default-features = false }

# Alloc feature
[dependencies.heapless]
version = "0.7"
optional = true

[dependencies.alloc]
version = "1.0"
optional = true
package = "alloc"

# Std feature
[dependencies.std]
version = "1.0"
optional = true
package = "std"
```

---

## 7. Testing Strategy

### 7.1 Unit Tests

**Coverage:**
- All crypto primitives
- Packet encoding/decoding
- Hash functions
- Identity operations

**Approach:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_packet_pack_unpack() {
        let packet = Packet { /* ... */ };
        let packed = packet.pack();
        let unpacked = Packet::unpack(&packed).unwrap();
        assert_eq!(packet, unpacked);
    }

    // Test against Python reference implementation
    #[test]
    fn test_identity_compatibility() {
        let rust_identity = Identity::new();
        let python_hash = /* load from file */;
        assert_eq!(rust_identity.hash(), python_hash);
    }
}
```

### 7.2 Integration Tests

**Test Network Topology:**
```
[Node A - Rust] <---> [Node B - Python] <---> [Node C - Rust]
```

**Test Cases:**
- Single-hop broadcast
- Multi-hop routing (3+ hops)
- Link establishment Rust ↔ Python
- Resource transfer both directions
- Announce propagation
- Path discovery

**Implementation:**
- Use `std::process::Command` to spawn Python nodes
- Use pipes/sockets for communication
- Parse logs for verification

### 7.3 Embedded Tests

**Target Hardware:**
- QEMU ARM (cortex-m4)
- STM32F4 Discovery board
- nRF52840 DK
- Raspberry Pi Pico (RP2040)

**Test Approach:**
- `defmt` for logging over RTT
- `probe-rs` for debugging
- Hardware-in-the-loop (HIL) with real radios

### 7.4 Fuzzing

**Targets:**
- Packet parser: `cargo fuzz run parse_packet`
- Crypto operations: `cargo fuzz run identity_ops`
- MessagePack parser: `cargo fuzz run msgpack_parse`

**Corpus Generation:**
- Capture packets from Python implementation
- Use mutation fuzzing on valid packets
- Structure-aware fuzzing (grammar-based)

### 7.5 Property-Based Testing

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn packet_roundtrip(data: Vec<u8>) {
        let packet = Packet::new(dest, &data);
        let packed = packet.pack();
        let unpacked = Packet::unpack(&packed)?;
        prop_assert_eq!(unpacked.data, data);
    }
}
```

---

## 8. Performance Targets

### 8.1 Throughput

| Metric | Target | Python Baseline |
|--------|--------|-----------------|
| Packet encode | <100μs | ~500μs |
| Packet decode | <100μs | ~500μs |
| Encrypt SINGLE | <1ms | ~2ms |
| Sign (Ed25519) | <500μs | ~1ms |
| Path lookup | <10μs | ~50μs |
| Announce process | <2ms | ~5ms |

### 8.2 Memory Usage

| Component | Target | Minimal | Notes |
|-----------|--------|---------|-------|
| Core stack | 64KB | 16KB | Protocol only |
| Packet buffers | 4KB | 2KB | 4×500 or 2×500 |
| Path table | 10KB | 1.5KB | 200 or 32 entries |
| Link state | 2KB | 400B | 8 or 2 links |
| Crypto temp | 4KB | 4KB | During ops |
| **Total** | **84KB** | **24KB** | |

### 8.3 Power Consumption

**Target (nRF52840 @ 64MHz):**
- Idle (RX): <5mA
- TX (LoRa): <120mA (peak)
- Sleep: <5μA

**Optimizations:**
- Use WFI (Wait For Interrupt) in idle
- Batch crypto operations
- Reduce announcement frequency

---

## 9. Migration Path & Compatibility

### 9.1 Protocol Compatibility

**100% Wire-Compatible:**
- Packet format identical
- Crypto algorithms identical
- Hashes must match exactly
- Timing constants preserved

**Verification:**
- Cross-version test suite
- Packet capture comparison
- Hash collision tests

### 9.2 Configuration Compatibility

**Goal:** Rust can load Python config files

**Approach:**
- Parse INI-style config (configobj format)
- Map Python types to Rust types
- Provide migration warnings for unsupported options

**Example:**
```ini
[interfaces]
  [[RNode LoRa]]
    type = RNodeInterface
    port = /dev/ttyUSB0
    frequency = 915000000
    bandwidth = 125000
    spreadingfactor = 7
```

### 9.3 API Compatibility

**Principle:** Similar feel, not identical API

**Python:**
```python
import RNS

reticulum = RNS.Reticulum()
identity = RNS.Identity()
destination = RNS.Destination(identity, RNS.Destination.IN,
                               RNS.Destination.SINGLE, "app", "aspect")
```

**Rust (std):**
```rust
use reticulum::{Reticulum, Identity, Destination, DestinationType, Direction};

let reticulum = Reticulum::new()?;
let identity = Identity::new();
let destination = Destination::new(
    &identity, Direction::In, DestinationType::Single, "app", "aspect"
)?;
```

**Rust (no_std):**
```rust
use reticulum_core::{Identity, Destination};

let mut rng = platform::rng();
let identity = Identity::generate(&mut rng);
let destination = Destination::single(&identity, b"app", b"aspect");
```

---

## 10. Risks & Mitigation

### 10.1 Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Crypto incompatibility | Medium | High | Early cross-version testing |
| Memory exhaustion | Low | High | Bounded collections, static allocation |
| Timing attacks | Medium | High | Constant-time audit, fuzzing |
| Hardware support | Medium | Medium | HAL abstraction, multiple backends |
| Performance issues | Low | Medium | Early benchmarking, optimization |

### 10.2 Project Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Scope creep | High | Medium | Phased approach, MVP first |
| Dependency breakage | Low | Medium | Version pinning, vendoring |
| Community adoption | Medium | Low | Good docs, examples |
| Protocol changes | Low | High | Version negotiation support |

---

## 11. Success Metrics

### 11.1 Technical Metrics

- ✅ Compiles for `thumbv7em-none-eabihf`
- ✅ Core library <64KB flash
- ✅ Memory usage <100KB RAM
- ✅ 100% protocol compatibility
- ✅ Performance within 2x of Python
- ✅ Zero security vulnerabilities

### 11.2 Functional Metrics

- ✅ Can send/receive packets with Python nodes
- ✅ Can establish links with Python nodes
- ✅ Can route packets across multi-hop network
- ✅ Announces propagate correctly
- ✅ Resource transfers work reliably

### 11.3 Quality Metrics

- ✅ Test coverage >80%
- ✅ Documentation complete (API + guides)
- ✅ CI passes on all targets
- ✅ Fuzzing runs 24h without crashes
- ✅ Security audit passed

---

## 12. Timeline Summary

| Phase | Duration | Deliverable | Milestone |
|-------|----------|-------------|-----------|
| 1. Foundation & Crypto | 3 weeks | Crypto primitives, Identity | M1: Core crypto working |
| 2. Packet Layer | 2 weeks | Packet encode/decode | M2: Packets compatible |
| 3. Interface Abstraction | 2 weeks | Serial + LoRa interfaces | M3: HW communication |
| 4. Transport - Basic | 3 weeks | Routing, announces | M4: Multi-hop routing |
| 5. Link Establishment | 3 weeks | Links working | M5: Encrypted links |
| 6. Resource Transfer | 2 weeks | File transfers | M6: Large data |
| 7. Config & Persistence | 2 weeks | Storage layer | M7: Persistence |
| 8. High-Level APIs | 2 weeks | Channel, Buffer | M8: Ergonomic API |
| 9. Advanced Interfaces | 2 weeks | TCP/UDP/I2P | M9: All interfaces |
| 10. Test & Optimize | 3 weeks | Full test suite | M10: Production ready |
| **Total** | **24 weeks** | | **~6 months** |

---

## 13. Post-Launch Roadmap

### Phase 11: Optimizations (Optional)

- Hardware crypto acceleration (AES-NI, ARM Crypto Extensions)
- Assembly optimization for critical paths
- Profile-guided optimization (PGO)
- Link-time optimization (LTO)

### Phase 12: Advanced Features (Optional)

- Multi-threaded transport (for std)
- WebAssembly target (browser nodes)
- Formal verification of crypto code
- Hardware security module (HSM) support

### Phase 13: Ecosystem (Optional)

- C FFI bindings (C header generation)
- Python bindings (PyO3)
- JavaScript bindings (WASM)
- Mobile SDKs (iOS/Android)

---

## 14. Getting Started Guide

### 14.1 Minimal Example (no_std)

```rust
#![no_std]
#![no_main]

use panic_halt as _;
use cortex_m_rt::entry;
use reticulum_core::{Identity, Packet, Destination};

#[entry]
fn main() -> ! {
    // Initialize platform
    let mut rng = init_rng();

    // Create identity
    let identity = Identity::generate(&mut rng);

    // Create destination
    let dest = Destination::single(&identity, b"app", b"test");

    // Create and send packet
    let data = b"Hello Reticulum!";
    let packet = Packet::new(&dest, data);

    loop {
        // Process packets
    }
}
```

### 14.2 Full Example (std)

```rust
use reticulum::{Reticulum, Identity, Destination, Link};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Initialize Reticulum
    let reticulum = Reticulum::new()?;

    // Create identity
    let identity = Identity::new();

    // Create destination
    let dest = Destination::single(&identity, "example", "echo")?;

    // Announce
    dest.announce()?;

    // Handle incoming links
    dest.set_link_established_callback(|link| {
        println!("Link established: {}", link.id());

        // Send data over link
        link.send(b"Hello over encrypted link!")?;
        Ok(())
    });

    // Run forever
    reticulum.run()?;
    Ok(())
}
```

---

## 15. Conclusion

This plan outlines a comprehensive approach to porting Reticulum to Rust with no_std constraints. The phased approach allows for incremental development and validation, with each phase building on the previous. The focus on embedded systems and resource constraints requires careful design decisions around memory management, cryptography, and I/O handling.

**Key Success Factors:**
1. **Early validation**: Cross-version testing from Phase 1
2. **Bounded resources**: All collections have compile-time or runtime limits
3. **Hardware abstraction**: Trait-based interfaces allow platform flexibility
4. **Security first**: Constant-time crypto, regular audits
5. **Documentation**: Comprehensive guides for embedded developers

**Estimated Effort:**
- ~6 months for full implementation (1-2 developers)
- ~3 months for minimal profile (embedded only)
- ~2 months for additional features/optimization

This Rust implementation will enable Reticulum to run on resource-constrained devices like LoRa nodes, satellite modems, and IoT devices, significantly expanding its deployment possibilities while maintaining full protocol compatibility with the Python implementation.
