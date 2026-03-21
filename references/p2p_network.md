# P2P Network Guide — Lightweight Edition

## Architecture

```
┌──────────────────────────────────────────┐
│              Application                  │
│        (Block sync, Tx broadcast)         │
├──────────────────────────────────────────┤
│          Flood Gossip (broadcast)         │
├──────────────────────────────────────────┤
│           Peer Manager                    │
├──────────────────────────────────────────┤
│         JSON-Line Protocol                │
│    (magic prefix + JSON + newline)        │
├──────────────────────────────────────────┤
│      TLS (optional, ssl stdlib)           │
├──────────────────────────────────────────┤
│             Raw TCP                       │
└──────────────────────────────────────────┘
```

Pure Python stdlib: `socket`, `ssl`, `threading`, `json`, `struct`.
Zero compiled dependencies.

---

## Protocol Magic Bytes

Every OFC TCP connection starts with a 4-byte magic handshake to immediately
reject non-OFC connections (port scanners, other protocols, etc.):

```
MAGIC = b'\x4f\x46\x43\x21'   # ASCII "OFC!" = 0x4F464321
```

### Why Magic Bytes Matter

Without magic bytes, common failure modes include:
- HTTP bots connecting to port 18333 and sending `GET /`
- SSH scanners sending binary garbage
- Other blockchain nodes (BTC, ETH) trying to handshake
- Accidental connections from other services on the same port

The first thing both sides do after TCP connect is send + verify magic bytes.
If the first 4 bytes don't match, close immediately.

### Implementation

```python
MAGIC = b'\x4f\x46\x43\x21'  # "OFC!"

def send_magic(sock):
    sock.sendall(MAGIC)

def recv_magic(sock) -> bool:
    """Returns True if peer sent correct magic. Timeout: 5 seconds."""
    sock.settimeout(5.0)
    try:
        data = b''
        while len(data) < 4:
            chunk = sock.recv(4 - len(data))
            if not chunk:
                return False
            data += chunk
        return data == MAGIC
    except socket.timeout:
        return False
    finally:
        sock.settimeout(None)
```

---

## Connection State Machine

Every peer connection follows this exact sequence. Failing at any step means
disconnect.

```
      TCP Connect
          │
          ▼
    ┌────────────┐
    │ SEND MAGIC │──── send b'\x4f\x46\x43\x21'
    └─────┬──────┘
          │
          ▼
    ┌────────────┐     timeout 5s
    │ RECV MAGIC │──── receive 4 bytes, verify == MAGIC
    └─────┬──────┘     FAIL → close("bad magic")
          │ OK
          ▼
    ┌────────────┐
    │SEND VERSION│──── {"type":"version","data":{...}}
    └─────┬──────┘
          │
          ▼
    ┌────────────┐     timeout 10s
    │RECV VERSION│──── parse JSON, check fields
    └─────┬──────┘     FAIL → close("bad version")
          │ OK
          ▼
    ┌────────────────┐
    │ VERIFY GENESIS │──── data.genesis_hash == our genesis?
    └─────┬──────────┘     FAIL → close("wrong network")
          │ OK
          ▼
    ┌────────────┐
    │SEND VERACK │──── {"type":"verack","data":{}}
    └─────┬──────┘
          │
          ▼
    ┌────────────┐     timeout 10s
    │RECV VERACK │──── verify type == "verack"
    └─────┬──────┘     FAIL → close("no verack")
          │ OK
          ▼
    ┌────────────┐
    │ CONNECTED  │──── now accept all message types
    └────────────┘
```

### Version Message — Complete Specification

```python
VERSION_MSG = {
    "type": "version",
    "data": {
        "protocol": 1,                    # Protocol version (integer)
        "agent": "ofc-py/2.0.0",          # Software identifier
        "height": 142,                     # Our best block height
        "genesis_hash": "0000d5d1702ee...", # FULL 64-char hex of genesis hash
        "port": 18333,                     # Our listening port
        "timestamp": 1710000000,           # Unix timestamp (our clock)
        "services": 1,                     # Bitfield: 1=full_node, 2=miner
        "nonce": 8471625390,               # Random u64 (detect self-connect)
    }
}
```

### Version Field Validation

```python
def validate_version(msg, our_genesis_hash):
    """Returns (ok: bool, reason: str)"""
    d = msg.get('data', {})

    # Required fields
    for field in ['protocol', 'genesis_hash', 'height', 'agent']:
        if field not in d:
            return False, f"missing field: {field}"

    # Protocol version
    if d['protocol'] != 1:
        return False, f"unsupported protocol: {d['protocol']}"

    # Genesis hash — THIS IS THE MOST IMPORTANT CHECK
    if d['genesis_hash'] != our_genesis_hash:
        return False, (
            f"genesis mismatch: peer={d['genesis_hash'][:16]}... "
            f"ours={our_genesis_hash[:16]}..."
        )

    # Self-connection detection
    if d.get('nonce') == our_nonce:
        return False, "self-connection detected"

    # Timestamp sanity (reject if peer's clock is >2 hours off)
    import time
    clock_diff = abs(time.time() - d.get('timestamp', 0))
    if clock_diff > 7200:
        return False, f"clock skew: {clock_diff:.0f}s"

    return True, "ok"
```

---

## Complete Handshake Implementation

