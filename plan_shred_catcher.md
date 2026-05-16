# Shred-Catcher: Solana Shred Viewer & Streamer

## What is this?

A **library + CLI** for capturing, viewing, and streaming raw Solana shreds from the live network.
No dependency on diet-rpc-validator or tinydancer. Uses modern official Solana crates (v3.1.x).

**Designed as a crate first** — tinydancer (or any Rust project) can import `shred-catcher` as a
dependency to get raw shreds without running a full validator.

## Why?

- Solana validators discard raw shreds after block assembly — they're ephemeral
- No public RPC exposes raw shreds
- Shreds contain cryptographic proofs (Merkle + leader signature) that enable light client verification
- This tool captures them before they're gone

## Use Cases

1. **As a library (crate)** — `use shred_catcher::ShredCatcher` in tinydancer or any Rust project
2. **View** — inspect individual shreds (structure, signature, Merkle proof, payload)
3. **Stream** — real-time feed of shreds for a slot or all slots
4. **Verify** — cryptographically verify shreds against leader signature + Merkle proof
5. **Serve** — expose a `getShreds` JSON-RPC endpoint for light clients
6. **Export** — dump shreds to file for offline analysis

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  shred-catcher                                          │
│                                                         │
│  ┌──────────────┐                                       │
│  │ gossip       │  Join network, discover validators    │
│  │ (solana-     │  Auto-detect shred version            │
│  │  gossip)     │  Get leader schedule from gossip      │
│  └──────┬───────┘                                       │
│         │ peer list                                     │
│  ┌──────▼───────┐                                       │
│  │ turbine      │  Receive shreds via UDP broadcast     │
│  │ receiver     │  Passive — sits in Turbine tree       │
│  └──────┬───────┘                                       │
│         │                                               │
│  ┌──────▼───────┐  ┌─────────────┐                      │
│  │ repair       │  │ shred store │  In-memory ring      │
│  │ requester    │──│ (DashMap)   │  buffer, last N      │
│  │              │  │             │  slots                │
│  └──────────────┘  └──────┬──────┘                      │
│                           │                             │
│         ┌─────────────────┼─────────────────┐           │
│         │                 │                 │           │
│  ┌──────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐   │
│  │ CLI viewer  │  │ RPC server   │  │ WebSocket    │   │
│  │ (terminal)  │  │ (getShreds)  │  │ stream       │   │
│  └─────────────┘  └──────────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Dependencies (NO diet-rpc-validator)

All from official Solana/Agave crates published on crates.io:

| Crate | Version | Purpose |
|---|---|---|
| `solana-gossip` | 3.1.x | Gossip protocol, peer discovery, ClusterInfo |
| `solana-ledger` | 3.1.x | Shred types, deserialization, Merkle verification |
| `solana-sdk` | 2.x / 3.x | Pubkey, Keypair, Signature, Clock |
| `solana-streamer` | 3.1.x | UDP packet handling |
| `solana-net-utils` | 3.1.x | IP echo, port binding |
| `solana-turbine` | 3.1.x | Turbine tree logic (optional, for understanding placement) |
| `jsonrpsee` | 0.24.x | JSON-RPC server |
| `tokio` | 1.x | Async runtime |
| `clap` | 4.x | CLI |
| `dashmap` | 6.x | Concurrent shred store |
| `crossterm` | 0.28.x | Terminal UI for viewer |

**Key point**: Using v3.1.x crates means gossip will work with current Devnet (v4.0.0)
and Mainnet validators. The gossip protocol is backwards-compatible within major versions.

---

## Project Structure (library-first design)

The crate exposes a public library (`lib.rs`) and an optional CLI binary (`main.rs`).
Tinydancer imports only the library — the CLI is for standalone use.

