# OpenAI Secure MCP Tunnel สำหรับ Serena

คู่มือการตั้งค่า ChatGPT (web) ให้เข้าถึง workspace ในเครื่องผ่าน OpenAI Secure MCP Tunnel

---

## Architecture

```
ChatGPT (web)
    │
    │  HTTPS  (outbound เท่านั้น — ไม่เปิด port สาธารณะ)
    ▼
api.openai.com/v1/mcp/{tunnel_id}
    │
    │  long-poll + forward
    ▼
tunnel-client  (host, systemd user service)
    binary: ~/.local/bin/tunnel-client  v0.0.16
    service: ~/.config/systemd/user/serena-tunnel-client.service
    admin UI: http://127.0.0.1:8080/ui
    │
    │  HTTP  127.0.0.1:9122/mcp  (localhost only)
    ▼
serena-mcp  (Docker container)
    compose: docker-compose.chatgpt.yml
    port: 127.0.0.1:9122 → container:9122
    endpoint: /mcp  (MCP Streamable HTTP)
    │
    │  volume mount  read-write
    ▼
/home/sun/projects  (workspace)
```

**ทำไม tunnel-client รันบน host (ไม่ใช่ Docker)**
- ใช้ `tunnel-client doctor` debug ได้โดยตรงจาก terminal
- admin UI ที่ `http://localhost:8080/ui` เปิดใน browser ได้ทันที
- ดู log แบบ live ด้วย `journalctl --user -f`
- ไม่ต้อง publish port เพิ่ม — tunnel-client เชื่อมต่อออกไปข้างนอกเท่านั้น

**ทำไม serena-mcp อยู่ใน Docker**
- isolation จาก host — shell command ที่ agent รันไม่กระทบ host โดยตรง
- dependency (pyright, language servers) จัดการใน container
- `restart: unless-stopped` ดูแลให้เอง

---

## Prerequisites

| สิ่งที่ต้องมี | ที่ได้มา |
|---|---|
| Docker + Docker Compose v2 | ติดตั้งแล้วบนเครื่อง |
| OpenAI account (ChatGPT Plus หรือ Team/Enterprise) | https://platform.openai.com |
| Tunnel ID | https://platform.openai.com/settings/organization/tunnels (สร้างใหม่) |
| Runtime API key | https://platform.openai.com/settings/organization/api-keys (ต้องมีสิทธิ Tunnels Read + Use) |
| tunnel-client binary | ดาวน์โหลดจาก GitHub releases (ดูขั้นตอนด้านล่าง) |

**สิทธิ์ที่ต้องการ (OpenAI Platform RBAC)**

| งาน | สิทธิ์ที่ต้องการ |
|---|---|
| สร้าง / แก้ไข tunnel | Tunnels **Read** + **Manage** |
| รัน tunnel-client / สร้าง runtime API key | Tunnels **Read** + **Use** |

---

## การติดตั้งครั้งแรก

### 1. สร้าง Tunnel และ Runtime API Key

1. เปิด https://platform.openai.com/settings/organization/tunnels
2. คลิก **Create tunnel** — ตั้งชื่อที่จำได้ เช่น `home_server_workspace`
3. บันทึก `tunnel_id` (รูปแบบ `tunnel_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`)
4. เปิด https://platform.openai.com/settings/organization/api-keys
5. สร้าง API key ใหม่ที่มีสิทธิ **Tunnels Read + Use**
6. บันทึก key (แสดงครั้งเดียว — เก็บให้ดี)

### 2. ตั้งค่าไฟล์ `.env`

```bash
cd /home/sun/projects/serena
cp .env.example .env
```

แก้ค่าในไฟล์ `.env`:

```dotenv
# Runtime API key จาก Platform → API keys (ต้องมีสิทธิ Tunnels Read + Use)
CONTROL_PLANE_API_KEY=sk-proj-...

# Tunnel ID จาก Platform → Tunnels
CONTROL_PLANE_TUNNEL_ID=tunnel_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# workspace ที่จะให้ agent เข้าถึง (mount เป็น /workspace ใน container)
WORKSPACE_DIR=/home/sun/projects
```

> **ข้อควรระวัง**: ไฟล์ `.env` มี API key — ห้าม commit เข้า git  
> ตรวจสอบว่า `.env` อยู่ใน `.gitignore` แล้ว (`grep '\.env' .gitignore`)

### 3. ดาวน์โหลดและติดตั้ง tunnel-client binary

```bash
# ตรวจสอบ architecture (ควรได้ x86_64)
uname -m

# ดาวน์โหลด release ล่าสุด (linux-amd64)
# ดู version ล่าสุดได้ที่ https://github.com/openai/tunnel-client/releases
VERSION=v0.0.16
curl -fLO "https://github.com/openai/tunnel-client/releases/download/${VERSION}/tunnel-client-${VERSION}-linux-amd64.zip"

# แตกไฟล์และติดตั้ง
unzip tunnel-client-${VERSION}-linux-amd64.zip tunnel-client
mkdir -p ~/.local/bin
mv tunnel-client ~/.local/bin/tunnel-client
chmod +x ~/.local/bin/tunnel-client

# ทำความสะอาด
rm -f tunnel-client-${VERSION}-linux-amd64.zip

# ตรวจสอบ
tunnel-client --version
```