```python
MAGIC = b'\x4f\x46\x43\x21'
GENESIS_HASH = "0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f"

import socket, json, time, os, struct

def handshake_outbound(sock, our_height, our_port):
    """Outbound connection: we initiate."""
    our_nonce = struct.unpack('<Q', os.urandom(8))[0]

    # Step 1: Magic exchange
    sock.sendall(MAGIC)
    if not recv_magic(sock):
        raise ConnectionError("Peer did not send OFC magic bytes. "
                              "Is this an OFC node? Check port number.")

    # Step 2: Send our version
    version_data = {
        "protocol": 1,
        "agent": "ofc-py/2.0.0",
        "height": our_height,
        "genesis_hash": GENESIS_HASH,
        "port": our_port,
        "timestamp": int(time.time()),
        "services": 1,
        "nonce": our_nonce,
    }
    send_json(sock, {"type": "version", "data": version_data})

    # Step 3: Receive peer's version
    peer_version = recv_json(sock, timeout=10)
    if peer_version is None:
        raise ConnectionError("Peer did not send version message (timeout 10s). "
                              "Possible: peer crashed, firewall dropped response, "
                              "or port is open but no OFC node is running.")
    if peer_version.get('type') != 'version':
        raise ConnectionError(f"Expected 'version', got '{peer_version.get('type')}'. "
                              "Peer may be running incompatible software.")

    ok, reason = validate_version(peer_version, GENESIS_HASH)
    if not ok:
        raise ConnectionError(f"Version rejected: {reason}")

    # Step 4: Verack exchange
    send_json(sock, {"type": "verack", "data": {}})

    peer_verack = recv_json(sock, timeout=10)
    if peer_verack is None or peer_verack.get('type') != 'verack':
        raise ConnectionError("Peer did not send verack. Handshake incomplete.")

    return peer_version['data']  # Return peer info


def handshake_inbound(sock, our_height, our_port):
    """Inbound connection: peer initiates."""
    our_nonce = struct.unpack('<Q', os.urandom(8))[0]

    # Step 1: Receive magic from peer (they connect to us)
    if not recv_magic(sock):
        raise ConnectionError("Inbound: bad magic bytes. Not an OFC connection.")

    # Send our magic back
    sock.sendall(MAGIC)

    # Step 2: Receive peer's version
    peer_version = recv_json(sock, timeout=10)
    if peer_version is None or peer_version.get('type') != 'version':
        raise ConnectionError("Inbound: no version message received.")

    ok, reason = validate_version(peer_version, GENESIS_HASH)
    if not ok:
        # Send reject before closing so peer gets a useful error
        send_json(sock, {"type": "reject", "data": {"reason": reason}})
        raise ConnectionError(f"Inbound version rejected: {reason}")

    # Step 3: Send our version
    version_data = {
        "protocol": 1,
        "agent": "ofc-py/2.0.0",
        "height": our_height,
        "genesis_hash": GENESIS_HASH,
        "port": our_port,
        "timestamp": int(time.time()),
        "services": 1,
        "nonce": our_nonce,
    }
    send_json(sock, {"type": "version", "data": version_data})

    # Step 4: Verack exchange
    send_json(sock, {"type": "verack", "data": {}})

    peer_verack = recv_json(sock, timeout=10)
    if peer_verack is None or peer_verack.get('type') != 'verack':
        raise ConnectionError("Inbound: no verack received.")

    return peer_version['data']


def send_json(sock, obj):
    line = json.dumps(obj) + "\n"
    sock.sendall(line.encode('utf-8'))

def recv_json(sock, timeout=30):
    sock.settimeout(timeout)
    try:
        reader = sock.makefile('r', buffering=1)
        line = reader.readline()
        if not line:
            return None
        return json.loads(line.strip())
    except (socket.timeout, json.JSONDecodeError):
        return None
    finally:
        sock.settimeout(None)

def recv_magic(sock) -> bool:
    sock.settimeout(5.0)
    try:
        data = b''
        while len(data) < 4:
            chunk = sock.recv(4 - len(data))
            if not chunk:
                return False
            data += chunk
        return data == MAGIC
    except socket.timeout:
        return False
    finally:
        sock.settimeout(None)
```

---

## Reject Message

When a peer sends something invalid, respond with a reject before disconnecting.
This helps the remote side diagnose the issue.

```python
{"type": "reject", "data": {
    "original_type": "version",            # What message was rejected
    "code": "genesis_mismatch",            # Machine-readable error code
    "reason": "genesis mismatch: peer=abcd... ours=0000d5d1..."  # Human-readable
}}
```

### Error Codes

```
genesis_mismatch      Genesis hash differs — not the same network
protocol_unsupported  Protocol version not supported
bad_magic             Magic bytes did not match OFC!
clock_skew            Peer's clock is >2 hours off
self_connect          Detected connection to self
invalid_block         Block failed validation
invalid_tx            Transaction failed validation
too_many_peers        Connection limit reached
banned                Peer is banned
```

---

## All Message Types — Complete Reference

```
CONNECTION SETUP:
  magic             4 raw bytes: 0x4F464321 ("OFC!")
  version           Handshake with chain identity + node info
  verack            Handshake acknowledgement
  reject            Error with code + reason

BLOCK PROPAGATION:
  new_block         Announce a newly mined/received block
  get_block         Request a specific block by hash
  block             Response with full block data

TRANSACTIONS:
  new_tx            Broadcast a transaction

CHAIN SYNC:
  get_headers       Request headers from a height
  headers           Response with serialized headers
  get_blocks        Request blocks by hash list
  blocks            Response with full blocks

PEER DISCOVERY:
  get_peers         Request known peer addresses
  peers             Response with peer list
  get_status        Request chain tip info
  status            Response with height + hashes

KEEPALIVE:
  ping              Liveness check (with nonce)
  pong              Reply to ping (echo nonce)
```

---

## Message Dispatcher — Handling ALL Incoming Requests

This is the **most critical section** for multi-node interoperability. Every node
MUST respond to every request type. A node that only broadcasts but doesn't respond
to requests is a bad citizen — other nodes cannot sync from it.

### The Core Problem This Solves

Without a proper dispatcher, nodes behave like this:
- Node A mines a block → broadcasts `new_block` → OK
- Node B joins later, asks Node A for historical blocks → **silence**
- Node B can never sync because Node A ignores `get_blocks`

Every node MUST be both a **client** (requesting data) AND a **server** (responding
to requests). This is peer-to-peer, not client-server.

