# Week 3: Broadcast Communication (LAN-Level)

## 📋 Overview
Week 3 เรียนรู้เกี่ยวกับ **UDP Broadcast** การส่งข้อมูลแบบ one-to-many ไปยังทุก node ในเครือข่าย LAN

## 🎯 Learning Objectives
- เข้าใจ Broadcast Communication
- เรียนรู้ Broadcast Address
- ฝึกใช้ SO_BROADCAST socket option
- เข้าใจข้อจำกัดของ Broadcast

## 📚 Key Concepts

### Broadcast Types
| Type | Address | Scope |
|------|---------|-------|
| Limited Broadcast | 255.255.255.255 | Local network only |
| Directed Broadcast | 192.168.1.255 | Specific subnet |
| Subnet Broadcast | x.x.x.255 | Subnet specific |

### Broadcast Characteristics
- **One-to-Many**: Sender เดียว, Multiple receivers
- **No Reply**: Broadcast เป็น one-way (ไม่คาดหวังคำตอบ)
- **LAN Only**: Router ไม่ forward broadcast
- **Network Noise**: มากเกินไปอาจทำให้เครือข่ายช้า

### Network Diagram
```
                    ┌──────────────┐
                    │  Broadcaster │
                    │   (Sender)   │
                    └──────┬───────┘
                           │
              ═════════════╪══════════════
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
   ┌──────────┐     ┌──────────┐     ┌──────────┐
   │ Listener │     │ Listener │     │ Listener │
   │   Port   │     │   Port   │     │   Port   │
   │   7000   │     │   7000   │     │   7000   │
   └──────────┘     └──────────┘     └──────────┘
```

## 💻 Implementation

### Broadcaster Code (broadcaster.py)
```python
import socket
from config import BROADCAST_IP, PORT

# สร้าง UDP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# เปิดใช้งาน Broadcast
sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)

message = "DISCOVERY: Who is online?"

# ส่ง Broadcast
sock.sendto(message.encode(), (BROADCAST_IP, PORT))

print("[BROADCASTER] Message sent")
sock.close()
```

### Listener Code (listener.py)
```python
import socket
from config import PORT, BUFFER_SIZE

# สร้าง UDP Socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# Bind ทุก interface (empty string = all interfaces)
sock.bind(("", PORT))

print(f"[LISTENER] Listening for broadcast on port {PORT}")

# รอรับ broadcast
while True:
    data, addr = sock.recvfrom(BUFFER_SIZE)
    print(f"[LISTENER] From {addr}: {data.decode()}")
```

### Configuration (config.py)
```python
BROADCAST_IP = "255.255.255.255"
PORT = 7000
BUFFER_SIZE = 1024
```

## 🧪 Execution & Verification

### รัน Listener (หลาย terminal)
```bash
# Terminal 1
python listener.py

# Terminal 2
python listener.py

# Terminal 3
python listener.py
```

### รัน Broadcaster
```bash
python broadcaster.py
```

### Expected Output
**All Listeners:**
```
[LISTENER] Listening for broadcast on port 7000
[LISTENER] From ('192.168.1.100', 54321): DISCOVERY: Who is online?
```

**Broadcaster:**
```
[BROADCASTER] Message sent
```

## 📝 What I Learned

### Technical Skills
1. **SO_BROADCAST** - ต้อง enable ก่อนส่ง broadcast
2. **Binding to All Interfaces** - ใช้ `""` แทน HOST เพื่อรับทุก interface
3. **Broadcast Address** - เข้าใจการใช้ 255.255.255.255
4. **Port Selection** - เลือก port ที่เหมาะสมสำหรับ service

### Conceptual Understanding
- **Broadcast Domain**: Broadcast ทำงานได้เฉพาะในเครือข่ายเดียวกัน
- **No Routing**: Router ไม่ forward broadcast packets
- **Network Impact**: Broadcast มากเกินไป = network congestion
- **Service Discovery**: ใช้สำหรับค้นหา services ในเครือข่าย

### Broadcast Flow
```
Sender: Create socket → Set SO_BROADCAST → sendto(255.255.255.255)
        │
        ▼
    ┌───────────────────────────┐
    │   Network Switch/Hub      │
    └─────────────┬─────────────┘
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
 Receiver 1    Receiver 2    Receiver 3
 (Port 7000)   (Port 7000)   (Port 7000)
```

## 🔧 Extensions

### Extension A: Broadcast Discovery + Reply
```python
# Listener ส่ง reply กลับ
listener_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
reply = f"PRESENT: {socket.gethostname()}"
listener_socket.sendto(reply.encode(), addr)  # ส่งกลับไปยัง sender
```

### Extension B: Periodic Broadcast
```python
import time

while True:
    sock.sendto(b"HEARTBEAT", (BROADCAST_IP, PORT))
    time.sleep(5)  # ส่งทุก 5 วินาที
```

### Extension C: Subnet Broadcast
```python
# ใช้ subnet broadcast address แทน
BROADCAST_IP = "192.168.1.255"  # สำหรับเครือข่าย 192.168.1.0/24
```

## 🌍 Real-World Applications
- **DHCP Discovery** - Client ค้นหา DHCP server
- **ARP** - แปลง IP เป็น MAC address
- **Network Scanning** - ค้นหา devices ในเครือข่าย
- **Service Announcement** - Printer, file server ประกาศตัว
- **Gaming** - LAN party game discovery

## ⚠️ Common Mistakes

| Problem | Cause | Solution |
|---------|-------|----------|
| Permission denied | SO_BROADCAST ไม่ถูก set | `setsockopt(SOL_SOCKET, SO_BROADCAST, 1)` |
| No response | Firewall block | เปิด port ใน firewall |
| Across subnets | Router ไม่ forward | ใช้ multicast แทน |
| Too much traffic | Broadcast บ่อยเกินไป | ลด frequency หรือใช้ multicast |

## 📖 References
- [UDP Broadcast Tutorial](https://www.geeksforgeeks.org/udp-broadcast-python/)
- [Socket Options](https://man7.org/linux/man-pages/man7/socket.7.html)