```
shred-catcher/
├── Cargo.toml               # [lib] + [[bin]] sections
├── README.md
└── src/
    ├── lib.rs               # Public API — ShredCatcher, ShredStore, types
    ├── main.rs              # CLI binary (uses lib.rs, optional)
    │
    ├── gossip/
    │   ├── mod.rs           # Gossip node setup
    │   └── leader.rs        # Leader schedule from gossip/RPC
    │
    ├── receiver/
    │   ├── mod.rs           # Turbine UDP listener
    │   └── repair.rs        # Repair protocol for missed shreds
    │
    ├── store/
    │   ├── mod.rs           # ShredStore (DashMap ring buffer)
    │   └── types.rs         # Shred metadata, slot stats
    │
    ├── server/
    │   ├── mod.rs           # JSON-RPC server (getShreds endpoint)
    │   └── ws.rs            # WebSocket stream (real-time shred feed)
    │
    └── viewer/
        ├── mod.rs           # Terminal shred viewer (CLI only)
        └── formatter.rs     # Pretty-print shred fields
```

### Cargo.toml layout

```toml
[package]
name = "shred-catcher"
version = "0.1.0"

[lib]
name = "shred_catcher"
path = "src/lib.rs"

[[bin]]
name = "shred-catcher"
path = "src/main.rs"

# CLI-only deps behind a feature flag so library consumers don't pull them in
[features]
default = ["cli"]
cli = ["clap", "crossterm"]
```

---

## Library API (what tinydancer imports)

### Core types

```rust
use shred_catcher::{ShredCatcher, ShredCatcherConfig, CatcherShred, SlotShreds};
```

### ShredCatcher — the main handle

```rust
/// Config for starting the shred catcher
pub struct ShredCatcherConfig {
    pub cluster: Cluster,           // Devnet, Mainnet, Custom(entrypoint)
    pub max_slots: usize,           // how many slots to keep in memory (default: 32)
    pub rpc_url: Option<String>,    // optional RPC for leader schedule fallback
}

/// The main shred catcher — start it, then query for shreds
pub struct ShredCatcher { /* ... */ }

impl ShredCatcher {
    /// Start the shred catcher (joins gossip, begins receiving shreds)
    pub async fn start(config: ShredCatcherConfig) -> Result<Self>;

    /// Get shreds for a slot by indices
    pub fn get_shreds(&self, slot: u64, indices: &[u32]) -> Vec<Option<CatcherShred>>;

    /// Get all shreds for a slot
    pub fn get_slot(&self, slot: u64) -> Option<SlotShreds>;

    /// Get the leader pubkey for a slot
    pub async fn get_leader(&self, slot: u64) -> Result<Pubkey>;

    /// Verify a shred (signature + Merkle proof)
    pub fn verify_shred(shred: &CatcherShred, leader: &Pubkey) -> VerifyResult;

    /// Subscribe to new shreds (returns a channel receiver)
    pub fn subscribe(&self) -> Receiver<CatcherShred>;

    /// Get list of stored slots
    pub fn stored_slots(&self) -> Vec<u64>;

    /// Shutdown
    pub async fn shutdown(self);
}
```

### Usage in tinydancer

```rust
// In tinydancer's Cargo.toml:
// shred-catcher = { path = "../shred-catcher", default-features = false }
//   (no "cli" feature → doesn't pull in clap/crossterm)

use shred_catcher::{ShredCatcher, ShredCatcherConfig, Cluster};

async fn verify_slot(slot: u64) -> bool {
    let catcher = ShredCatcher::start(ShredCatcherConfig {
        cluster: Cluster::Devnet,
        max_slots: 32,
        rpc_url: Some("https://api.devnet.solana.com".into()),
    }).await.unwrap();

    // Wait for shreds to arrive
    tokio::time::sleep(Duration::from_secs(5)).await;

    // Get shreds and verify
    let leader = catcher.get_leader(slot).await.unwrap();
    let shreds = catcher.get_shreds(slot, &[0, 1, 2, 3, 4]);

    let all_valid = shreds.iter()
        .flatten()
        .all(|s| ShredCatcher::verify_shred(s, &leader).is_valid());

    catcher.shutdown().await;
    all_valid
}
```

### Usage as a long-running service alongside tinydancer