### Complete Dispatcher Implementation

```python
class MessageDispatcher:
    """Routes incoming messages to handler functions.
    Every message type MUST have a handler. If a type is received
    without a handler, log a warning (don't crash)."""

    def __init__(self, node):
        self.node = node
        self.handlers = {
            # Connection (handled separately in handshake)
            'version':     None,  # Handled in handshake, not here
            'verack':      None,  # Handled in handshake, not here
            'reject':      self.on_reject,

            # Block propagation
            'new_block':   self.on_new_block,
            'get_block':   self.on_get_block,     # ← MUST respond
            'block':       self.on_block,

            # Transactions
            'new_tx':      self.on_new_tx,

            # Chain sync — ALL of these MUST respond
            'get_headers': self.on_get_headers,   # ← MUST respond
            'headers':     self.on_headers,
            'get_blocks':  self.on_get_blocks,    # ← MUST respond
            'blocks':      self.on_blocks,

            # Peer discovery
            'get_peers':   self.on_get_peers,     # ← MUST respond
            'peers':       self.on_peers,
            'get_status':  self.on_get_status,    # ← MUST respond
            'status':      self.on_status,

            # Keepalive
            'ping':        self.on_ping,          # ← MUST respond with pong
            'pong':        self.on_pong,
        }

    def dispatch(self, peer, msg):
        msg_type = msg.get('type', '')
        handler = self.handlers.get(msg_type)
        if handler is None:
            log.warning(f"[MSG] Unknown message type: {msg_type} from {peer.addr}")
            return
        try:
            handler(peer, msg.get('data', {}))
        except Exception as e:
            log.error(f"[MSG] Handler error for {msg_type}: {e}")
            peer.score -= 5
```

### Block Serving Handlers (nodes MUST implement these)

```python
    # ─── RESPOND TO get_block: Return a single block by hash ───

    def on_get_block(self, peer, data):
        """Peer requests a specific block. We MUST respond."""
        block_hash = bytes.fromhex(data.get('hash', ''))
        block = self.node.chain.get_block_by_hash(block_hash)

        if block:
            peer.send('block', {'block': block.to_hex()})
            log.debug(f"[SERVE] Sent block {data['hash'][:16]}... to {peer.addr}")
        else:
            # We don't have it — respond with empty so peer doesn't hang
            peer.send('block', {'block': None, 'not_found': True})
            log.debug(f"[SERVE] Block {data['hash'][:16]}... not found, sent not_found")

    # ─── RESPOND TO get_blocks: Return multiple blocks by hash list ───

    def on_get_blocks(self, peer, data):
        """Peer requests a batch of blocks. We MUST respond with as many as we have.
        This is the primary mechanism for chain sync (IBD phase 2)."""
        hashes = data.get('hashes', [])

        # Limit batch size to prevent memory abuse
        MAX_BATCH = 50
        if len(hashes) > MAX_BATCH:
            hashes = hashes[:MAX_BATCH]
            log.warning(f"[SERVE] Capped get_blocks to {MAX_BATCH} from {peer.addr}")

        blocks = []
        for h in hashes:
            block = self.node.chain.get_block_by_hash(bytes.fromhex(h))
            if block:
                blocks.append(block.to_hex())
            # If we don't have a block, skip it (don't include None)

        peer.send('blocks', {
            'blocks': blocks,
            'requested': len(hashes),
            'returned': len(blocks),
        })
        log.info(f"[SERVE] Sent {len(blocks)}/{len(hashes)} blocks to {peer.addr}")

    # ─── RESPOND TO get_headers: Return headers from a height ───

    def on_get_headers(self, peer, data):
        """Peer requests headers for chain sync (IBD phase 1).
        We MUST respond — this is how new nodes discover the chain."""
        from_height = data.get('from_height', 0)
        count = min(data.get('count', 500), 500)  # Cap at 500

        headers = []
        for h in range(from_height, from_height + count):
            block = self.node.chain.get_block_by_height(h)
            if block is None:
                break  # We don't have blocks past this height
            headers.append(block.header_hex())

        peer.send('headers', {
            'headers': headers,
            'from_height': from_height,
            'count': len(headers),
        })
        log.info(f"[SERVE] Sent {len(headers)} headers "
                 f"(height {from_height}..{from_height + len(headers) - 1}) to {peer.addr}")

    # ─── RESPOND TO get_status: Return our chain tip ───

    def on_get_status(self, peer, data):
        """Peer wants to know our chain state. Always respond."""
        peer.send('status', {
            'height': self.node.chain.get_height(),
            'best_hash': self.node.chain.get_best_hash().hex(),
            'genesis_hash': GENESIS_HASH,
        })

    # ─── RESPOND TO get_peers: Share our known peers ───

    def on_get_peers(self, peer, data):
        """Share our peer list (excluding the asking peer)."""
        peer_list = []
        for addr, p in self.node.peers.items():
            if p != peer and p.port:
                host, port_str = addr.rsplit(':', 1)
                peer_list.append({'host': host, 'port': int(port_str)})
        peer.send('peers', {'peers': peer_list[:50]})  # Cap at 50

    # ─── RESPOND TO ping ───

    def on_ping(self, peer, data):
        """Always respond to ping with pong echoing the nonce."""
        peer.send('pong', {'nonce': data.get('nonce', 0)})
```

### Block Propagation Handlers

