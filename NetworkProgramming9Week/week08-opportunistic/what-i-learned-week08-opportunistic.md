# Week 8: Opportunistic Routing

## 📋 Overview
Week 8 เรียนรู้เกี่ยวกับ **Opportunistic Routing** การ forward ข้อความตามโอกาสและความน่าจะเป็นที่จะส่งสำเร็จ

## 🎯 Learning Objectives
- เข้าใจ Opportunistic Routing Concept
- เรียนรู้ Delivery Probability Tables
- ฝึกใช้ Encounter-Based Forwarding
- เข้าใจ Probabilistic vs Deterministic Routing

## 📚 Key Concepts

### Deterministic vs Opportunistic Routing
```
Deterministic (Traditional):
    A ──────► B ──────► C
    (fixed path, must exist)

Opportunistic:
    A ──┬──► B (if available)
        ├──► C (if better probability)
        └──► D (store if no good option)
    (choose best available option)
```

### Key Metrics
| Metric | Description |
|--------|-------------|
| Delivery Probability | ความน่าจะเป็นที่จะส่งสำเร็จ (0-1) |
| Encounter Rate | ความถี่ที่พบ peer |
| Forward Threshold | ค่าขั้นต่ำที่จะ forward |
| Queue Management | จัดการข้อความขณะรอโอกาส |

### Probability Update Example
```
Initial: P(B) = 0.5, P(C) = 0.6

After successful delivery to C:
Updated: P(C) = 0.7 (increase)

After failed delivery to B:
Updated: P(B) = 0.4 (decrease)

Next forwarding decision:
Choose C (higher probability)
```

## 💻 Implementation

### Delivery Table (delivery_table.py)
```python
class DeliveryTable:
    def __init__(self):
        self.table = {}  # {peer_port: probability}
    
    def update_probability(self, peer, prob):
        """อัพเดทความน่าจะเป็นของ peer"""
        self.table[peer] = prob
    
    def get_probability(self, peer):
        """ดึงความน่าจะเป็นของ peer"""
        return self.table.get(peer, 0.0)
    
    def get_best_candidates(self, threshold):
        """หา peers ที่มี probability สูงกว่า threshold"""
        return [
            peer for peer, prob in self.table.items() 
            if prob >= threshold
        ]
    
    def reinforce(self, peer, delta):
        """เพิ่ม probability เมื่อส่งสำเร็จ"""
        current = self.table.get(peer, 0.5)
        self.table[peer] = min(1.0, current + delta)
    
    def decay(self, peer, delta):
        """ลด probability เมื่อส่งล้มเหลว"""
        current = self.table.get(peer, 0.5)
        self.table[peer] = max(0.0, current - delta)
```

### Node Code (node.py)
```python
import socket
import threading
import time
from config import (
    HOST, BASE_PORT, PEER_PORTS, BUFFER_SIZE,
    FORWARD_THRESHOLD, UPDATE_INTERVAL
)
from delivery_table import DeliveryTable

delivery_table = DeliveryTable()
message_queue = []

# ส่งข้อความ
def send_message(peer_port, message):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(2)
        s.connect((HOST, peer_port))
        s.sendall(message.encode())
        s.close()
        print(f"[NODE {BASE_PORT}] Sent: {message} to {peer_port}")
        delivery_table.reinforce(peer_port, 0.1)  # เพิ่ม probability
        return True
    except (ConnectionRefusedError, socket.timeout):
        print(f"[NODE {BASE_PORT}] Failed to send to {peer_port}")
        delivery_table.decay(peer_port, 0.1)  # ลด probability
        return False

# Opportunistic forwarding loop
def forward_loop():
    while True:
        # หา peers ที่ดีพอจะ forward
        candidates = delivery_table.get_best_candidates(FORWARD_THRESHOLD)
        
        for msg in message_queue[:]:
            for peer in candidates:
                if send_message(peer, msg):
                    message_queue.remove(msg)
                    break  # ส่งสำเร็จแล้วหยุด
        
        time.sleep(UPDATE_INTERVAL)

# Server - รับข้อความเข้า queue
def start_server():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind((HOST, BASE_PORT))
    server.listen()
    print(f"[NODE {BASE_PORT}] Listening for incoming messages...")
    while True:
        conn, addr = server.accept()
        data = conn.recv(BUFFER_SIZE).decode()
        print(f"[NODE {BASE_PORT}] Received: {data} from {addr}")
        message_queue.append(data)
        conn.close()

# Main
if __name__ == "__main__":
    threading.Thread(target=start_server, daemon=True).start()
    threading.Thread(target=forward_loop, daemon=True).start()
    
    # Initialize delivery probabilities
    for peer in PEER_PORTS:
        delivery_table.update_probability(peer, 0.6)  # Default probability
    
    # ส่งข้อความเริ่มต้น
    for peer in PEER_PORTS:
        msg = f"Hello from node {BASE_PORT}"
        if not send_message(peer, msg):
            print(f"[NODE {BASE_PORT}] Could not deliver to {peer}, storing in queue")
            message_queue.append(msg)
    
    while True:
        time.sleep(1)
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"
BASE_PORT = 9000  # เปลี่ยนสำหรับแต่ละ node
PEER_PORTS = [9001, 9002]
BUFFER_SIZE = 1024
FORWARD_THRESHOLD = 0.5  # Forward ถ้า probability >= 0.5
UPDATE_INTERVAL = 5  # ตรวจสอบทุก 5 วินาที
```

