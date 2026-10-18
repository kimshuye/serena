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
tunnel-client  (Docker, network_mode: host)
    compose: docker-compose.chatgpt.yml
    container: serena-tunnel-client
    admin UI: http://127.0.0.1:8080/ui
    │
    │  HTTP  localhost:9122/mcp
    ▼
serena-mcp  (host, systemd user service)
    binary: ~/.local/bin/serena  v2.0.1.dev0
    service: ~/.config/systemd/user/serena-mcp.service
    port: 0.0.0.0:9122
    endpoint: /mcp  (MCP Streamable HTTP)
    config: /home/sun/projects/serena/config/serena_config.yml
    │
    │  filesystem access โดยตรง (ไม่ผ่าน volume mount)
    ▼
/home/sun/projects  (workspace)
```

**ทำไม serena-mcp รันบน host (ไม่ใช่ Docker)**
- เข้าถึง filesystem ได้โดยตรง ไม่ต้องจัดการ volume mount
- language server (LSP) รันในสภาพแวดล้อม host จริง — ตรงกับที่ developer ใช้งาน
- debug ง่ายกว่า ดู log ด้วย `journalctl --user -f`
- อัพเดท serena ด้วย `uv tool upgrade serena-agent` ได้ทันที โดยไม่ต้อง rebuild image

**ทำไม tunnel-client อยู่ใน Docker**
- binary ออกแบบมาสำหรับ container — ไม่มี native package manager install
- `restart: unless-stopped` ดูแลให้เอง หลัง reboot Docker ก็ start ได้เอง
- isolate credential จาก host shell

> **หมายเหตุ port**: port 9121 ถูกใช้โดย `serena-serena-1` (compose.yaml สำหรับ IDE clients)  
> serena-mcp สำหรับ ChatGPT จึงใช้ port **9122** แทน

---

## Prerequisites

| สิ่งที่ต้องมี | ที่ได้มา |
|---|---|
| Docker + Docker Compose v2 | ติดตั้งแล้วบนเครื่อง |
| `uv` | https://astral.sh/uv (ดูขั้นตอนด้านล่าง) |
| OpenAI account (ChatGPT Plus หรือ Team/Enterprise) | https://platform.openai.com |
| Tunnel ID | https://platform.openai.com/settings/organization/tunnels (สร้างใหม่) |
| Runtime API key | https://platform.openai.com/settings/organization/api-keys (ต้องมีสิทธิ Tunnels Read + Use) |

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
```

> **ข้อควรระวัง**: ไฟล์ `.env` มี API key — ห้าม commit เข้า git  
> ตรวจสอบว่า `.env` อยู่ใน `.gitignore` แล้ว (`grep '\.env' .gitignore`)

### 3. ติดตั้ง uv และ serena บน host

```bash
# ติดตั้ง uv (Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# reload PATH (หรือเปิด terminal ใหม่)
source ~/.bashrc   # หรือ source ~/.zshrc

# ติดตั้ง serena จาก source repo (ได้เวอร์ชัน dev ล่าสุด)
uv tool install -p 3.14 --from /home/sun/projects/serena serena-agent

# ตรวจสอบ
serena --version
# → Serena 2.0.1.dev0
```

> ถ้าต้องการติดตั้งจาก PyPI แทน (เวอร์ชัน stable):
> ```bash
> uv tool install -p 3.13 serena-agent
> ```

### 4. ติดตั้ง serena-mcp เป็น systemd user service

```bash
# ติดตั้ง service file
mkdir -p ~/.config/systemd/user
cp /home/sun/projects/serena/serena-mcp.service ~/.config/systemd/user/

# โหลดและเปิดใช้งาน
systemctl --user daemon-reload
systemctl --user enable serena-mcp.service
systemctl --user start serena-mcp.service

# เปิด lingering เพื่อให้ service รันต่อแม้ไม่ได้ login
loginctl enable-linger $USER
```

ตรวจสอบว่าทำงาน:
```bash
systemctl --user status serena-mcp.service
# ควรเห็น: Active: active (running)
# และบรรทัด: Uvicorn running on http://0.0.0.0:9122

# ทดสอบ MCP endpoint โดยตรง
curl -s -o /dev/null -w "%{http_code}" \
  -X POST http://localhost:9122/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
# ควรได้: 200
```