```python
    # ─── RECEIVE new_block: Validate and relay ───

    def on_new_block(self, peer, data):
        """A peer announces a new block. Validate, store, and relay."""
        block_hash = data.get('hash', '')

        # Dedup: skip if we already have this block
        if block_hash in self.node.seen_blocks:
            return
        self.node.seen_blocks.add(block_hash)

        block_hex = data.get('block')
        if not block_hex:
            # Peer sent announcement without block data — request it
            peer.send('get_block', {'hash': block_hash})
            return

        block = Block.from_hex(block_hex)
        if self.node.chain.validate_and_add(block):
            peer.score += 10
            log.info(f"[BLOCK] Accepted block height={block.height} "
                     f"hash={block_hash[:16]}... from {peer.addr}")

            # *** RELAY TO ALL OTHER PEERS ***
            # Include full block data so receivers don't need to request it
            self.node.broadcast('new_block', {
                'hash': block_hash,
                'height': block.height,
                'block': block_hex,    # ← Include full block, not just hash
            }, exclude_peer=peer)
        else:
            peer.score -= 50
            log.warning(f"[BLOCK] REJECTED block {block_hash[:16]}... from {peer.addr}")

    # ─── RECEIVE block: Response to our get_block request ───

    def on_block(self, peer, data):
        """Response to a get_block we sent earlier."""
        if data.get('not_found'):
            return  # Peer doesn't have it, try another peer

        block_hex = data.get('block')
        if block_hex:
            block = Block.from_hex(block_hex)
            if self.node.chain.validate_and_add(block):
                peer.score += 5

    # ─── RECEIVE blocks: Batch response for IBD ───

    def on_blocks(self, peer, data):
        """Batch block response during IBD sync."""
        blocks_hex = data.get('blocks', [])
        requested = data.get('requested', len(blocks_hex))
        returned = data.get('returned', len(blocks_hex))

        log.info(f"[SYNC] Received {returned}/{requested} blocks from {peer.addr}")

        accepted = 0
        for bh in blocks_hex:
            block = Block.from_hex(bh)
            if self.node.chain.validate_and_add(block):
                accepted += 1
            else:
                log.warning(f"[SYNC] Block validation failed during IBD, stopping batch")
                peer.score -= 20
                break

        peer.score += accepted
        log.info(f"[SYNC] Accepted {accepted}/{returned} blocks")

    # ─── RECEIVE headers: Response for IBD phase 1 ───

    def on_headers(self, peer, data):
        """Headers batch during IBD. Validate and store for later block download."""
        headers_hex = data.get('headers', [])
        from_height = data.get('from_height', 0)

        validated = 0
        for i, h_hex in enumerate(headers_hex):
            header = BlockHeader.from_hex(h_hex)
            if self.node.chain.validate_header(header):
                self.node.chain.store_header(header)
                validated += 1
            else:
                log.warning(f"[SYNC] Invalid header at height {from_height + i}")
                peer.score -= 20
                break

        log.info(f"[SYNC] Validated {validated}/{len(headers_hex)} headers "
                 f"from height {from_height}")
```

### Transaction Handler

```python
    def on_new_tx(self, peer, data):
        """Receive and relay a new transaction."""
        tx_hex = data.get('tx', '')
        tx_hash = hashlib.sha256(bytes.fromhex(tx_hex)).hexdigest()

        if tx_hash in self.node.seen_txs:
            return
        self.node.seen_txs.add(tx_hash)

        tx = Transaction.from_hex(tx_hex)
        if self.node.mempool.add(tx):
            peer.score += 1
            self.node.broadcast('new_tx', data, exclude_peer=peer)
        else:
            peer.score -= 10
```

### Remaining Handlers

```python
    def on_reject(self, peer, data):
        """Peer rejected something we sent."""
        log.warning(f"[REJECT] From {peer.addr}: code={data.get('code')} "
                    f"reason={data.get('reason')}")

    def on_peers(self, peer, data):
        """Received peer list — add unknown peers for future connection."""
        for p in data.get('peers', []):
            addr = f"{p['host']}:{p['port']}"
            if addr not in self.node.peers and addr not in self.node.known_addrs:
                self.node.known_addrs.add(addr)

    def on_status(self, peer, data):
        """Received peer's chain status — update our knowledge of their height."""
        peer.peer_height = data.get('height', 0)
        peer.best_hash = data.get('best_hash', '')

    def on_pong(self, peer, data):
        """Pong received — peer is alive. Record latency."""
        peer.last_pong = time.time()
```

---

## Improved Chain Sync (IBD) — Full Implementation

The IBD (Initial Block Download) process MUST work as a proper client that
sends requests AND the peer MUST respond using the handlers above.

