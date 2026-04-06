# Kiến Trúc Hệ Thống — Haravan MCP

---

## Tổng quan

```
User ←→ AI Assistant (Claude/Cursor/Trae)
              │
              │ MCP Protocol
              ▼
         MCP Server (Node.js)
              │
              ├── Smart Tools (hrv_*)
              │   ├── Pagination engine
              │   ├── Rate limiter (leaky bucket)
              │   └── Aggregation logic
              │
              ├── Base Tools (haravan_*)
              │   └── 1:1 Haravan REST API mapping
              │
              └── Middleware Chain (Koa-style onion)
                  ├── Error Handler
                  ├── Validation
                  ├── Rate Limiter
                  ├── Pagination
                  └── API Handler
```

---

## Transport Modes

### stdio (mặc định)
- Dùng cho Claude Desktop, Cursor, Trae
- Giao tiếp qua stdin/stdout
- Mỗi instance = 1 session

### HTTP Streamable (cho Docker/web)
- Express server trên port 3000
- Endpoint: `POST/GET/DELETE /mcp`
- Session management: mỗi client tạo session riêng qua `mcp-session-id` header
- Health check: `GET /health`

```
Client                    Server
  │                         │
  ├── POST /mcp ──────────► │  Initialize (no session-id)
  │                         │  ← Response + mcp-session-id header
  │                         │
  ├── POST /mcp ──────────► │  tools/list (with session-id)
  │  (mcp-session-id: xxx)  │  ← Tool catalog
  │                         │
  ├── POST /mcp ──────────► │  tools/call (with session-id)
  │  (mcp-session-id: xxx)  │  ← Tool result (SSE stream)
  │                         │
  ├── DELETE /mcp ─────────►│  Close session
```

---

## Middleware Chain

Mỗi tool call đi qua middleware chain theo Koa-style onion model:

```
Request ──► Error Handler ──► Validation ──► Rate Limiter ──► Pagination ──► API Handler
                                                                                │
Response ◄── Error Handler ◄── Validation ◄── Rate Limiter ◄── Pagination ◄────┘
```

### Error Handler (`middleware/error-handler.ts`)
- Catch tất cả exceptions từ layers bên trong
- Format error response: `{ error: true, message: "..." }`
- Map HTTP status codes: 401 → token expired, 429 → rate limited

### Validation (`middleware/validation.ts`)
- Sanitize input parameters
- Type checking via Zod schemas
- Prevent injection attacks

### Rate Limiter (`middleware/rate-limiter.ts`)
- Tuân thủ Haravan leaky bucket: 80 request bucket, 4 req/s drain
- Đọc header `X-Haravan-Api-Call-Limit` từ mỗi response
- Auto-throttle khi bucket usage > 60: sleep 1s
- Auto-throttle khi bucket usage > 75: sleep 2s

### Pagination (`middleware/pagination.ts`)
- Trigger khi `fetchAll=true` trong params
- Haravan API max 50 records/page
- Auto-loop pages cho đến khi hết data hoặc đạt limit
- Max 5000 records (safety cap)

### API Handler (`handler.ts`)
- Xây dựng Haravan API request từ tool definition
- Bearer token authentication
- JSON response parsing

---

## Smart Tools vs Base Tools

### Smart Tools (`hrv_*`)
- **Bypass middleware chain** — có `customHandler`
- Tự quản lý pagination qua helper `fetchAll()`
- Tự quản lý rate limiting (sleep giữa pages)
- Return aggregated summary (~200-800 tokens)
- Client (Claude) nhận kết quả compact, tiết kiệm context

### Base Tools (`haravan_*`)
- Đi qua middleware chain đầy đủ
- 1:1 mapping với Haravan REST API endpoint
- Return raw API response (stripped null fields)
- Dùng cho detail view, CRUD operations, drill-down

### fetchAll Helper

```typescript
// Haravan API max 50/page → loop until done
async function fetchAll(client, path, resourceKey, params, options) {
  const PAGE_SIZE = 50;
  for (let page = 1; page <= maxPages; page++) {
    const response = await client.get(path, { ...params, page, limit: PAGE_SIZE });
    const items = response.data[resourceKey];
    allItems.push(...items);
    if (items.length < PAGE_SIZE) break;  // Last page
    await sleep(250);  // Throttle between pages
  }
  return { items: allItems, apiCalls };
}
```

---

## Authentication Flow

### Private Token
```
haravan-mcp mcp -t <token>
     │
     ▼
  Bearer <token> in Authorization header
     │
     ▼
  Haravan API (apis.haravan.com)
```

### OAuth 2.0
```
haravan-mcp login -a <app_id> -s <secret>
     │
     ▼
  Open browser → accounts.haravan.com/connect/authorize
     │                              │
     │                              ▼
     │                    User authorizes app
     │                              │
     │                              ▼
     │                    Redirect to localhost:3000/callback
     │                              │
     ▼                              ▼
  Exchange code for token → Store in ~/.haravan-mcp/tokens.json
     │
     ▼
  haravan-mcp mcp -a <app_id> --oauth
     │
     ▼
  Load stored token → Bearer auth
```

---

## Configuration

### mergeConfig Priority
```
CLI flags > Environment variables > Defaults
```

