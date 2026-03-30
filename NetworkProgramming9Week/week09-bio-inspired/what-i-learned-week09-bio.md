# Week 9: Bio-Inspired Networking

## 📋 Overview
Week 9 เรียนรู้เกี่ยวกับ **Bio-Inspired Networking** ใช้อัลกอริทึมจากธรรมชาติ (เช่น มด) มาประยุกต์ใช้กับการ routing

## 🎯 Learning Objectives
- เข้าใจ Bio-Inspired Concepts
- เรียนรู้ Pheromone-Based Routing
- ฝึกใช้ Reinforcement และ Decay
- เข้าใจ Adaptive Routing Systems

## 📚 Key Concepts

### Ant Colony Optimization (ACO)
```
Real Ant Behavior:
1. Ants explore random paths
2. Find food → return to nest
3. Drop pheromones on path
4. Other ants follow stronger pheromone trails
5. Shorter paths get more pheromones (faster reinforcement)

Network Application:
1. Nodes explore paths
2. Successful delivery → reinforce path
3. Pheromone values stored in tables
4. Forward to paths with higher pheromones
5. Unused paths decay over time
```

### Pheromone Mechanics
| Mechanism | Description | Formula |
|-----------|-------------|---------|
| Reinforcement | เพิ่มเมื่อส่งสำเร็จ | `pheromone += 0.1` |
| Decay | ลดลงตามเวลา | `pheromone *= 0.9` |
| Evaporation | หายไปเมื่อไม่ใช้ | `pheromone → 0` |
| Selection | เลือก path ที่ดี | `if pheromone >= threshold` |

### Pheromone Update Cycle
```
Initial State:
  Path A: 1.0    Path B: 1.0

After successful delivery on Path A:
  Path A: 1.1    Path B: 1.0

After decay (all paths):
  Path A: 0.99   Path B: 0.90

After multiple successes on Path A:
  Path A: 1.5    Path B: 0.8

Result: Traffic prefers Path A
```

## 💻 Implementation

### Pheromone Table (pheromone_table.py)
```python
from config import DECAY_FACTOR, REINFORCEMENT

class PheromoneTable:
    def __init__(self):
        self.table = {}  # {peer_port: pheromone_value}
    
    def reinforce(self, peer, value=REINFORCEMENT):
        """เพิ่ม pheromone เมื่อส่งสำเร็จ"""
        current = self.table.get(peer, 0)
        self.table[peer] = current + value
        print(f"[PHEROMONE] {peer} reinforced: {current:.2f} → {self.table[peer]:.2f}")
    
    def decay(self):
        """ลด pheromone ของทุก peer (evaporation)"""
        for peer in self.table:
            old_value = self.table[peer]
            self.table[peer] *= DECAY_FACTOR
            if self.table[peer] < 0.01:
                self.table[peer] = 0.01  # Minimum value
    
    def get_pheromone(self, peer):
        """ดึงค่า pheromone ของ peer"""
        return self.table.get(peer, 0.0)
    
    def get_best_candidates(self, threshold):
        """หา peers ที่มี pheromone สูงกว่า threshold"""
        return [
            peer for peer, pher in self.table.items()
            if pher >= threshold
        ]
    
    def get_best_peer(self):
        """หา peer ที่มี pheromone สูงสุด"""
        if not self.table:
            return None
        return max(self.table, key=self.table.get)
    
    def print_table(self):
        """พิมพ์ตาราง pheromone"""
        print("[PHEROMONE TABLE]")
        for peer, value in sorted(self.table.items(), key=lambda x: x[1], reverse=True):
            print(f"  Peer {peer}: {value:.3f}")
```