```rust
// Start once, share across tinydancer's lifetime
let catcher = Arc::new(ShredCatcher::start(config).await?);

// Pass to sampler, verify loop, etc.
let catcher_clone = catcher.clone();
tokio::spawn(async move {
    loop {
        let slot = get_current_slot().await;
        let shreds = catcher_clone.get_shreds(slot, &random_indices(5));
        let leader = catcher_clone.get_leader(slot).await.unwrap();
        for shred in shreds.iter().flatten() {
            let result = ShredCatcher::verify_shred(shred, &leader);
            println!("Slot {} Shred {}: {:?}", slot, shred.index(), result);
        }
    }
});
```

---

## Implementation Phases

### Phase 1: Gossip + Peer Discovery

**Goal**: Join the network, discover validators, auto-detect shred version.

- Create a gossip spy node using `solana-gossip` (v3.1.x)
- Connect to Devnet/Mainnet entrypoint
- Discover TVU peers (validators) and their addresses
- Auto-detect cluster shred version
- Extract leader schedule from gossip data

**Verification**: Run the binary, see validator list with IPs and shred version.

```bash
shred-catcher --cluster devnet discover
# Output:
# Discovered 45 validators, shred_version=52394
# Validator dv4AC... TVU=64.130.33.238:8001 Repair=64.130.33.238:8008
# ...
```

### Phase 2: Shred Receiver (Turbine + Repair)

**Goal**: Actually receive raw shreds from the network.

**Path A — Turbine (passive)**:
- Bind a TVU UDP socket
- Advertise it in our ContactInfo via gossip
- Receive shreds as UDP packets from the Turbine tree
- Deserialize with `Shred::new_from_serialized_shred()`
- Store in ShredStore
- Caveat: zero-stake node is low priority in Turbine tree

**Path B — Repair (active, more reliable)**:
- Monitor slot progression via gossip slot updates
- For each new slot, send Repair requests to validators
- Repair request format: `RepairProtocol::WindowIndex { slot, shred_index }`
- Send to validators' `serve_repair` port (discovered via gossip)
- Validators respond with the requested shred
- This is outbound-only — works behind NAT

**Both paths**: Deserialize packets → Shred structs → store in DashMap

**Verification**: See shred count increasing per slot in logs.

```bash
shred-catcher --cluster devnet stream
# Output:
# Slot 462500001: 64 data shreds, 32 coding shreds (receiving...)
# Slot 462500002: 48 data shreds, 24 coding shreds (receiving...)
```

### Phase 3: CLI Viewer

**Goal**: Human-readable shred inspection in the terminal.

Commands:
```bash
# View a specific shred
shred-catcher view --slot 462500001 --index 0
# Output:
# ┌─ Shred [Slot 462500001, Index 0] ──────────────────┐
# │ Type:        Data (Merkle)                          │
# │ Slot:        462500001                              │
# │ Index:       0 / 64                                 │
# │ FEC Set:     0                                      │
# │ Parent:      462500000                              │
# │ Leader:      dv4ACNkpYPcE3aKmYDq...                 │
# │ Signature:   3xK9vBq...  ✓ VALID                   │
# │ Merkle Root: 7YhT2nm...  ✓ VALID                   │
# │ Payload:     1051 bytes                             │
# │ Flags:       LAST_IN_SLOT                           │
# └─────────────────────────────────────────────────────┘

# Verify a slot (random sample)
shred-catcher verify --slot 462500001 --samples 5
# Output:
# Verifying 5 random shreds for Slot 462500001...
# Shred[3]:  Signature ✓  Merkle ✓
# Shred[17]: Signature ✓  Merkle ✓
# Shred[42]: Signature ✓  Merkle ✓
# Shred[8]:  Signature ✓  Merkle ✓
# Shred[55]: Signature ✓  Merkle ✓
# Slot 462500001: 5/5 samples VALID ✓

# Stream shreds in real-time
shred-catcher stream --cluster devnet
# (real-time feed of arriving shreds)

# List recent slots and shred counts
shred-catcher slots
# Output:
# Slot 462500001: 64 data + 32 coding = 96 shreds
# Slot 462500002: 48 data + 24 coding = 72 shreds
# ...
```

