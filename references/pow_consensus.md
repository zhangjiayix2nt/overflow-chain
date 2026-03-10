# Proof of Work Consensus Guide

## Mining Algorithm

### Double-SHA-256

```
block_hash = SHA-256(SHA-256(header_bytes))
```

The header is exactly 80 bytes in canonical little-endian format. The miner
increments the nonce until the resulting hash, interpreted as a 256-bit
big-endian integer, is less than the target.

```rust
fn mine(header: &mut BlockHeader, threads: usize) {
    let target = compact_to_target(header.difficulty_bits);
    let chunk = u32::MAX / threads as u32;

    crossbeam::scope(|s| {
        for t in 0..threads {
            let mut h = header.clone();
            s.spawn(move |_| {
                h.nonce = chunk * t as u32;
                loop {
                    let hash = double_sha256(&h.to_bytes());
                    if hash_le_target(&hash, &target) {
                        // Found! Signal other threads to stop
                        return Some((h.nonce, hash));
                    }
                    h.nonce = h.nonce.wrapping_add(1);
                    if h.nonce == chunk * t as u32 {
                        return None; // Exhausted this thread's range
                    }
                }
            });
        }
    });
}
```

When the 4-byte nonce space (2^32) is exhausted, update the timestamp or
modify the coinbase transaction's extra_nonce field, which changes the
merkle root and opens a fresh 2^32 nonce search space.

---

## Tokenomics — Overflow Coin (OFC)

### The Number: 184,467,440,737.09551616

This total supply comes from CVE-2010-5139. On August 15, 2010, a crafted Bitcoin
transaction in block 74638 exploited an integer overflow to create exactly
184,467,440,737.09551616 BTC — the value of 2^64 satoshis. Satoshi rolled it back,
but Overflow Chain adopts this number as its total supply by design.

Due to integer truncation in halving math, the actual mined total is
184,467,440,737.09140015 OFC (differs from 2^64 by only 0.004 OFC — an
unavoidable artifact of the floor division in reward halving).

### Fixed Parameters

```
Ticker            OFC (Overflow Coin)
Smallest unit     "bit" = 10^-8 OFC
INITIAL_REWARD    = 43,920,819,223,131 bits = 439,208.19223131 OFC
HALVING_INTERVAL  = 210,000 blocks
TOTAL_SUPPLY      ≈ 184,467,440,737.09 OFC (≈ 2^64 bits)
```

### Block Reward Function

```rust
const INITIAL_REWARD: u64 = 43_920_819_223_131;
const HALVING_INTERVAL: u64 = 210_000;

fn block_reward(height: u64) -> u64 {
    let halvings = height / HALVING_INTERVAL;
    if halvings >= 46 { return 0; }
    INITIAL_REWARD >> halvings
}
```

### Complete Halving Schedule

```
Phase  Block Range            Reward/Block (coins)     Phase Total (coins)         Cumulative
─────  ─────────────────────  ───────────────────────  ────────────────────────    ────────────────────────
  0    0 – 209,999            439,208.19223131         92,233,720,368.57510376    92,233,720,368.57510376
  1    210,000 – 419,999      219,604.09611565         46,116,860,184.28650665    138,350,580,552.86160278
  2    420,000 – 629,999      109,802.04805782         23,058,430,092.14219666    161,409,010,645.00378418
  3    630,000 – 839,999       54,901.02402891         11,529,215,046.07109833    172,938,225,691.07492065
  4    840,000 – 1,049,999     27,450.51201445          5,764,607,523.03450012    178,702,833,214.10940552
  5    1,050,000 – 1,259,999   13,725.25600722          2,882,303,761.51620007    181,585,136,975.62561035
  ...  (continues halving)
 45    9,450,000 – 9,659,999        0.00000001                    0.00210000     184,467,440,737.09140015
 46+   9,660,000+                   0.00000000           (fees only)

Total coins ever mined: 184,467,440,737.09140015
Total smallest units:   18,446,744,073,709,140,015  (≈ 2^64)
```

### Comparison with Bitcoin

```
Parameter            Bitcoin                    Overflow Chain (OFC)
───────────          ──────────────────         ──────────────────────
Total supply         21,000,000 BTC             184,467,440,737 OFC
Initial reward       50 BTC                     439,208.19223131 OFC
Smallest unit        1 satoshi (10^-8)          1 bit (10^-8 OFC)
Halving interval     210,000 blocks             210,000 blocks
Target block time    10 minutes                 10 minutes
Difficulty adjust    Every 2,016 blocks         Every 2,016 blocks
Halving curve        Geometric ÷2               Identical geometric ÷2
Max in smallest      2,100,000,000,000,000      ≈ 2^64
Port                 8333                       18333
Namesake             —                          CVE-2010-5139
```

The halving curve is geometrically identical to Bitcoin's. Each phase
produces exactly half the coins of the previous phase.

---

## Difficulty System

### Compact Target Encoding

The 256-bit target is stored in 4 bytes using the same "bits" format as Bitcoin:

```
bits = (exponent << 24) | coefficient

target = coefficient × 2^(8 × (exponent - 3))

Example: bits = 0x1d00ffff
  exponent = 0x1d = 29
  coefficient = 0x00ffff
  target = 0x00000000FFFF0000000000000000000000000000000000000000000000000000
```

### Conversion Functions