### Node Code (node.py)
```python
import socket
import threading
import time
from config import (
    HOST, BASE_PORT, PEER_PORTS, BUFFER_SIZE,
    FORWARD_THRESHOLD, UPDATE_INTERVAL,
    REINFORCEMENT, DECAY_FACTOR
)
from pheromone_table import PheromoneTable

pheromone_table = PheromoneTable()
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
        
        # Reinforce successful path
        pheromone_table.reinforce(peer_port, REINFORCEMENT)
        return True
    except (ConnectionRefusedError, socket.timeout):
        print(f"[NODE {BASE_PORT}] Failed to send to {peer_port}")
        return False

# Forwarding loop with pheromone decay
def forward_loop():
    while True:
        # Decay pheromones (evaporation)
        pheromone_table.decay()
        
        # Find best candidates
        candidates = pheromone_table.get_best_candidates(FORWARD_THRESHOLD)
        
        # Forward messages to best paths
        for msg in message_queue[:]:
            for peer in candidates:
                if send_message(peer, msg):
                    message_queue.remove(msg)
                    break
        
        time.sleep(UPDATE_INTERVAL)

# Server - รับข้อความ
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
    
    # Initialize pheromones for all peers
    print(f"[NODE {BASE_PORT}] Initializing pheromone table")
    for peer in PEER_PORTS:
        pheromone_table.reinforce(peer, 1.0)
    pheromone_table.print_table()
    
    # ส่งข้อความเริ่มต้น
    for peer in PEER_PORTS:
        msg = f"Hello from node {BASE_PORT}"
        if not send_message(peer, msg):
            message_queue.append(msg)
    
    # Print pheromone table periodically
    while True:
        time.sleep(10)
        pheromone_table.print_table()
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"
BASE_PORT = 10000  # เปลี่ยนสำหรับแต่ละ node
PEER_PORTS = [10001, 10002]
BUFFER_SIZE = 1024

# Pheromone parameters
INITIAL_PHEROMONE = 1.0
DECAY_FACTOR = 0.9  # ลด 10% ทุก update
REINFORCEMENT = 0.1  # เพิ่มเมื่อสำเร็จ
FORWARD_THRESHOLD = 0.2  # Forward ถ้า pheromone >= 0.2
UPDATE_INTERVAL = 5  # วินาที
```

## 🧪 Execution & Verification

### รัน 3 Nodes
```bash
# Terminal 1 - Node 10000
python node.py

# Terminal 2 - Node 10001
python node.py

# Terminal 3 - Node 10002
python node.py
```

### Expected Output

**Node 10000 (เริ่มต้น):**
```
[NODE 10000] Listening for incoming messages...
[NODE 10000] Initializing pheromone table
[PHEROMONE] 10001 reinforced: 0.00 → 1.00
[PHEROMONE] 10002 reinforced: 0.00 → 1.00
[PHEROMONE TABLE]
  Peer 10001: 1.000
  Peer 10002: 1.000
```

**หลังส่งข้อความสำเร็จ:**
```
[NODE 10000] Sent: Hello from node 10000 to 10001
[PHEROMONE] 10001 reinforced: 1.00 → 1.10
```

**หลัง Decay (ทุก 5 วินาที):**
```
[PHEROMONE TABLE]
  Peer 10001: 0.990
  Peer 10002: 0.900
```

**หลังหลายรอบ (ถ้า 10001 ส่งสำเร็จตลอด):**
```
[PHEROMONE TABLE]
  Peer 10001: 1.500  ← preferred path
  Peer 10002: 0.500  ← less preferred
```

## 📝 What I Learned

### Technical Skills
1. **Pheromone Tables** - จัดการค่า pheromone
2. **Reinforcement Logic** - เพิ่มค่าเมื่อสำเร็จ
3. **Decay Mechanism** - ลดค่าเมื่อเวลาผ่าน
4. **Adaptive Selection** - เลือก path ตามค่า pheromone

### Conceptual Understanding
- **Self-Organization**: ระบบจัดตัวเองโดยไม่ต้องควบคุมกลาง
- **Positive Feedback**: เส้นทางที่ดีได้รับ reinforcement
- **Negative Feedback**: เส้นทางที่ไม่ใช้ decay ลง
- **Emergent Behavior**: routing ที่ดีเกิดจากกฎง่ายๆ

