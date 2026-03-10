# P2P Network Guide

## Architecture Overview

The network layer uses **libp2p** — the same networking stack used by Ethereum 2.0,
Filecoin, IPFS, and Polkadot. This gives the prototype real-world networking patterns
rather than toy socket code.

```
┌──────────────────────────────────────────────┐
│                  Application                  │
│         (Block sync, Tx broadcast)            │
├──────────────────────────────────────────────┤
│              GossipSub (Pub/Sub)              │
│        Topics: /blocks, /transactions         │
├──────────────────────────────────────────────┤
│           Kademlia DHT + mDNS                 │
│             (Peer Discovery)                  │
├──────────────────────────────────────────────┤
│        Request/Response Protocol              │
│    (Chain sync, block/header requests)        │
├──────────────────────────────────────────────┤
│     Yamux (Stream Multiplexing)               │
├──────────────────────────────────────────────┤
│     Noise (Encryption & Authentication)       │
├──────────────────────────────────────────────┤
│              TCP Transport                    │
└──────────────────────────────────────────────┘
```

---

## WAN Networking: Connecting Across the Internet

### The NAT Problem

Most home/office computers sit behind a router doing NAT (Network Address Translation).
Two nodes both behind NAT cannot directly connect to each other — neither knows the
other's real address, and incoming connections are blocked by default.

```
Scenario A — Both have public IPs (cloud servers):
  Node A (1.2.3.4:8333) ←──── TCP ────→ Node B (5.6.7.8:8333)
  ✅ Works directly. Just add each other's IP to seednode config.

Scenario B — One public, one behind NAT:
  Node A (public 1.2.3.4) ←──── TCP ────→ Node B (behind NAT)
  ✅ Works: B connects outbound to A. A accepts inbound.
  ❌ A cannot initiate connection to B (NAT blocks inbound).

Scenario C — Both behind NAT:
  Node A (behind NAT) ──╳──── TCP ────╳──→ Node B (behind NAT)
  ❌ Neither can reach the other directly.
  → Requires NAT traversal techniques (see below).
```

### Solution 1: Port Forwarding (simplest, recommended for testing)

On your router, forward external port 8333 to your machine's local IP:

```
Router admin → Port Forwarding:
  External port: 8333 → Internal IP: 192.168.1.100, port: 8333

mychain.conf:
  port=8333
  externalip=YOUR_PUBLIC_IP:8333
```

Then share your public IP with peers. Find it via `curl ifconfig.me`.

### Solution 2: libp2p Relay (for NAT-to-NAT)

libp2p has a built-in relay protocol (Circuit Relay v2). A public node acts
as a relay — both NAT'd nodes connect outbound to the relay, which bridges
their traffic.

```
Node A (NAT) ──outbound──→ Relay Server (public) ←──outbound── Node B (NAT)
                           Relay bridges A ↔ B
```

Implementation:

```rust
// Cargo.toml: add "relay" to libp2p features
libp2p = { version = "0.54", features = [
    "tcp", "noise", "yamux", "gossipsub", "mdns", "kad",
    "relay",           // ← Add this
    "dcutr",           // ← Direct Connection Upgrade (hole punching)
    "identify", "macros", "tokio",
] }
```

```rust
use libp2p::{relay, dcutr};

#[derive(NetworkBehaviour)]
struct BlockchainBehaviour {
    gossipsub: gossipsub::Behaviour,
    mdns: mdns::tokio::Behaviour,
    kademlia: kad::Behaviour<kad::store::MemoryStore>,
    relay_client: relay::client::Behaviour,  // ← Connect via relay
    dcutr: dcutr::Behaviour,                 // ← Then try hole-punch
    identify: identify::Behaviour,
    // ... other behaviours
}
```

Configure relay nodes in `mychain.conf`:

```ini
relay=/ip4/RELAY_PUBLIC_IP/tcp/8333/p2p/RELAY_PEER_ID
```

### Solution 3: DCUtR Hole Punching (automatic, best experience)

After connecting through a relay, libp2p's DCUtR (Direct Connection Upgrade
through Relay) attempts to establish a direct connection using UDP/TCP hole
punching. If successful, the relay is no longer needed.

