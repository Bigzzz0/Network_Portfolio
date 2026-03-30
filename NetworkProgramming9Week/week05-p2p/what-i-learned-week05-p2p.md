# Week 5: Peer-to-Peer Networking

## 📋 Overview
Week 5 เรียนรู้เกี่ยวกับ **Peer-to-Peer (P2P) Networking** ซึ่งแต่ละ node เป็นทั้ง client และ server พร้อมกัน

## 🎯 Learning Objectives
- เข้าใจ P2P Architecture
- เรียนรู้ Symmetric Roles
- ฝึกใช้ Threading สำหรับ Concurrent Operations
- เข้าใจ Decentralized Communication

## 📚 Key Concepts

### Client-Server vs Peer-to-Peer
```
Client-Server:
        ┌──────────┐
        │  Server  │
        └────┬─────┘
             │
    ┌────────┼────────┐
    │        │        │
    ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐
│Client│ │Client│ │Client│
└──────┘ └──────┘ └──────┘
(centralized)

Peer-to-Peer:
    ┌──────┐         ┌──────┐
    │ Peer │◄───────►│ Peer │
    │  1   │         │  2   │
    └──┬───┘         └──┬───┘
       │                │
       └───────┬────────┘
               │
          ┌────▼────┐
          │  Peer   │
          │   3     │
          └─────────┘
(decentralized, symmetric)
```

### P2P Characteristics
- **Symmetric Roles**: ทุก peer เท่าเทียมกัน
- **No Central Authority**: ไม่มี server กลาง
- **Self-Organizing**: Nodes จัดการตัวเอง
- **Fault Tolerant**: ไม่มี single point of failure

## 💻 Implementation

### Peer Code (peer.py)
```python
import socket
import threading
import sys
from config import HOST, BASE_PORT, BUFFER_SIZE

# รับ peer ID จาก command line
peer_id = int(sys.argv[1])
PORT = BASE_PORT + peer_id

# Listener Thread - รับ incoming connections
def listen():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.bind((HOST, PORT))
    sock.listen(5)
    print(f"[PEER {peer_id}] Listening on port {PORT}")

    while True:
        conn, addr = sock.accept()
        data = conn.recv(BUFFER_SIZE)
        print(f"[PEER {peer_id}] From {addr}: {data.decode()}")
        conn.close()

# Sender Function - ส่งข้อความไปยัง peer อื่น
def send_message(target_peer_id, message):
    target_port = BASE_PORT + target_peer_id
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        sock.connect((HOST, target_port))
        sock.sendall(message.encode())
        print(f"[PEER {peer_id}] Sent to Peer {target_peer_id}")
    except ConnectionRefusedError:
        print(f"[PEER {peer_id}] Peer {target_peer_id} not available")
    finally:
        sock.close()

# Start listener thread
threading.Thread(target=listen, daemon=True).start()

# Main loop - ส่งข้อความ
while True:
    try:
        target = int(input("Send to peer ID: "))
        msg = input("Message: ")
        send_message(target, msg)
    except KeyboardInterrupt:
        print("\nShutting down...")
        break
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"
BASE_PORT = 9000
BUFFER_SIZE = 1024
```

## 🧪 Execution & Verification

### รัน Peer 3 ตัว (3 terminals)
```bash
# Terminal 1
python peer.py 1
# Output: [PEER 1] Listening on port 9001

# Terminal 2
python peer.py 2
# Output: [PEER 2] Listening on port 9002

# Terminal 3
python peer.py 3
# Output: [PEER 3] Listening on port 9003
```

### ส่งข้อความระหว่าง Peers
```
# ใน Terminal 1
Send to peer ID: 2
Message: Hello from Peer 1

# ใน Terminal 2 จะเห็น:
[PEER 2] From ('127.0.0.1', 54321): Hello from Peer 1
```

## 📝 What I Learned

### Technical Skills
1. **Dual Role Sockets** - ทั้ง listen และ connect ในโปรแกรมเดียวกัน
2. **Threading** - ใช้ thread แยกสำหรับ listener
3. **Port Management** - จัดการ port ตาม peer ID
4. **Error Handling** - จัดการ connection refused

### Conceptual Understanding
- **Symmetry**: ทุก peer มี capability เท่ากัน
- **Concurrency**: ต้องรับและส่งพร้อมกันได้
- **Discovery**: ต้องรู้ว่า peer อื่นอยู่ที่ไหน
- **Independence**: ไม่มี central coordinator

### P2P Communication Flow
```
Peer 1                          Peer 2
  │                               │
  ├─► listen() on 9001           ├─► listen() on 9002
  │                               │
  ├─► connect(9002) ────────────►│
  │         send("Hello")         │
  │                               ├─► recv()
  │                               │   print message
  │                               │
  │◄──────── connect(9001) ──────┤
  │         recv()                │   send("Hi back")
  │   print message               │
```

## 🔧 Extensions

### Extension A: Peer List Management
```python
class PeerTable:
    def __init__(self):
        self.peers = {}
    
    def add_peer(self, peer_id, host, port):
        self.peers[peer_id] = (host, port)
    
    def get_peer(self, peer_id):
        return self.peers.get(peer_id)
    
    def broadcast(self, message):
        for peer_id, (host, port) in self.peers.items():
            # send to each peer
```

### Extension B: Message Relay
```python
def relay_message(message, source_peer, target_peer):
    """Forward message from one peer to another"""
    if target_peer in peer_table:
        send_message(target_peer, f"[Relayed from {source_peer}] {message}")
```

### Extension C: Graceful Shutdown
```python
def shutdown():
    """Notify all peers before shutting down"""
    broadcast("BYE")
    server_socket.close()
    sys.exit(0)
```

## 🌍 Real-World Applications
- **File Sharing** - BitTorrent, eMule
- **Blockchain** - Bitcoin, Ethereum nodes
- **Messaging** - decentralized chat apps
- **CDN** - Peer-assisted content delivery
- **Video Streaming** - P2P live streaming

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| Port conflict | Same port for multiple peers | Use BASE_PORT + peer_id |
| Blocking input | Can't receive while input() | Use separate thread for input |
| Deadlock | Both peers waiting | Use non-blocking or timeout |
| No peer discovery | Don't know peer addresses | Implement discovery protocol |

## 📖 References
- [P2P Networking Overview](https://www.investopedia.com/terms/p/peertopeer-p2p.asp)
- [Python Threading](https://docs.python.org/3/library/threading.html)
