# Week 1: TCP Client-Server Communication (Unicast)

## 📋 Overview
Week 1 เป็นพื้นฐานของการเขียนโปรแกรมเครือข่าย โดยเรียนรู้เกี่ยวกับ **TCP Client-Server Communication** ซึ่งเป็นรูปแบบการสื่อสารที่พื้นฐานที่สุด

## 🎯 Learning Objectives
- เข้าใจหลักการทำงานของ TCP (Transmission Control Protocol)
- เรียนรู้ Client-Server Architecture
- ฝึกใช้ Socket Programming ด้วย Python
- เข้าใจเรื่อง Protocol Discipline

## 📚 Key Concepts

### TCP Socket Functions
| Function | Description |
|----------|-------------|
| `socket()` | สร้าง socket ใหม่ |
| `bind()` | ผูก socket กับ address และ port |
| `listen()` | ตั้งค่า socket ให้รอรับ connection |
| `accept()` | ยอมรับ connection จาก client |
| `connect()` | เชื่อมต่อไปยัง server |
| `send()` / `sendall()` | ส่งข้อมูล |
| `recv()` | รับข้อมูล |
| `close()` | ปิด connection |

### TCP Communication Flow
```
Server: socket() → bind() → listen() → accept() → recv() → send() → close()
Client: socket() → connect() → send() → recv() → close()
```

## 💻 Implementation

### Server Code (server.py)
```python
import socket
from config import HOST, PORT, BUFFER_SIZE

# สร้าง TCP Socket
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Bind socket กับ address
server_socket.bind((HOST, PORT))

# เริ่ม listen
server_socket.listen(1)
print(f"[SERVER] Listening on {HOST}:{PORT}")

# รับ connection จาก client
conn, addr = server_socket.accept()
print(f"[SERVER] Connection from {addr}")

# รับข้อมูลจาก client
data = conn.recv(BUFFER_SIZE)
message = data.decode()
print(f"[SERVER] Received: {message}")

# ส่งคำตอบกลับ
reply = f"ACK: {message}"
conn.sendall(reply.encode())

# ปิด connection
conn.close()
server_socket.close()
print("[SERVER] Closed connection")
```

### Client Code (client.py)
```python
import socket
from config import HOST, PORT, BUFFER_SIZE

# สร้าง TCP Socket
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# เชื่อมต่อไปยัง server
client_socket.connect((HOST, PORT))

# ส่งข้อมูล
message = "Hello Server"
client_socket.sendall(message.encode())

# รับคำตอบจาก server
response = client_socket.recv(BUFFER_SIZE)
print(f"[CLIENT] Received: {response.decode()}")

# ปิด connection
client_socket.close()
```

### Configuration (config.py)
```python
HOST = "127.0.0.1"  # localhost
PORT = 5000
BUFFER_SIZE = 1024
```

## 🧪 Execution & Verification

### รัน Server
```bash
python server.py
```

### รัน Client (ใน terminal ใหม่)
```bash
python client.py
```

### Expected Output
**Server:**
```
[SERVER] Listening on 127.0.0.1:5000
[SERVER] Connection from ('127.0.0.1', 12345)
[SERVER] Received: Hello Server
[SERVER] Closed connection
```

**Client:**
```
[CLIENT] Received: ACK: Hello Server
```

## 📝 What I Learned

### Technical Skills
1. **TCP Handshake** - เข้าใจกระบวนการ 3-way handshake ก่อนส่งข้อมูล
2. **Blocking I/O** - เรียนรู้ว่า `recv()` จะ block จนกว่าจะได้รับข้อมูล
3. **Connection Management** - การจัดการ connection อย่างถูกต้อง (open/close)
4. **Protocol Discipline** - TCP ต้องทำตามลำดับ: listen → accept → connect

### Conceptual Understanding
- **Reliability**: TCP รับประกันว่าข้อมูลจะถึงและเรียงลำดับถูกต้อง
- **Connection-Oriented**: ต้องสร้าง connection ก่อนส่งข้อมูล
- **Stream-based**: ข้อมูลเป็น stream ของ bytes

### Common Mistakes & Solutions
| Problem | Cause | Solution |
|---------|-------|----------|
| Port already in use | Server ยังไม่ปิด socket | ใช้ `SO_REUSEADDR` หรือรอ OS release port |
| recv() blocks forever | Client ไม่ส่งข้อมูล | ตั้ง timeout หรือตรวจสอบ logic |
| Connection refused | Server ไม่ listen | ตรวจสอบว่า server รันก่อน client |

## 🔧 Extensions

### Extension A: Multiple Clients
```python
# Server รับ multiple clients แบบ sequential
while True:
    conn, addr = server_socket.accept()
    data = conn.recv(BUFFER_SIZE)
    print(f"[SERVER] From {addr}: {data.decode()}")
    conn.sendall(b"ACK")
    conn.close()
```

### Extension B: Message Validation
```python
if not data:
    conn.sendall(b"ERROR: Empty message")
elif len(data) > BUFFER_SIZE:
    conn.sendall(b"ERROR: Message too long")
else:
    conn.sendall(f"ACK: {message}".encode())
```

### Extension C: Timeout Handling
```python
server_socket.settimeout(30)  # Timeout หลัง 30 วินาที
try:
    conn, addr = server_socket.accept()
except socket.timeout:
    print("[SERVER] Timeout waiting for client")
```

## 🌍 Real-World Applications
- **Web Servers** (HTTP/HTTPS)
- **Email Servers** (SMTP, IMAP)
- **File Transfer** (FTP)
- **Database Connections**

## 📖 References
- [Python Socket Documentation](https://docs.python.org/3/library/socket.html)
- [TCP/IP Guide](http://www.tcpipguide.com/)
