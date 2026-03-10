# ⛓ Overflow Chain (OFC)

**From a bug, a chain is born.**

On August 15, 2010, Bitcoin block 74638 contained a transaction that created **184,467,440,737.09551616 BTC** out of thin air — exploiting an integer overflow in the output sum check ([CVE-2010-5139](https://en.bitcoin.it/wiki/CVE-2010-5139)). Satoshi patched it within five hours and the chain forked back to sanity. That number — exactly **2⁶⁴ satoshis** — was erased from Bitcoin's history.

Overflow Chain brings it back. Our total supply is that exact number: **184,467,440,737.09551616 OFC**. What was a bug becomes a feature.

---

## What is this?

This is an **OpenClaw / Claude skill** that teaches an AI agent to build, run, and deploy a fully functional Overflow Chain node — including:

- ⛏ **SHA-256 double-hash Proof of Work** (same algorithm as Bitcoin)
- 🔑 **Ed25519 wallets** with encrypted storage
- 🌳 **Merkle tree** transaction verification
- 🌐 **libp2p P2P networking** (GossipSub, Kademlia DHT, NAT traversal)
- 📦 **Bitcoin-style storage** (blk*.dat files, LevelDB indexes)

## Chain Parameters

| Parameter | Value |
|---|---|
| **Ticker** | OFC |
| **Total Supply** | 184,467,440,737.09551616 OFC (≈ 2⁶⁴ smallest units) |
| **Smallest Unit** | 10⁻⁸ OFC (called "bit") |
| **Initial Block Reward** | 439,208.19223131 OFC |
| **Halving Interval** | 210,000 blocks |
| **Block Time** | 10 minutes |
| **Difficulty Adjustment** | Every 2,016 blocks |
| **Consensus** | SHA-256² Proof of Work |
| **Signatures** | Ed25519 |
| **P2P** | libp2p (TCP + Noise + Yamux + GossipSub) |
| **Default Port** | 18333 |
| **Namesake** | [CVE-2010-5139](https://nvd.nist.gov/vuln/detail/CVE-2010-5139) |

## Install

### For OpenClaw users

Paste this repo URL directly into your OpenClaw chat:

```
https://github.com/YOUR_USERNAME/overflow-chain
```

OpenClaw will automatically detect the SKILL.md and install it.

Or manually:

```bash
git clone https://github.com/YOUR_USERNAME/overflow-chain.git
cp -r overflow-chain ~/.openclaw/skills/
```

Then start a new OpenClaw session.

### For Claude users

Download the `.skill` file from [Releases](https://github.com/YOUR_USERNAME/overflow-chain/releases) and upload it in Claude's skill settings.

### Via ClawHub

```bash
clawhub install overflow-chain
```

## Usage

After installing, just tell your AI agent:

- *"Build me an Overflow Chain node"*
- *"Mine some OFC"*
- *"Create an OFC wallet and send 100 OFC to ofc1..."*
- *"Set up a 3-node OFC testnet"*
- *"Connect my OFC node to my friend's node across the internet"*

The skill handles language selection (Rust/Python/TypeScript), project scaffolding, and all implementation details.

## File Structure

```
overflow-chain/
├── SKILL.md                    ← Main skill file (AI reads this)
├── references/
│   ├── pow_consensus.md        ← Mining algorithm, difficulty, halving schedule
│   ├── p2p_network.md          ← libp2p setup, NAT traversal, chain sync
│   └── explorer_ui.md          ← Block explorer REST API & UI templates
├── .github/
│   └── workflows/
│       └── publish.yml         ← Auto-publish to ClawHub on release
├── LICENSE
└── README.md
```

## Halving Schedule

The reward curve is geometrically identical to Bitcoin's:

| Phase | Blocks | Reward / Block | Phase Total |
|-------|--------|----------------|-------------|
| 0 | 0 – 209,999 | 439,208.19 OFC | 92.2B OFC |
| 1 | 210,000 – 419,999 | 219,604.10 OFC | 46.1B OFC |
| 2 | 420,000 – 629,999 | 109,802.05 OFC | 23.1B OFC |
| 3 | 630,000 – 839,999 | 54,901.02 OFC | 11.5B OFC |
| ... | ... | halves each phase | ... |
| 45 | 9,450,000 – 9,659,999 | 0.00000001 OFC | 0.002 OFC |
| 46+ | 9,660,000+ | 0 (fees only) | — |

**Total mined: ≈ 184,467,440,737.09 OFC**

## Contributing

Issues and PRs welcome. If you're extending the skill (new consensus module, UTXO model, smart contracts), please keep the SKILL.md under 500 lines and put details in `references/`.

## License

MIT
