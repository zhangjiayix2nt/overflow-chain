---
name: overflow-chain
description: Build and run Overflow Chain (OFC) nodes — a Bitcoin-inspired blockchain whose total supply of 184,467,440,737.09551616 OFC (exactly 2^64 smallest units) is a homage to CVE-2010-5139. Uses SHA-256 double-hash PoW, Ed25519 wallets, Merkle trees, and TCP P2P networking. Lightweight pure-Python implementation runs on machines with as little as 1GB RAM and 1 CPU. Use when the user mentions OFC, Overflow Coin, overflow chain, proof of work, mining, blockchain node, crypto wallet, P2P network, block explorer, or wants to build, run, join, or mine on a blockchain. Also trigger for general blockchain prototype requests.
version: 2.0.0
metadata: {"openclaw":{"emoji":"⛓","homepage":"https://github.com/overflow-chain","requires":{"bins":["python3"]}}}
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

## Genesis Block (hardcoded — all nodes MUST use this exact genesis)

This is the identity of the Overflow Chain network. Nodes with a different genesis
are on a different network and will refuse to peer.

```
Genesis Block Hash:  0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f
Header (80 bytes):   0100000000000000000000000000000000000000000000000000000000000000
                     000000003312d701d543e9f5b868de9412bf601c2d6185fbd902d0e4f634d3b8
                     e2385e29002e674cffff002020100000

Version:             1
Previous Hash:       0000000000000000000000000000000000000000000000000000000000000000
Merkle Root:         3312d701d543e9f5b868de9412bf601c2d6185fbd902d0e4f634d3b8e2385e29
Timestamp:           1281830400 (2010-08-15 00:00:00 UTC — the date of CVE-2010-5139)
Difficulty Bits:     0x2000ffff
Nonce:               4128

Coinbase Message:    "CVE-2010-5139: The value overflow incident.
                      184467440737.09551616 BTC created from nothing. 2010-08-15"
```

**Implementation — the genesis function must produce this exact block:**

```python
import struct, hashlib

def genesis_block():
    coinbase_msg = (
        b"CVE-2010-5139: The value overflow incident. "
        b"184467440737.09551616 BTC created from nothing. 2010-08-15"
    )
    coinbase_tx_hash = double_sha256(coinbase_msg)

    header = struct.pack('<I', 1)           # version
    header += b'\x00' * 32                  # prev_hash (none)
    header += coinbase_tx_hash              # merkle_root
    header += struct.pack('<I', 1281830400) # timestamp: 2010-08-15
    header += struct.pack('<I', 0x2000ffff) # difficulty_bits
    header += struct.pack('<I', 4128)       # nonce (pre-mined)

    block_hash = double_sha256(header)
    assert block_hash[::-1].hex().startswith("0000d5d1"), "Genesis hash mismatch!"
    return header, coinbase_msg, block_hash
```

**Any node whose genesis hash ≠ `0000d5d1...fc8f` is not on the OFC network.**

---

## Joining an Existing OFC Network

When a user says "join the Overflow Chain network", "connect to OFC", or
"run an OFC node", follow these steps:

### For OpenClaw Users

Tell the user:

```bash
pip install ed25519
python3 ofc.py init
python3 ofc.py start --seednode=<SEED_IP>:18333
```

Where `<SEED_IP>` is the IP of a known running node.

### How Seed Nodes Work

First node bootstraps the network:
```bash
python3 ofc.py init
python3 ofc.py start --mine    # No seeds needed — it IS the seed
# Share your IP with others
```

All subsequent nodes join by pointing to an existing node:
```bash
python3 ofc.py init
python3 ofc.py start --seednode=FIRST_NODE_IP:18333
```

After connecting, nodes exchange peer lists automatically.

### Network Bootstrap Scenarios

```
Scenario 1: Friend has a running node
  → Get their IP (or Tailscale IP)
  → python3 ofc.py start --seednode=FRIEND_IP:18333

Scenario 2: Start a new private network
  → python3 ofc.py init && python3 ofc.py start --mine
  → Share your IP

Scenario 3: VPS as always-on seed
  → VPS: python3 ofc.py start --mine
  → Local: python3 ofc.py start --seednode=VPS_IP:18333

Scenario 4: Two users behind NAT, no VPS
  → Both: tailscale up
  → A: python3 ofc.py start --mine
  → B: python3 ofc.py start --seednode=A_TAILSCALE_IP:18333
```

