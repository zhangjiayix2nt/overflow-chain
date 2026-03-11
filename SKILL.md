---
name: overflow-chain
description: Build and run Overflow Chain (OFC) nodes — a Bitcoin-inspired blockchain whose total supply of 184,467,440,737.09551616 OFC (exactly 2^64 smallest units) is a homage to CVE-2010-5139. Uses SHA-256 double-hash PoW, Ed25519 wallets, Merkle trees, and TCP P2P networking. Lightweight pure-Python implementation runs on machines with as little as 1GB RAM and 1 CPU. Use when the user mentions OFC, Overflow Coin, overflow chain, proof of work, mining, blockchain node, crypto wallet, P2P network, block explorer, or wants to build, run, join, or mine on a blockchain. Also trigger for general blockchain prototype requests.
version: 2.0.0
metadata: {"openclaw":{"emoji":"⛓","homepage":"https://github.com/zhangjiayix2nt/overflow-chain","requires":{"bins":["python3"]}}}
---

# Overflow Chain (OFC) — Lightweight Edition

## Origin Story

On August 15, 2010, Bitcoin block 74638 created **184,467,440,737.09551616 BTC**
out of nothing via an integer overflow (CVE-2010-5139). Satoshi patched it in five hours.
Overflow Chain immortalizes that number as its total supply.

**Overflow Coin (OFC)** — *from a bug, a chain is born.*

## Design Philosophy: Run Anywhere

This edition targets **minimal hardware**: 1 CPU, 1GB RAM, no compiler needed.

```
Previous edition (v1)          This edition (v2 Lite)
───────────────────            ──────────────────────
Rust + cargo build             Pure Python 3.8+ (no compile)
libp2p (~200 crates)           Raw TCP + JSON protocol
LevelDB (C++ native)           SQLite (Python stdlib)
~200MB runtime RAM             ~30-50MB runtime RAM
Needs 2GB+ to compile          pip install ed25519 (only dep)
```

**All chain parameters, consensus rules, and protocol behavior are identical.**
Only the implementation stack changed. A v1 Rust node and a v2 Python node
speaking the same protocol can interoperate.

---

## Chain Parameters (unchanged)

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
P2P Protocol          TCP + JSON-line protocol
Default Port          18333
Default RPC Port      18332
Data Directory        ~/.overflowchain/
Magic Bytes           0x4F 0x46 0x43 0x21  ("OFC!")
```

---

## Requirements

```
Python 3.8+      (pre-installed on most Linux/macOS)
ed25519           (pip install ed25519 — pure Python, ~20KB)
                  OR: PyNaCl for better performance if available
```

No compiler, no Rust, no C libraries, no LevelDB, no libp2p.
Everything else uses Python standard library: hashlib, sqlite3, socket,
threading, json, struct, os, argparse.

---

## Quick Decision

| User Says | Build |
|-----------|-------|
| "build overflow chain" / "run OFC node" | Full node |
| "mine OFC" / "start mining" | Miner |
| "create wallet" / "send OFC" | Wallet & tx |
| "set up network" / "connect nodes" | P2P setup |
| "block explorer" / "dashboard" | Explorer UI |

---

## Data Directory

```
~/.overflowchain/
├── overflow.conf           # Configuration (INI format)
├── chain.db                # SQLite: blocks, chainstate, tx index (ALL IN ONE)
├── wallet.db               # SQLite: encrypted keys, labels
├── mempool.json            # Pending transactions
├── peers.json              # Known peer list
├── banlist.json            # Banned peers
└── debug.log               # Log file
```

**Key simplification**: v1 used separate blk*.dat files + LevelDB indexes +
LevelDB chainstate (3 storage systems). v2 uses **one SQLite file** for
everything. SQLite is built into Python, needs zero install, handles concurrent
reads, and works on every OS.

### SQLite Schema

```sql
-- chain.db

CREATE TABLE blocks (
    height      INTEGER PRIMARY KEY,
    hash        BLOB NOT NULL UNIQUE,       -- 32 bytes
    prev_hash   BLOB NOT NULL,
    merkle_root BLOB NOT NULL,
    timestamp   INTEGER NOT NULL,
    diff_bits   INTEGER NOT NULL,
    nonce       INTEGER NOT NULL,
    raw_header  BLOB NOT NULL,              -- 80 bytes canonical
    raw_body    BLOB NOT NULL,              -- serialized transactions
    chain_work  TEXT NOT NULL               -- cumulative work (hex string)
);
CREATE INDEX idx_blocks_hash ON blocks(hash);