```
Step 1: A ──relay──→ Relay ←──relay── B   (relayed connection)
Step 2: A ──────── hole punch ────────→ B  (DCUtR upgrades to direct)
Step 3: A ←────── direct TCP ─────────→ B  (relay dropped)
```

This works for ~80% of NAT types (cone NAT). Symmetric NAT (common in
carrier-grade NAT / mobile networks) cannot be hole-punched.

### Solution 4: VPN / Overlay Network (fallback)

If hole punching fails (symmetric NAT), use a VPN to create a virtual LAN:

- **Tailscale** (easiest): `tailscale up` on both machines → they get virtual IPs
  on the same network. Zero config. Free for personal use.
- **WireGuard**: More control, manual setup. Create a tunnel between machines.
- **ZeroTier**: Similar to Tailscale, creates a virtual LAN.

```ini
# mychain.conf using Tailscale IPs:
seednode=100.64.0.1:8333    # Tailscale IP of other node
```

### Recommended Setup for Two Friends Testing

```
Option A: One person has public IP (or sets up port forwarding)
  → Other person adds their IP as seednode
  → Simple, reliable

Option B: Both behind NAT, no port forwarding
  → Deploy one relay node on a cheap VPS ($5/month)
  → Both nodes connect to relay
  → DCUtR will likely upgrade to direct connection

Option C: Both behind NAT, no VPS
  → Both install Tailscale (free)
  → Use Tailscale IPs as seednodes
  → Works like a LAN, zero NAT issues
```

### Network Stack Summary

```
┌──────────────────────────────────────────────────────┐
│                    Application                        │
│            (Block sync, Tx broadcast)                 │
├──────────────────────────────────────────────────────┤
│                 GossipSub (Pub/Sub)                   │
├──────────────────────────────────────────────────────┤
│              Kademlia DHT + mDNS                      │
├──────────────────────────────────────────────────────┤
│          Request/Response Protocol                    │
├──────────────────────────────────────────────────────┤
│            Yamux (Stream Multiplexing)                 │
├──────────────────────────────────────────────────────┤
│          Noise (Encryption & Authentication)           │
├──────────────────────────────────────────────────────┤
│   TCP Transport ──or── Circuit Relay v2 + DCUtR       │
│   (direct)            (NAT traversal)                 │
└──────────────────────────────────────────────────────┘
```

---

## Setup by Language

### Rust (recommended for full prototype)

```toml
# Cargo.toml
[dependencies]
libp2p = { version = "0.54", features = [
    "tcp",
    "noise",
    "yamux",
    "gossipsub",
    "mdns",
    "kad",
    "request-response",
    "identify",
    "macros",
    "tokio",
] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
use libp2p::{
    gossipsub, identify, kad, mdns, noise,
    request_response, swarm::NetworkBehaviour,
    tcp, yamux, SwarmBuilder, Multiaddr,
};

#[derive(NetworkBehaviour)]
struct BlockchainBehaviour {
    gossipsub: gossipsub::Behaviour,
    mdns: mdns::tokio::Behaviour,
    kademlia: kad::Behaviour<kad::store::MemoryStore>,
    request_response: request_response::cbor::Behaviour<ChainRequest, ChainResponse>,
    identify: identify::Behaviour,
}
```

### Python (for rapid prototyping)

```bash
pip install libp2p py-multiaddr
```

Note: Python libp2p (`py-libp2p`) is less mature than Rust. For Python prototypes,
consider using a simplified direct TCP approach with the same message protocol:

```python
import asyncio
import json

class P2PNode:
    def __init__(self, host='0.0.0.0', port=0):
        self.host = host
        self.port = port
        self.peers = {}          # peer_id -> (reader, writer)
        self.handlers = {}       # message_type -> handler_fn
        self.peer_id = generate_peer_id()
    
    async def start(self):
        self.server = await asyncio.start_server(
            self._handle_connection, self.host, self.port
        )
        self.port = self.server.sockets[0].getsockname()[1]
    
    async def connect(self, host, port):
        reader, writer = await asyncio.open_connection(host, port)
        await self._handshake(reader, writer)
    
    async def broadcast(self, message_type, payload):
        msg = json.dumps({"type": message_type, "data": payload})
        for peer_id, (_, writer) in self.peers.items():
            writer.write((msg + "\n").encode())
            await writer.drain()
```