### 5. รัน tunnel-client (Docker)

```bash
cd /home/sun/projects/serena
docker compose -f docker-compose.chatgpt.yml --env-file .env up -d

# ตรวจสอบว่า tunnel เชื่อมต่อสำเร็จ
docker logs serena-tunnel-client | grep -E "🟢|started|ERROR"
# ควรเห็น: 🟢 tunnel-client started
```

ตรวจสอบ health:
```bash
curl http://127.0.0.1:8080/healthz   # → live
curl http://127.0.0.1:8080/readyz    # → ready
```

### 6. ติดตั้ง tunnel-client service file เป็น backup (optional)

ถ้าต้องการให้ tunnel-client รันเป็น systemd service แทน Docker (ต้องมี binary บน host ด้วย):

```bash
mkdir -p ~/.config/systemd/user
cp /home/sun/projects/serena/serena-tunnel-client.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable serena-tunnel-client.service
systemctl --user start serena-tunnel-client.service
```

> ดาวน์โหลด tunnel-client binary:
> ```bash
> VERSION=v0.0.16
> curl -fLO "https://github.com/openai/tunnel-client/releases/download/${VERSION}/tunnel-client-${VERSION}-linux-amd64.zip"
> unzip tunnel-client-${VERSION}-linux-amd64.zip tunnel-client
> mv tunnel-client ~/.local/bin/tunnel-client
> chmod +x ~/.local/bin/tunnel-client
> rm tunnel-client-${VERSION}-linux-amd64.zip
> ```

### 7. เชื่อมต่อกับ ChatGPT

1. เปิด ChatGPT → เมนู Plugins → **Add custom MCP server**
2. เลือก **Connection: Tunnel**
3. เลือก tunnel ที่สร้างไว้จากรายการ (หรือวาง `tunnel_id` โดยตรง)
4. ตั้งค่า authentication ตามที่ ChatGPT ถาม
5. คลิก **I understand and want to continue** → **Create as a plugin**