```rust
fn compact_to_target(bits: u32) -> [u8; 32] {
    let exp = (bits >> 24) as usize;
    let coeff = bits & 0x007fffff;
    let mut target = [0u8; 32];
    if exp >= 3 && exp <= 32 {
        let pos = 32 - exp;
        target[pos]     = ((coeff >> 16) & 0xff) as u8;
        if pos + 1 < 32 { target[pos + 1] = ((coeff >> 8) & 0xff) as u8; }
        if pos + 2 < 32 { target[pos + 2] = (coeff & 0xff) as u8; }
    }
    target
}

fn target_to_compact(target: &[u8; 32]) -> u32 {
    let mut first = 0;
    while first < 32 && target[first] == 0 { first += 1; }
    if first == 32 { return 0; }
    let exp = (32 - first) as u32;
    let coeff = ((target[first] as u32) << 16)
        | ((*target.get(first + 1).unwrap_or(&0) as u32) << 8)
        | (*target.get(first + 2).unwrap_or(&0) as u32);
    (exp << 24) | coeff
}
```

### Difficulty Adjustment

```rust
const ADJUSTMENT_INTERVAL: u64 = 2_016;
const TARGET_TIMESPAN: u64 = 2_016 * 600; // 2 weeks in seconds

fn retarget(old_bits: u32, actual_timespan: u64) -> u32 {
    let mut timespan = actual_timespan;
    // Clamp to [1/4, 4x] of expected
    if timespan < TARGET_TIMESPAN / 4 { timespan = TARGET_TIMESPAN / 4; }
    if timespan > TARGET_TIMESPAN * 4 { timespan = TARGET_TIMESPAN * 4; }

    let old_target = compact_to_target(old_bits);
    // new_target = old_target × actual_timespan / expected_timespan
    let new_target = bigint_mul_div(&old_target, timespan, TARGET_TIMESPAN);
    // Clamp to max target (easiest difficulty)
    let clamped = min_target(&new_target, &MAX_TARGET);
    target_to_compact(&clamped)
}
```

---

## Coinbase Transaction

```rust
fn create_coinbase(height: u64, miner_addr: &Address, fees: u64) -> Transaction {
    Transaction {
        version: 1,
        sender: [0u8; 20],           // Coinbase: no sender
        recipient: miner_addr.bytes(),
        amount: block_reward(height) + fees,
        fee: 0,
        nonce: 0,
        timestamp: now(),
        payload: height.to_le_bytes().to_vec(), // BIP34: height in coinbase
        signature: [0u8; 64],         // No signature for coinbase
    }
}
```

Coinbase outputs cannot be spent until 100 blocks of maturity (prevents
spending rewards from blocks that might be orphaned).

---

## Block Validation Checklist

When receiving a new block, verify in order (reject at first failure):

1. Header is exactly 80 bytes
2. `double_sha256(header) < target` — PoW is valid
3. `prev_block_hash` references a known block
4. `timestamp` > median of last 11 blocks
5. `timestamp` < current_time + 2 hours
6. `difficulty_bits` matches expected difficulty for this height
7. First transaction is a valid coinbase
8. Coinbase amount ≤ `block_reward(height) + sum(fees)`
9. `merkle_root` matches computed Merkle root of transactions
10. All non-coinbase transactions have valid Ed25519 signatures
11. No double-spends within the block
12. All senders have sufficient balance and correct nonce
13. Block size ≤ MAX_BLOCK_SIZE

---

## Genesis Block

Hardcode a genesis block with:

```rust
fn genesis_block() -> Block {
    let header = BlockHeader {
        version: 1,
        prev_block_hash: [0u8; 32],          // No previous block
        merkle_root: coinbase_merkle_root,
        timestamp: GENESIS_TIMESTAMP,          // Fixed launch time
        difficulty_bits: INITIAL_DIFFICULTY,    // Easy starting difficulty
        nonce: GENESIS_NONCE,                  // Pre-mined
    };
    Block {
        header,
        transactions: vec![genesis_coinbase()],
    }
}
```

All nodes must have the same genesis block. Its hash serves as the chain
identifier — peers with a different genesis hash are on a different network.

---

## Security Properties

### Double-Spend Probability

Attacker hashrate fraction = q, confirmations = z:

```
P(reversal) = (q / (1-q))^z   when q < 0.5

q=0.1:  z=6  → P = 0.0002%
q=0.3:  z=6  → P = 0.63%
q=0.3:  z=12 → P = 0.004%
q=0.45: z=6  → P = 32%     ← still dangerous near 50%
```

### 51% Attack

An attacker with >50% can always build a longer chain privately and
release it to rewrite history. This is fundamental to all PoW systems.

### Selfish Mining

Profitable above ~33% hashrate (Eyal & Sirer, 2014). Mitigated by:
- Short propagation times (GossipSub mesh < 1 second)
- Monitoring for withheld blocks

---

## Implementation by Language

### Rust

```toml
[dependencies]
sha2 = "0.10"
ed25519-dalek = "2"
bs58 = "0.5"
leveldb = "0.8"
libp2p = { version = "0.54", features = ["tcp","noise","yamux","gossipsub","mdns","kad","tokio"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
clap = { version = "4", features = ["derive"] }
```

### Python

```
pip install hashlib ed25519 base58 plyvel libp2p
```

### TypeScript

```
npm install @noble/hashes @noble/ed25519 bs58 level libp2p
    @libp2p/tcp @libp2p/noise @libp2p/yamux @chainsafe/libp2p-gossipsub
```