> ถ้า `tunnel-client` ไม่พบใน PATH ให้เพิ่ม `export PATH="$HOME/.local/bin:$PATH"` ใน `~/.bashrc` แล้วรัน `source ~/.bashrc`

### 4. รัน serena-mcp (Docker)

```bash
cd /home/sun/projects/serena
docker compose -f docker-compose.chatgpt.yml --env-file .env up -d

# รอให้ healthy (ประมาณ 30–120 วินาทีครั้งแรก เพราะต้องโหลด language server)
docker compose -f docker-compose.chatgpt.yml ps
```

ผลที่ควรเห็น:
```
NAME         STATUS                    PORTS
serena-mcp   Up X minutes (healthy)   127.0.0.1:9122->9122/tcp
```

ตรวจสอบ endpoint โดยตรง:
```bash
curl -X POST http://localhost:9122/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
# ควรได้ HTTP 200 และ event stream กลับมา
```

### 5. ตั้งค่า tunnel-client เป็น systemd user service

```bash
# ติดตั้ง service file
mkdir -p ~/.config/systemd/user
cp /home/sun/projects/serena/serena-tunnel-client.service ~/.config/systemd/user/

# โหลด และเปิดใช้งาน
systemctl --user daemon-reload
systemctl --user enable serena-tunnel-client.service
systemctl --user start serena-tunnel-client.service

# เปิด lingering เพื่อให้ service รันต่อแม้ไม่ได้ login
loginctl enable-linger $USER
```

ตรวจสอบว่าทำงาน:
```bash
systemctl --user status serena-tunnel-client.service
# ควรเห็น Active: active (running)

curl http://127.0.0.1:8080/healthz   # → live
curl http://127.0.0.1:8080/readyz    # → ready
```

### 6. เชื่อมต่อกับ ChatGPT

1. เปิด ChatGPT → เมนู Plugins → **Add custom MCP server**
2. เลือก **Connection: Tunnel**
3. เลือก tunnel ที่สร้างไว้ จากรายการ (หรือวาง `tunnel_id` โดยตรง)
4. ตั้งค่า authentication ตามที่ ChatGPT ถาม
5. คลิก **I understand and want to continue** → **Create as a plugin**

