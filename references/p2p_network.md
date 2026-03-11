# P2P Network Guide — Lightweight Edition

## Architecture

Pure TCP + JSON Lines. No libp2p, no compiled dependencies. Uses only Python
stdlib: `socket`, `ssl`, `threading`, `json`.

```
┌──────────────────────────────────────────┐
│              Application                  │
│        (Block sync, Tx broadcast)         │
├──────────────────────────────────────────┤
│          Flood Gossip (broadcast)         │
│      Topics: blocks, transactions         │
├──────────────────────────────────────────┤
│           Peer Manager                    │
│    (connect, discover, score, ban)        │
├──────────────────────────────────────────┤
│         JSON-Line Protocol                │
│    (one JSON object per line, \n delim)   │
├──────────────────────────────────────────┤
│      TLS (optional, ssl stdlib)           │
├──────────────────────────────────────────┤
│             Raw TCP                       │
└──────────────────────────────────────────┘

Memory footprint: ~500 bytes per connection + message buffers
Max connections: 20 (configurable, memory-safe default)
```

---

## Connection Lifecycle

```python
import socket, json, threading

class PeerConnection:
    def __init__(self, sock, addr):
        self.sock = sock
        self.addr = addr
        self.reader = sock.makefile('r', buffering=1)  # line-buffered
        self.writer = sock.makefile('w', buffering=1)
        self.peer_height = 0
        self.peer_agent = ""
        self.score = 0

    def send(self, msg_type, data=None):
        msg = json.dumps({"type": msg_type, "data": data or {}})
        self.writer.write(msg + "\n")
        self.writer.flush()

    def recv(self):
        line = self.reader.readline()
        if not line:
            raise ConnectionError
        return json.loads(line)

    def close(self):
        try:
            self.sock.shutdown(socket.SHUT_RDWR)
        except OSError:
            pass
        self.sock.close()
```

---

## Protocol Messages

All messages are JSON objects, one per line. Each has `type` and `data` fields.

### Handshake

```
→ {"type":"version","data":{"protocol":1,"height":42,"agent":"ofc-py/2.0","port":18333}}
← {"type":"verack","data":{}}
← {"type":"version","data":{"protocol":1,"height":100,"agent":"ofc-py/2.0","port":18333}}
→ {"type":"verack","data":{}}
```

Both sides send version, both sides reply verack. After mutual verack,
the connection is established.

### Block Propagation

```
{"type":"new_block","data":{"hash":"<hex>","height":43,"block":"<hex-encoded full block>"}}
{"type":"get_block","data":{"hash":"<hex>"}}
{"type":"block","data":{"block":"<hex>"}}
```

### Transactions

```
{"type":"new_tx","data":{"tx":"<hex-encoded transaction>"}}
```

### Chain Sync (Headers-First)

```
{"type":"get_headers","data":{"from_height":0,"count":500}}
{"type":"headers","data":{"headers":["<80-byte hex>","<80-byte hex>",...]}}
{"type":"get_blocks","data":{"hashes":["<hex>","<hex>",...]}}
{"type":"blocks","data":{"blocks":["<hex>","<hex>",...]}}
```

Batch size capped at 500 headers / 50 blocks per message to limit memory.

### Peer Discovery

```
{"type":"get_peers","data":{}}
{"type":"peers","data":{"peers":[{"host":"1.2.3.4","port":18333},...]}}
```

### Status

```
{"type":"get_status","data":{}}
{"type":"status","data":{"height":142,"best_hash":"<hex>","genesis_hash":"<hex>"}}
```

### Ping/Pong

```
{"type":"ping","data":{"nonce":12345}}
{"type":"pong","data":{"nonce":12345}}
```

---

## Node Server

```python
class Node:
    def __init__(self, config):
        self.host = config.get('host', '0.0.0.0')
        self.port = config.get('port', 18333)
        self.max_peers = config.get('maxconnections', 20)
        self.peers = {}       # addr_str -> PeerConnection
        self.seen_blocks = set()  # block hashes we already have (dedup)
        self.seen_txs = set()     # tx hashes we already have

    def start(self):
        srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind((self.host, self.port))
        srv.listen(self.max_peers)

        # Accept inbound
        threading.Thread(target=self._accept_loop, args=(srv,), daemon=True).start()

        # Connect outbound to seeds
        for seed in self.config.get('seednodes', []):
            threading.Thread(target=self._connect_to, args=(seed,), daemon=True).start()

    def _accept_loop(self, srv):
        while True:
            sock, addr = srv.accept()
            if len(self.peers) >= self.max_peers:
                sock.close()
                continue
            threading.Thread(target=self._handle_peer, args=(sock, addr), daemon=True).start()

    def _handle_peer(self, sock, addr):
        peer = PeerConnection(sock, addr)
        try:
            self._handshake(peer)
            self.peers[f"{addr[0]}:{addr[1]}"] = peer
            while True:
                msg = peer.recv()
                self._dispatch(peer, msg)
        except (ConnectionError, json.JSONDecodeError):
            pass
        finally:
            self.peers.pop(f"{addr[0]}:{addr[1]}", None)
            peer.close()
```

