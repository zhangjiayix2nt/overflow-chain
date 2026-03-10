---
name: overflow-chain
description: Build and run Overflow Chain (OFC) nodes — a Bitcoin-inspired blockchain whose total supply of 184,467,440,737.09551616 OFC (exactly 2^64 smallest units) is a homage to CVE-2010-5139. Uses SHA-256 double-hash PoW, Ed25519 wallets, Merkle trees, libp2p P2P networking, and Bitcoin-identical halving curve and storage architecture. Use when the user mentions OFC, Overflow Coin, overflow chain, proof of work, mining, blockchain node, crypto wallet, P2P network, libp2p, block explorer, or wants to build, run, join, or mine on a blockchain. Also trigger for general blockchain prototype requests.
version: 1.0.0
metadata: {"openclaw":{"emoji":"⛓","homepage":"https://github.com/overflow-chain","requires":{"anyBins":["cargo","python3","node"]},"install":[{"id":"rust","kind":"download","url":"https://sh.rustup.rs","label":"Install Rust toolchain"}]}}
---

# Overflow Chain (OFC)

## Origin Story

On August 15, 2010, Bitcoin block 74638 contained a transaction that created
**184,467,440,737.09551616 BTC** out of nothing — an integer overflow in the
output sum check (CVE-2010-5139). Satoshi patched it within five hours.

Overflow Chain immortalizes that number. Total supply = exactly
**184,467,440,737.09551616 OFC** — the same 2^64 smallest units that once
briefly existed on Bitcoin's "bad chain".

**Overflow Coin (OFC)** — *from a bug, a chain is born.*

---

## Chain Parameters

```
Name                  Overflow Chain
Ticker                OFC
Total Supply          184,467,440,737.09551616 OFC (2^64 smallest units)
Smallest Unit         10^-8 OFC (8 decimal places, called "bits")
Initial Block Reward  439,208.19223131 OFC (43,920,819,223,131 bits)
Halving Interval      210,000 blocks
Number of Halvings    45 (then reward = 0, fees only)
Target Block Time     600 seconds (10 minutes)
Difficulty Adjustment Every 2,016 blocks (~2 weeks)
Consensus             SHA-256 double-hash Proof of Work
Signatures            Ed25519
Addresses             Base58Check(SHA-256(pubkey)[0..20])
P2P Protocol          libp2p (TCP + Noise + Yamux + GossipSub)
Default Port          18333
Default RPC Port      18332
Data Directory        ~/.overflowchain/
Magic Bytes           0x4F 0x46 0x43 0x21  ("OFC!")
```

---

## Quick Decision

| User Says | Build |
|-----------|-------|
| "build overflow chain" / "run OFC node" | Full node + network join |
| "mine OFC" / "start mining" | PoW miner setup |
| "create wallet" / "send OFC" | Wallet & transaction |
| "set up network" / "connect nodes" | P2P multi-node setup |
| "block explorer" / "dashboard" | Explorer UI |
| "build a blockchain" / "make a coin" | Full stack from scratch |

---

## Language Selection

- **User specified**: Use that.
- **Full node with P2P**: **Rust** (best libp2p, production-grade).
- **Rapid prototyping**: **Python**.
- **Web demo**: **TypeScript**.

---

## First-Run: Joining the Overflow Network

### Step 1: Initialize

```bash
overflowchain init
```

Creates:

```
~/.overflowchain/
├── overflow.conf                  # Configuration
├── blocks/
│   ├── blk00000.dat               # Raw blocks (append-only, ≤128MB/file)
│   └── index/                     # LevelDB: block_hash → file position
├── chainstate/                    # LevelDB: address → {balance, nonce}
├── wallets/
│   └── wallet.dat                 # Default wallet (scrypt + AES-256-CBC)
├── mempool.dat                    # Persisted mempool
├── peers.dat                      # Known peer addresses
├── banlist.json                   # Banned peers
└── debug.log                      # Node log
```

### Step 2: Connect

```bash
overflowchain start
```

1. Load `peers.dat` or fall back to seed nodes in `overflow.conf`
2. Connect to 8 outbound peers (libp2p TCP + Noise + Yamux)
3. Accept up to 125 inbound connections
4. Exchange Version/VerAck handshake

### Step 3: Sync (IBD)

1. Download headers in batches of 2,000
2. Validate header chain (PoW, prev_hash, timestamps)
3. Download full blocks in parallel
4. Validate blocks (signatures, Merkle root, balances, coinbase)
5. Update chainstate DB
6. Switch to real-time GossipSub at chain tip

### Step 4: Mine

```bash
overflowchain start --mine --address ofc1YourAddress...
```

---

## Module 1: Proof of Work Consensus

Read `{baseDir}/references/pow_consensus.md` for full spec.

### Mining

```
block_hash = SHA-256(SHA-256(header_80_bytes))
Find nonce: block_hash < target
```

### Block Header (80 bytes, little-endian)

```
version          4B    u32
prev_block_hash  32B   double-SHA-256 of previous header
merkle_root      32B
timestamp        4B    unix epoch u32
difficulty_bits  4B    compact target
nonce            4B    u32
```

### Halving

```rust
const INITIAL_REWARD: u64 = 43_920_819_223_131;
const HALVING_INTERVAL: u64 = 210_000;

fn block_reward(height: u64) -> u64 {
    let halvings = height / HALVING_INTERVAL;
    if halvings >= 46 { return 0; }
    INITIAL_REWARD >> halvings
}
```

