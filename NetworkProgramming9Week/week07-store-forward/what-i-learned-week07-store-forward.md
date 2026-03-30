# Week 7: Store-and-Forward Communication

## 📋 Overview
Week 7 เรียนรู้เกี่ยวกับ **Store-and-Forward** เทคนิคการส่งข้อความที่เก็บข้อความไว้เมื่อ peer ไม่พร้อม และส่งต่อเมื่อ peer พร้อม

## 🎯 Learning Objectives
- เข้าใจ Store-and-Forward Concept
- เรียนรู้ Message Queuing
- ฝึกใช้ Retry Logic
- เข้าใจ Delay-Tolerant Networking (DTN)

## 📚 Key Concepts

### Store-and-Forward Model
```
Traditional (Real-time):
    Sender ──────► Receiver
    (must be online simultaneously)

Store-and-Forward:
    Sender ──► Queue ──► Receiver
    (can be offline at different times)
```

### Key Components
| Component | Description |
|-----------|-------------|
| Message Queue | เก็บข้อความที่ยังส่งไม่ได้ |
| Link Detection | ตรวจสอบว่า peer พร้อมหรือไม่ |
| Retry Logic | พยายามส่งซ้ำตามช่วงเวลา |
| Backoff Strategy | ลดความถี่เมื่อล้มเหลวต่อเนื่อง |

### Use Cases
- **Intermittent Connectivity**: เครือข่ายไม่เสถียร
- **Delay-Tolerant**: ไม่ต้องส่งทันที
- **Offline Operations**: Peer offline บ่อย
- **Long-Distance**: เช่น satellite communication

## 💻 Implementation

### Message Queue (message_queue.py)
```python
import time
from collections import deque

class MessageQueue:
    def __init__(self):
        self.queue = deque()
    
    def add_message(self, message, peer_port):
        """เพิ่มข้อความเข้า queue"""
        self.queue.append({
            "message": message,
            "peer": peer_port,
            "timestamp": time.time()
        })
    
    def get_messages(self):
        """ดึงรายการข้อความทั้งหมด"""
        return list(self.queue)
    
    def remove_message(self, msg):
        """ลบข้อความที่ส่งสำเร็จแล้ว"""
        self.queue.remove(msg)
    
    def is_empty(self):
        return len(self.queue) == 0
```

### Node Code (node.py)
```python
import socket
import threading
import time
from config import HOST, BASE_PORT, PEER_PORTS, BUFFER_SIZE, RETRY_INTERVAL
from message_queue import MessageQueue

queue = MessageQueue()

# ส่งข้อความ
def send_message(peer_port, message):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(2)
        s.connect((HOST, peer_port))
        s.sendall(message.encode())
        s.close()
        return True
    except (ConnectionRefusedError, socket.timeout):
        return False

# Forward loop - ตรวจสอบและส่งข้อความใน queue
def forward_loop():
    while True:
        for msg in queue.get_messages():
            success = send_message(msg["peer"], msg["message"])
            if success:
                print(f"[NODE {BASE_PORT}] Sent stored message to {msg['peer']}")
                queue.remove_message(msg)
        time.sleep(RETRY_INTERVAL)

# Server - รับข้อความ
def start_server():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.bind((HOST, BASE_PORT))
    server.listen()
    print(f"[NODE {BASE_PORT}] Listening for messages...")
    while True:
        conn, addr = server.accept()
        data = conn.recv(BUFFER_SIZE).decode()
        print(f"[NODE {BASE_PORT}] Received: {data} from {addr}")
        conn.close()

# Main
if __name__ == "__main__":
    threading.Thread(target=start_server, daemon=True).start()
    threading.Thread(target=forward_loop, daemon=True).start()
    
    # ส่งข้อความเริ่มต้น
    for peer in PEER_PORTS:
        msg = f"Hello from node {BASE_PORT}"
        if not send_message(peer, msg):
            print(f"[NODE {BASE_PORT}] Peer {peer} unavailable, storing message")
            queue.add_message(msg, peer)
    
    while True:
        time.sleep(1)
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"
BASE_PORT = 8000  # เปลี่ยนสำหรับแต่ละ node
PEER_PORTS = [8001, 8002]  # Ports ของ peers
BUFFER_SIZE = 1024
RETRY_INTERVAL = 5  # วินาทีระหว่าง retry
```

## 🧪 Execution & Verification

### Test Scenario: Peer Offline แล้ว Online

```bash
# Terminal 1 - Node 8000
python node.py

# Terminal 2 - Node 8001 (ยังไม่รัน)

# Terminal 3 - Node 8002
python node.py
```