---

## Gossip: Simple Flood

For networks under ~100 nodes, flood is optimal: minimal code, minimal memory,
and propagation is fast (one hop to all peers).

```python
def broadcast(self, msg_type, data, exclude_peer=None):
    """Send to all connected peers except the one who sent it to us."""
    for addr, peer in list(self.peers.items()):
        if peer != exclude_peer:
            try:
                peer.send(msg_type, data)
            except:
                self._disconnect(addr)

def on_new_block(self, peer, data):
    block_hash = data['hash']
    if block_hash in self.seen_blocks:
        return  # Dedup: already have it
    self.seen_blocks.add(block_hash)

    block = Block.from_hex(data['block'])
    if self.chain.validate_and_add(block):
        self.broadcast('new_block', data, exclude_peer=peer)
    else:
        peer.score -= 10  # Penalty for invalid block
```

Dedup via `seen_blocks` set prevents infinite rebroadcast. Memory: ~32 bytes
per block hash × chain length.

---

## Chain Sync (IBD)

```python
def initial_sync(self):
    """Headers-first sync from the best peer."""
    best_peer = max(self.peers.values(), key=lambda p: p.peer_height)
    local_height = self.chain.get_height()

    if best_peer.peer_height <= local_height:
        return  # Already synced

    # Phase 1: Download headers
    height = local_height + 1
    while height <= best_peer.peer_height:
        best_peer.send('get_headers', {'from_height': height, 'count': 500})
        resp = best_peer.recv()
        if resp['type'] != 'headers':
            break
        headers = resp['data']['headers']
        for h_hex in headers:
            header = BlockHeader.from_hex(h_hex)
            if not self.chain.validate_header(header):
                raise SyncError("Invalid header")
        height += len(headers)

    # Phase 2: Download full blocks
    missing = self.chain.get_missing_block_hashes()
    for batch in chunks(missing, 50):
        best_peer.send('get_blocks', {'hashes': batch})
        resp = best_peer.recv()
        for b_hex in resp['data']['blocks']:
            block = Block.from_hex(b_hex)
            self.chain.validate_and_add(block)
```

---

## TLS Encryption (optional)

For WAN connections, wrap TCP sockets with TLS:

```python
import ssl

# Server side
context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.load_cert_chain('node.crt', 'node.key')
secure_sock = context.wrap_socket(sock, server_side=True)

# Client side
context = ssl.create_default_context()
context.check_hostname = False
context.verify_mode = ssl.CERT_NONE  # Self-signed OK for P2P
secure_sock = context.wrap_socket(sock)

# Generate self-signed cert:
# openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
#   -keyout node.key -out node.crt -days 365 -nodes -subj "/CN=ofc-node"
```

---

## WAN / NAT Traversal

**Option A: Port forwarding** — Forward router port 18333 to local machine.

**Option B: Tailscale** (recommended for NAT-to-NAT):
```bash
# Both machines:
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Then use Tailscale IPs in overflow.conf:
seednode=100.64.x.x:18333
```

**Option C: SSH tunnel** (no install needed):
```bash
# On machine B, tunnel to machine A's port:
ssh -L 18333:localhost:18333 user@machine-a
# Now B can connect to localhost:18333 which tunnels to A
```

---

## Peer Scoring

```python
class PeerScore:
    def __init__(self):
        self.score = 0

    def reward(self, points):
        self.score = min(self.score + points, 100)

    def penalize(self, points):
        self.score = max(self.score - points, -100)

    def should_ban(self):
        return self.score < -50

# Events:
# +10: valid new block received
# +1:  valid transaction received
# -50: invalid block sent
# -10: invalid transaction sent
# -5:  request timeout
```

---

## Resource Limits

```python
# Memory-safe defaults for 1GB machines:
MAX_CONNECTIONS = 20           # ~10KB per connection
MAX_MEMPOOL_SIZE = 1000        # transactions
MAX_SEEN_HASHES = 10000        # dedup cache (320KB)
HEADER_BATCH_SIZE = 500        # sync batch
BLOCK_BATCH_SIZE = 50          # sync batch
MAX_MESSAGE_SIZE = 2 * 1024 * 1024  # 2MB per JSON line
```