```python
class ChainSync:
    """Handles initial block download from peers."""

    def __init__(self, node):
        self.node = node
        self.syncing = False

    def start_sync(self):
        """Called after handshake if peer has higher chain."""
        if self.syncing:
            return
        self.syncing = True

        try:
            best_peer = self._find_best_peer()
            if not best_peer:
                log.info("[SYNC] No peers with higher chain, we're up to date")
                return

            local_height = self.node.chain.get_height()
            peer_height = best_peer.peer_height

            log.info(f"[SYNC] Starting IBD: local={local_height} "
                     f"peer={peer_height} gap={peer_height - local_height} blocks")

            # Phase 1: Download and validate headers
            log.info("[SYNC] Phase 1: Downloading headers...")
            height = local_height + 1
            total_headers = 0
            while height <= peer_height:
                best_peer.send('get_headers', {
                    'from_height': height,
                    'count': 500,
                })
                resp = self._wait_for(best_peer, 'headers', timeout=30)
                if resp is None:
                    log.warning("[SYNC] Header download timed out, retrying with another peer")
                    best_peer = self._find_next_peer(best_peer)
                    if not best_peer:
                        break
                    continue

                headers = resp.get('headers', [])
                if not headers:
                    break

                total_headers += len(headers)
                height += len(headers)
                log.info(f"[SYNC] Headers: {total_headers} downloaded, up to height {height - 1}")

            # Phase 2: Download full blocks in batches
            log.info("[SYNC] Phase 2: Downloading full blocks...")
            missing_hashes = self.node.chain.get_headers_without_blocks()
            total_blocks = len(missing_hashes)
            downloaded = 0

            for i in range(0, len(missing_hashes), 50):  # Batches of 50
                batch = missing_hashes[i:i+50]

                # Try multiple peers for resilience
                peer = self._find_peer_with_blocks(batch)
                if not peer:
                    peer = best_peer

                peer.send('get_blocks', {'hashes': [h.hex() for h in batch]})
                resp = self._wait_for(peer, 'blocks', timeout=60)

                if resp is None:
                    log.warning(f"[SYNC] Block batch timed out at offset {i}")
                    continue

                blocks_hex = resp.get('blocks', [])
                returned = resp.get('returned', len(blocks_hex))
                downloaded += returned
                log.info(f"[SYNC] Blocks: {downloaded}/{total_blocks} "
                         f"({100*downloaded//max(total_blocks,1)}%)")

            log.info(f"[SYNC] IBD complete: synced to height {self.node.chain.get_height()}")

        finally:
            self.syncing = False

    def _find_best_peer(self):
        local_h = self.node.chain.get_height()
        peers = [(p, p.peer_height) for p in self.node.peers.values()
                 if p.peer_height > local_h]
        if not peers:
            return None
        return max(peers, key=lambda x: x[1])[0]

    def _find_next_peer(self, exclude):
        """Find another peer for failover."""
        local_h = self.node.chain.get_height()
        for p in self.node.peers.values():
            if p != exclude and p.peer_height > local_h:
                return p
        return None

    def _find_peer_with_blocks(self, hashes):
        """Find any peer that likely has these blocks (by height)."""
        # Any peer with height >= our request height should have them
        return self._find_best_peer()

    def _wait_for(self, peer, expected_type, timeout=30):
        """Wait for a specific message type from a peer.
        Other message types received during the wait are dispatched normally."""
        import time
        deadline = time.time() + timeout
        while time.time() < deadline:
            msg = peer.recv_with_timeout(timeout=max(1, deadline - time.time()))
            if msg is None:
                return None
            if msg.get('type') == expected_type:
                return msg.get('data', {})
            # Not what we're waiting for — dispatch it normally
            self.node.dispatcher.dispatch(peer, msg)
        return None
```

---

## New Block Relay — Include Full Data

When relaying a new block, ALWAYS include the full block data, not just the
hash. This prevents a cascade of `get_block` requests:

```
BAD (announcement only — forces N extra round trips):
  Node A mines block
  A → B: {"type":"new_block","data":{"hash":"abc..."}}
  B → A: {"type":"get_block","data":{"hash":"abc..."}}     ← extra request
  A → B: {"type":"block","data":{"block":"<hex>"}}          ← extra response
  B → C: {"type":"new_block","data":{"hash":"abc..."}}
  C → B: {"type":"get_block","data":{"hash":"abc..."}}     ← extra request
  ... multiplied by every node in the network

GOOD (full relay — zero extra round trips):
  Node A mines block
  A → B: {"type":"new_block","data":{"hash":"abc...","height":42,"block":"<hex>"}}
  B validates, then:
  B → C: {"type":"new_block","data":{"hash":"abc...","height":42,"block":"<hex>"}}
  C validates, then:
  C → D: ... (propagation complete in one hop per node)
```

Implementation:

```python
def broadcast_new_block(self, block):
    """Broadcast a newly mined or received block WITH full data."""
    block_hex = block.to_hex()
    msg_data = {
        'hash': block.hash_hex(),
        'height': block.height,
        'block': block_hex,      # ← ALWAYS include full block
    }
    self.broadcast('new_block', msg_data)
    log.info(f"[RELAY] Broadcasting block height={block.height} "
             f"hash={block.hash_hex()[:16]}... size={len(block_hex)//2}B "
             f"to {len(self.peers)} peers")
```

---

## Sync-Friendliness Checklist

Every generated node MUST satisfy ALL of these. Missing any one breaks
multi-node operation:

```
SERVING (responding to peer requests):
  [x] on_get_block    → Return single block by hash (or not_found)
  [x] on_get_blocks   → Return batch of blocks by hash list (up to 50)
  [x] on_get_headers  → Return headers from a height (up to 500)
  [x] on_get_status   → Return our height + best hash + genesis hash
  [x] on_get_peers    → Return our known peer list
  [x] on_ping         → Always reply with pong (echo nonce)

RELAYING (forwarding to other peers):
  [x] new_block       → Include FULL block data, not just hash
  [x] new_tx          → Include full transaction data

SYNCING (requesting from peers):
  [x] IBD Phase 1     → Request headers in batches of 500
  [x] IBD Phase 2     → Request blocks in batches of 50
  [x] Failover        → If one peer times out, try another
  [x] Progress log    → Show sync progress: "Blocks: 150/1000 (15%)"
```

**If any [x] above is missing from the generated code, it is a bug. Fix it.**

---

## Connection Logging

Every connection event should be logged with enough detail to diagnose problems.
The generated code MUST include structured logging:

```python
import logging, time

log = logging.getLogger('ofc.network')

def log_connect_attempt(addr, direction):
    log.info(f"[CONN] {direction} → {addr[0]}:{addr[1]}")

def log_magic_result(addr, ok):
    if ok:
        log.info(f"[MAGIC] {addr[0]}:{addr[1]} ✓ OFC magic verified")
    else:
        log.warning(f"[MAGIC] {addr[0]}:{addr[1]} ✗ Bad magic bytes — not an OFC node")

def log_version_result(addr, ok, reason, peer_data=None):
    if ok:
        log.info(f"[VERSION] {addr[0]}:{addr[1]} ✓ "
                 f"agent={peer_data.get('agent')} "
                 f"height={peer_data.get('height')} "
                 f"genesis={peer_data.get('genesis_hash','')[:16]}...")
    else:
        log.warning(f"[VERSION] {addr[0]}:{addr[1]} ✗ {reason}")

def log_handshake_complete(addr, peer_data):
    log.info(f"[PEER] Connected to {addr[0]}:{addr[1]} "
             f"(height={peer_data.get('height')}, agent={peer_data.get('agent')})")

def log_disconnect(addr, reason):
    log.info(f"[DISC] {addr[0]}:{addr[1]} — {reason}")

def log_message(addr, direction, msg_type, size):
    log.debug(f"[MSG] {direction} {addr[0]}:{addr[1]} type={msg_type} size={size}B")
```

