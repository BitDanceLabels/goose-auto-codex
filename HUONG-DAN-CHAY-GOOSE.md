# 🦢 HƯỚNG DẪN CHẠY GOOSE AI AGENT

> **Goose** là AI Agent tự động hóa các tác vụ engineering
> Hỗ trợ: OpenAI, Anthropic (Claude), Google (Gemini), Ollama

---

## 📋 CÁCH 1: CHẠY TỪ PRE-BUILT DOCKER (Nhanh nhất)

```powershell
# Pull image từ GitHub Container Registry
docker pull ghcr.io/block/goose:latest

# Test chạy
docker run --rm ghcr.io/block/goose:latest --version
```

---

## 📋 CÁCH 2: CHẠY VỚI DOCKER COMPOSE (Khuyến nghị)

### Bước 1: Cấu hình môi trường

Sửa file `E:\workspace-bumbee\API-Gateway-microservices-bumbee\setup-workspace-7777\deploy-all-in-one\allinone\.env-goose`:

```env
# Port để truy cập Goose Web UI
GOOSE_PORT=3300

# Log level
GOOSE_LOG_LEVEL=info

# Thư mục làm việc của Goose
GOOSE_WORKSPACE=E:/bumbee_workspace

# === CHỌN 1 TRONG CÁC PROVIDER SAU ===

# Option 1: OpenAI (GPT-4)
OPENAI_API_KEY=sk-your-openai-api-key-here
GOOSE_PROVIDER=openai
GOOSE_MODEL=gpt-4o

# Option 2: Anthropic (Claude)
# ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
# GOOSE_PROVIDER=anthropic
# GOOSE_MODEL=claude-sonnet-4-20250514

# Option 3: Google (Gemini)
# GOOGLE_API_KEY=your-google-api-key
# GOOSE_PROVIDER=google
# GOOSE_MODEL=gemini-2.0-flash

# Option 4: Ollama (Local - Miễn phí)
# OLLAMA_HOST=http://host.docker.internal:11434
# GOOSE_PROVIDER=ollama
# GOOSE_MODEL=llama3.2
```

### Bước 2: Tạo network Docker (nếu chưa có)

```powershell
docker network create bumbee-ai-net
```

### Bước 3: Chạy Goose

```powershell
cd E:\workspace-bumbee\API-Gateway-microservices-bumbee\setup-workspace-7777\deploy-all-in-one\docker-bumbee-studio-ai

# Chạy với docker-compose
docker-compose -f docker-compose.goose.yml --env-file ../allinone/.env-goose up -d

# Xem logs
docker logs -f goose-ai
```

### Bước 4: Truy cập

- **Web UI**: http://localhost:3300
- **Health Check**: http://localhost:3300/api/health
- **Sessions**: http://localhost:3300/api/sessions

---

## 📋 CÁCH 3: BUILD VÀ CHẠY LOCAL (Từ source)

### Bước 1: Build Docker image

```powershell
cd E:\workspace-bumbee\goose-auto-codex

# Build image
docker build -t goose:local .
```

### Bước 2: Chạy Goose CLI

```powershell
# Hiển thị help
docker run --rm goose:local --help

# Chạy một task
docker run --rm `
  -e GOOSE_PROVIDER=openai `
  -e GOOSE_MODEL=gpt-4o `
  -e OPENAI_API_KEY=$env:OPENAI_API_KEY `
  goose:local run -t "Explain Docker containers"
```

### Bước 3: Chạy Web Server

```powershell
docker run -d `
  --name goose-web `
  -p 3300:3000 `
  -e GOOSE_PROVIDER=openai `
  -e GOOSE_MODEL=gpt-4o `
  -e OPENAI_API_KEY=$env:OPENAI_API_KEY `
  -v E:/bumbee_workspace:/workspace `
  goose:local web --host 0.0.0.0 --port 3000