> tunnel-client ต้องรันอยู่ก่อน ถ้า tunnel ไม่ขึ้นในรายการให้ตรวจสอบว่า tunnel associate กับ ChatGPT workspace แล้ว (ดู [Troubleshooting](#troubleshooting))

---

## การใช้งานประจำวัน

### ดูสถานะ

```bash
# สถานะ tunnel-client (systemd)
systemctl --user status serena-tunnel-client.service

# ดู log live
journalctl --user -u serena-tunnel-client.service -f

# health endpoint
curl http://127.0.0.1:8080/healthz   # live = ทำงานอยู่
curl http://127.0.0.1:8080/readyz    # ready = พร้อมรับ request

# admin UI (เปิดใน browser)
xdg-open http://127.0.0.1:8080/ui

# สถานะ serena-mcp (Docker)
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml ps
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml logs -f serena-mcp
```

### หยุด / เริ่มใหม่

```bash
# tunnel-client
systemctl --user stop serena-tunnel-client.service
systemctl --user start serena-tunnel-client.service
systemctl --user restart serena-tunnel-client.service

# serena-mcp (Docker)
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml down
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml --env-file .env up -d
```

### ดู log ย้อนหลัง

```bash
# 50 บรรทัดล่าสุด
journalctl --user -u serena-tunnel-client.service -n 50 --no-pager

# ช่วงเวลาเฉพาะ
journalctl --user -u serena-tunnel-client.service --since "1 hour ago"

# log ของ serena-mcp
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml logs --tail 50 serena-mcp
```

---

## Troubleshooting

### tunnel-client ไม่ start / crash ทันที

```bash
# ดู log เพื่อหาสาเหตุ
journalctl --user -u serena-tunnel-client.service -n 30 --no-pager
```

สาเหตุที่พบบ่อย:
- **API key ผิด / หมดอายุ** — เห็น `401 Unauthorized` ใน log → สร้าง key ใหม่ใน Platform
- **Tunnel ID ผิด** — เห็น `tunnel not found` → ตรวจค่า `CONTROL_PLANE_TUNNEL_ID` ใน `.env`
- **serena-mcp ยังไม่พร้อม** — tunnel-client จะ retry เอง รอสักครู่

### tunnel ไม่ขึ้นใน ChatGPT

1. ตรวจว่า tunnel associate กับ **ChatGPT workspace** ด้วย (ไม่ใช่แค่ Platform organization):
   - Platform → Tunnels → เลือก tunnel → แก้ไข → เพิ่ม workspace
2. ตรวจว่าผู้ใช้ที่สร้าง plugin มีสิทธิ **Tunnels Read + Use**
3. รอ 30 นาทีหลังเพิ่มสิทธิ์ใหม่ (propagation time)
4. ตรวจ readiness: `curl http://127.0.0.1:8080/readyz` ต้องได้ `ready`

### serena-mcp unhealthy

```bash
# ดู log ของ container
docker logs serena-mcp --tail 30

# ทดสอบ endpoint โดยตรง
curl -X POST http://localhost:9122/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'

# restart
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml restart serena-mcp
```

### debug ด้วย tunnel-client doctor

```bash
# สร้าง profile ก่อน (ครั้งแรก)
tunnel-client init \
  --profile serena-chatgpt \
  --tunnel-id tunnel_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  --mcp-server-url http://localhost:9122/mcp

# รัน doctor
tunnel-client doctor --profile serena-chatgpt --explain
```

### Port 9122 ถูกยึด

```bash
# ดูว่าใครยึด
ss -tlnp | grep 9122

# ถ้าไม่ใช่ serena-mcp ให้เปลี่ยน port ใน docker-compose.chatgpt.yml และ service file
# แล้วรัน: systemctl --user daemon-reload && systemctl --user restart serena-tunnel-client.service
```

---

## ไฟล์ที่เกี่ยวข้อง

| ไฟล์ | ที่อยู่ | คำอธิบาย |
|---|---|---|
| `docker-compose.chatgpt.yml` | repo root | compose สำหรับ serena-mcp (port 9122) |
| `.env` | repo root | API key + Tunnel ID (ห้าม commit) |
| `.env.example` | repo root | template สำหรับ .env |
| `serena-tunnel-client.service` | repo root | systemd service template |
| `~/.config/systemd/user/serena-tunnel-client.service` | host | service ที่ติดตั้งจริง |
| `~/.local/bin/tunnel-client` | host | tunnel-client binary v0.0.16 |

---

## ความปลอดภัย

- **Port 9122 bind `127.0.0.1` เท่านั้น** — เครื่องอื่นใน LAN เข้าไม่ถึง serena-mcp โดยตรง
- **tunnel-client ไม่เปิด inbound port** — ติดต่อออกไป `api.openai.com:443` เท่านั้น
- **`.env` ไม่ commit** — อยู่ใน `.gitignore` แล้ว
- **Workspace ที่ mount เข้า container** — ChatGPT มีสิทธิ์อ่าน/เขียน/ลบไฟล์ได้ทุกอย่างใน `WORKSPACE_DIR` ควรกำหนดให้แคบที่สุดที่ต้องการ
- **ปิด tunnel เมื่อไม่ใช้** — `systemctl --user stop serena-tunnel-client.service` หรือลบ plugin ออกจาก ChatGPT

### จำกัด workspace ให้แคบลง (แนะนำ)

แก้ค่าใน `.env`:
```dotenv
# จำกัดเฉพาะโปรเจกต์ที่ต้องการแทนทุกโปรเจกต์
WORKSPACE_DIR=/home/sun/projects/serena
```

### ปิด shell command ถ้าไม่ต้องการ

ใน `.serena/project.yml` (สร้างถ้ายังไม่มี):
```yaml
read_only: true
# หรือเลือก disable เฉพาะ tool
excluded_tools:
  - execute_shell_command
```

---

## อัปเกรด tunnel-client

```bash
# หยุด service ก่อน
systemctl --user stop serena-tunnel-client.service

# ดาวน์โหลด version ใหม่
VERSION=vX.Y.Z   # เปลี่ยนเป็น version ล่าสุด
curl -fLO "https://github.com/openai/tunnel-client/releases/download/${VERSION}/tunnel-client-${VERSION}-linux-amd64.zip"
unzip -o tunnel-client-${VERSION}-linux-amd64.zip tunnel-client
mv tunnel-client ~/.local/bin/tunnel-client
chmod +x ~/.local/bin/tunnel-client
rm tunnel-client-${VERSION}-linux-amd64.zip

# เริ่มใหม่
systemctl --user start serena-tunnel-client.service
tunnel-client --version
```

---

## อ้างอิง

- [OpenAI Secure MCP Tunnel Guide](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
- [tunnel-client GitHub Repository](https://github.com/openai/tunnel-client)
- [tunnel-client Deployment Docs](https://github.com/openai/tunnel-client/blob/master/docs/deployment/overview.md)
- [Serena MCP Documentation](https://oraios.github.io/serena/02-usage/020_running.html)
- [Platform Tunnel Settings](https://platform.openai.com/settings/organization/tunnels)