### overflow.conf Seeds

```ini
[network]
seednode=192.168.1.100:18333
seednode=100.64.0.1:18333
seednode=203.0.113.50:18333
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

## ⚠️ IMPORTANT: Check Before Build (read this FIRST)

Before generating ANY code, ALWAYS check if an OFC project already exists locally.

### Step 1: Search for existing project

```bash
# Check common locations in order:
ls ./overflow-chain/ofc.py 2>/dev/null
ls ~/overflow-chain/ofc.py 2>/dev/null
ls ~/.overflowchain/ofc.py 2>/dev/null
find ~ -maxdepth 3 -name "ofc.py" -path "*/overflow-chain/*" 2>/dev/null
```

### Step 2: Decide

```
IF ofc.py is found:
  → DO NOT regenerate code
  → Tell the user: "Found existing OFC project at <path>"
  → Work with the existing code (run it, debug it, modify it, etc.)
  → If user asks to "build/create/set up" a node, ask:
    "You already have an OFC project at <path>. Do you want to
     use the existing one, or start fresh?"

IF ofc.py is NOT found:
  → Generate the full project from scratch following this skill
  → Write all files to ./overflow-chain/ (or user-specified path)
  → After generating, tell the user the project location
```

### Step 3: Also check for running node

```bash
# Check if an OFC node is already running
lsof -i :18333 2>/dev/null || ss -tlnp | grep 18333
ps aux | grep "ofc.py" | grep -v grep
```

```
IF node is already running on port 18333:
  → DO NOT start another instance
  → Tell the user: "An OFC node is already running (PID: X, port 18333)"
  → Ask what they want to do: check status, stop it, connect to it, etc.
```

This check prevents:
- Regenerating code that already exists (wasting time, risking incompatibility)
- Running duplicate nodes on the same port (will crash)
- Losing existing chain data (the user may have mined blocks already)

### Step 4: Verify project integrity (if existing project found)

If an existing project is found, quickly verify it has the essential files:

```bash
# Required files — if any are missing, offer to regenerate just that file
REQUIRED="ofc.py
consensus/__init__.py
consensus/pow.py
consensus/difficulty.py
consensus/coinbase.py
wallet/__init__.py
wallet/keys.py
wallet/address.py
wallet/transaction.py
chain/__init__.py
chain/block.py
chain/merkle.py
chain/blockchain.py
chain/chaindb.py
network/__init__.py
network/node.py
network/protocol.py
network/sync.py"

cd <project_path>
MISSING=""
for f in $REQUIRED; do
    [ ! -f "$f" ] && MISSING="$MISSING $f"
done
```

```
IF all files present:
  → Project is complete, use as-is

IF some files missing:
  → Tell user: "Your OFC project is missing: <list>"
  → Offer to regenerate ONLY the missing files (not the whole project)
  → This preserves any customizations the user made to other files
```

### Step 5: Verify genesis compatibility

After finding an existing project, verify it uses the correct genesis hash:

```bash
python3 -c "
import sys; sys.path.insert(0, '<project_path>')
from chain.block import genesis_block
_, _, h = genesis_block()
expected = '0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f'
print('Genesis OK' if h[::-1].hex() == expected else 'GENESIS MISMATCH — not OFC network!')
"
```

```
IF genesis matches:
  → Compatible with the OFC network ✓

IF genesis does NOT match:
  → WARN the user: "This project uses a different genesis block.
    It cannot connect to the OFC network."
  → Ask: regenerate with correct genesis, or keep as separate network?
```

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

Read `{baseDir}/references/p2p_network.md` for complete protocol specification,
full handshake code, and detailed troubleshooting guide.

### Connection Protocol (strict order — every step must pass)

```
1. TCP connect
2. Both sides send 4-byte magic: 0x4F464321 ("OFC!")
   → If wrong magic received → close (not an OFC node)
3. Both sides send version JSON (includes genesis_hash)
   → If genesis_hash mismatches → send reject + close (different network)