---

## Troubleshooting P2P Connections

### Built-in Diagnostic Command

The generated CLI MUST include a `diagnose` subcommand:

```bash
python3 ofc.py diagnose <target_ip> [--port 18333]
```

This command runs a step-by-step connectivity check:

```python
def diagnose_connection(host, port=18333):
    """Step-by-step P2P connectivity diagnostic."""
    results = []

    # Test 1: DNS / IP resolution
    print(f"[1/7] Resolving {host}...")
    try:
        ip = socket.gethostbyname(host)
        print(f"  ✓ Resolved to {ip}")
        results.append(("DNS", True, ip))
    except socket.gaierror as e:
        print(f"  ✗ DNS resolution failed: {e}")
        print(f"  → Check: Is the hostname correct? Try using IP directly.")
        results.append(("DNS", False, str(e)))
        return results

    # Test 2: TCP port reachable
    print(f"[2/7] TCP connect to {ip}:{port}...")
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(10)
    try:
        sock.connect((ip, port))
        print(f"  ✓ TCP connection established")
        results.append(("TCP", True, "connected"))
    except socket.timeout:
        print(f"  ✗ Connection timed out (10s)")
        print(f"  → Check: Is port {port} open? Firewall? NAT?")
        print(f"  → Run on remote: ss -tlnp | grep {port}")
        print(f"  → Run on remote: sudo ufw status  (or iptables -L)")
        results.append(("TCP", False, "timeout"))
        return results
    except ConnectionRefusedError:
        print(f"  ✗ Connection refused")
        print(f"  → Port {port} is reachable but nothing is listening")
        print(f"  → Check: Is the OFC node running? python3 ofc.py start")
        results.append(("TCP", False, "refused"))
        return results
    except OSError as e:
        print(f"  ✗ Network error: {e}")
        print(f"  → Check: Network connectivity, routing, VPN")
        results.append(("TCP", False, str(e)))
        return results

    # Test 3: Magic bytes
    print(f"[3/7] Sending OFC magic bytes (0x4F464321)...")
    try:
        sock.sendall(MAGIC)
        sock.settimeout(5)
        data = sock.recv(4)
        if data == MAGIC:
            print(f"  ✓ Peer responded with correct magic: {data.hex()}")
            results.append(("MAGIC", True, data.hex()))
        elif len(data) == 0:
            print(f"  ✗ Peer closed connection after receiving magic")
            print(f"  → Peer may have a different magic / protocol version")
            results.append(("MAGIC", False, "closed"))
        else:
            print(f"  ✗ Wrong magic: got {data.hex()} expected 4f464321")
            print(f"  → This is NOT an OFC node, or magic bytes are misconfigured")
            # Try to decode what we got
            try:
                text = data.decode('ascii', errors='replace')
                print(f"  → Received text: {repr(text)}")
                if text.startswith("HTTP"):
                    print(f"  → Looks like an HTTP server, not an OFC node")
                elif text.startswith("SSH"):
                    print(f"  → Looks like an SSH server")
            except:
                pass
            results.append(("MAGIC", False, f"got:{data.hex()}"))
    except socket.timeout:
        print(f"  ✗ No response to magic bytes (5s timeout)")
        print(f"  → Peer may not be running OFC, or connection is one-way")
        results.append(("MAGIC", False, "timeout"))
        sock.close()
        return results

    # Test 4: Version exchange
    print(f"[4/7] Sending version message...")
    try:
        version_msg = {
            "type": "version",
            "data": {
                "protocol": 1,
                "agent": "ofc-diag/1.0",
                "height": 0,
                "genesis_hash": GENESIS_HASH,
                "port": 0,
                "timestamp": int(time.time()),
                "services": 0,
                "nonce": struct.unpack('<Q', os.urandom(8))[0],
            }
        }
        send_json(sock, version_msg)
        print(f"  ✓ Version sent")

        peer_version = recv_json(sock, timeout=10)
        if peer_version is None:
            print(f"  ✗ No version response (10s timeout)")
            print(f"  → Peer accepted magic but didn't reply with version")
            print(f"  → Possible: peer crashed, protocol bug")
            results.append(("VERSION", False, "timeout"))
            sock.close()
            return results

        if peer_version.get('type') == 'reject':
            reason = peer_version.get('data', {}).get('reason', 'unknown')
            print(f"  ✗ Peer rejected us: {reason}")
            results.append(("VERSION", False, f"rejected: {reason}"))
            sock.close()
            return results

        if peer_version.get('type') != 'version':
            print(f"  ✗ Unexpected message type: {peer_version.get('type')}")
            results.append(("VERSION", False, f"wrong type: {peer_version.get('type')}"))
            sock.close()
            return results

        pd = peer_version['data']
        print(f"  ✓ Peer version received:")
        print(f"    agent:   {pd.get('agent', '?')}")
        print(f"    height:  {pd.get('height', '?')}")
        print(f"    genesis: {pd.get('genesis_hash', '?')[:16]}...")
        results.append(("VERSION", True, pd))

    except Exception as e:
        print(f"  ✗ Version exchange failed: {e}")
        results.append(("VERSION", False, str(e)))
        sock.close()
        return results

    # Test 5: Genesis hash match
    print(f"[5/7] Verifying genesis hash...")
    peer_genesis = pd.get('genesis_hash', '')
    if peer_genesis == GENESIS_HASH:
        print(f"  ✓ Genesis match — same OFC network!")
        results.append(("GENESIS", True, "match"))
    else:
        print(f"  ✗ GENESIS MISMATCH!")
        print(f"    Ours:   {GENESIS_HASH[:32]}...")
        print(f"    Theirs: {peer_genesis[:32]}...")
        print(f"  → You and the peer are on DIFFERENT networks")
        print(f"  → Causes: different code generated at different times,")
        print(f"    or genesis block parameters were modified")
        print(f"  → Fix: both nodes must use the same genesis block.")
        print(f"    See SKILL.md 'Genesis Block' section for the canonical genesis.")
        results.append(("GENESIS", False, f"ours={GENESIS_HASH[:16]} theirs={peer_genesis[:16]}"))
        sock.close()
        return results

    # Test 6: Verack
    print(f"[6/7] Completing handshake (verack)...")
    try:
        send_json(sock, {"type": "verack", "data": {}})
        verack = recv_json(sock, timeout=10)
        if verack and verack.get('type') == 'verack':
            print(f"  ✓ Handshake complete!")
            results.append(("VERACK", True, "ok"))
        else:
            print(f"  ✗ No verack received")
            results.append(("VERACK", False, "no verack"))
    except Exception as e:
        print(f"  ✗ Verack failed: {e}")
        results.append(("VERACK", False, str(e)))

    # Test 7: Status query
    print(f"[7/7] Querying peer status...")
    try:
        send_json(sock, {"type": "get_status", "data": {}})
        status = recv_json(sock, timeout=10)
        if status and status.get('type') == 'status':
            sd = status['data']
            print(f"  ✓ Peer status:")
            print(f"    height:    {sd.get('height', '?')}")
            print(f"    best_hash: {sd.get('best_hash', '?')[:16]}...")
            results.append(("STATUS", True, sd))
        else:
            print(f"  ⚠ No status response (optional — peer may not support it)")
            results.append(("STATUS", False, "no response"))
    except:
        results.append(("STATUS", False, "error"))

    sock.close()

    # Summary
    print(f"\n{'='*50}")
    print(f"DIAGNOSTIC SUMMARY")
    print(f"{'='*50}")
    passed = sum(1 for _, ok, _ in results if ok)
    total = len(results)
    for name, ok, detail in results:
        symbol = "✓" if ok else "✗"
        print(f"  {symbol} {name}: {detail if isinstance(detail, str) else 'ok'}")
    print(f"\nResult: {passed}/{total} checks passed")

    if passed == total:
        print("🎉 Connection fully healthy — you can sync and mine!")
    elif passed >= 5:
        print("⚠ Connection mostly works, minor issues detected")
    else:
        first_fail = next((name for name, ok, _ in results if not ok), None)
        print(f"❌ Connection failed at: {first_fail}")

    return results
```

