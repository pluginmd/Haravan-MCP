# Hướng Dẫn Thiết Kế — MCP Server & Claude Skill

---

## Tư duy phát triển MCP Application

### Nguyên tắc #1: MCP Server là "cánh tay", Claude Skill là "bộ não"

```
❌ SAI: MCP server tính toán mọi thứ, trả insight cho Claude đọc lại
✅ ĐÚNG: MCP server fetch + aggregate data, Claude tự phân tích + viết insight
```

**Lý do**: Claude AI hiểu context, ngành, và ngôn ngữ tốt hơn bất kỳ server logic nào. Server chỉ nên làm những gì Claude KHÔNG THỂ tự làm (pagination lớn, batch API calls, full-population scoring).

### Nguyên tắc #2: Tránh overlap giữa 2 lớp

| Nếu Claude có thể... | Thì ĐỪNG viết smart tool cho nó |
|----------------------|-------------------------------|
| Group by 5 categories từ 1 JSON | ❌ Không cần server-side grouping |
| Filter cancelled orders từ status field | ❌ Không cần cancel_analysis tool |
| Tính % từ 2 con số | ❌ Không cần server-side percentage |
| Viết insight bằng tiếng Việt | ❌ Không cần server-side templating |

| Nếu cần... | Thì BẮT BUỘC viết smart tool |
|-------------|------------------------------|
| Fetch 2000 orders (40 pages) | ✅ Server pagination + rate limiting |
| Score 9000 khách hàng theo quintile | ✅ Full population cần sort toàn bộ |
| Check inventory 200 variants | ✅ 200 API calls + throttling |

### Nguyên tắc #3: Token efficiency

```
Không tối ưu:
  Claude gọi haravan_orders_list page=1 → 50 orders (5000 tokens)
  Claude gọi haravan_orders_list page=2 → 50 orders (5000 tokens)
  ... × 40 pages = 200,000 tokens input
  Claude tự tổng hợp: "Tổng DT = X, AOV = Y"

Tối ưu:
  Claude gọi hrv_orders_summary → 300 tokens output
  Claude viết insight: "Doanh thu tăng 12%..."

Tiết kiệm: 99.85% tokens = 99.85% chi phí
```

### Nguyên tắc #4: Smart tool output phải compact

- Return summary, KHÔNG return raw records
- Target: 200-800 tokens per response
- Include `_meta` (api_calls_used, generated_at) cho transparency
- Strip null/empty values

---

## Thiết kế Smart Tool

### Khi nào nên viết smart tool mới?

Hỏi 3 câu:

1. **Claude có thể tự làm từ base tools không?**
   - Nếu CÓ → không cần smart tool. Hướng dẫn trong SKILL.md thay thế
   - Nếu KHÔNG (cần pagination lớn, batch calls, full-population) → viết smart tool

2. **Output size bao nhiêu?**
   - <1000 tokens → smart tool hợp lý
   - >5000 tokens → đang return quá nhiều raw data, cần aggregate thêm

3. **Có overlap với smart tool khác không?**
   - Nếu data nguồn giống nhau (cùng fetch orders) → xem xét merge
   - Mỗi smart tool nên có data source RIÊNG hoặc logic aggregation KHÁC BIỆT

### Template smart tool

```typescript
// 1. Handler function
async function mySmartHandler(ctx: MiddlewareContext) {
  try {
    // Parse params
    const { date_from, date_to } = ctx.params;
    const from = parseDate(date_from, 30);
    const to = date_to ?? new Date().toISOString();

    // Fetch data (server handles pagination + rate limiting)
    const client = clientFromCtx(ctx);
    const { items, apiCalls } = await fetchAll(client, '/com/orders.json', 'orders',
      { created_at_min: from, created_at_max: to, status: 'any' },
      { fields: 'id,total_price,financial_status' }  // ONLY needed fields
    );

    // Aggregate (the value-add of smart tool)
    const result = computeMetrics(items);

    // Return compact summary
    return ok({
      period: { from, to },
      ...result,
      _meta: { api_calls_used: apiCalls, generated_at: new Date().toISOString() },
    });
  } catch (e: any) {
    return err(`my_tool failed: ${e?.message ?? e}`);
  }
}

// 2. Tool definition
export const myTool: McpTool = {
  name: 'hrv_my_analysis',       // hrv_ prefix
  project: 'smart',
  description: 'English description — what it returns, not how',
  schema: z.object({
    date_from: z.string().optional().describe('Start date ISO 8601'),
    date_to: z.string().optional().describe('End date ISO 8601'),
  }),
  httpMethod: 'GET',
  path: '/smart',
  scopes: ['com.read_orders'],
  customHandler: mySmartHandler,
};

// 3. Register in registry.ts
import { myTool } from './tools/smart/my-smart';
export const allTools = [...existingTools, myTool];
```