| Config | CLI | Env | Default |
|--------|-----|-----|---------|
| Access Token | `-t <token>` | `HARAVAN_ACCESS_TOKEN` | — |
| App ID | `-a <appId>` | `HARAVAN_APP_ID` | — |
| App Secret | `-s <secret>` | `HARAVAN_APP_SECRET` | — |
| API Domain | `-d <domain>` | — | `https://apis.haravan.com` |
| Webhook Domain | `--webhook-domain` | — | `https://webhook.haravan.com` |
| Transport | `-m stdio\|http` | — | `stdio` |
| Port | `-p <port>` | — | `3000` |
| Tools Filter | `--tools <filter>` | — | `preset.default` |
| Debug | `--debug` | — | `false` |

---

## Haravan API Constraints

| Constraint | Giá trị | Ảnh hưởng |
|-----------|---------|-----------|
| Rate Limit | 80 bucket, 4 req/s drain | Smart tools throttle tự động |
| Max per page | 50 records | fetchAll loop 50/page |
| Date format | ISO 8601 | Skill layer tính date ranges |
| Currency | VND (ISO 4217) | Format với dấu phẩy nghìn |
| Auth | Bearer token | Header mỗi request |

---

---

## Claude Skill Layer

### Vai trò: "Bộ não" điều phối MCP Server ("Cánh tay")

```
User câu hỏi tự nhiên
        │
        ▼
  Claude Skill (SKILL.md)
  ├── Phân loại intent (Store Pulse / Inventory / Customer / Action...)
  ├── Chọn tools phù hợp (smart > base)
  ├── Xác định params (date_from, date_to, limit...)
  └── Gọi song song khi tools độc lập nhau
        │ MCP tool calls
        ▼
  MCP Server — thực thi heavy lifting
  ├── Smart Tools: pagination lớn, batch calls, full-population scoring
  └── Base Tools: CRUD, detail lookup, drill-down
        │ Compact JSON response (~200-800 tokens)
        ▼
  Claude Skill — xử lý output
  ├── Áp dụng formulas (ODR, RFM, ABC-FSN, GMROI...)
  ├── So sánh với benchmarks (Cancel <3%, COD Fail <15%...)
  ├── Viết insight: [Metric] + [Con số] + [Context] → [Hành động] + [Impact]
  └── Format markdown: bảng, ↑↓ arrows, ⚠️🔴 alerts
        │
        ▼
  User nhận output có ngữ cảnh, actionable
```

### Decision Tree Flow

```
Câu hỏi về doanh thu / đơn hàng
  → hrv_orders_summary (+ compare period nếu có)
  → hrv_top_products (nếu hỏi sản phẩm)
  → hrv_order_cycle_time (nếu hỏi tốc độ xử lý)

Câu hỏi về khách hàng
  → hrv_customer_segments (RFM 8 phân khúc)
  → haravan_customers_search (nếu tìm khách cụ thể)

Câu hỏi về tồn kho
  → hrv_inventory_health (phân loại ABC-FSN)
  → hrv_stock_reorder_plan (cần nhập hàng)
  → hrv_inventory_imbalance (chuyển kho)

Câu hỏi tổng quan / scorecard
  → Gọi song song: hrv_orders_summary + hrv_customer_segments
    + hrv_inventory_health + hrv_top_products

Hành động (tạo/sửa/xóa)
  → Xác nhận user trước → gọi base tool tương ứng
```

### Anti-patterns Skill ngăn chặn

| Anti-pattern | Hậu quả | Skill fix |
|-------------|---------|----------|
| Gọi `haravan_orders_list` trong vòng lặp để đếm đơn | 200,000+ tokens bị đốt | Dùng `hrv_orders_summary` thay thế |
| Gọi detail tool trong loop để "count" records | N × detail calls thừa | Dùng smart tool có aggregation sẵn |
| Không truyền `date_from`/`date_to` | Fetch toàn bộ lịch sử = timeout | Skill luôn tính date range trước |
| Ghi vào store (create/update/delete) không xác nhận | Thao tác không thể hoàn tác | Skill bắt buộc confirmation step |
| Gọi >6 tools cho 1 câu hỏi | Context window bị lãng phí | Skill giới hạn và dùng smart tools |

### Error Handling Flow

```
Tool call thất bại
  │
  ├── 401 Unauthorized → Token hết hạn → Hướng dẫn refresh token
  ├── 403 Forbidden    → Thiếu scope → Liệt kê scope cần thêm
  ├── 429 Too Many Requests → Rate limited → Thông báo + chờ Retry-After
  └── 500 Server Error → Haravan API lỗi → Thử lại 1 lần, báo cáo nếu tiếp tục
```

### Skill Files

```
claudeskill/haravan-mcp/
├── SKILL.md                        # 718 dòng: rules, decision tree, 10 kịch bản
└── references/
    ├── mcp-tools.md                # Tool catalog + benchmarks
    ├── insights-formulas.md        # 20+ formulas (ODR, RFM, ABC-FSN, GMROI...)
    └── examples.md                 # 5 ví dụ output hoàn chỉnh
```

---

*English: This document describes the internal architecture of Haravan MCP server including transport modes, middleware chain, smart vs base tool design, and Haravan API constraints.*