### Bio-Inspired Routing Cycle
```
┌─────────────────┐
│  Initialize     │
│  Pheromones     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Forward to     │
│  Best Path      │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│Success│ │ Fail  │
└───┬───┘ └───┬───┘
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│Reinforce│ │ Nothing│
└────┬───┘ └────┬───┘
     │          │
     └────┬─────┘
          │
          ▼
┌─────────────────┐
│  Decay All      │
│  (Evaporation)  │
└─────────────────┘
```

## 🔧 Extensions

### Extension A: Dynamic Learning (RTT-based)
```python
class AdaptivePheromoneTable(PheromoneTable):
    def __init__(self):
        super().__init__()
        self.rtt_history = {}  # {peer: [rtt1, rtt2, ...]}
    
    def record_rtt(self, peer, rtt):
        if peer not in self.rtt_history:
            self.rtt_history[peer] = []
        self.rtt_history[peer].append(rtt)
        
        # คำนวณ reinforcement จาก RTT (ยิ่งเร็ว ยิ่งได้มาก)
        avg_rtt = sum(self.rtt_history[peer][-10:]) / len(self.rtt_history[peer][-10:])
        reinforcement = 1.0 / (1.0 + avg_rtt)  # Faster = more reinforcement
        self.reinforce(peer, reinforcement)
```

### Extension B: Multi-Hop Paths
```python
class PathPheromoneTable(PheromoneTable):
    def __init__(self):
        super().__init__()
        self.path_table = {}  # {(dest, next_hop): pheromone}
    
    def reinforce_path(self, destination, next_hop, value):
        key = (destination, next_hop)
        self.path_table[key] = self.path_table.get(key, 0) + value
    
    def get_best_next_hop(self, destination):
        candidates = [
            (next_hop, pher)
            for (dest, next_hop), pher in self.path_table.items()
            if dest == destination
        ]
        if not candidates:
            return None
        return max(candidates, key=lambda x: x[1])[0]
```

### Extension C: Logging & Visualization
```python
import csv

class PheromoneLogger:
    def __init__(self, filename="pheromone_log.csv"):
        self.filename = filename
        with open(filename, 'w', newline='') as f:
            writer = csv.writer(f)
            writer.writerow(['timestamp', 'peer', 'pheromone'])
    
    def log(self, pheromone_table):
        import time
        with open(self.filename, 'a', newline='') as f:
            writer = csv.writer(f)
            timestamp = time.time()
            for peer, value in pheromone_table.table.items():
                writer.writerow([timestamp, peer, value])
    
    def plot(self):
        import pandas as pd
        import matplotlib.pyplot as plt
        
        df = pd.read_csv(self.filename)
        for peer in df['peer'].unique():
            peer_data = df[df['peer'] == peer]
            plt.plot(peer_data['timestamp'], peer_data['pheromone'], label=f'Peer {peer}')
        
        plt.xlabel('Time')
        plt.ylabel('Pheromone Value')
        plt.legend()
        plt.savefig('pheromone_evolution.png')
```

## 🌍 Real-World Applications
- **Sensor Networks** - Adaptive routing in IoT
- **Disaster Response** - Self-healing emergency networks
- **Traffic Routing** - Dynamic path selection
- **Load Balancing** - Distribute traffic adaptively
- **Swarm Robotics** - Multi-robot coordination

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| All paths equal | No decay | Implement decay mechanism |
| One path dominates | Too much reinforcement | Reduce REINFORCEMENT value |
| Slow convergence | Decay too fast | Adjust DECAY_FACTOR closer to 1.0 |
| Oscillation | Threshold too sensitive | Increase FORWARD_THRESHOLD |

## 📖 References
- [Ant Colony Optimization](https://en.wikipedia.org/wiki/Ant_colony_optimization)
- [Bio-Inspired Networking](https://www.sciencedirect.com/topics/computer-science/bio-inspired-networking)
