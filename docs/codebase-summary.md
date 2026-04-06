# Tóm Tắt Codebase — Haravan MCP

---

## Thống kê

| Metric | Giá trị |
|--------|---------|
| Ngôn ngữ | TypeScript (strict mode) |
| Source files | 32 files trong `src/` |
| Total tools | 70 (7 smart + 63 base) |
| Dependencies | 7 runtime + 7 dev |
| Docker image | ~154MB (Node 20 Alpine) |
| Transports | 2 (stdio, HTTP streamable) |

---

## File quan trọng

### Entry Points

| File | Vai trò |
|------|---------|
| `src/cli.ts` | CLI entry — 5 commands (mcp, login, logout, whoami, tools) |
| `src/index.ts` | Public API exports cho library usage |
| `src/mcp-server/init.ts` | Server init: register tools, build middleware, create factory |

### Core Logic

| File | Vai trò | Dòng code ước tính |
|------|---------|-------------------|
| `src/mcp-tool/registry.ts` | Tool catalog, presets, filterTools() | ~200 |
| `src/mcp-tool/handler.ts` | Default API handler cho base tools | ~50 |
| `src/mcp-tool/middleware/chain.ts` | Koa-style middleware composition | ~40 |
| `src/mcp-tool/middleware/rate-limiter.ts` | Haravan leaky bucket compliance | ~60 |
| `src/mcp-tool/middleware/pagination.ts` | Auto-pagination fetchAll=true | ~120 |
| `src/utils/http-client.ts` | HaravanClient class (Axios + rate limit headers) | ~120 |

### Smart Tools

| File | Tools | Logic chính |
|------|-------|------------|
| `src/mcp-tool/tools/smart/helpers.ts` | fetchAll, clientFromCtx, ok, err | Pagination engine (50/page, throttle) |
| `src/mcp-tool/tools/smart/orders-smart.ts` | hrv_orders_summary, hrv_top_products, hrv_order_cycle_time | Order aggregation + prior period comparison |
| `src/mcp-tool/tools/smart/customers-smart.ts` | hrv_customer_segments | RFM quintile scoring + 8 segments |
| `src/mcp-tool/tools/smart/inventory-smart.ts` | hrv_inventory_health, hrv_stock_reorder_plan, hrv_inventory_imbalance | Batch inventory calls + classification |

### Base Tools

| File | Tools | Resources |
|------|-------|-----------|
| `src/mcp-tool/tools/customers.ts` | 14 | Customer CRUD + addresses |
| `src/mcp-tool/tools/orders.ts` | 13 | Order CRUD + transactions |
| `src/mcp-tool/tools/products.ts` | 11 | Product + variant CRUD |
| `src/mcp-tool/tools/inventory.ts` | 5 | Inventory adjustments + locations |
| `src/mcp-tool/tools/shop.ts` | 6 | Shop info, locations, users |
| `src/mcp-tool/tools/webhooks.ts` | 3 | Webhook subscribe/unsubscribe |
| `src/mcp-tool/tools/content.ts` | 11 | Pages, blogs, articles, script tags |

---

## Dependencies

### Runtime

| Package | Version | Vai trò |
|---------|---------|---------|
| @modelcontextprotocol/sdk | 1.12.1 | MCP server framework |
| axios | 1.8.4 | HTTP client cho Haravan API |
| commander | 13.1.0 | CLI argument parsing |
| dotenv | 16.4.7 | Environment variable loading |
| express | 5.1.0 | HTTP transport server |
| open | 8.4.2 | Open browser cho OAuth callback |
| zod | 3.24.0 | Schema validation |

### Dev

| Package | Vai trò |
|---------|---------|
| typescript 5.x | Compiler |
| jest 29.x | Test framework |
| ts-jest | TypeScript test support |
| ts-node | Dev mode runner |
| prettier | Code formatting |

---

## Data Flow

```
1. CLI parses args (commander)
   ↓
2. mergeConfig() combines CLI + env + defaults
   ↓
3. initHaravanMcpServer(config)
   ├── resolveToolFilter() → list of tool names
   ├── filterTools() → enabled McpTool[]
   ├── Build middleware chain
   └── Register each tool on McpServer
   ↓
4. Start transport
   ├── stdio: startStdioTransport(server)
   └── http: startHttpTransport(createServer, port)
   ↓
5. Tool call comes in
   ├── Smart tool (customHandler): direct execution
   │   └── fetchAll() handles pagination internally
   └── Base tool: middleware chain
       └── error → validation → rate-limit → pagination → api-handler
   ↓
6. Response sent back to client
```

---

## Claude Skill Layer

```
claudeskill/haravan-mcp/
├── SKILL.md                        # 718 dòng: rules, decision tree, 10 kịch bản, output templates
└── references/
    ├── mcp-tools.md                # Tool catalog + "Claude tự làm" guidance + benchmarks
    ├── insights-formulas.md        # 20+ formulas: ODR, RFM, ABC-FSN, GMROI, DSR/DOS, Shrinkage...
    └── examples.md                 # 5 ví dụ output hoàn chỉnh (Store Pulse, COD, Inventory...)
```

### Skill coverage — 10 kịch bản phân tích

| # | Kịch bản | Tools chính | Output |
|---|---------|------------|--------|
| 1 | Store Pulse | hrv_orders_summary | DT, đơn, AOV, kênh, cancel |
| 2 | Order Pipeline | hrv_order_cycle_time | Time-to-confirm, đơn stuck |
| 3 | Stock Health | hrv_inventory_health | ABC-FSN, out-of-stock, dead |
| 4 | Customer RFM | hrv_customer_segments | 8 phân khúc + actions |
| 5 | Product Performance | hrv_top_products | Top N + Catalog Health Score |
| 6 | Weekly Scorecard | 4 smart tools song song | KPI dashboard |
| 7 | COD Monitor | hrv_orders_summary | COD fail rate, Revenue at Risk |
| 8 | Inventory Reorder | hrv_stock_reorder_plan | DSR, DOS, reorder qty |
| 9 | Smart Search | haravan_* base tools | Tìm đơn/khách/sản phẩm |
| 10 | Store Action | haravan_* write tools | CRUD với user confirmation |

### Formulas tích hợp trong Skill

- **ODR** (Order Defect Rate) = (cancel + return) / total_orders × 100
- **Revenue at Risk** = ODR × avg_revenue_per_order × orders_at_risk
- **Payment Collection Efficiency** = collected / invoiced × 100
- **RFM Quintile Scoring** — server-side, full population
- **Customer Concentration Risk** = top_10% revenue / total_revenue
- **Purchase Gap Analysis** = avg_days_between_orders per segment
- **ABC-FSN Classification** — Inventory by value + movement frequency
- **DSR** (Daily Sales Rate) = units_sold / days_in_period
- **DOS** (Days of Stock) = current_stock / DSR
- **GMROI** = gross_margin / avg_inventory_value
- **Shrinkage Detection** = (expected_stock - actual_stock) / expected × 100
- **Catalog Health Score** — 12 criteria, 0-100 scale

---

## Docker

```
docker/
├── Dockerfile          # Multi-stage: builder (npm ci + tsc) → production (dist only)
├── docker-compose.yml  # haravan-mcp service + optional ngrok
└── .dockerignore       # Exclude node_modules, tests, docs
```

**Build**: `docker compose -f docker/docker-compose.yml up -d --build`
**Port**: 4567 (host) → 3000 (container)

---

*English: 32 source files, 70 tools (7 smart + 63 base), TypeScript strict, Koa-style middleware, multi-stage Docker, Claude skill layer with decision tree and examples.*