> tunnel-client ต้องรันอยู่และ status เป็น `ready` ก่อน ถ้า tunnel ไม่ขึ้นในรายการดู [Troubleshooting](#troubleshooting)

---

## การใช้งาน serena ใน ChatGPT

### การ activate project

serena-mcp บน host ไม่ได้ load project อัตโนมัติตอน startup เพื่อหลีกเลี่ยงปัญหา permission  
ให้ ChatGPT เรียก tool `activate_project` ก่อนเริ่มงานทุกครั้ง:

```
ขอให้ ChatGPT:
"กรุณา activate project /home/sun/projects/serena ก่อนเริ่มทำงาน"
```

หรือ ChatGPT จะเรียก `initial_instructions` tool เองโดยอัตโนมัติตาม system prompt

### เพิ่ม project ให้ serena รู้จัก

ถ้า project ยังไม่มี `.serena/project.yml` ให้ run ผ่าน terminal ก่อน:
```bash
cd /home/sun/projects/MY_PROJECT
serena add-project .
```

จากนั้น ChatGPT สามารถ activate ด้วย:
```
activate_project("/home/sun/projects/MY_PROJECT")
```

### ดู project ที่ serena รู้จัก

```bash
serena list-projects
```

### ตรวจสอบ log ขณะ ChatGPT ทำงาน

```bash
# ดู log แบบ live
journalctl --user -u serena-mcp.service -f

# หรือดู log file ที่ serena เขียนไว้
ls /home/sun/projects/serena/config/logs/$(date +%Y-%m-%d)/
```

---

## การใช้งานประจำวัน

### ดูสถานะทั้งหมด

```bash
# serena-mcp (host service)
systemctl --user status serena-mcp.service

# tunnel-client (Docker)
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml ps

# ตรวจสอบ tunnel พร้อมรับ request
curl http://127.0.0.1:8080/healthz   # live
curl http://127.0.0.1:8080/readyz    # ready

# admin UI (เปิดใน browser)
xdg-open http://127.0.0.1:8080/ui
```

### หยุด / เริ่มใหม่

```bash
# serena-mcp (host)
systemctl --user stop serena-mcp.service
systemctl --user start serena-mcp.service
systemctl --user restart serena-mcp.service

# tunnel-client (Docker)
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml restart tunnel-client

# หยุดทั้งคู่
systemctl --user stop serena-mcp.service
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml down

# เริ่มใหม่ทั้งคู่
systemctl --user start serena-mcp.service
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml up -d
```

### ดู log

```bash
# serena-mcp — live
journalctl --user -u serena-mcp.service -f

# serena-mcp — 50 บรรทัดล่าสุด
journalctl --user -u serena-mcp.service -n 50 --no-pager

# serena-mcp — ช่วงเวลาเฉพาะ
journalctl --user -u serena-mcp.service --since "1 hour ago"

# tunnel-client — live
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml logs -f tunnel-client

# tunnel-client — ล่าสุด
docker logs serena-tunnel-client --tail 30
```

---

## อัปเกรด serena

```bash
# อัปเกรดจาก local source (ใช้ commit ล่าสุดใน repo)
cd /home/sun/projects/serena
git pull

uv tool install -p 3.14 --from /home/sun/projects/serena serena-agent --reinstall

# ตรวจสอบ version
serena --version

# restart service เพื่อใช้ version ใหม่
systemctl --user restart serena-mcp.service
```

---

## Troubleshooting

### serena-mcp ไม่ start / crash

```bash
# ดู log ย้อนหลัง
journalctl --user -u serena-mcp.service -n 50 --no-pager
```

สาเหตุที่พบบ่อย:

| อาการ | สาเหตุ | วิธีแก้ |
|---|---|---|
| `Address already in use` | port 9122 ถูกยึด | `ss -tlnp \| grep 9122` หาว่าใคร |
| `Permission denied: .../project.yml` | project.yml เป็นของ root | ดู [ปัญหา permission](#ปัญหา-permission-ของ-projectyml) |
| `No module named serena` | ติดตั้งไม่สำเร็จ | `uv tool install --reinstall ...` |
| ไม่มี error แต่ process หยุดทันที | config ผิด | ตรวจ `SERENA_HOME` ใน service file |

### ปัญหา permission ของ project.yml

ถ้า serena-mcp container (root) เคยสร้างไฟล์ไว้ก่อน ไฟล์เหล่านั้นจะเป็นของ root:

```bash
# ตรวจสอบ
ls -la /home/sun/projects/.serena/

# แก้โดยให้ root chown ให้ (ต้องใช้ sudo)
sudo chown -R sun:sun /home/sun/projects/.serena/
```

> ถ้าใช้ sudo ไม่ได้ใน session นี้ ให้ข้ามไปก่อน — serena จะทำงานได้ปกติโดยไม่ระบุ `--project` ตอน startup ChatGPT activate project ได้ผ่าน MCP tool เอง

### tunnel-client ไม่ start หรือ log แสดง error

```bash
# ดู log
docker logs serena-tunnel-client --tail 30
```

สาเหตุที่พบบ่อย:
- **API key ผิด / หมดอายุ** — เห็น `401 Unauthorized` → สร้าง key ใหม่ใน Platform แล้วแก้ `.env`
- **Tunnel ID ผิด** — เห็น `tunnel not found` → ตรวจ `CONTROL_PLANE_TUNNEL_ID` ใน `.env`
- **serena-mcp ยังไม่พร้อม** — tunnel-client จะ retry อัตโนมัติ รอสักครู่

หลังแก้ `.env` ต้อง recreate container:
```bash
cd /home/sun/projects/serena
docker compose -f docker-compose.chatgpt.yml down
docker compose -f docker-compose.chatgpt.yml --env-file .env up -d
```

### tunnel ไม่ขึ้นใน ChatGPT

1. ตรวจว่า `readyz` ตอบกลับ: `curl http://127.0.0.1:8080/readyz` ต้องได้ `ready`
2. ตรวจว่า tunnel associate กับ **ChatGPT workspace** ด้วย:
   - Platform → Tunnels → เลือก tunnel → แก้ไข → เพิ่ม workspace
3. ตรวจว่าผู้ใช้ที่สร้าง plugin มีสิทธิ **Tunnels Read + Use**
4. รอ 30 นาทีหลังเพิ่มสิทธิ์ใหม่ (propagation time)

### debug ด้วย tunnel-client doctor (ถ้า binary อยู่บน host)

```bash
tunnel-client doctor --explain
```

### ทดสอบ MCP endpoint โดยตรง

```bash
curl -v -X POST http://localhost:9122/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
# ควรได้ HTTP 200 และ event stream
```

### Port 9122 ถูกยึด

```bash
# ดูว่าใครยึด
ss -tlnp | grep 9122

# ถ้าจำเป็นต้องเปลี่ยน port ให้แก้ใน 3 ที่พร้อมกัน:
# 1. serena-mcp.service  — ค่า --port
# 2. serena-tunnel-client.service  — ค่า MCP_SERVER_URL
# 3. docker-compose.chatgpt.yml    — ค่า MCP_SERVER_URL
# แล้ว reload:
cp /home/sun/projects/serena/serena-mcp.service ~/.config/systemd/user/
systemctl --user daemon-reload && systemctl --user restart serena-mcp.service
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml restart tunnel-client
```

---

## ไฟล์ที่เกี่ยวข้อง

| ไฟล์ | ที่อยู่ | คำอธิบาย |
|---|---|---|
| `docker-compose.chatgpt.yml` | repo root | compose สำหรับ tunnel-client |
| `serena-mcp.service` | repo root | systemd service template สำหรับ serena-mcp |
| `serena-tunnel-client.service` | repo root | systemd service template สำหรับ tunnel-client (ใช้แทน Docker ได้) |
| `.env` | repo root | API key + Tunnel ID (ห้าม commit) |
| `.env.example` | repo root | template สำหรับ .env |
| `~/.config/systemd/user/serena-mcp.service` | host | service ที่ติดตั้งจริง |
| `~/.config/systemd/user/serena-tunnel-client.service` | host | tunnel-client service (ถ้าใช้แทน Docker) |
| `config/serena_config.yml` | repo root | global Serena config ที่ serena-mcp อ่าน |

---

## ความปลอดภัย

- **tunnel-client ไม่เปิด inbound port** — ติดต่อออกไป `api.openai.com:443` เท่านั้น
- **port 9122 bind `0.0.0.0`** แต่ tunnel-client ใช้ `localhost` เพื่อเข้าถึง — เครื่องอื่นใน LAN ไม่ควรเข้าถึงได้โดยตรง (ปิดด้วย firewall ถ้าต้องการความปลอดภัยสูง)
- **`.env` ไม่ commit** — อยู่ใน `.gitignore` แล้ว
- **ChatGPT มีสิทธิ์เข้าถึง filesystem** โดยตรงผ่าน serena — เฉพาะไฟล์ที่ `sun` อ่าน/เขียนได้เท่านั้น (ไม่มี container isolation)
- **ปิด tunnel เมื่อไม่ใช้**:
  ```bash
  docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml down
  systemctl --user stop serena-mcp.service
  ```

### จำกัด shell command ถ้าไม่ต้องการ

ใน `config/serena_config.yml` หรือ per-project `.serena/project.yml`:
```yaml
excluded_tools:
  - execute_shell_command
```

---

## อัปเกรด tunnel-client (Docker)

Docker จะดึง image ใหม่เองเมื่อมีการ pull:

```bash
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml pull tunnel-client
docker compose -f /home/sun/projects/serena/docker-compose.chatgpt.yml up -d
```

---

## อ้างอิง

- [OpenAI Secure MCP Tunnel Guide](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
- [tunnel-client GitHub Repository](https://github.com/openai/tunnel-client)
- [tunnel-client Deployment Docs](https://github.com/openai/tunnel-client/blob/master/docs/deployment/overview.md)
- [Serena MCP Documentation](https://oraios.github.io/serena/02-usage/020_running.html)
- [Platform Tunnel Settings](https://platform.openai.com/settings/organization/tunnels)
