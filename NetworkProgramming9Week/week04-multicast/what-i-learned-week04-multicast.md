# Week 4: Multicast Communication

## 📋 Overview
Week 4 เรียนรู้เกี่ยวกับ **IP Multicast** การส่งข้อมูลแบบ selective one-to-many โดยผู้รับต้อง join group ก่อน

## 🎯 Learning Objectives
- เข้าใจความแตกต่างระหว่าง Broadcast และ Multicast
- เรียนรู้ Multicast Group Membership
- ฝึกใช้ Multicast Sockets
- เข้าใจเรื่อง TTL และ Scoping

## 📚 Key Concepts

### Multicast Address Range
| Range | Description |
|-------|-------------|
| 224.0.0.0 - 224.0.0.255 | Local network control (TTL=1) |
| 224.0.1.0 - 238.255.255.255 | Globally scoped |
| 239.0.0.0 - 239.255.255.255 | Administratively scoped (local org) |

### Comparison: Unicast vs Broadcast vs Multicast
```
Unicast:     Sender ──────> Receiver 1
             (one-to-one)

Broadcast:   Sender ──┬───> Receiver 1
                       ├───> Receiver 2
                       └───> Receiver 3
             (one-to-all, no choice)

Multicast:   Sender ──┬───> Receiver 1 (joined)
                       └───> Receiver 3 (joined)
                       X───> Receiver 2 (not joined)
             (one-to-many, selective)
```

### Key Multicast Concepts
- **Group Membership**: Receiver ต้อง join group ด้วย `IP_ADD_MEMBERSHIP`
- **TTL (Time-To-Live)**: ควบคุม scope ของ packet
- **No Guarantee**: เช่น UDP ไม่รับประกัน delivery

## 💻 Implementation

### Receiver Code (receiver.py)
```python
import socket
import struct
from config import MULTICAST_GROUP, PORT, BUFFER_SIZE

# สร้าง UDP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

# Bind socket
sock.bind(("", PORT))

# Join multicast group
mreq = struct.pack("4sl", socket.inet_aton(MULTICAST_GROUP), socket.INADDR_ANY)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)

print(f"[RECEIVER] Joined {MULTICAST_GROUP}:{PORT}")

# รอรับข้อมูล
while True:
    data, addr = sock.recvfrom(BUFFER_SIZE)
    print(f"[RECEIVER] {addr} -> {data.decode()}")
```

### Sender Code (sender.py)
```python
import socket
from config import MULTICAST_GROUP, PORT, TTL

# สร้าง UDP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)

# ตั้งค่า TTL
sock.setsockopt(socket.IPPROTO_IP, socket.IP_MULTICAST_TTL, TTL)

message = "MULTICAST: Hello subscribers"

# ส่ง multicast
sock.sendto(message.encode(), (MULTICAST_GROUP, PORT))

print("[SENDER] Multicast sent")
sock.close()
```

### Configuration (config.py)
```python
MULTICAST_GROUP = "224.1.1.1"
PORT = 8000
BUFFER_SIZE = 1024
TTL = 1  # Local network only
```

## 🧪 Execution & Verification

### รัน Receivers (หลาย terminal)
```bash
# Terminal 1
python receiver.py

# Terminal 2
python receiver.py
```

### รัน Sender
```bash
python sender.py
```

### Expected Output
**Receivers:**
```
[RECEIVER] Joined 224.1.1.1:8000
[RECEIVER] ('192.168.1.100', 54321) -> MULTICAST: Hello subscribers
```

**Sender:**
```
[SENDER] Multicast sent
```

### Test: Non-Member Does Not Receive
```bash
# รันโปรแกรมที่ไม่ join group
# จะไม่ได้รับข้อความ multicast
```

## 📝 What I Learned

### Technical Skills
1. **Group Membership** - ใช้ `IP_ADD_MEMBERSHIP` join group
2. **TTL Configuration** - ควบคุม scope ด้วย `IP_MULTICAST_TTL`
3. **Multicast Address** - เลือก address ที่เหมาะสม
4. **struct.pack** - Pack address สำหรับ membership

### Conceptual Understanding
- **Selective Delivery**: เฉพาะสมาชิก group เท่านั้นที่ได้รับ
- **Scalability**: คุ้มค่ากว่า broadcast เมื่อมีผู้รับจำนวนมาก
- **No Membership Protocol**: Sender ไม่รู้ว่าใครเป็นสมาชิก
- **TTL Scoping**:
  - TTL=1: Local network only
  - TTL>1: Can cross routers

### Multicast Architecture
```
                    ┌─────────────┐
                    │   Sender    │
                    │  sendto()   │
                    └──────┬──────┘
                           │  Multicast Packet
                           │  Group: 224.1.1.1
                           ▼
                    ┌─────────────┐
                    │   Router    │
                    │  (if TTL>1) │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
   ┌──────────┐     ┌──────────┐     ┌──────────┐
   │ Joined   │     │ Joined   │     │   Not    │
   │ Member   │     │ Member   │     │  Member  │
   └──────────┘     └──────────┘     └──────────┘
        ✓                ✓                ✗
```

## 🔧 Extensions

### Extension A: Periodic Multicast Stream
```python
import time

for i in range(10):
    message = f"Update #{i}: {time.time()}"
    sock.sendto(message.encode(), (MULTICAST_GROUP, PORT))
    time.sleep(1)
```

### Extension B: Multiple Groups/Channels
```python
# Join multiple groups
groups = ["224.1.1.1", "224.1.1.2", "224.1.1.3"]
for group in groups:
    mreq = struct.pack("4sl", socket.inet_aton(group), socket.INADDR_ANY)
    sock.setsockopt(socket.IPPROTO_IP, socket.IP_ADD_MEMBERSHIP, mreq)
```

### Extension C: Dynamic Join/Leave
```python
# Leave group
mreq = struct.pack("4sl", socket.inet_aton(MULTICAST_GROUP), socket.INADDR_ANY)
sock.setsockopt(socket.IPPROTO_IP, socket.IP_DROP_MEMBERSHIP, mreq)
print(f"[RECEIVER] Left {MULTICAST_GROUP}")
```

## 🌍 Real-World Applications
- **IPTV/Video Streaming** - Live TV channels
- **Financial Data** - Stock market feeds
- **Multiplayer Games** - Game state distribution
- **Conference Systems** - Video conferencing
- **Software Distribution** - Update notifications

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| No data received | Not joined group | Call `IP_ADD_MEMBERSHIP` before recv |
| Wrong address | Using invalid multicast IP | Use 224.0.0.0 - 239.255.255.255 |
| TTL too high | Packet goes beyond intended scope | Set TTL=1 for local |
| Firewall blocked | Multicast traffic blocked | Configure firewall rules |

## 📖 References
- [IP Multicast Technical Overview](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipconfig_mcast/icm-np-21/icm-np-21-book.pdf)
- [Python Multicast Example](https://pymotw.com/3/socket/multicast.html)