## 🧪 Execution & Verification

### รัน 3 Nodes
```bash
# Terminal 1 - Node 9000
python node.py

# Terminal 2 - Node 9001
python node.py

# Terminal 3 - Node 9002
python node.py
```

### Expected Output
**Node 9000:**
```
[NODE 9000] Listening for incoming messages...
[NODE 9000] Sent: Hello from node 9000 to 9001
[NODE 9000] Failed to send to 9002
[NODE 9000] Could not deliver to 9002, storing in queue
```

**หลังจาก 5 วินาที:**
```
[NODE 9000] Sent: Hello from node 9000 to 9002
```

### Test Probability Updates
```
Initial state:
  P(9001) = 0.6, P(9002) = 0.6

After successful send to 9001:
  P(9001) = 0.7

After failed send to 9002:
  P(9002) = 0.5

Next decision:
  Choose 9001 (higher probability)
```

## 📝 What I Learned

### Technical Skills
1. **Probability Tables** - จัดการ delivery probabilities
2. **Threshold-Based Forwarding** - ตัดสินใจตาม threshold
3. **Dynamic Updates** - อัพเดท probability ตามผลลัพธ์
4. **Queue Management** - จัดการข้อความขณะรอโอกาส

### Conceptual Understanding
- **Probabilistic Decision**: เลือก forward ตามความน่าจะเป็น
- **Learning System**: เรียนรู้จากประวัติการส่ง
- **Opportunistic**: ใช้โอกาสเมื่อ peer พร้อม
- **Tradeoff**: รอ peer ที่ดี vs ส่งทันที

### Opportunistic Routing Flow
```
┌─────────────┐
│   Message   │
│    Queue    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Check      │
│  Probabilities│
└──────┬──────┘
       │
       ├──────────────┐
       │              │
       ▼              ▼
┌─────────────┐ ┌─────────────┐
│ P(peer) >=  │ │ P(peer) <   │
│ Threshold   │ │ Threshold   │
└──────┬──────┘ └──────┬──────┘
       │              │
       ▼              ▼
┌─────────────┐ ┌─────────────┐
│  Forward    │ │   Wait &    │
│  Now        │ │   Store     │
└─────────────┘ └─────────────┘
```

## 🔧 Extensions

### Extension A: Dynamic Probability Updates
```python
class AdaptiveDeliveryTable(DeliveryTable):
    def __init__(self):
        super().__init__()
        self.encounter_history = {}  # {peer: [success, fail, ...]}
    
    def record_encounter(self, peer, success):
        if peer not in self.encounter_history:
            self.encounter_history[peer] = []
        self.encounter_history[peer].append(1 if success else 0)
        
        # คำนวณ probability จากประวัติ 10 ครั้งล่าสุด
        recent = self.encounter_history[peer][-10:]
        prob = sum(recent) / len(recent)
        self.update_probability(peer, prob)
```

### Extension B: Message TTL
```python
class MessageWithTTL:
    def __init__(self, message, peer, ttl=10):
        self.message = message
        self.peer = peer
        self.ttl = ttl
        self.timestamp = time.time()
    
    def is_expired(self):
        return self.ttl <= 0 or time.time() - self.timestamp > 300  # 5 min
    
    def decrement_ttl(self):
        self.ttl -= 1
```

### Extension C: Logging & Statistics
```python
class RoutingStats:
    def __init__(self):
        self.attempts = 0
        self.successes = 0
        self.failures = 0
    
    def record_attempt(self, success):
        self.attempts += 1
        if success:
            self.successes += 1
        else:
            self.failures += 1
    
    def delivery_rate(self):
        return self.successes / max(self.attempts, 1)
    
    def print_stats(self):
        print(f"Delivery Rate: {self.delivery_rate():.2%}")
        print(f"Attempts: {self.attempts}, Success: {self.successes}, Fail: {self.failures}")
```

## 🌍 Real-World Applications
- **Wildlife Tracking** - Sensor networks on animals
- **Disaster Networks** - Emergency response teams
- **Vehicular Networks** - Car-to-car communication
- **Opportunistic WiFi** - WiFi offloading
- **Mobile Social Networks** - Phone-to-phone data sharing

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| Never forwarding | Threshold too high | Lower FORWARD_THRESHOLD |
| Always waiting | Probabilities not updating | Add reinforcement/decay |
| Queue overflow | No message expiry | Implement TTL |
| Stale probabilities | No decay over time | Add time-based decay |

## 📖 References
- [Opportunistic Networks](https://www.comet-center.at/opportunistic-networks/)
- [Probabilistic Routing Protocol](https://en.wikipedia.org/wiki/Probabilistic_Routing_Protocol)