### Expected Output Sequence

**Node 8000 (เริ่มต้น):**
```
[NODE 8000] Listening for messages...
[NODE 8000] Peer 8001 unavailable, storing message
[NODE 8000] Sent stored message to 8002
```

**หลังจาก 5 วินาที (retry):**
```
[NODE 8000] Sent stored message to 8001
```

**รัน Node 8001 (หลังจาก Node 8000 เก็บข้อความแล้ว):**
```bash
# Terminal 2
python node.py
```

**Node 8000 จะเห็น:**
```
[NODE 8000] Sent stored message to 8001
```

## 📝 What I Learned

### Technical Skills
1. **Queue Management** - ใช้ deque สำหรับ message queue
2. **Retry Logic** - พยายามส่งซ้ำตาม interval
3. **Timeout Handling** - ตั้ง timeout สำหรับ connection
4. **Background Threads** - รัน forward loop พื้นหลัง

### Conceptual Understanding
- **Asynchronous Communication**: Sender และ receiver ไม่ต้อง online พร้อมกัน
- **Persistence**: ข้อความไม่หายแม้ peer offline
- **Eventual Delivery**: ข้อความจะส่งได้เมื่อ peer พร้อม
- **Resource Tradeoff**: ใช้ memory เก็บข้อความ vs reliability

### Store-and-Forward Flow
```
┌─────────────┐
│   Sender    │
│  (Node A)   │
└──────┬──────┘
       │ Send to B
       ▼
┌─────────────┐     ┌─────────────┐
│  Peer B     │     │   Queue     │
│  (Offline)  │     │  [msg for B]│
└─────────────┘     └──────┬──────┘
       │                   │ Retry every 5s
       │                   │
       ▼                   ▼
┌─────────────┐     ┌─────────────┐
│  Peer B     │     │   Queue     │
│  (Online)   │     │   [empty]   │
└─────────────┘     └─────────────┘
       ▲
       │ Message delivered
```

## 🔧 Extensions

### Extension A: Persistent Storage
```python
import json

class PersistentQueue(MessageQueue):
    def __init__(self, filename="queue.json"):
        super().__init__()
        self.filename = filename
        self.load()
    
    def save(self):
        with open(self.filename, 'w') as f:
            json.dump(list(self.queue), f)
    
    def load(self):
        try:
            with open(self.filename, 'r') as f:
                self.queue = deque(json.load(f))
        except FileNotFoundError:
            pass
    
    def add_message(self, message, peer_port):
        super().add_message(message, peer_port)
        self.save()
```

### Extension B: Exponential Backoff
```python
class RetryWithBackoff:
    def __init__(self):
        self.retry_counts = {}
        self.base_interval = 5
    
    def get_interval(self, peer_port):
        count = self.retry_counts.get(peer_port, 0)
        return self.base_interval * (2 ** count)  # 5, 10, 20, 40...
    
    def on_success(self, peer_port):
        self.retry_counts[peer_port] = 0
    
    def on_failure(self, peer_port):
        self.retry_counts[peer_port] = self.retry_counts.get(peer_port, 0) + 1
```

### Extension C: Priority Queues
```python
class PriorityMessageQueue:
    def __init__(self):
        self.high_priority = deque()
        self.normal_priority = deque()
    
    def add_message(self, message, peer_port, priority="normal"):
        msg_obj = {"message": message, "peer": peer_port}
        if priority == "high":
            self.high_priority.append(msg_obj)
        else:
            self.normal_priority.append(msg_obj)
    
    def get_next_message(self):
        if self.high_priority:
            return self.high_priority.popleft()
        elif self.normal_priority:
            return self.normal_priority.popleft()
        return None
```

## 🌍 Real-World Applications
- **Email Systems** (SMTP) - Store and forward emails
- **Deep Space Communication** - Mars-Earth messages
- **Disaster Networks** - เมื่อ infrastructure พัง
- **Rural IoT** - เครือข่ายไม่เสถียร
- **Messaging Apps** - WhatsApp, Telegram (offline messages)

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| Duplicate delivery | Not removing sent messages | Remove from queue after success |
| Queue grows infinitely | Never delivering | Implement TTL or expiry |
| Network flooding | Retry too frequent | Use exponential backoff |
| Blocking I/O | Can't receive while sending | Use threads or async |

## 📖 References
- [Delay-Tolerant Networking](https://www.dtntutorial.org/)
- [RFC 4838 - DTN Architecture](https://datatracker.ietf.org/doc/html/rfc4838)