---

## Thiết kế Claude Skill

### Nguyên tắc viết insight (SAI vs ĐÚNG)

```
❌ SAI — chung chung, không actionable:
  "Doanh thu đang tốt"
  "Tỷ lệ hủy có dấu hiệu tăng"
  "Nên xem lại tồn kho"

✅ ĐÚNG — công thức đầy đủ:
  [Metric] + [Con số] + [Context] + [Nguyên nhân] → [Hành động] + [Impact]

  "Cancel rate tăng 2.9% → 3.8% (↑0.9 điểm, cao hơn benchmark 3%).
   6/16 đơn hủy do inventory: AT-M-W, QJ-32-B, SN-42-W.
   Revenue at Risk: ~8.5M VND/tuần.
   → Nhập khẩn 3 SKU: 40 + 26 + 27 units. Lead time 7 ngày → hành động hôm nay."

  "Catalog Health Score: 62/100 (dưới benchmark 80).
   Thiếu: 23 sản phẩm không có ảnh, 31 sản phẩm thiếu mô tả.
   Impact: -18% conversion rate ước tính theo benchmark Haravan.
   → Ưu tiên 10 sản phẩm top-seller: thêm ảnh + mô tả trước cuối tuần."
```

### Công thức Insight

```
Insight = [Metric] + [Con số] + [Context] + [Nguyên nhân] → [Hành động] + [Impact]

Checklist trước khi viết:
✅ Có metric cụ thể? (Cancel Rate, Revenue, AOV...)
✅ Có con số thực tế? (3.8%, 8.5M VND, 23 sản phẩm...)
✅ Có context / trend? (tăng từ X, cao hơn benchmark Y)
✅ Có nguyên nhân? (do inventory, do COD, do seasonal...)
✅ Có hành động cụ thể? (nhập N units, gọi điện N khách...)
✅ Có impact ước tính? (tiết kiệm X VND, phục hồi Y% DT...)
```

### Khi nào thêm Smart Tool mới vs để Skill tự xử lý

| Tình huống | Quyết định | Lý do |
|-----------|-----------|-------|
| Cần fetch >100 pages data | Viết smart tool | Claude không thể loop API |
| Cần score toàn bộ dataset (quintile) | Viết smart tool | Full population cần server-side sort |
| Cần batch 200+ API calls (inventory) | Viết smart tool | Rate limiting phức tạp |
| Group by 5 categories từ JSON có sẵn | Để Skill xử lý | Claude đọc output và tự group |
| Tính % từ 2 con số | Để Skill xử lý | 1 phép tính đơn giản |
| Viết insight + format markdown | Để Skill xử lý | Claude AI làm tốt hơn template |
| Filter theo status từ 1 response | Để Skill xử lý | Không cần thêm API call |

### Cách thêm kịch bản phân tích mới vào SKILL.md

```markdown
## Bước 1: Thêm vào Decision Tree
Xác định trigger phrase → tools cần gọi → params mặc định

## Bước 2: Viết Output Template
### [Tên Kịch Bản]
| Metric | Giá trị | Trend | Benchmark | Trạng thái |
|--------|---------|-------|-----------|-----------|
| ...    | ...     | ...   | ...       | ✅/⚠️/🔴  |

**Insights:**
1. [Insight theo công thức đầy đủ]
2. ...

**Actions:**
- [ ] Hành động ưu tiên cao (deadline, người thực hiện)

## Bước 3: Thêm vào references/examples.md
Viết 1 ví dụ output hoàn chỉnh với data thực tế mẫu

## Bước 4: Cập nhật references/mcp-tools.md
Nếu kịch bản dùng tool mới → ghi rõ "Claude tự làm từ output này"
```