4. Both sides send verack
5. Connection established — now accept all message types
```

Every failure MUST be logged with the specific step and reason.

### Magic Bytes — First Line of Defense

```python
MAGIC = b'\x4f\x46\x43\x21'  # "OFC!" — first 4 raw bytes of every connection
```

Send immediately after TCP connect. If first 4 received bytes ≠ `4F464321`,
close at once. This rejects HTTP bots, SSH scanners, and non-OFC nodes
within milliseconds before wasting any further resources.

### Version Message — Network Identity

```python
{"type": "version", "data": {
    "protocol": 1,
    "agent": "ofc-py/2.0.0",
    "height": 42,
    "genesis_hash": "0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f",
    "port": 18333,
    "timestamp": 1710000000,
    "nonce": 847162539
}}
```

**genesis_hash is mandatory.** If two nodes have different genesis hashes, they
are on different networks and MUST NOT connect. Send a `reject` message with
code `genesis_mismatch` before closing so the peer gets a clear error.

### Reject Message (send before disconnecting on errors)

```python
{"type": "reject", "data": {
    "code": "genesis_mismatch",    # machine-readable
    "reason": "peer=abcd... ours=0000d5d1..."  # human-readable
}}
# Codes: genesis_mismatch, bad_magic, protocol_unsupported,
#        clock_skew, self_connect, banned, too_many_peers
```

### Required Diagnostic CLI Commands

The generated code MUST include these:

```
python3 ofc.py diagnose <ip> [--port 18333]
    Step-by-step check: DNS → TCP → Magic → Version → Genesis → Verack → Status
    Pinpoints exactly which step fails with specific fix suggestions

python3 ofc.py netcheck
    Local health: Python version, ed25519, port 18333, firewall, external IP

python3 ofc.py peers list
    Connected peers: height, agent, score, uptime
```

### Quick Troubleshooting Reference

```
"Connection refused"    → Node not running, or bound to 127.0.0.1 not 0.0.0.0
"Connection timed out"  → Firewall / NAT / cloud security group blocking 18333
"Bad magic bytes"       → Not an OFC node, or code version mismatch
"Genesis mismatch"      → Different genesis — #1 cause of multi-node failures
                          Both sides must have genesis hash 0000d5d1...
"No version response"   → Peer crashed, or JSON line format mismatch
"Clock skew"            → sudo timedatectl set-ntp true
"Address already in use" → Old node still running: lsof -i :18333
```

See `{baseDir}/references/p2p_network.md` for full diagnostic code and fixes.

### NAT / WAN

- **Port forwarding**: router forward 18333 → local IP
- **Tailscale**: `tailscale up` on both machines, use 100.x.x.x IPs
- **SSH tunnel**: `ssh -L 18333:localhost:18333 user@remote`

### Checklist

- [ ] 4-byte magic exchange (0x4F464321) as first bytes after TCP connect
- [ ] Version message with genesis_hash field (mandatory)
- [ ] Genesis hash validation — reject + disconnect on mismatch
- [ ] Reject message with error code before every error-disconnect
- [ ] Structured logging for every connection event ([MAGIC], [VERSION], etc.)
- [ ] `diagnose` command — step-by-step remote connectivity check
- [ ] `netcheck` command — local environment verification
- [ ] Flood gossip with dedup (seen_blocks / seen_txs)
- [ ] Headers-first IBD
- [ ] Peer scoring + banning
- [ ] `peers.json` persistence
- [ ] Ping/pong keepalive every 60s

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
python3 ofc.py diagnose <ip> [--port N]    # Step-by-step P2P connectivity test
python3 ofc.py netcheck                    # Local environment health check
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
7. **CLI** — argparse commands, INCLUDING `diagnose` and `netcheck`
8. **Self-Test** — Run the verification below BEFORE presenting to user
9. **Explorer** (optional) — http.server stdlib RPC

---

## ⚠️ Post-Generation Self-Test (MANDATORY)

After generating all code files, run these checks BEFORE telling the user
the project is ready. If any check fails, fix the code and re-test.

### Test 1: Genesis Block Integrity

```bash
cd <project_path>
python3 -c "
from chain.block import genesis_block, double_sha256
header, coinbase_msg, block_hash = genesis_block()

# Verify hash
expected = '0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f'
actual = block_hash[::-1].hex()
assert actual == expected, f'GENESIS FAIL: {actual} != {expected}'

# Verify header is 80 bytes
assert len(header) == 80, f'HEADER SIZE FAIL: {len(header)} != 80'

# Verify coinbase message
assert b'CVE-2010-5139' in coinbase_msg, 'COINBASE MSG FAIL'

