# Quy Chuẩn Code — Haravan MCP

---

## Ngôn ngữ & Framework

| Stack | Version | Vai trò |
|-------|---------|---------|
| TypeScript | 5.x | Ngôn ngữ chính, strict mode |
| Node.js | ≥18 | Runtime |
| Express | 5.x | HTTP transport |
| Commander.js | 13.x | CLI framework |
| Zod | 3.x | Schema validation |
| Axios | 1.x | HTTP client |
| @modelcontextprotocol/sdk | 1.x | MCP framework |

---

## Cấu trúc thư mục

```
src/
├── cli.ts                    # CLI entry — Commander.js commands
├── index.ts                  # Public API exports
├── auth/
│   ├── oauth.ts              # OAuth 2.0 flow (browser → callback → token)
│   └── token-store.ts        # Token persistence (~/.haravan-mcp/)
├── mcp-server/
│   ├── init.ts               # Server init: register tools, build middleware
│   └── transport/
│       ├── stdio.ts           # stdin/stdout transport
│       └── sse.ts             # HTTP streamable transport
├── mcp-tool/
│   ├── types.ts               # McpTool, MiddlewareContext interfaces
│   ├── registry.ts            # allTools[], PRESETS, filterTools()
│   ├── presets.ts             # resolveToolFilter() logic
│   ├── handler.ts             # Default Haravan API handler
│   ├── middleware/
│   │   ├── chain.ts           # composeMiddleware() — Koa-style onion
│   │   ├── error-handler.ts   # Catch + format errors
│   │   ├── rate-limiter.ts    # Haravan leaky bucket compliance
│   │   ├── pagination.ts      # Auto-paginate when fetchAll=true
│   │   └── validation.ts      # Input sanitization
│   └── tools/
│       ├── customers.ts       # 14 customer + address tools
│       ├── orders.ts          # 13 order + transaction tools
│       ├── products.ts        # 11 product + variant tools
│       ├── inventory.ts       # 5 inventory tools
│       ├── shop.ts            # 6 shop/location/user tools
│       ├── webhooks.ts        # 3 webhook tools
│       ├── content.ts         # 11 content tools
│       └── smart/
│           ├── helpers.ts     # fetchAll, clientFromCtx, ok, err
│           ├── orders-smart.ts    # 3 order aggregation tools
│           ├── customers-smart.ts # 1 RFM segmentation tool
│           └── inventory-smart.ts # 3 inventory analysis tools
└── utils/
    ├── http-client.ts         # HaravanClient class (Axios wrapper)
    ├── config.ts              # mergeConfig()
    └── logger.ts              # Logger with levels
```

---

## Quy tắc viết Tool

### Base Tool (1:1 API mapping)

```typescript
export const myTools: McpTool[] = [
  {
    name: 'haravan_resource_action',      // prefix: haravan_
    project: 'resource_category',          // group name
    description: 'Mô tả ngắn gọn bằng English',
    schema: z.object({                     // Zod schema cho input
      resource_id: z.number().describe('Resource ID'),
      fields: z.string().optional().describe('Comma-separated fields'),
    }),
    httpMethod: 'GET',                     // GET | POST | PUT | DELETE
    path: '/com/resources/{resource_id}.json',  // Haravan API path
    scopes: ['com.read_resources'],        // Required OAuth scopes
  },
];
```

### Smart Tool (aggregation)

```typescript
export const mySmartTools: McpTool[] = [
  {
    name: 'hrv_analysis_name',            // prefix: hrv_
    project: 'smart',                      // always 'smart'
    description: 'Mô tả English — nêu rõ output gì',
    schema: z.object({ /* input params */ }),
    httpMethod: 'GET',
    path: '/smart',                        // placeholder — not used
    scopes: ['com.read_orders'],
    customHandler: myAnalysisHandler,      // bypass middleware chain
  },
];
```

### Naming Convention

| Prefix | Dùng cho | Ví dụ |
|--------|---------|-------|
| `haravan_` | Base tools (1:1 API) | `haravan_orders_get` |
| `hrv_` | Smart tools (aggregation) | `hrv_orders_summary` |

### Smart Tool Handler Pattern

```typescript
async function myHandler(ctx: MiddlewareContext) {
  try {
    const client = clientFromCtx(ctx);
    const { items, apiCalls } = await fetchAll(client, path, key, params);
    // ... aggregate logic ...
    return ok({ result, _meta: { api_calls: apiCalls, generated_at: new Date().toISOString() } });
  } catch (e: any) {
    return err(`tool_name failed: ${e?.message ?? e}`);
  }
}
```