---

## Common Problems and Fixes

### Problem 1: "Connection refused"

```
Symptom:  python3 ofc.py start --seednode=1.2.3.4:18333
          → ConnectionRefusedError: [Errno 111] Connection refused

Meaning:  TCP reached the machine, but nothing is listening on port 18333.

Check on the REMOTE machine:
  ss -tlnp | grep 18333
  # Should show: LISTEN  0  20  0.0.0.0:18333

If nothing shows:
  → OFC node is not running. Start it: python3 ofc.py start
  → Node is running on a different port. Check overflow.conf
  → Node crashed. Check debug.log

If it shows 127.0.0.1:18333 (not 0.0.0.0):
  → Node is bound to localhost only. Change overflow.conf:
    [network]
    host=0.0.0.0
```

### Problem 2: "Connection timed out"

```
Symptom:  Connect hangs for 10+ seconds, then times out.

Meaning:  TCP packets are being dropped (never reach the peer).

Check sequence:
  1. Can you ping the peer?
     ping 1.2.3.4
     → If no reply: network/routing issue, check VPN/Tailscale

  2. Is the port blocked by firewall?
     On remote: sudo ufw allow 18333/tcp
     On remote: sudo iptables -I INPUT -p tcp --dport 18333 -j ACCEPT

  3. Is NAT port forwarding set up?
     On router: forward external 18333 → internal_ip:18333
     Verify: from outside, use diagnose tool:
       python3 ofc.py diagnose PUBLIC_IP --port 18333

  4. Cloud provider security group?
     AWS: EC2 → Security Groups → allow inbound TCP 18333
     GCP: VPC → Firewall → allow tcp:18333
     Azure: NSG → add inbound rule for 18333
```

### Problem 3: "Bad magic bytes"

```
Symptom:  [MAGIC] 1.2.3.4:18333 ✗ Bad magic bytes

Meaning:  Something is listening on port 18333, but it's not an OFC node.

Causes:
  - Another service using port 18333 (unlikely but possible)
  - The peer is running OFC code without magic byte support
    (code generated before this feature was added)
  - The peer is running a different blockchain on the same port

Fix:
  - Verify both sides are running the same code version
  - Check what's on the port: curl http://1.2.3.4:18333 -v
    If you get HTTP response → it's a web server, not OFC
  - Change port in overflow.conf and retry
```

### Problem 4: "Genesis mismatch"

```
Symptom:  [VERSION] ✗ genesis mismatch: peer=abcd1234... ours=0000d5d1...

Meaning:  Your node and the peer have DIFFERENT genesis blocks.
          You are literally on different blockchains.

This is the #1 problem when AI generates code at different times.

Fix:
  Both nodes MUST use the exact same genesis block:
    hash: 0000d5d1702ee9723c2b769189a962f75f55c08a8ff572740f905c61bb11fc8f

  Verify your genesis:
    python3 -c "
    from chain.block import genesis_block
    _, _, h = genesis_block()
    print('Genesis:', h[::-1].hex())
    "

  If it doesn't print 0000d5d1... :
    Your code has a different genesis. Regenerate following the SKILL.md spec,
    or copy the genesis_block() function from SKILL.md directly.

  After fixing genesis:
    Delete old chain data: rm ~/.overflowchain/chain.db
    Reinitialize: python3 ofc.py init
```