### TypeScript

```bash
npm install @libp2p/tcp @libp2p/noise @libp2p/yamux
npm install @chainsafe/libp2p-gossipsub @libp2p/mdns
npm install @libp2p/kad-dht libp2p
```

```typescript
import { createLibp2p } from 'libp2p'
import { tcp } from '@libp2p/tcp'
import { noise } from '@libp2p/noise'
import { yamux } from '@libp2p/yamux'
import { gossipsub } from '@chainsafe/libp2p-gossipsub'
import { mdns } from '@libp2p/mdns'

const node = await createLibp2p({
  addresses: { listen: ['/ip4/0.0.0.0/tcp/0'] },
  transports: [tcp()],
  connectionEncrypters: [noise()],
  streamMuxers: [yamux()],
  services: {
    pubsub: gossipsub({ emitSelf: false }),
    mdns: mdns(),
  },
})
```

---

## Message Protocol

### Serialization

Use a compact binary format for efficiency. Recommended:
- **Rust**: `serde` + CBOR (via `serde_cbor` or libp2p's built-in CBOR request-response)
- **Python**: JSON over newline-delimited TCP (simpler, fine for prototype)
- **TypeScript**: Protocol Buffers or JSON

### Message Types

```rust
#[derive(Serialize, Deserialize)]
enum NetworkMessage {
    // === Pub/Sub (GossipSub) ===
    
    /// Broadcast when a new block is mined
    NewBlock {
        block: Block,
        sender_height: u64,
    },
    
    /// Broadcast when a new transaction enters the mempool
    NewTransaction {
        transaction: Transaction,
    },
    
    // === Request/Response ===
    
    /// Request block headers starting from a height
    GetHeaders {
        start_height: u64,
        max_count: u32,   // Cap at 2000
    },
    
    /// Response with headers
    Headers {
        headers: Vec<BlockHeader>,
    },
    
    /// Request full blocks by hash
    GetBlocks {
        hashes: Vec<[u8; 32]>,  // Cap at 128
    },
    
    /// Response with full blocks
    Blocks {
        blocks: Vec<Block>,
    },
    
    /// Request the peer's chain status
    GetStatus,
    
    /// Response with chain status
    Status {
        best_height: u64,
        best_hash: [u8; 32],
        genesis_hash: [u8; 32],
    },
}
```

---

## GossipSub Topics

Define two topics for pub/sub messaging:

```
/blockchain/blocks/1.0.0     — New block announcements
/blockchain/txs/1.0.0        — New transaction broadcasts
```

Configuration for GossipSub:

```rust
let gossipsub_config = gossipsub::ConfigBuilder::default()
    .heartbeat_interval(Duration::from_secs(1))
    .validation_mode(gossipsub::ValidationMode::Strict)
    .mesh_n(6)              // Target mesh peers
    .mesh_n_low(4)          // Min mesh peers
    .mesh_n_high(12)        // Max mesh peers
    .gossip_lazy(6)         // Peers for gossip propagation
    .max_transmit_size(2 * 1024 * 1024)  // 2 MB max message
    .build()
    .expect("valid gossipsub config");
```

---

## Peer Discovery

### Local Development: mDNS

mDNS discovers peers on the same local network automatically. Perfect for testing
multiple nodes on localhost.

```rust
// mDNS events
SwarmEvent::Behaviour(BehaviourEvent::Mdns(mdns::Event::Discovered(peers))) => {
    for (peer_id, addr) in peers {
        swarm.behaviour_mut().gossipsub.add_explicit_peer(&peer_id);
        swarm.dial(addr)?;
    }
}
```

### Production-like: Kademlia DHT

For wider networks (or to demonstrate the real protocol), use Kademlia:

```rust
// Bootstrap with known peers
let bootnode: Multiaddr = "/ip4/127.0.0.1/tcp/4001".parse()?;
swarm.behaviour_mut().kademlia.add_address(&bootnode_peer_id, bootnode);
swarm.behaviour_mut().kademlia.bootstrap()?;
```

---

## Chain Synchronization (Initial Block Download)

When a node starts, it needs to catch up to the network. Implement headers-first sync:

```
1. Connect to peers, exchange Status messages
2. Find the peer with the highest chain
3. Download headers in batches (GetHeaders, starting from our tip)
4. Validate header chain (each header's previous_hash links correctly)
5. Download full blocks in parallel for validated headers (GetBlocks)
6. Validate and apply each block to local chain
7. Once caught up, switch to listening for NewBlock gossip
```

### Sync State Machine

```
        ┌──────────┐
        │  IDLE     │──── peer connected ────►┌──────────────┐
        └──────────┘                          │ EXCHANGING   │
                                              │ STATUS       │
              ┌───────────── peer ahead ──────┤              │
              │                               └──────┬───────┘
              ▼                                      │
        ┌──────────────┐                    peer same/behind
        │ DOWNLOADING  │                             │
        │ HEADERS      │                             ▼
        └──────┬───────┘                      ┌──────────┐
               │                              │  SYNCED   │
               ▼                              │ (gossip)  │
        ┌──────────────┐                      └──────────┘
        │ DOWNLOADING  │                             ▲
        │ BLOCKS       │─── all blocks applied ──────┘
        └──────────────┘
```

---

## Testing Multi-Node Locally

Provide a script to launch multiple nodes for local testing:

```bash
#!/bin/bash
# launch_testnet.sh — Starts N local nodes

NUM_NODES=${1:-3}
BASE_PORT=9000

for i in $(seq 1 $NUM_NODES); do
    PORT=$((BASE_PORT + i))
    echo "Starting node $i on port $PORT..."
    ./target/release/my-blockchain \
        --port $PORT \
        --data-dir "./testnet/node-$i" \
        --bootnodes "/ip4/127.0.0.1/tcp/$((BASE_PORT + 1))" \
        &
    sleep 1  # Give each node time to start
done

echo "Testnet running with $NUM_NODES nodes"
echo "Node 1 (seed): localhost:$((BASE_PORT + 1))"
```

### Integration Test Pattern

```rust
#[tokio::test]
async fn test_block_propagation() {
    // 1. Spawn 3 nodes
    let node_a = spawn_node(9001).await;
    let node_b = spawn_node(9002).await;
    let node_c = spawn_node(9003).await;
    
    // 2. Connect them: A <-> B <-> C
    node_b.connect(node_a.addr()).await;
    node_c.connect(node_b.addr()).await;
    tokio::time::sleep(Duration::from_secs(2)).await;
    
    // 3. Mine block on node A
    let block = node_a.mine_block(vec![]).await;
    
    // 4. Wait for propagation
    tokio::time::sleep(Duration::from_secs(5)).await;
    
    // 5. Verify all nodes have the block
    assert_eq!(node_b.chain_height(), 1);
    assert_eq!(node_c.chain_height(), 1);
    assert_eq!(node_b.best_hash(), block.hash);
    assert_eq!(node_c.best_hash(), block.hash);
}
```

---

## Peer Scoring

Protect against malicious peers:

```rust
struct PeerScore {
    peer_id: PeerId,
    score: f64,              // Starts at 0, range [-100, 100]
    invalid_blocks: u32,     // Blocks that failed validation
    invalid_txs: u32,        // Transactions that failed validation
    timeouts: u32,           // Requests that timed out
    useful_blocks: u32,      // Valid new blocks received
    useful_txs: u32,         // Valid new transactions received
}

impl PeerScore {
    fn update_score(&mut self) {
        self.score = (self.useful_blocks as f64 * 10.0
                    + self.useful_txs as f64 * 1.0
                    - self.invalid_blocks as f64 * 50.0
                    - self.invalid_txs as f64 * 10.0
                    - self.timeouts as f64 * 5.0)
                    .clamp(-100.0, 100.0);
    }
    
    fn should_disconnect(&self) -> bool {
        self.score < -50.0
    }
}
```