### Phase 4: RPC Server + WebSocket

**Goal**: Serve shreds to external clients (like tinydancer).

- JSON-RPC `getShreds(slot, indices)` endpoint — compatible with tinydancer's format
- WebSocket subscription for real-time shred streaming
- REST endpoint for slot stats

```bash
# Start as a server
shred-catcher serve --cluster devnet --port 8899

# Clients can then:
curl http://localhost:8899 -d '{"jsonrpc":"2.0","id":1,"method":"getShreds","params":[462500001,[0,1,2]]}'
```

### Phase 5: Tinydancer Integration

Two integration paths (choose based on effort):

**Path A — Library import (recommended)**:
- Add `shred-catcher = { path = "../shred-catcher", default-features = false }` to tinydancer's Cargo.toml
- Update tinydancer's Solana deps from v1.15.0 → v3.1.x (required for type compatibility)
- Replace `pull_and_verify_shreds` with `ShredCatcher::get_shreds()` + `ShredCatcher::verify_shred()`
- No RPC server needed — direct in-process access to shreds
- Shred-catcher runs as a background task inside tinydancer

**Path B — RPC bridge (if tinydancer stays on v1.15.0)**:
- Run shred-catcher as a separate process with `shred-catcher serve --port 8899`
- Point tinydancer's `--cluster http://localhost:8899` at it
- Tinydancer's existing `request_shreds` / `getShreds` works as-is
- No dep changes in tinydancer needed
- Downside: extra process, serialization overhead

**Path A** is cleaner. **Path B** works today without touching tinydancer's deps.

---

## Key Technical Decisions

### Gossip: Spy node vs Full node

| | Spy Node | Full Gossip Node |
|---|---|---|
| NAT/firewall | Works (outbound only) | Needs open ports |
| Peer discovery | Yes (via pull) | Yes (push + pull) |
| Turbine shreds | No (not in tree) | Maybe (bottom of tree) |
| Repair requests | Yes (outbound) | Yes |
| **Recommendation** | **Use this** | Only if on public server |

### Shred acquisition: Turbine vs Repair

| | Turbine (passive) | Repair (active) |
|---|---|---|
| How | Receive UDP broadcasts | Request specific shreds |
| Reliability | Low for zero-stake | High (direct request) |
| Latency | Lowest (~200ms) | Higher (~500ms-1s) |
| NAT-friendly | No (needs inbound UDP) | Yes (outbound only) |
| **Recommendation** | Nice-to-have bonus | **Primary strategy** |

### Storage: Memory vs Disk

| | In-memory (DashMap) | Disk (RocksDB) |
|---|---|---|
| Speed | Fastest | Fast |
| Capacity | ~32 slots (~10MB) | Unlimited |
| Persistence | Lost on restart | Survives restart |
| Complexity | Simple | More setup |
| **Recommendation** | **Phase 1-3** | Phase 4+ optional |

---

## System Requirements

| Resource | Requirement |
|---|---|
| CPU | 2 cores |
| RAM | 2-4 GB |
| Disk | Minimal (in-memory store) |
| Network | ~5 MB/s inbound |
| Ports | None (outbound only with spy + repair) |
| OS | Linux, macOS |

---

## Milestones

| Phase | What | Deliverable | Depends on |
|---|---|---|---|
| 1 | Gossip + discovery | `shred-catcher discover` shows validators | — |
| 2 | Shred receiver | Shreds flowing into store via library API | Phase 1 |
| 3 | CLI viewer | `shred-catcher view/verify/stream/slots` | Phase 2 |
| 4 | RPC + WebSocket server | `shred-catcher serve` (Path B for tinydancer) | Phase 2 |
| 5 | Tinydancer integration | `use shred_catcher::ShredCatcher` (Path A) or RPC bridge (Path B) | Phase 2 or 4 |

---

## Open Questions

