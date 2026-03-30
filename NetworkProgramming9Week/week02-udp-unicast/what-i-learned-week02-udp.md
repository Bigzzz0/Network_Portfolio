# Week 2: UDP Communication (Unicast)

## 📋 Overview
Week 2 เรียนรู้เกี่ยวกับ **UDP (User Datagram Protocol)** ซึ่งแตกต่างจาก TCP โดยไม่มี handshake และไม่มีประกันความน่าเชื่อถือ

## 🎯 Learning Objectives
- เข้าใจความแตกต่างระหว่าง TCP และ UDP
- เรียนรู้ Connectionless Communication
- ฝึกใช้ UDP Sockets
- เข้าใจเรื่อง Reliability Tradeoffs

## 📚 Key Concepts

### TCP vs UDP
| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | Connection-oriented | Connectionless |
| Reliability | Guaranteed | No guarantee |
| Ordering | Ordered | No ordering |
| Speed | Slower | Faster |
| Overhead | High | Low |
| Use Case | Web, Email | Streaming, Gaming |

### UDP Socket Functions
| Function | Description |
|----------|-------------|
| `socket(AF_INET, SOCK_DGRAM)` | สร้าง UDP socket |
| `sendto(data, address)` | ส่ง datagram ไปยัง address |
| `recvfrom(buffer)` | รับ datagram (return data + address) |
| `bind()` | ผูก socket กับ port |

## 💻 Implementation

### Receiver Code (receiver.py)
```python
import socket
from config import HOST, PORT, BUFFER_SIZE

# สร้าง UDP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Bind socket
sock.bind((HOST, PORT))

print(f"[RECEIVER] Listening on {HOST}:{PORT}")

# รับข้อมูล (ไม่จำกัด sender)
while True:
    data, addr = sock.recvfrom(BUFFER_SIZE)
    print(f"[RECEIVER] From {addr}: {data.decode()}")
```

### Sender Code (sender.py)
```python
import socket
from config import HOST, PORT

# สร้าง UDP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

message = "Hello via UDP"

# ส่งข้อมูล (ไม่ต้อง connect)
sock.sendto(message.encode(), (HOST, PORT))

print("[SENDER] Message sent")
sock.close()
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"
PORT = 6000
BUFFER_SIZE = 1024
```

## 🧪 Execution & Verification

### รัน Receiver
```bash
python receiver.py
```

### รัน Sender (ใน terminal ใหม่)
```bash
python sender.py
```

### Expected Output
**Receiver:**
```
[RECEIVER] Listening on 127.0.0.1:6000
[RECEIVER] From ('127.0.0.1', 54321): Hello via UDP
```

**Sender:**
```
[SENDER] Message sent
```

### Test Packet Loss
```bash
# รัน sender หลายครั้งโดยไม่มี receiver
python sender.py
python sender.py
python sender.py
# ไม่เกิด error เพราะ UDP ไม่สนใจว่า receiver มีหรือไม่
```

## 📝 What I Learned

### Technical Skills
1. **Connectionless Communication** - ส่งข้อมูลโดยไม่ต้องสร้าง connection
2. **Datagram Handling** - ข้อมูลส่งเป็น packet แยกกัน
3. **Address Handling** - `recvfrom()` return ทั้ง data และ sender address
4. **No State** - Server ไม่จำ client

### Conceptual Understanding
- **Best Effort Delivery**: UDP ไม่รับประกันว่าข้อมูลจะถึง
- **No Ordering**: Packet อาจถึงไม่เรียงลำดับ
- **No Congestion Control**: ส่งได้ทันทีโดยไม่รอ
- **Application Responsibility**: Application ต้องจัดการ reliability เอง

### UDP Characteristics
```
┌─────────────┐
│   Sender    │
│  sendto()   │
└──────┬──────┘
       │  Datagram 1
       │  Datagram 2
       │  Datagram 3 (may be lost)
       │  Datagram 4 (may arrive out of order)
       ▼
┌─────────────┐
│  Receiver   │
│  recvfrom() │
└─────────────┘
```

## 🔧 Extensions

### Extension A: Sequence Numbers
```python
# Sender เพิ่ม sequence number
seq_num = 0
for i in range(10):
    message = f"{seq_num}:Data{i}"
    sock.sendto(message.encode(), (HOST, PORT))
    seq_num += 1

# Receiver ตรวจสอบ sequence
expected_seq = 0
data, addr = sock.recvfrom(BUFFER_SIZE)
seq, content = data.decode().split(':')
if int(seq) != expected_seq:
    print(f"Packet loss detected! Expected {expected_seq}, got {seq}")
```

### Extension B: Manual ACK
```python
# Receiver ส่ง ACK
sock.sendto(b"ACK", addr)

# Sender รอ ACK
sock.settimeout(2)
try:
    ack, _ = sock.recvfrom(BUFFER_SIZE)
    print("Received ACK")
except socket.timeout:
    print("Timeout, retrying...")
```

### Extension C: Rate Control
```python
import time

for i in range(100):
    sock.sendto(f"Packet {i}".encode(), (HOST, PORT))
    time.sleep(0.01)  # Delay 10ms ระหว่าง packet
```

## 🌍 Real-World Applications
- **DNS Queries** - Fast, small requests
- **Video Streaming** - Tolerate some loss for speed
- **Online Gaming** - Low latency over reliability
- **VoIP** - Real-time voice communication
- **IoT Telemetry** - Sensor data streaming

## 📖 References
- [UDP Wikipedia](https://en.wikipedia.org/wiki/User_Datagram_Protocol)
- [Python Socket HOWTO](https://docs.python.org/3/howto/sockets.html)
