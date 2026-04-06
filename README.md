# Haravan MCP

Haravan E-commerce OpenAPI MCP Server — Kết nối AI assistants với Haravan stores.

MCP (Model Context Protocol) server cho phép AI assistants (Claude, Cursor, Trae) tương tác trực tiếp với Haravan APIs: quản lý sản phẩm, đơn hàng, khách hàng, tồn kho, webhooks và nhiều hơn.

*[English](#english) below.*

---

## Bắt đầu nhanh

### Sử dụng với Claude / Cursor / Trae

Thêm vào file cấu hình MCP:

```json
{
  "mcpServers": {
    "haravan-mcp": {
      "command": "npx",
      "args": ["-y", "haravan-mcp", "mcp", "-t", "<your_access_token>"]
    }
  }
}
```

### Sử dụng với Docker (HTTP mode)

```bash
# Clone repo
git clone <repo-url> && cd HaravanMCP

# Cấu hình token
cp docker/.env.example docker/.env
# Sửa docker/.env → điền HARAVAN_ACCESS_TOKEN

# Chạy
docker compose -f docker/docker-compose.yml up -d

# MCP endpoint: http://localhost:4567/mcp
# Health check: http://localhost:4567/health
```

### Sử dụng với OAuth 2.0

```bash
# Bước 1: Login
npx haravan-mcp login -a <app_id> -s <app_secret>

# Bước 2: Cấu hình MCP
{
  "mcpServers": {
    "haravan-mcp": {
      "command": "npx",
      "args": ["-y", "haravan-mcp", "mcp", "-a", "<app_id>", "--oauth"]
    }
  }
}
```

---

## Kiến trúc 2 lớp

```
Claude Skill Layer (AI reasoning, insights, formatting)
         │
         │ MCP Protocol
         ▼
MCP Server Layer (pagination, rate limiting, aggregation)
         │
         │ HTTPS + Bearer Token
         ▼
Haravan REST API (apis.haravan.com)
```

**Smart Tools (7)**: Server-side pagination + aggregation cho data lớn
**Base Tools (63)**: 1:1 mapping với Haravan API cho detail/CRUD

---

## Smart Tools

| Tool | Chức năng |
|------|-----------|
| `hrv_orders_summary` | Tổng DT, AOV, status/source/cancel breakdown, so sánh kỳ trước |
| `hrv_top_products` | Top N sản phẩm theo doanh thu + variant breakdown |
| `hrv_order_cycle_time` | Median/p90 time-to-confirm/close, đơn stuck |
| `hrv_customer_segments` | RFM 8 segments + action suggestions |
| `hrv_inventory_health` | Phân loại: out_of_stock / low / dead / healthy |
| `hrv_stock_reorder_plan` | DSR + reorder point + suggested qty per variant |
| `hrv_inventory_imbalance` | Cross-location imbalance + transfer suggestions |

---

## Base Tools (63 tổng)

| Category | Tools | Mô tả |
|----------|-------|-------|
| Customers | 14 | CRUD khách hàng + địa chỉ |
| Orders | 13 | CRUD đơn hàng + transactions |
| Products | 11 | CRUD sản phẩm + variants |
| Inventory | 5 | Kiểm kho + locations |
| Shop | 6 | Shop info, kho, nhân viên |
| Content | 11 | Pages, blogs, articles, script tags |
| Webhooks | 3 | Subscribe/unsubscribe |

---

## Presets

```bash
# Smart tools only (dashboard/analytics)
haravan-mcp mcp -t <token> --tools "preset.smart"

# Default (smart + detail drill-down)
haravan-mcp mcp -t <token> --tools "preset.default"

# Full CRUD cho specific resources
haravan-mcp mcp -t <token> --tools "preset.products,preset.orders"

# Tất cả tools
haravan-mcp mcp -t <token> --tools "all"

# Mix presets + individual tools
haravan-mcp mcp -t <token> --tools "preset.smart,haravan_webhooks_list"
```

| Preset | Tools | Use case |
|--------|-------|---------|
| `preset.default` | 17 | AI assistant — smart + drill-down |
| `preset.smart` | 7 | Dashboard, analytics |
| `preset.light` | 7 | Read-only minimal |
| `preset.products` | 11 | Product management |
| `preset.orders` | 13 | Order management |
| `preset.customers` | 14 | Customer management |
| `preset.inventory` | 7 | Inventory management |
| `preset.content` | 11 | Content management |
| `preset.webhooks` | 3 | Webhook management |

---

## CLI Commands

```bash
haravan-mcp mcp -t <token>                   # Start server (stdio)
haravan-mcp mcp -t <token> -m http -p 3000   # Start server (HTTP)
haravan-mcp mcp -t <token> --tools "preset.smart"  # Custom tools

haravan-mcp login -a <app_id> -s <secret>    # OAuth login
haravan-mcp whoami                            # Show stored tokens
haravan-mcp logout -a <app_id>               # Remove token
haravan-mcp logout --all                      # Remove all tokens

haravan-mcp tools                             # List all tools
haravan-mcp tools --presets                   # Show presets
haravan-mcp tools --project products          # Filter by category
```

---

## Claude Skill — Tri thức tối ưu

Ngoài MCP Server (70 tools), dự án bao gồm **Claude Skill** (`claudeskill/haravan-mcp/`) — bộ não phân tích được upload lên claude.ai, dạy Claude cách sử dụng MCP tools hiệu quả nhất.

### Token Efficiency — Lý do tồn tại

| Scenario | Không tối ưu | Có Skill + Smart Tools | Tiết kiệm |
|----------|-------------|------------------------|------------|
| "Doanh thu tháng này" | 17 API calls, ~250,000 tokens | 1 tool call, ~300 tokens | **99.9%** |
| "Top 10 sản phẩm" | 17+342 calls, ~800,000 tokens | 1 tool call, ~500 tokens | **99.9%** |
| "Phân tích RFM khách hàng" | 50+ calls, ~500,000 tokens | 1 tool call, ~800 tokens | **99.8%** |
| "Tổng quan cửa hàng" | 100+ calls, ~1,500,000 tokens | 3 calls song song, ~1,200 tokens | **99.9%** |

### Tri thức tích hợp trong Skill

**20+ công thức vận hành e-commerce** (từ `references/insights-formulas.md`):
- **Order Operations**: ODR (Order Defect Rate), Revenue at Risk, Payment Collection Efficiency, Order Cycle Time Breakdown
- **Inventory Intelligence**: ABC-FSN Analysis (phân loại SKU), DSR/DOS (tốc độ bán/ngày tồn kho), Shrinkage Detection, GMROI
- **Customer Analytics**: RFM Scoring (quintile), Purchase Gap Analysis, Customer Concentration Risk, Acquisition vs Retention Economics
- **Product Intelligence**: Catalog Health Score (12 tiêu chí, thang 0-100), Variant Performance Matrix, Product Lifecycle Detection, Price-Volume Curve

**Benchmarks ngành e-commerce Việt Nam** (từ `references/mcp-tools.md`):
| Metric | Tốt | TB | Cần cải thiện |
|--------|-----|-----|---------------|
| Cancel Rate | <3% | 3-5% | >5% |
| Repeat Purchase Rate | >30% | 20-30% | <20% |
| COD Fail Rate | <15% | 15-25% | >25% |
| Catalog Health Score | >80 | 60-80 | <60 |
| Discount Penetration | 10-20% | 20-40% | >40% |

**10 kịch bản phân tích có sẵn** (Decision Tree trong SKILL.md):
1. **Store Pulse** — tổng quan cửa hàng (3 calls song song)
2. **Revenue Breakdown** — phân tích đa chiều: kênh, sản phẩm, địa lý (4 calls)
3. **Order Pipeline** — bottleneck, cycle time, cancel analysis (3 calls)
4. **Stock Health** — phân loại kho, đề xuất nhập, cân bằng đa chi nhánh (3 calls)
5. **Customer RFM** — 7 segments + marketing actions cho từng segment (2 calls)
6. **Product Performance** — best sellers, catalog health, discount ROI
7. **Operations Scorecard** — chấm điểm 10 chỉ số (1-10), top 3 mạnh/yếu (5-6 calls)
8. **COD Monitor** — fail rate theo tỉnh, risk scoring
9. **Smart Search** — tìm đơn/khách/sản phẩm bằng ngôn ngữ tự nhiên
10. **Store Action** — thao tác nhanh với xác nhận trước khi write

**5 ví dụ output thực tế** (từ `references/examples.md`) với data giả — Claude học cách format bảng, viết insight, xử lý lỗi.

### Cách cài Skill

**Claude.ai**: Upload `haravan-mcp-skill.zip` tại claude.ai/skills

**Claude Code**: Copy thư mục `claudeskill/haravan-mcp/` vào `~/.claude/skills/haravan-mcp/`

### Cấu trúc Skill

```
claudeskill/haravan-mcp/
├── SKILL.md (718 dòng)
│   ├── Phần 1: 5 quy tắc bắt buộc
│   ├── Phần 2: Decision tree — câu hỏi → tool nào
│   ├── Phần 3: 10 kịch bản phân tích chi tiết (output templates)
│   ├── Phần 4: Xử lý lỗi (401/403/429/500/network)
│   ├── Phần 5: Multi-turn drill-down
│   ├── Phần 6: Cách viết insight xuất sắc (SAI vs ĐÚNG)
│   └── Phần 7: Anti-patterns
└── references/
    ├── mcp-tools.md — Tool catalog + benchmarks ngành
    ├── insights-formulas.md — 20+ công thức: ODR, RFM, DSR, GMROI, ABC-FSN...
    └── examples.md — 5 ví dụ output hoàn chỉnh với data thực tế
```

---

## Development

```bash
npm install          # Install dependencies
npm run build        # Build TypeScript
npm run dev -- mcp -t <token>   # Dev mode
npm test             # Run tests
npm run format       # Prettier format
```

---

## Tài liệu

| Doc | Nội dung |
|-----|---------|
| [docs/project-overview-pdr.md](docs/project-overview-pdr.md) | Tổng quan dự án, tầm nhìn, roadmap |
| [docs/system-architecture.md](docs/system-architecture.md) | Kiến trúc: transport, middleware, auth flow |
| [docs/code-standards.md](docs/code-standards.md) | Quy chuẩn code, cấu trúc thư mục, patterns |
| [docs/design-guidelines.md](docs/design-guidelines.md) | Tư duy thiết kế MCP + Claude Skill |
| [docs/deployment-guide.md](docs/deployment-guide.md) | Docker, Cloudflare Tunnel, troubleshooting |
| [docs/codebase-summary.md](docs/codebase-summary.md) | Tóm tắt codebase, dependencies, data flow |

---

## Environment Variables

| Variable | Mô tả |
|----------|-------|
| `HARAVAN_ACCESS_TOKEN` | Private app token |
| `HARAVAN_APP_ID` | OAuth App ID |
| `HARAVAN_APP_SECRET` | OAuth App Secret |

---

## Tác giả

**Nguyễn Ngọc Tuấn**
Founder — Transform Group | Lark Platinum Partner
[Facebook](https://www.facebook.com/khongphaituan)

## License

MIT

---

<a name="english"></a>
## English

### What is Haravan MCP?

An MCP (Model Context Protocol) server that connects AI assistants (Claude, Cursor, Trae) to Haravan e-commerce stores. Manage products, orders, customers, inventory, and more through natural language.

### Architecture

Two-layer design:
- **MCP Server** (7 smart tools): Server-side pagination, rate limiting, aggregation for large datasets
- **Claude Skill** (SKILL.md): AI reasoning, insights, formatting — handles simple analysis from smart tool outputs

### Quick Start

```json
{
  "mcpServers": {
    "haravan-mcp": {
      "command": "npx",
      "args": ["-y", "haravan-mcp", "mcp", "-t", "<your_token>"]
    }
  }
}
```

### Docker

```bash
docker compose -f docker/docker-compose.yml up -d
# MCP endpoint: http://localhost:4567/mcp
```

### Documentation

All docs are in Vietnamese (default) with English summaries at the bottom of each file. See [docs/](docs/) directory.