---

## Quy tắc Middleware

- Middleware theo Koa-style: `async (ctx, next) => { ... await next() ... }`
- Luôn call `next()` trừ khi muốn short-circuit
- Error handler là middleware ngoài cùng (catch-all)
- Smart tools (có `customHandler`) bypass middleware chain

---

## Quy tắc Haravan API

| Quy tắc | Chi tiết |
|---------|---------|
| Rate limit | Check `X-Haravan-Api-Call-Limit` header. Sleep khi >60/80 |
| Pagination | Max 50/page. Dùng `page` param. Break khi `items.length < 50` |
| Fields filtering | Luôn truyền `fields` param để giảm response size |
| Date filtering | Dùng `created_at_min` / `created_at_max` (ISO 8601) |
| Error handling | 401 = token expired, 429 = rate limited (check Retry-After), 500 = server error |

---

## Testing

```bash
npm test              # Run Jest tests
npm run test:watch    # Watch mode
npm run test:coverage # Coverage report
```

---

## Formatting

```bash
npm run format        # Prettier format all .ts files
```

**Prettier config**: Default settings, single quotes, no semicolons (follow existing codebase).

---

---

## Quy chuẩn Claude Skill

### SKILL.md Frontmatter Format

```markdown
---
name: haravan-mcp
description: "Activate khi user hỏi về cửa hàng Haravan: doanh thu, đơn hàng, tồn kho, khách hàng, sản phẩm"
---
```

- `name`: kebab-case, khớp với tên thư mục trong `~/.claude/skills/`
- `description`: mô tả rõ trigger conditions để Claude biết khi nào activate

### References Directory Structure

```
claudeskill/haravan-mcp/
├── SKILL.md                     # Main file — decision tree, rules, output templates
└── references/
    ├── mcp-tools.md             # Tool catalog với "Claude tự làm từ output này"
    ├── insights-formulas.md     # Công thức + ngưỡng benchmark
    └── examples.md              # Ví dụ output hoàn chỉnh (ít nhất 3-5 examples)
```

**Quy tắc references:**
- `mcp-tools.md`: mỗi tool 1 entry — tên, params quan trọng, output fields, cách Claude dùng
- `insights-formulas.md`: format `Formula = A / B × 100`, kèm benchmark và threshold
- `examples.md`: output thực tế không lược bỏ — dùng để Claude học format

### Output Format Standards

**Số tiền VND:**
```
✅ 125,400,000 VND    (dấu phẩy phân cách nghìn)
✅ 125.4M VND         (viết tắt M = triệu, B = tỷ)
❌ 125400000          (không có dấu phân cách)
❌ 125,400,000đ       (không dùng ký hiệu đ)
```

**Trend indicators:**
```
↑ 12.3%   — tăng (màu xanh trong context)
↓ 5.1%    — giảm (màu đỏ trong context)
→ 0.0%    — không đổi
```

**Alert levels:**
```
✅ Tốt      — metric đạt benchmark
⚠️ Cảnh báo — metric cần chú ý (warning zone)
🔴 Nguy hiểm — metric vượt ngưỡng nguy hiểm
```

**Markdown table format:**
```markdown
| Metric | Giá trị | vs Kỳ trước | Benchmark | Trạng thái |
|--------|---------|-------------|-----------|-----------|
| DT | 125.4M VND | ↑12.3% | — | ✅ |
| Cancel Rate | 3.8% | ↑0.9pp | <3% | ⚠️ |
| COD Fail | 22% | ↓3pp | <15% | 🔴 |
```

### Insight Quality Standards

Mỗi insight PHẢI đáp ứng 3 tiêu chí:

1. **Actionable** — có hành động cụ thể (không chỉ "nên xem xét")
2. **Numbered** — có con số thực tế từ data, không ước chừng mơ hồ
3. **Impact** — ước tính tác động kinh doanh (VND tiết kiệm, % cải thiện, N khách phục hồi)

```
Checklist kiểm tra insight trước khi xuất:
☐ Metric cụ thể được nêu tên
☐ Con số thực tế từ data (không phải placeholder)
☐ So sánh với benchmark hoặc kỳ trước
☐ Nguyên nhân được xác định (nếu có thể)
☐ Hành động cụ thể với deadline / người thực hiện
☐ Impact ước tính (VND, %, số lượng)
```

---

*English: TypeScript strict mode, Koa-style middleware, Zod validation, Haravan API rate limit compliance, smart tool pattern with fetchAll helper.*
