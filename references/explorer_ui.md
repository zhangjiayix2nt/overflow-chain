# Block Explorer UI Guide

## Overview

A lightweight web UI that connects to a running node's REST API. Shows real-time
blockchain state: blocks, transactions, addresses, and network peers.

---

## Node REST API Endpoints

The node should expose these HTTP endpoints (default port: 3030):

```
GET  /api/status              → { height, best_hash, peers_count, mining }
GET  /api/blocks              → [Block] (latest 20, paginated)
GET  /api/blocks/:hash        → Block (full block with transactions)
GET  /api/blocks/height/:h    → Block
GET  /api/transactions/:hash  → Transaction (with block inclusion info)
GET  /api/address/:addr       → { balance, nonce, tx_count }
GET  /api/address/:addr/txs   → [Transaction] (paginated)
GET  /api/mempool             → [Transaction] (pending)
GET  /api/peers               → [{ peer_id, addr, score, connected_since }]
GET  /api/consensus           → { difficulty, target_block_time, last_adjustment }

POST /api/transaction         → Submit a signed transaction
POST /api/mine                → Trigger mining (for testing)

WS   /ws/events               → Real-time events stream
```

### WebSocket Events

```json
{ "type": "new_block",       "data": { "height": 42, "hash": "...", "tx_count": 3 } }
{ "type": "new_transaction", "data": { "hash": "...", "from": "...", "to": "...", "amount": 100 } }
{ "type": "peer_connected",  "data": { "peer_id": "...", "addr": "..." } }
{ "type": "peer_disconnected", "data": { "peer_id": "..." } }
{ "type": "mining_started",  "data": { "task_type": "math", "difficulty": 5 } }
{ "type": "mining_complete", "data": { "block_hash": "...", "time_ms": 2340 } }
```

---

## UI Components

Build as a single-page React app (or plain HTML + vanilla JS for simplicity).

### Dashboard Page (/)

```
┌─────────────────────────────────────────────────────────────┐
│  🔗 MyChain Explorer                          [Network: 3 peers] │
├──────────────┬──────────────┬───────────────┬───────────────┤
│  Height      │  Difficulty  │  Mempool      │  Block Time   │
│  142         │  7           │  12 pending   │  ~8.3s avg    │
├──────────────┴──────────────┴───────────────┴───────────────┤
│                                                              │
│  Latest Blocks                                               │
│  ┌─────┬──────────────┬──────┬───────────┬────────────────┐ │
│  │ #   │ Hash         │ Txs  │ Time      │ Miner          │ │
│  ├─────┼──────────────┼──────┼───────────┼────────────────┤ │
│  │ 142 │ 0xa3f2...    │ 5    │ 3s ago    │ 1BvB...        │ │
│  │ 141 │ 0x9c1e...    │ 2    │ 12s ago   │ 1KxP...        │ │
│  │ 140 │ 0x7b4d...    │ 0    │ 20s ago   │ 1BvB...        │ │
│  └─────┴──────────────┴──────┴───────────┴────────────────┘ │
│                                                              │
│  Latest Transactions                                         │
│  ┌──────────────┬───────────┬───────────┬────────┬────────┐ │
│  │ Hash         │ From      │ To        │ Amount │ Status │ │
│  ├──────────────┼───────────┼───────────┼────────┼────────┤ │
│  │ 0xf2a1...    │ 1BvB...   │ 1KxP...   │ 50     │ ✅     │ │
│  │ 0xd3c7...    │ 1KxP...   │ 1MnQ...   │ 10     │ ⏳     │ │
│  └──────────────┴───────────┴───────────┴────────┴────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Block Detail Page (/block/:hash)

Show: block header fields, AI consensus proof details, list of transactions,
merkle tree visualization (optional but impressive).

### Address Page (/address/:addr)

Show: balance, transaction history, nonce.

### Network Page (/network)

Show: connected peers (peer_id, address, score), network topology visualization
(D3.js force-directed graph).

---

## Styling

Keep it minimal and functional — dark theme works well for blockchain explorers.
Use a monospace font for hashes and addresses. Truncate long hashes with ellipsis
and copy-on-click.

```css
:root {
    --bg-primary: #0d1117;
    --bg-secondary: #161b22;
    --text-primary: #e6edf3;
    --text-secondary: #8b949e;
    --accent: #58a6ff;
    --success: #3fb950;
    --warning: #d29922;
    --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
}
```

---

## Implementation Notes

For the prototype, the explorer can be:

1. **A React artifact** (fastest for demo) — Single `.jsx` file that polls the REST API.
   Good for showing to users quickly.

2. **A bundled HTML file** — Served by the node itself on a separate port. No build step
   needed. Uses fetch() + vanilla JS.

3. **A separate app** — If the user wants a production-like setup, create a proper
   Vite + React project.

Option 1 or 2 is recommended for prototyping. Build option 3 only if requested.
