# Week 6: Ad-Hoc Networking (MANET Simulation)

## 📋 Overview
Week 6 เรียนรู้เกี่ยวกับ **Mobile Ad-Hoc Networks (MANET)** เครือข่ายที่ node เคลื่อนที่ได้และสื่อสารกันโดยไม่ต้องใช้ infrastructure

## 🎯 Learning Objectives
- เข้าใจ Ad-Hoc Network Concepts
- เรียนรู้ Neighbor Discovery
- ฝึกใช้ Probabilistic Forwarding
- เข้าใจ TTL และ Loop Prevention

## 📚 Key Concepts

### MANET Characteristics
| Feature | Description |
|---------|-------------|
| Infrastructure-less | ไม่มี access point หรือ base station |
| Dynamic Topology | Node เคลื่อนที่ได้ตลอดเวลา |
| Multi-hop | ข้อความ hop ผ่าน intermediate nodes |
| Limited Resources | พลังงานและ bandwidth จำกัด |
| Self-Configuring | Nodes จัดการตัวเอง |

### MANET vs Traditional Networks
```
Traditional (Infrastructure):
    ┌──────────┐
    │   Base   │
    │  Station │
    └────┬─────┘
         │
    ┌────┼────┐
    │    │    │
    ▼    ▼    ▼
┌────┐ ┌────┐ ┌────┐
│Node│ │Node│ │Node│
└────┘ └────┘ └────┘

Ad-Hoc (No Infrastructure):
    ┌────┐         ┌────┐
    │ A  │◄───────►│ B  │
    └─┬──┘         └─┬──┘
      │              │
      └──────┬───────┘
             │
          ┌──▼──┐
          │  C  │
          └─────┘
    (Nodes communicate directly)
```

### Key Algorithms
1. **Neighbor Discovery**: ค้นหา nodes ที่อยู่ใกล้เคียง
2. **Probabilistic Forwarding**: Forward packet ด้วยความน่าจะเป็น
3. **TTL (Time-To-Live)**: จำกัดจำนวน hop

## 💻 Implementation

### Node Code (node.py)
```python
import socket
import threading
import random
from config import HOST, BASE_PORT, BUFFER_SIZE, NEIGHBORS, FORWARD_PROBABILITY, TTL

# Neighbor table
neighbor_table = set(NEIGHBORS)

# Handle incoming messages
def handle_incoming(conn, addr):
    data = conn.recv(BUFFER_SIZE).decode()
    msg, ttl = data.split('|')
    ttl = int(ttl)
    print(f"[NODE {BASE_PORT}] Received from {addr}: {msg} (TTL={ttl})")
    conn.close()

    # Forward probabilistically
    if ttl > 0 and random.random() < FORWARD_PROBABILITY:
        forward_message(msg, ttl - 1, exclude=addr[1])

# Start server
def start_server(port):
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind((HOST, port))
    server.listen()
    print(f"[NODE {port}] Listening for neighbors...")
    while True:
        conn, addr = server.accept()
        threading.Thread(target=handle_incoming, args=(conn, addr)).start()

# Forward message to neighbors
def forward_message(message, ttl, exclude=None):
    for peer_port in neighbor_table:
        if peer_port == exclude:
            continue
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect((HOST, peer_port))
            s.sendall(f"{message}|{ttl}".encode())
            s.close()
        except ConnectionRefusedError:
            print(f"[NODE {BASE_PORT}] Peer {peer_port} unreachable")

# Main
if __name__ == "__main__":
    threading.Thread(target=start_server, args=(BASE_PORT,), daemon=True).start()
    
    # Send test message
    test_message = f"Hello from node {BASE_PORT}"
    forward_message(test_message, TTL)
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"
BASE_PORT = 7000  # เปลี่ยนสำหรับแต่ละ node
BUFFER_SIZE = 1024
NEIGHBORS = [7001, 7002]  # Ports ของ neighbors
FORWARD_PROBABILITY = 0.5  # 50% chance to forward
TTL = 3  # Maximum hops
```

## 🧪 Execution & Verification

### รัน 3 Nodes
```bash
# Terminal 1 (Node 7000)
python node.py
# BASE_PORT = 7000, NEIGHBORS = [7001, 7002]

# Terminal 2 (Node 7001)
python node.py
# BASE_PORT = 7001, NEIGHBORS = [7000, 7002]

# Terminal 3 (Node 7002)
python node.py
# BASE_PORT = 7002, NEIGHBORS = [7000, 7001]
```

### Expected Output
```
[NODE 7000] Listening for neighbors...
[NODE 7000] Received from ('127.0.0.1', 54321): Hello from node 7001 (TTL=2)
[NODE 7000] Peer 7002 unreachable
```

## 📝 What I Learned

### Technical Skills
1. **Dynamic Neighbor Table** - จัดการ neighbors ที่เปลี่ยนแปลง
2. **Probabilistic Logic** - ใช้ random สำหรับ forwarding decision
3. **TTL Management** - ลด TTL แต่ละ hop ป้องกัน loops
4. **Multi-threading** - รับและส่งพร้อมกัน

### Conceptual Understanding
- **Self-Organization**: Nodes จัดการ topology เอง
- **Probabilistic Forwarding**: ไม่ forward ทุก packet ลด congestion
- **Multi-Hop Routing**: ข้อความผ่าน intermediate nodes
- **Mobility Impact**: Links มาและไปตลอดเวลา

### Message Propagation Example
```
Node A (TTL=3)
    │
    ├─► Node B (TTL=2) ─────┐
    │                        │
    │                   (50% forward)
    │                        │
    └─► Node C (TTL=2) ◄────┘
            │
            ▼
       (TTL=1, may not forward)
```

## 🔧 Extensions

### Extension A: Neighbor Discovery Protocol
```python
def discover_neighbors():
    """Scan for active neighbors"""
    for port in range(7000, 7010):
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.settimeout(0.5)
            s.connect((HOST, port))
            neighbor_table.add(port)
            s.close()
        except:
            pass
```

### Extension B: Mobility Simulation
```python
import random

def simulate_mobility():
    """Randomly add/remove neighbors"""
    if random.random() < 0.1:
        # Remove random neighbor
        if neighbor_table:
            neighbor_table.pop()
    else:
        # Add new neighbor
        new_port = random.randint(7000, 7010)
        neighbor_table.add(new_port)
```

### Extension C: Routing Metrics
```python
class RoutingMetrics:
    def __init__(self):
        self.delivery_count = 0
        self.forward_count = 0
    
    def delivery_rate(self):
        return self.delivery_count / max(self.forward_count, 1)
    
    def adjust_probability(self, rate):
        if rate < 0.5:
            FORWARD_PROBABILITY *= 0.9  # Decrease
        else:
            FORWARD_PROBABILITY *= 1.1  # Increase
```

## 🌍 Real-World Applications
- **Disaster Recovery** - สื่อสารเมื่อ infrastructure พัง
- **Military Communications** - Tactical networks
- **Vehicular Networks** - Car-to-car communication
- **Sensor Networks** - Environmental monitoring
- **Search and Rescue** - Temporary emergency networks

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| Infinite loops | No TTL or TTL not decrementing | Always decrement TTL |
| Port conflicts | Same port for multiple nodes | Use different BASE_PORT |
| Network flooding | FORWARD_PROBABILITY too high | Reduce to 0.3-0.5 |
| Message not propagating | TTL too low | Increase TTL to 3-5 |

## 📖 References
- [MANET Overview](https://www.sciencedirect.com/topics/computer-science/mobile-ad-hoc-network)
- [AODV Routing Protocol](https://datatracker.ietf.org/doc/html/rfc3561)
