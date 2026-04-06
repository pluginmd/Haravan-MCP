# Hướng Dẫn Triển Khai — Haravan MCP

---

## 1. Local Development

```bash
# Clone & install
git clone <repo-url>
cd HaravanMCP
npm install

# Chạy development mode
npm run dev -- mcp -t <your_haravan_token>

# Build production
npm run build

# Chạy production
node dist/cli.js mcp -t <your_haravan_token>
```

---

## 2. Docker Deployment

### Cấu trúc thư mục

```
docker/
├── Dockerfile          # Multi-stage build (builder + production)
├── docker-compose.yml  # Services: haravan-mcp + ngrok (optional)
└── .dockerignore       # Exclude node_modules, tests, docs
```

### Build & Run

```bash
# Build và chạy
docker compose -f docker/docker-compose.yml up -d

# Xem logs
docker logs haravan-mcp -f

# Rebuild sau khi sửa code
docker compose -f docker/docker-compose.yml up -d --build

# Dừng
docker compose -f docker/docker-compose.yml down
```

### Cấu hình

```bash
# Copy file env mẫu, điền token thật
cp docker/.env.example docker/.env
```

Sửa `docker/.env`:

```env
HARAVAN_ACCESS_TOKEN=your_haravan_token_here
NGROK_AUTHTOKEN=your_ngrok_authtoken_here   # optional, chỉ cần nếu dùng ngrok
```

**Lưu ý**: `docker/.env` đã nằm trong `.gitignore` — không bao giờ bị push lên GitHub. Chỉ `docker/.env.example` (template không chứa key) được commit.

### Health Check

```bash
# Local
curl http://localhost:4567/health
# → {"status":"ok"}

# MCP endpoint
curl -X POST http://localhost:4567/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}'
```

---

## 3. Expose qua Internet

### Option A: Cloudflare Tunnel (khuyến nghị — miễn phí, không interstitial)

```bash
# Cài đặt
brew install cloudflared  # macOS
# hoặc: apt install cloudflared  # Linux

# Chạy tunnel
cloudflared tunnel --url http://localhost:4567

# Output: https://xxx-xxx-xxx.trycloudflare.com
# MCP endpoint: https://xxx-xxx-xxx.trycloudflare.com/mcp
```

**Lưu ý**: URL thay đổi mỗi lần restart (free plan). Dùng named tunnel cho URL cố định.

### Option B: ngrok (có interstitial page — có thể gây lỗi với MCP clients)

```bash
# Thêm ngrok service vào docker-compose.yml
ngrok:
  image: ngrok/ngrok:latest
  environment:
    - NGROK_AUTHTOKEN=<your_authtoken>
  command: http haravan-mcp:3000
  ports:
    - "4040:4040"
  depends_on:
    haravan-mcp:
      condition: service_healthy
```

**Cảnh báo**: ngrok free tier hiển thị interstitial page → Claude.ai / MCP clients có thể không connect được. Dùng Cloudflare Tunnel thay thế.

---

## 4. Cài đặt Claude Skill

Claude Skill là lớp tri thức giúp Claude dùng Haravan MCP đúng cách — chọn đúng tool, tránh lãng phí token, viết insight có giá trị. Cần cài riêng ngoài MCP Server.

### Claude.ai Web

1. Tải file `haravan-mcp-skill.zip` từ release hoặc build từ `claudeskill/haravan-mcp/`
2. Truy cập Claude.ai → **Settings** → **Claude Code** → **Skills** → **Upload Skill**
3. Upload file zip → Confirm activation
4. Kiểm tra: hỏi Claude *"tình hình cửa hàng tuần này"* → Claude nên tự chọn tools và trình bày dashboard đúng format

**Lưu ý**: Skill chỉ cần upload 1 lần. Tự động áp dụng cho mọi conversation có kết nối Haravan MCP.

### Claude Code CLI

```bash
# Copy skill vào thư mục skills của Claude Code
cp -r claudeskill/haravan-mcp/ ~/.claude/skills/haravan-mcp/

# Hoặc tạo symlink (tiện cho development)
ln -s /path/to/HaravanMCP/claudeskill/haravan-mcp/ ~/.claude/skills/haravan-mcp
```

Cấu trúc sau khi cài:
```
~/.claude/skills/haravan-mcp/
├── SKILL.md
└── references/
    ├── mcp-tools.md
    ├── insights-formulas.md
    └── examples.md
```

### Xác nhận Skill đã hoạt động

Sau khi cài, mở conversation mới với Claude (đã kết nối MCP Server) và hỏi:

```
"tình hình cửa hàng tuần này"
```

Claude nên tự động:
- Gọi `hrv_orders_summary` với date range tuần hiện tại
- Gọi `hrv_inventory_health` song song
- Trả về dashboard với bảng KPI, trend indicators (↑↓), alerts (⚠️🔴)
- Viết insight có con số và hành động cụ thể

Nếu Claude hỏi lại "bạn muốn xem gì?" thay vì tự chạy → Skill chưa được load đúng.

### Cập nhật Skill

```bash
# Pull bản mới và copy lại
git pull origin main
cp -r claudeskill/haravan-mcp/ ~/.claude/skills/haravan-mcp/
```

---

## 5. Kết nối với AI Assistants

### Claude Desktop / Cursor / Trae (stdio mode)

```json
{
  "mcpServers": {
    "haravan-mcp": {
      "command": "npx",
      "args": ["-y", "haravan-mcp", "mcp", "-t", "<token>"]
    }
  }
}
```

### Claude.ai Web (HTTP mode qua tunnel)

1. Chạy Docker container: `docker compose -f docker/docker-compose.yml up -d`
2. Expose qua Cloudflare: `cloudflared tunnel --url http://localhost:4567`
3. Thêm connector trên Claude.ai: URL = `https://xxx.trycloudflare.com/mcp`

### Custom Tool Selection

```json
{
  "mcpServers": {
    "haravan-mcp": {
      "command": "npx",
      "args": ["-y", "haravan-mcp", "mcp", "-t", "<token>", "--tools", "preset.smart,preset.orders"]
    }
  }
}
```

---

## 6. Troubleshooting

| Vấn đề | Nguyên nhân | Giải pháp |
|---------|------------|-----------|
| "Couldn't reach the MCP server" | ngrok interstitial / URL sai | Dùng Cloudflare Tunnel |
| "Parse error: Invalid JSON" | express.json() consume body | Đảm bảo `handleRequest(req, res, req.body)` |
| Tools không hiển thị | Session ID không lưu | Check sse.ts session flow |
| Chỉ 50 orders bất kể date range | Pagination break condition sai | Verify `PAGE_SIZE=50` trong helpers.ts |
| 429 Rate Limited | Quá nhiều API calls | Smart tools tự throttle; giảm tần suất gọi |
| Container restart loop | Port conflict / token sai | Check `docker logs haravan-mcp` |

---

## 7. Environment Variables

| Variable | Bắt buộc | Mô tả |
|----------|---------|-------|
| `HARAVAN_ACCESS_TOKEN` | ✅ | Private app token hoặc OAuth token |
| `HARAVAN_APP_ID` | OAuth only | App ID cho OAuth flow |
| `HARAVAN_APP_SECRET` | OAuth only | App Secret cho OAuth flow |
| `NODE_ENV` | ❌ | `production` để disable debug logs |

---

*English: This guide covers local development, Docker deployment, internet exposure via Cloudflare Tunnel, and connecting to AI assistants (Claude Desktop, Cursor, Claude.ai web).*