1. **Repair protocol compatibility** — Does the repair protocol require any form of authentication (signed nonces)? Need to check the Agave source for `RepairProtocol` enum and any handshake requirements.

2. **Rate limiting** — Will validators rate-limit repair requests from unknown nodes? May need to space out requests.

3. **Shred version detection** — Can we reliably detect shred version from gossip in spy mode? The entrypoint should provide it.

4. **crates.io vs git** — solana-gossip v3.1.11 is on crates.io. Need to verify it works without git dependency on Agave repo. If not, use `git = "https://github.com/anza-xyz/agave"` with a tag.

---

## How Shred-Catcher Captures Shreds

Shreds are NOT available via HTTP/WSS/RPC. They travel as **raw UDP packets** between
validators. Shred-catcher captures them through two methods:

### Method 1: Turbine — Passive UDP Listener (bonus)

```
Leader broadcasts shreds via Turbine (UDP tree)
│
├──► Validator A (retransmits to children)
├──► Validator B (retransmits to children)
├──► ...
└──► shred-catcher (our node, if in the Turbine tree)
         │
         │  Binds a UDP socket (TVU port)
         │  Advertises it via gossip so the network knows about us
         │  Receives raw shred packets (~1280 bytes each)
         │  Deserializes: Shred::new_from_serialized_shred(packet)
         ▼
    Shred captured in memory
```