### Problem 5: "No version response"

```
Symptom:  Magic bytes OK, but no version message comes back.

Causes:
  - Peer is running but frozen/crashed after magic exchange
  - Peer's version message format is different (code mismatch)
  - Network is very slow (increase timeout)
  - Peer's send buffer is full (too many connections)

Fix:
  - Check peer's debug.log for errors at the same timestamp
  - Verify both sides send version as JSON + newline after magic
  - Try with longer timeout: modify HANDSHAKE_TIMEOUT to 30s
```

### Problem 6: "Clock skew" / timestamps rejected

```
Symptom:  [VERSION] ✗ clock skew: 7500s

Meaning:  Your clock and the peer's clock differ by >2 hours.
          Block timestamps will be rejected if clocks are too far apart.

Fix:
  sudo timedatectl set-ntp true     # Enable NTP
  timedatectl status                # Verify synchronized
  # Or manually: sudo ntpdate pool.ntp.org
```

### Problem 7: "Address already in use"

```
Symptom:  OSError: [Errno 98] Address already in use

Meaning:  Port 18333 is already taken (maybe a previous node didn't shut down).

Fix:
  # Find what's using the port:
  lsof -i :18333
  # Or:
  ss -tlnp | grep 18333

  # Kill the old process:
  kill <PID>

  # Or use a different port:
  python3 ofc.py start --port 18334
```

---

## Quick Network Health Check Script

The generated code should include this as `python3 ofc.py netcheck`:

```python
def network_health_check():
    """Run from any machine to verify OFC networking basics."""
    import socket, subprocess, shutil

    print("=== OFC Network Health Check ===\n")

    # 1. Python version
    import sys
    print(f"[1] Python: {sys.version.split()[0]}", "✓" if sys.version_info >= (3,8) else "✗ Need 3.8+")

    # 2. ed25519 available
    try:
        import ed25519
        print(f"[2] ed25519: installed ✓")
    except ImportError:
        print(f"[2] ed25519: NOT INSTALLED ✗ → pip install ed25519")

    # 3. Port 18333 available
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        s.bind(('0.0.0.0', 18333))
        print(f"[3] Port 18333: available ✓")
    except OSError:
        print(f"[3] Port 18333: IN USE ✗ → another node running?")
    finally:
        s.close()

    # 4. Can reach internet
    try:
        socket.create_connection(("8.8.8.8", 53), timeout=3)
        print(f"[4] Internet: reachable ✓")
    except:
        print(f"[4] Internet: UNREACHABLE ✗ → check network")

    # 5. Firewall check (Linux)
    if shutil.which('ufw'):
        r = subprocess.run(['ufw', 'status'], capture_output=True, text=True)
        if '18333' in r.stdout and 'ALLOW' in r.stdout:
            print(f"[5] Firewall: port 18333 allowed ✓")
        elif 'inactive' in r.stdout:
            print(f"[5] Firewall: ufw inactive (all ports open) ✓")
        else:
            print(f"[5] Firewall: port 18333 NOT in allow list ⚠")
            print(f"    → sudo ufw allow 18333/tcp")
    elif shutil.which('iptables'):
        r = subprocess.run(['iptables', '-L', '-n'], capture_output=True, text=True)
        if '18333' in r.stdout:
            print(f"[5] Firewall: port 18333 rule found ✓")
        else:
            print(f"[5] Firewall: no explicit 18333 rule (may be open by default)")
    else:
        print(f"[5] Firewall: cannot check (no ufw/iptables)")

    # 6. External IP
    try:
        import urllib.request
        ip = urllib.request.urlopen('https://ifconfig.me', timeout=5).read().decode().strip()
        print(f"[6] External IP: {ip}")
    except:
        print(f"[6] External IP: cannot determine (offline or blocked)")

    # 7. Data directory
    import os
    datadir = os.path.expanduser('~/.overflowchain')
    if os.path.exists(datadir):
        chain_db = os.path.join(datadir, 'chain.db')
        if os.path.exists(chain_db):
            size = os.path.getsize(chain_db) / 1024
            print(f"[7] Data dir: {datadir} ✓ (chain.db: {size:.0f}KB)")
        else:
            print(f"[7] Data dir: {datadir} exists but no chain.db → run init")
    else:
        print(f"[7] Data dir: not found → run: python3 ofc.py init")

    print(f"\nDone. Fix any ✗ items before starting the node.")
```

---

## WAN / NAT Traversal

### Option A: Port Forwarding

```
Router admin → forward external 18333 → your_local_ip:18333

Verify from outside:
  python3 ofc.py diagnose YOUR_PUBLIC_IP
```

### Option B: Tailscale (recommended for NAT-to-NAT)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Use Tailscale IPs in overflow.conf:
[network]
seednode=100.64.x.x:18333
```

### Option C: SSH Tunnel

```bash
ssh -L 18333:localhost:18333 user@remote-machine
# Then connect to localhost:18333
```

---

## Peer Scoring

```python
# +10: valid new block received
# +1:  valid transaction received
# -50: invalid block sent
# -10: invalid transaction sent
# -5:  request timeout
# -20: bad magic bytes
# -30: genesis mismatch
# Ban threshold: score < -50
```

---

## Resource Limits

```python
MAX_CONNECTIONS = 20
MAX_MEMPOOL_SIZE = 1000
MAX_SEEN_HASHES = 10000
HEADER_BATCH_SIZE = 500
BLOCK_BATCH_SIZE = 50
MAX_MESSAGE_SIZE = 2 * 1024 * 1024  # 2MB per JSON line
HANDSHAKE_TIMEOUT = 10              # seconds
MAGIC_TIMEOUT = 5                   # seconds
PING_INTERVAL = 60                  # seconds
PING_TIMEOUT = 30                   # seconds
```