```

---

## 🔌 API ENDPOINTS

### Health Check
```
GET /api/health
Response: {"status": "ok", "service": "goose-web"}
```

### List Sessions
```
GET /api/sessions
Response: {
  "sessions": [
    {
      "name": "session-uuid",
      "path": "session-uuid",
      "description": "Web session",
      "message_count": 5,
      "working_dir": "/workspace"
    }
  ]
}
```

### Get Session Detail
```
GET /api/sessions/{session_id}
Response: {
  "metadata": {...},
  "messages": [...]
}
```

---

## 🔗 WEBSOCKET API (Real-time Chat)

### Connect
```
WS ws://localhost:3300/ws?token={ws_token}
```

### Send Message
```json
{
  "type": "message",
  "content": "Viết một hàm Python để đọc file CSV",
  "session_id": "your-session-id",
  "timestamp": 1706345678000
}
```

### Receive Response
```json
{
  "type": "response",
  "content": "Đây là code Python...",
  "role": "assistant",
  "timestamp": 1706345678123
}
```

### Tool Request (Goose muốn chạy tool)
```json
{
  "type": "tool_request",
  "id": "tool-call-id",
  "tool_name": "shell",
  "arguments": {"command": "ls -la"}
}
```

### Cancel Task
```json
{
  "type": "cancel",
  "session_id": "your-session-id"
}
```

---

## 🧪 TEST VỚI CURL

### Test Health
```bash
curl http://localhost:3300/api/health
```

### Test Sessions
```bash
curl http://localhost:3300/api/sessions
```

### Test với Authentication (nếu có)
```bash
curl -H "Authorization: Bearer your-token" http://localhost:3300/api/sessions
```

---

## 🔧 TÍCH HỢP VỚI BUMBEE STUDIO

### 1. Đăng ký Goose như một Provider trong core-service

Gọi API core-service để đăng ký:
```bash
POST http://localhost:8000/api/v1/providers/register
{
  "name": "goose",
  "provider_type": "agent",
  "wss_url": "ws://localhost:3300/ws",
  "api_url": "http://localhost:3300/api",
  "is_active": true
}
```

### 2. Gửi task từ Studio đến Goose

```javascript
// Frontend WebSocket
const ws = new WebSocket('ws://localhost:3300/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'message',
    content: 'Phân tích code trong thư mục này và đề xuất cải tiến',
    session_id: 'studio-session-123',
    timestamp: Date.now()
  }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Goose response:', data);
};
```

---

## 📁 CẤU TRÚC WORKSPACE

```
E:/bumbee_workspace/
├── projects/          # Các dự án
├── outputs/           # Kết quả từ Goose
├── logs/              # Logs
└── temp/              # Files tạm
```

---

## ❓ TROUBLESHOOTING

### Lỗi "No provider configured"
```
Cần set GOOSE_PROVIDER và GOOSE_MODEL trong environment
Hoặc chạy: goose configure
```

### Lỗi kết nối WebSocket
```
Kiểm tra token, port, và CORS settings
```

### Lỗi permission khi mount volume
```
Chạy docker với -u $(id -u):$(id -g) hoặc dùng WSL
```

---

## 🎯 CÁC LỆNH GOOSE CLI

```bash
# Hiển thị phiên bản
goose --version

# Cấu hình provider
goose configure

# Chạy một task đơn lẻ
goose run -t "Your task here"

# Bắt đầu session tương tác
goose session

# Chạy web server
goose web --host 0.0.0.0 --port 3000

# Hiển thị thông tin
goose info
```

---

## ✅ CHECKLIST CHẠY THÀNH CÔNG

- [ ] Docker đang chạy
- [ ] Network `bumbee-ai-net` đã tạo
- [ ] API key đã cấu hình trong .env-goose
- [ ] Container `goose-ai` running
- [ ] http://localhost:3300/api/health trả về OK
- [ ] WebSocket kết nối được

---

**Chúc bạn sử dụng Goose hiệu quả! 🦢**
