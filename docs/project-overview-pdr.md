# Haravan MCP — Tổng Quan Dự Án (PDR)

*Product Development Requirements — Phiên bản 0.1.0*

---

## Tầm nhìn

Haravan MCP là cầu nối giữa AI assistants (Claude, Cursor, Trae) và Haravan e-commerce stores, cho phép AI hiểu và vận hành cửa hàng trực tuyến thông qua Model Context Protocol (MCP).

**Vấn đề giải quyết**: Chủ shop Haravan phải tự đăng nhập admin, dò từng tab, export Excel rồi phân tích thủ công. Với Haravan MCP, họ chỉ cần hỏi bằng ngôn ngữ tự nhiên: *"Tuần này bán bao nhiêu?"*, *"Sản phẩm nào sắp hết hàng?"*, *"Khách nào sắp mất?"*

---

## Kiến trúc 2 lớp

```
┌──────────────────────────────────────────────┐
│              CLAUDE SKILL LAYER              │
│  (SKILL.md — AI reasoning, insights, UX)     │
│                                              │
│  • Decision tree: câu hỏi → tool nào         │
│  • Simple aggregation: group by, filter      │
│  • Insight generation: metric + action       │
│  • Output formatting: bảng, biểu đồ text    │
└──────────────────┬───────────────────────────┘
                   │ MCP Protocol (stdio/HTTP)
┌──────────────────▼───────────────────────────┐
│              MCP SERVER LAYER                │
│  (Node.js — heavy lifting)                   │
│                                              │
│  Smart Tools (7):                            │
│  • Pagination 100+ pages                     │
│  • Rate limiting (leaky bucket 4 req/s)      │
│  • RFM quintile scoring (full population)    │
│  • Batch inventory calls (200+ API calls)    │
│                                              │
│  Base Tools (63):                            │
│  • 1:1 mapping với Haravan REST API          │
│  • Detail view, CRUD, search                 │
└──────────────────┬───────────────────────────┘
                   │ HTTPS + Bearer Token
┌──────────────────▼───────────────────────────┐
│           HARAVAN REST API                   │
│  apis.haravan.com                            │
│  Rate: 80 bucket, 4 req/s leak              │
│  Pagination: 50/page (max)                   │
└──────────────────────────────────────────────┘
```

### Tại sao 2 lớp?

| Việc | MCP Server | Claude Skill |
|------|-----------|-------------|
| Fetch 2000 đơn hàng (40 pages) | ✅ Server pagination + throttle | ❌ Claude không thể loop API |
| RFM scoring 9000 khách | ✅ Quintile cần full population | ❌ Không fit context window |
| Group by kênh bán hàng | ❌ Quá đơn giản cho server | ✅ Claude đọc orders_by_source |
| Viết insight "Doanh thu tăng 12% nhờ..." | ❌ Server không hiểu context | ✅ Claude AI reasoning |
| Format bảng + emoji + tiếng Việt | ❌ Server trả JSON | ✅ Claude format markdown |

---

## Tool Catalog

### Smart Tools (7) — Server-side aggregation

| Tool | Chức năng | API calls tiết kiệm |
|------|-----------|---------------------|
| `hrv_orders_summary` | Tổng DT, đơn, AOV, status/source/cancel breakdown, so sánh kỳ trước | ~40 pages → 1 response |
| `hrv_top_products` | Top N sản phẩm theo DT, variant breakdown | ~40 pages → 1 response |
| `hrv_order_cycle_time` | Median/p90 time-to-confirm/close, đơn stuck | ~40 pages → 1 response |
| `hrv_customer_segments` | RFM 8 segments với action suggestions | ~100 pages → 1 response |
| `hrv_inventory_health` | Phân loại variant: out_of_stock/low/dead/healthy | 100+ inventory calls → 1 response |
| `hrv_stock_reorder_plan` | DSR, days_of_stock, reorder_qty per variant | 200+ inventory calls → 1 response |
| `hrv_inventory_imbalance` | Cross-location imbalance + transfer suggestions | 100+ inventory calls → 1 response |

### Base Tools (63) — Haravan API 1:1

| Category | Tools | Ví dụ |
|----------|-------|-------|
| Orders (13) | list, count, get, create, update, confirm, close, open, cancel, assign, transactions | Tra cứu đơn #12345 |
| Products (11) | list, count, get, create, update, delete, variants CRUD | Tạo sản phẩm mới |
| Customers (14) | list, search, count, get, create, update, delete, groups, addresses CRUD | Tìm khách theo SĐT |
| Inventory (5) | adjustments list/count/get, adjust_or_set, locations | Set tồn kho SKU-001 = 50 |
| Shop (6) | shop_get, locations, users, shipping_rates | Thông tin cửa hàng |
| Content (11) | pages, blogs, articles, script_tags CRUD | Tạo landing page |
| Webhooks (3) | list, subscribe, unsubscribe | Đăng ký webhook |

---

## Presets

