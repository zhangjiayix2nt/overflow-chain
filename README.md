# ⛓ Overflow Chain (OFC) — Lightweight Edition

**From a bug, a chain is born.**

On August 15, 2010, Bitcoin block 74638 created **184,467,440,737.09551616 BTC** out of thin air via an integer overflow ([CVE-2010-5139](https://en.bitcoin.it/wiki/CVE-2010-5139)). Overflow Chain immortalizes that number as its total supply.

## Why Lightweight?

The original v1 required Rust compilation (2GB+ RAM), libp2p (~200 crates), and LevelDB (C++ native). Many users on low-end VPS or Raspberry Pi couldn't run it.

This v2 Lite edition delivers **identical blockchain functionality** with radically simpler dependencies:

| | v1 (Rust + libp2p) | **v2 Lite (Python)** |
|---|---|---|
| Compile RAM | 2+ GB | **0 (interpreted)** |
| Runtime RAM | ~200 MB | **~30-50 MB** |
| External deps | ~200 crates | **1 pip package** |
| Install time | 10+ minutes | **5 seconds** |
| Min hardware | 2 CPU, 2GB RAM | **1 CPU, 512MB RAM** |

## Requirements

```bash
python3 --version   # 3.8 or higher
pip install ed25519  # only external dependency (~20KB, pure Python)
```

That's it. No compiler, no C libraries, no Rust, no Node.js.

## Chain Parameters

| Parameter | Value |
|---|---|
| **Ticker** | OFC |
| **Total Supply** | 184,467,440,737.09551616 OFC (≈ 2⁶⁴ smallest units) |
| **Initial Block Reward** | 439,208.19223131 OFC |
| **Halving Interval** | 210,000 blocks |
| **Block Time** | 10 minutes |
| **Consensus** | SHA-256² Proof of Work |
| **Signatures** | Ed25519 |
| **P2P** | TCP + JSON-line (TLS optional) |
| **Storage** | SQLite (single file, stdlib) |
| **Default Port** | 18333 |

## Quick Start

```bash
git clone https://github.com/zhangjiayix2nt/overflow-chain.git
cd overflow-chain
pip install ed25519

python3 ofc.py init                    # Create ~/.overflowchain/ + genesis block
python3 ofc.py start                   # Start node, connect to network
python3 ofc.py start --mine            # Start mining
python3 ofc.py wallet create           # Create a wallet
python3 ofc.py send ofc1... 100        # Send 100 OFC
```

## Install as OpenClaw Skill

Paste this URL into your OpenClaw chat:

```
https://github.com/zhangjiayix2nt/overflow-chain
```

Or manually:

```bash
git clone https://github.com/zhangjiayix2nt/overflow-chain.git
cp -r overflow-chain ~/.openclaw/skills/
```

## What Changed from v1

| Component | v1 | v2 Lite |
|---|---|---|
| Language | Rust | Python 3.8+ |
| P2P | libp2p (GossipSub, Kademlia, Noise, Yamux, Relay, DCUtR) | Raw TCP + JSON lines + flood gossip (+ optional TLS) |
| Storage | blk*.dat + LevelDB block index + LevelDB chainstate | Single SQLite file (`chain.db`) |
| Wallet crypto | scrypt (16MB RAM per call) | PBKDF2-SHA256 (stdlib, <1MB) |
| Peer discovery | mDNS + Kademlia DHT + DNS seeds | Seed nodes + peer exchange |
| NAT traversal | Circuit Relay + DCUtR hole punching | Port forward / Tailscale / SSH tunnel |

**All chain rules are identical**: same PoW algorithm, same halving curve, same total supply, same block format. A v1 and v2 node speaking the same wire protocol can interoperate.

## File Structure

```
overflow-chain/
├── SKILL.md                 ← AI skill definition
├── references/
│   ├── pow_consensus.md     ← Mining, difficulty, halving
│   ├── p2p_network.md       ← TCP protocol, sync, gossip
│   └── explorer_ui.md       ← Block explorer API & UI
├── LICENSE
└── README.md
```

## Contributing

PRs welcome. Keep external dependencies to zero if possible (stdlib only). The ed25519 package is the one exception.

## License

MIT