print('✓ Genesis block OK')
"
```

### Test 2: Block Reward Halving

```bash
python3 -c "
from consensus.coinbase import block_reward

# Check specific values
assert block_reward(0) == 43_920_819_223_131, 'Block 0 reward wrong'
assert block_reward(209_999) == 43_920_819_223_131, 'Block 209999 reward wrong'
assert block_reward(210_000) == 21_960_409_611_565, 'Block 210000 reward wrong'
assert block_reward(9_660_000) == 0, 'Block 9660000 should be 0'

# Check total supply
total = 0
for i in range(46):
    r = block_reward(i * 210_000)
    total += r * 210_000
assert abs(total - 18_446_744_073_709_140_015) < 210_000, f'Total supply wrong: {total}'

print('✓ Halving schedule OK')
"
```

### Test 3: Wallet Signing Round-Trip

```bash
python3 -c "
from wallet.keys import generate_keypair
from wallet.transaction import serialize_tx, sign_tx, verify_tx
import hashlib

sk, vk = generate_keypair()
tx = {
    'version': 1,
    'sender': hashlib.sha256(vk.to_bytes()).digest()[:20],
    'recipient': b'\\x00' * 20,
    'amount': 100,
    'fee': 1,
    'nonce': 0,
    'timestamp': 1710000000,
    'payload': b'',
}
signed = sign_tx(tx, sk)
assert verify_tx(signed, vk), 'SIGNATURE VERIFICATION FAIL'

print('✓ Wallet sign/verify OK')
"
```

### Test 4: Merkle Tree

```bash
python3 -c "
from chain.merkle import MerkleTree
import hashlib

def sha256(d): return hashlib.sha256(d).digest()

# 4 transactions
hashes = [sha256(f'tx{i}'.encode()) for i in range(4)]
tree = MerkleTree(hashes)

# Proof for tx2
proof = tree.get_proof(2)
assert tree.verify_proof(hashes[2], proof, tree.root), 'MERKLE PROOF FAIL'

# Tampered hash should fail
assert not tree.verify_proof(sha256(b'fake'), proof, tree.root), 'MERKLE TAMPER DETECTION FAIL'

print('✓ Merkle tree OK')
"
```

### Test 5: P2P Magic Bytes

```bash
python3 -c "
from network.protocol import MAGIC

assert MAGIC == b'\\x4f\\x46\\x43\\x21', f'MAGIC WRONG: {MAGIC.hex()}'
assert len(MAGIC) == 4, 'MAGIC LENGTH WRONG'

print('✓ Magic bytes OK: ' + MAGIC.hex())
"
```

### Test 6: P2P Handshake (loopback self-test)

```bash
python3 -c "
import socket, threading, json, time

MAGIC = b'\\x4f\\x46\\x43\\x21'
GENESIS = '0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f'

def make_version():
    return json.dumps({'type':'version','data':{
        'protocol':1,'agent':'test','height':0,
        'genesis_hash':GENESIS,'port':0,'timestamp':int(time.time()),'nonce':1
    }}) + '\n'

result = {'ok': False}

def server_side(srv):
    conn, addr = srv.accept()
    # Receive magic
    data = conn.recv(4)
    assert data == MAGIC, f'Server: bad magic {data.hex()}'
    # Send magic back
    conn.sendall(MAGIC)
    # Receive version
    reader = conn.makefile('r')
    line = reader.readline()
    msg = json.loads(line)
    assert msg['type'] == 'version', 'Server: expected version'
    assert msg['data']['genesis_hash'] == GENESIS, 'Server: genesis mismatch'
    # Send version
    conn.sendall(make_version().encode())
    # Verack exchange
    conn.sendall((json.dumps({'type':'verack','data':{}}) + '\n').encode())
    line = reader.readline()
    msg = json.loads(line)
    assert msg['type'] == 'verack', 'Server: expected verack'
    result['ok'] = True
    conn.close()

# Start server
srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(('127.0.0.1', 0))
port = srv.getsockname()[1]
srv.listen(1)
t = threading.Thread(target=server_side, args=(srv,), daemon=True)
t.start()