### Difficulty Adjustment

Every 2,016 blocks: `new_target = old_target × clamp(actual / expected, 0.25, 4.0)`

### Checklist

- [ ] Double-SHA-256 mining (multi-threaded)
- [ ] Compact bits ↔ 256-bit target conversion
- [ ] Difficulty retarget every 2,016 blocks
- [ ] Coinbase: `block_reward(height) + sum(fees)`, 100-block maturity
- [ ] Hashrate display

---

## Module 2: Wallet & Transactions

### wallet.dat (encrypted)

```
Header:  magic "OFC!" | version u32
Crypto:  scrypt(passphrase, salt) → AES-256-CBC key
Payload: Vec<{ ed25519_seed, pubkey, address, label, created_at }>
```

### Transaction

```
version u32 | sender [u8;20] | recipient [u8;20] | amount u64 | fee u64
nonce u64 | timestamp u64 | payload Vec<u8> ≤256B | signature [u8;64] Ed25519
tx_hash = SHA-256(all fields except signature)
```

### Checklist

- [ ] Ed25519 keys, encrypted wallet.dat
- [ ] Address: `Base58Check(SHA-256(pubkey)[0..20])`
- [ ] Account state: address → {balance, nonce}
- [ ] Mempool with fee-priority, persisted to `mempool.dat`

---

## Module 3: Chain Data Structures

### Block Storage (Bitcoin-style)

```
blocks/blkNNNNN.dat:
  [magic "OFC!" 4B][size 4B][block raw bytes] ...
blocks/index/ (LevelDB):
  "b"+hash → {file_num, offset, height, num_tx, size, header}
  "h"+height → hash
  "B" → best_block_hash
```

File rotation at 128 MB.

### Chainstate (LevelDB)

```
"a"+address → {balance: u64, nonce: u64}
"B" → best_block_hash, "H" → best_height
```

### Merkle Tree

Leaf = SHA-256(tx_hash), Internal = SHA-256(left ‖ right), odd → dup last.
Implement `get_proof()` + `verify_proof()`.

### Chain Selection

Most cumulative work: `chain_work = Σ(2^256 / target)`.

---

## Module 4: P2P Network

Read `{baseDir}/references/p2p_network.md` for full details including NAT traversal.

```
Transport:  TCP + Noise + Yamux (+ Circuit Relay v2 for NAT)
Discovery:  mDNS (local) + Kademlia DHT + DNS seeds
Messaging:  GossipSub → /ofc/blocks/1, /ofc/txs/1
Sync:       Headers-first IBD
Port:       18333
```

### WAN Setup

```
Option A: One has public IP → other adds seednode=PUBLIC_IP:18333
Option B: Both behind NAT → relay node on VPS + DCUtR hole punching
Option C: No VPS → both install Tailscale, use Tailscale IPs
```

### Checklist

- [ ] libp2p: TCP + Noise + Yamux + Relay + DCUtR
- [ ] GossipSub: `/ofc/blocks/1`, `/ofc/txs/1`
- [ ] mDNS + Kademlia + DNS seeds
- [ ] Headers-first IBD
- [ ] `peers.dat` persistence, peer scoring, `banlist.json`

---

## Configuration: overflow.conf

```ini
port=18333
rpcport=18332
maxconnections=125
seednode=seed1.overflowchain.example:18333
mine=true
mineraddress=ofc1YourAddress...
minerthreads=4
datadir=~/.overflowchain
loglevel=info
```

---

## CLI Commands

```
overflowchain init                          # Create data dir + genesis
overflowchain start [--mine]                # Run node
overflowchain wallet create [label]         # New keypair
overflowchain wallet list                   # Addresses + balances
overflowchain send <to> <amount> [fee]      # Send OFC
overflowchain balance [address]             # Check balance
overflowchain getblock <hash|height>        # Block details
overflowchain getinfo                       # Status, height, peers, hashrate
overflowchain peers list|add|ban            # Peer management
```

---

## Project Structure (Rust)

```
overflow-chain/
├── Cargo.toml
├── src/
│   ├── main.rs, config.rs
│   ├── consensus/ { pow.rs, difficulty.rs, coinbase.rs }
│   ├── wallet/    { keypair.rs, address.rs, transaction.rs, wallet_file.rs }
│   ├── chain/     { block.rs, merkle.rs, blockchain.rs, state.rs }
│   ├── storage/   { block_store.rs, block_index.rs, chainstate_db.rs }
│   ├── network/   { host.rs, protocol.rs, gossip.rs, sync.rs, peer_manager.rs, nat.rs }
│   └── api/       { rpc.rs }
├── tests/
└── explorer/index.html
```

---

## Implementation Order

1. **Core** — Types, SHA-256, Ed25519, serialization
2. **Storage** — blk*.dat, LevelDB block index, chainstate
3. **Chain** — Block, Merkle, genesis, validation, reorg
4. **Wallet** — Keys, encrypted wallet.dat, transactions, mempool
5. **Consensus** — PoW mining, difficulty, coinbase halving
6. **Network** — libp2p, handshake, GossipSub, IBD, NAT traversal
7. **CLI & Config** — Commands, overflow.conf
8. **Explorer** (optional) — RPC + web UI

---

## Reference Files

- `{baseDir}/references/pow_consensus.md` — Mining, difficulty, halving, security
- `{baseDir}/references/p2p_network.md` — libp2p, NAT traversal, sync
- `{baseDir}/references/explorer_ui.md` — RPC API, WebSocket, UI