### Cấu trúc SKILL.md

```markdown
---
name: skill-name
description: "Khi nào activate skill này"
---

# Skill Title

## Quy tắc bắt buộc
- Tool priority (smart > base)
- Parallel calling
- Date range rules
- Write confirmation

## Decision Tree
- Câu hỏi loại A → gọi tools X, Y, Z
- Câu hỏi loại B → gọi tools P, Q

## Output Templates
- Bảng markdown
- Insight format: [Metric] + [Con số] + [Action]

## Anti-patterns
- Những gì KHÔNG ĐƯỢC làm
```

### Nguyên tắc viết SKILL.md

1. **Explicit decision tree**: Mỗi loại câu hỏi → chỉ rõ tools nào, params gì
2. **Parallel by default**: Gọi song song khi tools không phụ thuộc nhau
3. **Max 6 calls**: Nếu cần >6 → đang gọi sai, dùng smart tool thay thế
4. **Reuse data**: Turn trước đã có data → đừng gọi lại
5. **Actionable insights**: Metric + con số + hành động cụ thể + impact ước tính

### Insight Quality Framework

```
❌ Chung chung: "Doanh thu đang tốt"
❌ Thiếu action: "Tỷ lệ hủy tăng 3.8%"
❌ Thiếu impact: "Nên nhập thêm hàng"

✅ Xuất sắc:
"Cancel rate tăng 2.9% → 3.8% (↑0.9 điểm).
 6/16 đơn hủy do hết hàng (inventory): AT-M-W, QJ-32-B, SN-42-W.
 Mất ~8.5M VND DT/tuần.
 → Nhập khẩn 3 SKU: 40 + 26 + 27 units. Lead time 7 ngày."
```

### References folder

```
claudeskill/my-skill/
├── SKILL.md              # Main skill definition
└── references/
    ├── mcp-tools.md      # Tool catalog + "Claude tự làm từ output này"
    ├── insights-formulas.md  # Formulas + benchmarks + thresholds
    └── examples.md       # 5-6 ví dụ output hoàn chỉnh
```

---

## Tư duy phát triển MCP nâng cao

### Data Flow Pipeline

```
User question
  → Skill decision tree (chọn tools)
    → MCP tool calls (parallel khi có thể)
      → Server pagination + aggregation
        → Haravan API calls (rate limited)
      ← Compact JSON response
    ← Combine multiple tool results
  ← Claude analysis + insight + formatting
← User-facing output (markdown tables, trends, actions)
```

### Cost Optimization

| Scenario | Không tối ưu | Tối ưu | Tiết kiệm |
|----------|-------------|--------|-----------|
| "Doanh thu tháng này" | 40 API calls, 250K tokens | 1 smart tool, 300 tokens | 99.9% |
| "Scorecard toàn diện" | 200 API calls, 1M tokens | 4 smart tools, 2K tokens | 99.8% |
| "Phân tích khách hàng" | 100 API calls, 500K tokens | 1 smart tool, 800 tokens | 99.8% |

### Scalability Pattern

```
Thêm resource mới (VD: Haravan thêm API mới):
1. Viết base tool (1:1 mapping) trong src/mcp-tool/tools/new-resource.ts
2. Register trong registry.ts
3. Thêm preset nếu cần
4. Update SKILL.md decision tree
5. Nếu cần aggregation lớn → viết smart tool

KHÔNG viết smart tool cho:
- Simple CRUD operations
- Single-record lookups
- Aggregation đơn giản mà Claude tự làm được
```

---

*English: This document covers MCP application design philosophy — server as "hands" vs skill as "brain", token efficiency, smart tool design patterns, Claude skill writing guidelines, and cost optimization strategies.*