| Preset | Tools | Use case |
|--------|-------|---------|
| `preset.default` | 17 | Claude AI assistant — smart + drill-down |
| `preset.smart` | 7 | Chỉ smart tools — dashboard/analytics |
| `preset.light` | 7 | Read-only minimal |
| `preset.products` | 11 | Quản lý sản phẩm |
| `preset.orders` | 13 | Quản lý đơn hàng |
| `preset.customers` | 14 | Quản lý khách hàng |
| `preset.inventory` | 7 | Quản lý tồn kho |
| `preset.content` | 11 | Quản lý nội dung |
| `preset.webhooks` | 3 | Developer webhook |

---

## Authentication

### Private Token (đơn giản)
1. Haravan Admin → Apps → Private apps → Create
2. Chọn permissions → Copy Token
3. `haravan-mcp mcp -t <token>`

### OAuth 2.0 (đầy đủ)
1. `haravan-mcp login -a <app_id> -s <app_secret>`
2. Browser mở → Authorize → Callback → Token stored tại `~/.haravan-mcp/`
3. `haravan-mcp mcp -a <app_id> --oauth`

---

## Roadmap

| Phase | Nội dung | Status |
|-------|---------|--------|
| v0.1.0 | Core: 70 tools, 2 transports, OAuth, presets | ✅ Done |
| v0.2.0 | Caching layer, webhook invalidation | 📋 Planned |
| v0.3.0 | Advanced smart tools (demand forecast, basket analysis) | 📋 Planned |
| v1.0.0 | Production-ready, npm publish | 📋 Planned |

---

---

## Claude Skill — Bộ tri thức tối ưu

Claude Skill là lớp tri thức hướng dẫn Claude cách dùng Haravan MCP hiệu quả nhất. Không cần cài thêm phần mềm — chỉ cần upload file vào Claude.ai hoặc copy vào thư mục `~/.claude/skills/`.

### Nội dung Skill

| File | Dòng | Nội dung |
|------|------|---------|
| `SKILL.md` | 718 | Decision tree, 10 kịch bản phân tích, quy tắc bắt buộc, output templates |
| `references/mcp-tools.md` | ~200 | Catalog 70 tools + "Claude tự làm từ output này" + benchmarks |
| `references/insights-formulas.md` | ~300 | 20+ công thức: ODR, RFM, ABC-FSN, GMROI, DSR/DOS, Shrinkage... |
| `references/examples.md` | ~250 | 5 ví dụ output hoàn chỉnh (Store Pulse, COD Monitor, Inventory...) |

### Token efficiency

| Kịch bản | Không có Skill | Có Skill | Tiết kiệm |
|---------|---------------|---------|-----------|
| "Doanh thu tuần này" | Claude tự loop 40 pages = 200,000 tokens | Gọi `hrv_orders_summary` = 300 tokens | 99.85% |
| "Phân khúc khách hàng" | Loop 100 pages + tự score = 500,000 tokens | Gọi `hrv_customer_segments` = 800 tokens | 99.84% |
| "Tồn kho cần nhập" | Loop 200 inventory calls = 1,000,000 tokens | Gọi `hrv_stock_reorder_plan` = 600 tokens | 99.94% |
| "Scorecard toàn diện" | 4 luồng × 250K = 1,000,000 tokens | 4 smart tools song song = 2,000 tokens | 99.80% |

### 10 kịch bản phân tích tích hợp sẵn

1. **Store Pulse** — tổng quan doanh thu, tỷ lệ hủy, kênh bán hàng
2. **Order Pipeline** — thời gian xử lý, đơn stuck, bottleneck
3. **Stock Health** — phân loại ABC-FSN, out-of-stock, dead stock
4. **Customer RFM** — 8 phân khúc, at-risk, loyal, lapsed
5. **Product Performance** — top sản phẩm, Catalog Health Score (0-100)
6. **Weekly Scorecard** — KPI dashboard 1 tuần vs kỳ trước
7. **COD Monitor** — tỷ lệ COD fail, Revenue at Risk
8. **Inventory Reorder** — DSR, Days of Stock, reorder quantity
9. **Smart Search** — tra cứu đơn/khách/sản phẩm theo intent
10. **Store Action** — thực thi CRUD với user confirmation

### Benchmarks tích hợp

| Metric | Tốt | Cảnh báo | Nguy hiểm |
|--------|-----|---------|----------|
| Cancel Rate | <3% | 3-7% | >7% |
| Repeat Purchase Rate | >30% | 15-30% | <15% |
| COD Fail Rate | <15% | 15-25% | >25% |
| Catalog Health Score | >80 | 60-80 | <60 |

### Cài đặt

**Claude.ai (web)**:
1. Tải file `haravan-mcp-skill.zip`
2. Claude.ai → Settings → Claude Code → Upload Skill
3. Xác nhận: hỏi *"tình hình cửa hàng tuần này"* → Skill tự kích hoạt

**Claude Code CLI**:
```bash
cp -r claudeskill/haravan-mcp/ ~/.claude/skills/haravan-mcp/
```

---

## Tác giả

**Nguyễn Ngọc Tuấn**
Founder — Transform Group | Lark Platinum Partner
[Facebook](https://www.facebook.com/khongphaituan)

---

*English version: See [README.md](../README.md) for English quick start.*