# Client side
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(('127.0.0.1', port))
s.sendall(MAGIC)
data = s.recv(4)
assert data == MAGIC, f'Client: bad magic {data.hex()}'
s.sendall(make_version().encode())
reader = s.makefile('r')
line = reader.readline()
msg = json.loads(line)
assert msg['data']['genesis_hash'] == GENESIS, 'Client: genesis mismatch'
s.sendall((json.dumps({'type':'verack','data':{}}) + '\n').encode())
line = reader.readline()
msg = json.loads(line)
assert msg['type'] == 'verack', 'Client: expected verack'

t.join(timeout=5)
assert result['ok'], 'Handshake did not complete'
s.close()
srv.close()

print('✓ P2P handshake loopback OK')
"
```

### Test 7: CLI Commands Exist

```bash
python3 ofc.py --help | grep -q "init" && echo "✓ init command exists" || echo "✗ init MISSING"
python3 ofc.py --help | grep -q "start" && echo "✓ start command exists" || echo "✗ start MISSING"
python3 ofc.py --help | grep -q "diagnose" && echo "✓ diagnose command exists" || echo "✗ diagnose MISSING"
python3 ofc.py --help | grep -q "netcheck" && echo "✓ netcheck command exists" || echo "✗ netcheck MISSING"
python3 ofc.py --help | grep -q "wallet" && echo "✓ wallet command exists" || echo "✗ wallet MISSING"
python3 ofc.py --help | grep -q "send" && echo "✓ send command exists" || echo "✗ send MISSING"
python3 ofc.py --help | grep -q "peers" && echo "✓ peers command exists" || echo "✗ peers MISSING"
```

### Test 8: Sync-Friendliness (node responds to all request types)

This test verifies the node is a good network citizen — it doesn't just broadcast,
it actually responds when other nodes ask for data.

```bash
python3 -c "
from network.protocol import MessageDispatcher

# Check that all required handler methods exist
dispatcher_cls = MessageDispatcher
required_handlers = [
    'on_get_block',     # Serve single block by hash
    'on_get_blocks',    # Serve batch of blocks (IBD phase 2)
    'on_get_headers',   # Serve headers (IBD phase 1)
    'on_get_status',    # Report our chain tip
    'on_get_peers',     # Share peer list
    'on_ping',          # Respond with pong
    'on_new_block',     # Accept and relay blocks
    'on_new_tx',        # Accept and relay transactions
    'on_blocks',        # Handle batch block response
    'on_headers',       # Handle headers response
]

missing = []
for h in required_handlers:
    if not hasattr(dispatcher_cls, h) or not callable(getattr(dispatcher_cls, h)):
        missing.append(h)

if missing:
    print(f'✗ SYNC FAIL: Missing handlers: {missing}')
    print(f'  Without these, other nodes cannot sync from this node!')
    exit(1)

# Verify on_new_block includes block data in relay (not just hash)
import inspect
src = inspect.getsource(dispatcher_cls.on_new_block)
if 'block' not in src or 'broadcast' not in src:
    print('✗ SYNC FAIL: on_new_block does not relay full block data')
    exit(1)

# Verify on_get_blocks returns actual blocks (not empty)
src = inspect.getsource(dispatcher_cls.on_get_blocks)
if 'get_block_by_hash' not in src and 'get_block' not in src:
    print('✗ SYNC FAIL: on_get_blocks does not look up blocks from chain')
    exit(1)

print('✓ Sync-friendliness OK: all 10 handlers present, relay includes full data')
"
```

### What To Do When Tests Fail

```
Test 1 fails → Genesis block code is wrong. Copy genesis_block() from this SKILL.md.
Test 2 fails → INITIAL_REWARD or halving logic wrong. Must be 43920819223131 >> halvings.
Test 3 fails → Ed25519 sign/verify broken. Check ed25519 package is installed.
Test 4 fails → Merkle tree implementation bug. Check odd-leaf duplication and hash order.
Test 5 fails → Magic constant wrong. Must be exactly b'\x4f\x46\x43\x21'.
Test 6 fails → Handshake protocol bug. Most common: missing newline in JSON, wrong message
              order, genesis_hash field missing from version message.
Test 7 fails → CLI subcommand not registered. Check argparse subparsers setup.
Test 8 fails → Message handler missing. The node will broadcast but not respond to
              requests — other nodes cannot sync. Must implement ALL handlers in the
              dispatcher. See p2p_network.md "Message Dispatcher" section for full code.
```

**Do NOT tell the user the project is ready until all 8 tests pass.**
If a test fails, fix the specific file and re-run only that test.

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