Limitation: Turbine tree is stake-weighted. Zero-stake node sits at the bottom,
may miss shreds. Also needs inbound UDP ports open (doesn't work behind NAT).

### Method 2: Repair — Active UDP Requests (primary strategy)

This is the main approach. Validators keep shreds in memory for a short window
(a few minutes) for their own repair protocol. During that window, any gossip
participant can request specific shreds.

**There is no URL.** The target is a raw UDP socket address discovered via gossip.

```
Step 1: Discover validators via gossip
│
│  shred-catcher joins gossip → discovers all validators
│  Each validator's ContactInfo includes:
│
│    ContactInfo {
│        id:           "dv4ACNkpYPcE3...",         // validator pubkey
│        gossip:       64.130.33.238:8001,          // gossip protocol
│        tvu:          64.130.33.238:8002,          // Turbine shreds port
│        repair:       64.130.33.238:8008,          // repair requests port
│        serve_repair: 64.130.33.238:8009,          // ← WE SEND REQUESTS HERE
│        rpc:          64.130.33.238:8899,          // standard JSON-RPC
│        ...
│    }
│
▼
Step 2: Monitor slot progression
│
│  Watch for new slots via gossip or RPC slotSubscribe
│  "Slot 462500001 just started"
│
▼
Step 3: Send repair request (raw UDP)
│
│  Our UDP socket ────── UDP packet ──────► 64.130.33.238:8009
│                                           (validator's serve_repair port)
│
│  Packet contains (serialized with bincode):
│    RepairProtocol::WindowIndex {
│        slot: 462500001,
│        shred_index: 42,
│    }
│
▼
Step 4: Receive response (raw UDP)
│
│                  ◄────── UDP packet ────── Validator responds with
│                                            raw shred bytes (~1280 bytes)
│
▼
Step 5: Deserialize and store
│
│  Shred::new_from_serialized_shred(response_bytes)
│  → Store in DashMap keyed by (slot, index)
│  → Available via ShredCatcher::get_shreds() API
```

**Why Repair works behind NAT**: We send the UDP request outbound. The validator's
response comes back on the same UDP socket (NAT hole-punching). No inbound ports needed.

**Why there's a time window**: Validators only keep shreds in memory for repair purposes
for a few minutes. After block assembly + confirmation, shreds are garbage collected.
Shred-catcher must capture them during this window — after that, they're gone forever.

**Why we request from any validator, not just the leader**: The leader could be malicious.
Every validator that received the shreds via Turbine has a copy. We request from random
validators (discovered via gossip) to avoid trusting any single node.

### What standard RPCs give you vs what shred-catcher gives you

```
Standard RPC (HTTP/WSS):                     Shred-Catcher (raw UDP):
  getBlock → assembled block                   Raw shred fragments
  getSlot → slot number                        Leader's ED25519 signature
  slotSubscribe → slot updates                 Merkle proof per shred
  getTransaction → parsed tx                   FEC coding shreds
                                               Shred headers (slot, index, FEC set)
  ✗ No Merkle proofs                           
  ✗ No leader signatures on data               ✓ Everything needed for
  ✗ No raw shred structure                       Data Availability Sampling
  ✗ Shreds already discarded                     (tinydancer's core verification)
```

---

# The Life of a Shred

## 1. Why Shreds Exist

Solana produces blocks every ~400ms. A block can be several MB of transaction data.
You can't send that as one giant UDP packet (max UDP payload ~1232 bytes for IPv6).
So blocks are **shredded** — split into small fragments that can be individually
transmitted, verified, and reassembled.

Think of it like sending a book page-by-page through the mail, where each page has
a stamp proving which book it belongs to and who wrote it.

## 2. Block to Shreds (Generation)

```
Leader validator produces a block for Slot 12345
│
│  Block contains: 500 transactions, ~2MB total
│
▼
STEP 1: Split into Entries
│  Entries = ordered groups of transactions with PoH hashes
│  [Entry0: tx1,tx2,tx3] [Entry1: tx4,tx5] [Entry2: tx6,tx7,tx8] ...
│
▼
STEP 2: Serialize entries into a byte stream
│  Raw bytes: [0x4a, 0x2f, 0x8c, 0x1b, ...]  (~2MB)
│
▼
STEP 3: Chunk into Data Shreds
│  Each data shred holds ~1051 bytes of payload
│  DataShred[0]: bytes[0..1051]      ← first fragment
│  DataShred[1]: bytes[1051..2102]   ← second fragment
│  DataShred[2]: bytes[2102..3153]   ← third fragment
│  ... (~200 data shreds for a 2MB block)
│  Last data shred has LAST_IN_SLOT flag set
│
▼
STEP 4: Generate Coding Shreds (Reed-Solomon FEC)
│  For every batch of K data shreds, generate M coding shreds
│  These are redundancy — if some data shreds are lost in transit,
│  you can reconstruct them from coding shreds (like RAID parity)
│  CodingShred[0], CodingShred[1], ...
│  Together: K data + M coding = one "FEC set"
│
▼
STEP 5: Build Merkle Tree (per FEC set)
│  All shreds in the FEC set (data + coding) become leaves
│
│       Merkle Root (this gets signed)
│           /          \
│         H01           H23
│        /   \         /   \
│     H0      H1    H2      H3
│     │       │     │       │
│  Data[0] Data[1] Code[0] Code[1]   ← leaves = shreds
│
│  Each shred gets a Merkle proof (the sibling hashes on the path to root)
│  Example: Data[0]'s proof = [H1, H23] — enough to recompute the root
│  This proves: "this shred belongs to this specific FEC set"
│
▼
STEP 6: Leader Signs the Merkle Root
│  The leader signs the root with their ED25519 private key
│  This signature is embedded in EVERY shred's header
│  This proves: "the slot leader actually produced this data"
│  If anyone modifies a shred → Merkle proof breaks
│  If anyone forges a shred → signature verification fails
```

## 3. Shred Structure (What's Inside Each ~1280 Byte Packet)

```
┌──────────────────────────────────────────────────────────┐
│ COMMON HEADER (every shred has this)                     │
│                                                          │
│   signature: [u8; 64]      ← leader's ED25519 signature │
│                               (signs the Merkle root)    │
│   shred_variant: u8        ← encodes type + auth method │
│                               (data/code, legacy/merkle) │
│   slot: u64                ← which slot (block height)   │
│   index: u32               ← position within the slot    │
│   version: u16             ← cluster shred version       │
│   fec_set_index: u32       ← which FEC group this is in │
├──────────────────────────────────────────────────────────┤
│ IF DATA SHRED:                                           │
│                                                          │
│   parent_offset: u16       ← slots back to parent block │
│   flags: u8                ← DATA_COMPLETE_SHRED,        │
│                               LAST_SHRED_IN_SLOT         │
│   payload: [u8; ~1051]     ← actual serialized entries   │
│                               (the real block content)    │
├──────────────────────────────────────────────────────────┤
│ IF CODING SHRED:                                         │
│                                                          │
│   num_data_shreds: u16     ← K (data shreds in FEC set) │
│   num_coding_shreds: u16   ← M (coding shreds in set)   │
│   position: u16            ← index within coding shreds  │
│   payload: [u8; ~1051]     ← Reed-Solomon parity data    │
├──────────────────────────────────────────────────────────┤
│ MERKLE PROOF (appended at end)                           │
│                                                          │
│   proof: Vec<[u8; 20]>     ← sibling hashes from leaf   │
│                               to Merkle root             │
│                               (typically 3-5 hashes)     │
│                                                          │
│   This allows anyone to verify this shred belongs to     │
│   the block without having all the other shreds          │
└──────────────────────────────────────────────────────────┘

Total size per shred: ~1228 bytes (fits in one UDP packet)
SIZE_OF_PAYLOAD = 1228 (IPv6 MTU 1280 - 48 byte IP header - 4 byte padding)
```

## 4. Shred to Network (Turbine Broadcast)

```
Leader has ~400 shreds for Slot 12345
│
▼
TURBINE PROTOCOL (stake-weighted UDP broadcast tree)
│
│  Validators are organized into a tree based on stake weight:
│  Higher stake = closer to leader = receives shreds first
│
│  Layer 0: Leader
│           │
│  Layer 1: ┌──────┼──────┐         (top validators by stake)
│           V1     V2     V3        receive directly from leader
│           │      │      │
│  Layer 2: ┌┤    ┌┤    ┌─┤         (mid-stake validators)
│           V4 V5 V6 V7 V8 V9      receive from Layer 1
│           │     │     │
│  Layer 3: ...  ...   ...          (lower stake validators)
│                                    receive from Layer 2
│
│  Each node retransmits to its children upon receipt
│  Fanout = 200 (each node sends to up to 200 children)
│  Entire network receives all shreds in 3-4 hops (~200ms)
│
▼
Each validator receives shreds as individual UDP packets
│  One UDP packet = one shred (~1280 bytes)
│  ~800-1000 shreds per second at current throughput
│  Shreds arrive out of order — Turbine doesn't guarantee ordering
│
▼
Validator stores shreds temporarily in Blockstore
│  Waiting until enough arrive to reassemble the block
│  Also keeps them for a few minutes for Repair protocol
│  (other nodes can request missing shreds during this window)
│
▼
Eventually: shreds are garbage collected
│  After block is assembled + confirmed + executed
│  Raw shreds are discarded — they're no longer needed
│  *** This is the window shred-catcher must capture them ***
```

## 5. Shred Verification (What Tinydancer / Shred-Catcher Does)

```
Received shred for Slot 12345, Index 42
(obtained via Repair request or Turbine)
│
▼
CHECK 1: Leader Signature Verification
│
│  "Was this shred actually produced by the leader of Slot 12345?"
│
│  1. Look up leader schedule → leader for slot 12345 = Pubkey(ABC...)
│     (leader schedule is deterministic, derived from stake distribution)
│  2. Extract signature from shred header (64 bytes)
│  3. Extract the signed payload (Merkle root hash)
│  4. Verify ED25519: verify(signature, merkle_root, leader_pubkey)
│
│  PASS → the slot leader produced this shred
│  FAIL → forged by someone who doesn't have the leader's private key
│
▼
CHECK 2: Merkle Proof Verification
│
│  "Does this shred actually belong to this block?"
│
│  1. Hash the shred data to get the leaf hash
│  2. Walk the Merkle proof (sibling hashes) upward:
│       leaf_hash + proof[0] → H_parent
│       H_parent + proof[1]  → H_grandparent
│       ...                  → computed_root
│  3. Compare computed_root with the signed Merkle root
│
│  PASS → this shred is genuinely part of this FEC set / block
│  FAIL → shred was modified, or swapped from a different block
│
▼
BOTH CHECKS PASS → Shred is cryptographically verified ✓
│
│  This proves two things:
│  1. The leader produced this data (signature)
│  2. This data belongs to this block (Merkle proof)
│
│  A malicious actor would need the leader's private key to forge shreds
│  — which they don't have (unless they ARE the leader, caught by consensus)
```

## 6. Shred to Block Reassembly (What Full Validators Do)

```
Validator collects shreds for Slot 12345 over ~400ms
│
│  Has: DataShred[0], DataShred[1], ... DataShred[199]
│       CodingShred[0], CodingShred[1], ...
│
▼
Check: do we have all data shreds?
├── YES → proceed to concatenation
├── NO, but have enough coding shreds
│       → Use Reed-Solomon decoding to recover missing data shreds
│         (can recover K data shreds from any K of K+M total shreds)
└── NO, not enough of either
        → Send Repair requests to other validators for missing shreds
          (this is the same Repair protocol shred-catcher uses!)
│
▼
Concatenate all data shred payloads in order:
│  DataShred[0].payload ++ DataShred[1].payload ++ ... ++ DataShred[199].payload
│  Result: the original serialized byte stream (~2MB)
│
▼
Deserialize back into Entries
│  Each Entry = PoH hash + list of transactions
│
▼
Execute all transactions against bank state
│  Process each transaction: debit accounts, credit accounts, run programs
│
▼
Vote on the block
│  If state transition is valid → validator votes to confirm
│  Vote is broadcast via gossip to the rest of the network
│
▼
Block confirmed when 2/3 supermajority of stake votes on it
│  Raw shreds are then garbage collected (no longer needed)
```

## 7. Full Lifecycle Summary

```
CREATION:
  Leader creates block from pending transactions
  → serializes into entries
  → chunks entries into Data Shreds (~1051 bytes each)
  → generates Coding Shreds (Reed-Solomon FEC for redundancy)
  → builds Merkle tree over each FEC set of shreds
  → signs the Merkle root with leader's ED25519 private key
  → embeds signature + Merkle proof in every shred

PROPAGATION:
  → broadcasts shreds via Turbine (stake-weighted UDP tree)
  → each validator receives shreds as raw UDP packets
  → validators retransmit to their Turbine children
  → entire network has all shreds within ~200ms

REPAIR (the window shred-catcher exploits):
  → validators keep shreds in memory for a few minutes
  → any gossip participant can request specific shreds via UDP
  → request: RepairProtocol::WindowIndex { slot, shred_index }
  → response: raw shred bytes
  → *** shred-catcher captures shreds here ***

VERIFICATION (what tinydancer does with captured shreds):
  → verify leader signature (ED25519 on Merkle root)
  → verify Merkle proof (shred belongs to this block)
  → sample r random shreds → probability of undetected attack = 1/2^r

ASSEMBLY:
  → validators reassemble block from data shreds
  → recover missing shreds via Reed-Solomon + Repair
  → execute transactions → update state

CONSENSUS:
  → validators vote on the block
  → 2/3 supermajority stake confirms it
  → shreds are garbage collected (gone forever from most validators)
```

## Why This Tool Can't Exist As An RPC Extension

Standard Solana RPCs serve **blocks** (the assembled result after shreds are discarded).
By the time you call `getBlock`, the raw shreds with their Merkle proofs and leader
signatures are already gone. There is no RPC method to retrieve them because validators
don't keep them.

The only way to get raw shreds is to be present on the network during the short
window between broadcast and garbage collection — either passively via Turbine or
actively via Repair. That's what shred-catcher does.

diet-rpc-validator solved this by modifying the validator to NOT discard shreds and
serving them via a custom `getShreds` RPC. But that requires running a full validator
(128GB RAM, 12+ CPU cores). Shred-catcher achieves the same result with 2-4GB RAM
by capturing shreds in real-time from the live network.