CREATE TABLE chainstate (
    address     BLOB PRIMARY KEY,           -- 20 bytes
    balance     INTEGER NOT NULL DEFAULT 0, -- in smallest units (u64)
    nonce       INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE tx_index (
    tx_hash     BLOB PRIMARY KEY,           -- 32 bytes
    block_height INTEGER NOT NULL,
    tx_offset   INTEGER NOT NULL            -- index within block
);

CREATE TABLE meta (
    key         TEXT PRIMARY KEY,
    value       TEXT
);
-- meta keys: "best_hash", "best_height", "genesis_hash"

-- wallet.db
CREATE TABLE keys (
    address     TEXT PRIMARY KEY,
    enc_seed    BLOB NOT NULL,              -- AES-encrypted Ed25519 seed
    pubkey      BLOB NOT NULL,
    label       TEXT DEFAULT '',
    created_at  INTEGER
);

CREATE TABLE wallet_meta (
    key         TEXT PRIMARY KEY,
    value       TEXT
);
-- wallet_meta keys: "salt", "iv", "kdf_iterations"
```

---

## Module 1: Proof of Work

Read `{baseDir}/references/pow_consensus.md` for full spec.

### Mining Loop (single-file, stdlib only)

```python
import hashlib, struct, time, threading

def double_sha256(data: bytes) -> bytes:
    return hashlib.sha256(hashlib.sha256(data).digest()).digest()

def mine_block(header_bytes: bytearray, target: bytes, start_nonce=0, stop_event=None):
    """Mine by iterating nonce in header bytes [76:80]. Returns nonce or None."""
    nonce = start_nonce
    while nonce < 0xFFFFFFFF:
        if stop_event and stop_event.is_set():
            return None
        struct.pack_into('<I', header_bytes, 76, nonce)
        h = double_sha256(bytes(header_bytes))
        if h[::-1] <= target:  # compare as big-endian
            return nonce
        nonce += 1
    return None
```

Multi-threaded: spawn N threads, each searching a nonce partition.
On 1 CPU, use 1 thread. No overhead.

### Halving (unchanged)

```python
INITIAL_REWARD = 43_920_819_223_131
HALVING_INTERVAL = 210_000

def block_reward(height: int) -> int:
    halvings = height // HALVING_INTERVAL
    if halvings >= 46:
        return 0
    return INITIAL_REWARD >> halvings
```

---

## Module 2: Wallet

### Crypto (stdlib + ed25519)

```python
import os, hashlib

# Key generation — only external dependency
import ed25519  # pip install ed25519 (pure Python, ~20KB)

def generate_keypair():
    signing_key, verifying_key = ed25519.create_keypair(entropy=os.urandom)
    return signing_key, verifying_key

def address_from_pubkey(pubkey_bytes: bytes) -> str:
    h = hashlib.sha256(pubkey_bytes).digest()[:20]
    return base58check_encode(b'\x00' + h)  # version byte 0x00
```

### Wallet Encryption (stdlib, low-memory)

```python
import hashlib, os

# Use PBKDF2 instead of scrypt — stdlib, uses <1MB RAM
def derive_key(passphrase: str, salt: bytes, iterations=100_000) -> bytes:
    return hashlib.pbkdf2_hmac('sha256', passphrase.encode(), salt, iterations)

# AES-CBC via PyCryptodome if available, fallback to XOR-based ChaCha20
# Or simplest: store seed encrypted with PBKDF2-derived key using XOR stream
```

### Transaction (unchanged format)

```python
import struct

def serialize_tx(tx: dict) -> bytes:
    """Canonical serialization for signing and hashing."""
    return struct.pack('<I', tx['version']) + \
           tx['sender'] + tx['recipient'] + \
           struct.pack('<QQQ', tx['amount'], tx['fee'], tx['nonce']) + \
           struct.pack('<Q', tx['timestamp']) + \
           struct.pack('<H', len(tx['payload'])) + tx['payload']
```

---

## Module 3: Chain Storage (SQLite)

```python
import sqlite3

class ChainDB:
    def __init__(self, path='~/.overflowchain/chain.db'):
        self.conn = sqlite3.connect(os.path.expanduser(path))
        self.conn.execute("PRAGMA journal_mode=WAL")  # concurrent reads
        self.conn.execute("PRAGMA synchronous=NORMAL") # faster writes
        self._create_tables()

    def add_block(self, block):
        """Atomic: insert block + update chainstate in one transaction."""
        with self.conn:
            self.conn.execute(
                "INSERT INTO blocks VALUES (?,?,?,?,?,?,?,?,?,?)",
                (block.height, block.hash, block.prev_hash, ...)
            )
            for tx in block.transactions:
                self._apply_tx(tx)
            self.conn.execute(
                "INSERT OR REPLACE INTO meta VALUES ('best_height', ?)",
                (str(block.height),)
            )

    def get_block_by_hash(self, h: bytes):
        return self.conn.execute(
            "SELECT * FROM blocks WHERE hash=?", (h,)
        ).fetchone()

    def get_balance(self, address: bytes) -> int:
        row = self.conn.execute(
            "SELECT balance FROM chainstate WHERE address=?", (address,)
        ).fetchone()
        return row[0] if row else 0
```

**Why SQLite over LevelDB:**
- Zero install (Python stdlib)
- Single file (easy backup: just copy chain.db)
- SQL queries (easy debugging: `sqlite3 chain.db "SELECT COUNT(*) FROM blocks"`)
- WAL mode handles concurrent reads from RPC while mining writes
- ~30MB base memory vs LevelDB's ~50MB+

---

## Module 4: P2P Network (Pure TCP)

Read `{baseDir}/references/p2p_network.md` for full details.

### Protocol: JSON Lines over TCP

```
Each message = one JSON object + newline (\n)
Connection encrypted with: TLS (ssl stdlib module) or plaintext for local testing
```

```python
import socket, ssl, json, threading

class PeerConnection:
    def __init__(self, sock):
        self.sock = sock
        self.reader = sock.makefile('r')
        self.writer = sock.makefile('w')

    def send(self, msg_type: str, payload: dict):
        msg = json.dumps({"type": msg_type, "data": payload}) + "\n"
        self.writer.write(msg)
        self.writer.flush()

    def recv(self) -> dict:
        line = self.reader.readline()
        if not line:
            raise ConnectionError("peer disconnected")
        return json.loads(line)
```

### Message Types (same semantics as v1, simpler encoding)

```python
# Handshake
{"type": "version", "data": {"protocol": 1, "height": 142, "agent": "ofc-py/2.0"}}
{"type": "verack",  "data": {}}

# Block propagation
{"type": "new_block", "data": {"block": "<hex-encoded block>"}}
{"type": "get_block", "data": {"hash": "<hex>"}}
{"type": "block",     "data": {"block": "<hex>"}}

# Transactions
{"type": "new_tx", "data": {"tx": "<hex>"}}

# Chain sync
{"type": "get_headers", "data": {"from_height": 0, "count": 500}}
{"type": "headers",     "data": {"headers": ["<hex>", ...]}}
{"type": "get_blocks",  "data": {"hashes": ["<hex>", ...]}}

# Peer discovery
{"type": "get_peers", "data": {}}
{"type": "peers",     "data": {"peers": [{"host": "1.2.3.4", "port": 18333}]}}
```

### Node Server

```python
class Node:
    def __init__(self, host='0.0.0.0', port=18333):
        self.server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server.bind((host, port))
        self.server.listen(20)  # 20 max connections (low-memory friendly)
        self.peers = {}         # peer_id -> PeerConnection
        self.chain = ChainDB()
        self.mempool = []

    def start(self):
        threading.Thread(target=self._accept_loop, daemon=True).start()
        # Connect to seed nodes from config
        for seed in self.config.get('seednodes', []):
            self._connect_to(seed)

    def _accept_loop(self):
        while True:
            sock, addr = self.server.accept()
            threading.Thread(target=self._handle_peer, args=(sock, addr), daemon=True).start()
```

### Gossip (simplified GossipSub)

```python
def broadcast_block(self, block):
    """Flood to all connected peers (simple but effective for small networks)."""
    msg = {"type": "new_block", "data": {"block": block.to_hex()}}
    seen = set()
    for peer_id, conn in self.peers.items():
        if peer_id not in seen:
            conn.send(msg["type"], msg["data"])
            seen.add(peer_id)
```

For small networks (<100 nodes), simple flood is sufficient and uses zero
extra memory compared to GossipSub's mesh state tracking.

### NAT / WAN

Same options as v1:
- Port forwarding (simplest)
- Tailscale/WireGuard VPN (zero-config for NAT-to-NAT)
- No relay needed — TCP direct connection works through VPN

TLS encryption (optional, recommended for WAN):
```python
import ssl
context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
context.load_cert_chain('node.crt', 'node.key')
secure_sock = context.wrap_socket(sock, server_side=True)
```

---

## Configuration: overflow.conf

```ini
[network]
port=18333
rpcport=18332
maxconnections=20
seednode=seed1.overflowchain.example:18333

[mining]
mine=true
mineraddress=ofc1YourAddress...
threads=1

[storage]
datadir=~/.overflowchain

[logging]
level=info
file=debug.log
```

---

## CLI

```
python3 ofc.py init                        # Create data dir + genesis
python3 ofc.py start [--mine]              # Run node
python3 ofc.py wallet create [label]       # New keypair
python3 ofc.py wallet list                 # Addresses + balances
python3 ofc.py send <to> <amount> [fee]    # Send OFC
python3 ofc.py balance [address]           # Check balance
python3 ofc.py getblock <hash|height>      # Block details
python3 ofc.py getinfo                     # Status, height, peers
python3 ofc.py peers list|add|ban          # Peer management
```

---

## Project Structure

```
overflow-chain/
├── ofc.py                  # CLI entry point (argparse)
├── config.py               # Config file parsing
├── consensus/
│   ├── __init__.py
│   ├── pow.py              # Mining loop (threading)
│   ├── difficulty.py       # Target & retarget
│   └── coinbase.py         # Rewards & halving
├── wallet/
│   ├── __init__.py
│   ├── keys.py             # Ed25519 (ed25519 package)
│   ├── address.py          # Base58Check
│   └── transaction.py      # Tx creation & signing
├── chain/
│   ├── __init__.py
│   ├── block.py            # Block structure & serialization
│   ├── merkle.py           # Merkle tree
│   ├── blockchain.py       # Chain management, reorg
│   └── chaindb.py          # SQLite storage (blocks + state + index)
├── network/
│   ├── __init__.py
│   ├── node.py             # TCP server + peer manager
│   ├── protocol.py         # JSON-line message handlers
│   └── sync.py             # Headers-first IBD
├── api/
│   └── rpc.py              # HTTP JSON-RPC (http.server stdlib)
└── tests/
    └── test_all.py
```

**Total external dependency: 1 package (ed25519)**. Everything else is stdlib.

---

## Implementation Order

1. **Core** — hashlib SHA-256, Ed25519 keys, serialization (struct)
2. **Storage** — SQLite schema, ChainDB class
3. **Chain** — Block, Merkle tree, genesis, validation
4. **Wallet** — Keys, PBKDF2 encryption, transactions
5. **Consensus** — Mining loop, difficulty, coinbase halving
6. **Network** — TCP server, JSON-line protocol, peer management, IBD
7. **CLI** — argparse commands
8. **Explorer** (optional) — http.server stdlib RPC

---

## Resource Usage

```
                    v1 (Rust+libp2p)    v2 Lite (Python)
──────────────      ────────────────    ────────────────
Compile RAM         2+ GB               0 (interpreted)
Runtime RAM         ~200 MB             ~30-50 MB
Disk (binary)       ~30 MB              ~50 KB source
Disk (data)         blk*.dat+LevelDB    single chain.db
External deps       ~200 crates         1 pip package
Min CPU             2 cores             1 core
Min RAM             2 GB                512 MB
Install time        10+ minutes         5 seconds
```

---

## Reference Files

- `{baseDir}/references/pow_consensus.md` — Mining, difficulty, halving, security
- `{baseDir}/references/p2p_network.md` — P2P protocol, NAT traversal, sync
- `{baseDir}/references/explorer_ui.md` — RPC API, WebSocket, UI
