# 综合运用

恭喜，你已经学会了如何与编码 agent 协作。

你现在知道如何使用 agent 来理解代码库、快速创建功能、在代码中搜索并找到 bug、更高效地审查代码，以及根据自己的需求自定义 agent。本章将通过一个示例展示如何把这些能力综合运用起来。

## 端到端示例

你加入了一个运营电商平台的团队，需要在结账流程中添加折扣码功能。代码库对你来说很陌生，而且这个功能必须在本周末前上线。

这是一个非常现实的场景：你**"构建前先探索，写代码前先规划，上线前先测试，合并前先评审"**。

## 步骤 1：理解代码库

在修改任何代码之前，先探索结账流程。使用 agent 来建立你的心智地图。

> **Ask mode example: 探索结账流程**
>
> I need to add discount codes to our checkout flow. Before I make any changes, walk me through how checkout works. What components, API routes, and services are involved? Show me the data flow from the cart to payment confirmation.

**Cursor 的响应：**

- **Searched codebase** checkout flow components and services
- Read `src/pages/Checkout.tsx`
- Read `src/services/OrderService.ts`
- Read `src/services/PricingService.ts`

**Checkout flow: cart to payment confirmation**

1. **Cart page** (`src/pages/Cart.tsx`) — 用户查看商品，点击 "Checkout"
2. **Checkout page** (`src/pages/Checkout.tsx`) — 收集收货地址和支付方式
3. **Order summary** (`src/components/OrderSummary.tsx`) — 显示行项目、小计、税费和总计。调用 `PricingService.calculateTotal()`
4. **Payment** — `Checkout.tsx:handleSubmit()` 调用 `OrderService.createOrder()`，进而调用 `PaymentService.charge()`
5. **Confirmation** — 成功后跳转到 `/orders/[id]/confirmation`

**折扣逻辑适配点：** `PricingService.calculateTotal()` 第 34 行从小计开始计算商品价格。折扣步骤应放在小计计算与税费计算（第 41 行）之间。`OrderSummary` 组件需要新增一行显示折扣金额。

---

通过进一步提问来补全信息空白：

> **Ask mode example: 找出折扣逻辑的位置**
>
> How does our pricing calculation work in `src/services/PricingService.ts`? Where do we compute the order total, and where would a discount be applied in that flow?

**Cursor 的响应：**

**Pricing calculation flow:**

```
calculateTotal(items) → {
  subtotal = sum(items.map(i => i.price * i.quantity))  // line 34
  tax = subtotal * taxRate                                // line 41
  total = subtotal + tax + shippingCost                   // line 45
}
```

**折扣插入点：** 在第 34 行和第 41 行之间。在计算税费前将折扣应用于 `subtotal`，这样税费将针对折扣后的金额进行计算。

**重要提示：** `PricingService` 也被 `InvoiceService` 和 `QuoteService` 调用。任何折扣逻辑都需要处理不应用折扣的情况（默认路径应保持不变）。

这会将我们在[理解你的代码库](02-understanding-your-codebase.md)中使用的 grep 和语义搜索结合起来，用于追踪结账流程，然后总结它找到的内容。

## 步骤 2：规划功能

在你已经熟悉代码库之后，先制定一个计划。切换到 Plan Mode（`Shift+Tab`），并概述要做的工作：

> **Plans: discount-codes.md [Build]**
>
> **折扣码**
>
> **概述**
> 为结账流程添加折扣码支持。用户输入折扣码，系统进行校验，然后在税前将折扣应用到订单总额。
>
> **实现方案**
> - 添加 `DiscountService`，并实现校验逻辑（过期时间、使用次数限制）
> - 修改 `PricingService.calculateTotal()` 以接受可选的折扣参数
> - 在 `OrderSummary` 组件中添加折扣码输入框
> - 添加 `/api/discount/validate` 接口
>
> **任务**
> - [ ] 创建 `discount_codes` 表的迁移脚本
> - [ ] 创建带有 `applyDiscount()` 的 `DiscountService`
> - [ ] 在 `PricingService.calculateTotal()` 中添加折扣处理步骤
> - [ ] 在 `OrderSummary` 组件中添加折扣码输入 UI
> - [ ] 添加 `/api/discount/validate` 路由
> - [ ] 为所有折扣类型和边界情况编写测试

检查这个计划。删掉对第一版来说过于激进的步骤。专注于核心流程：应用折扣码、计算折扣金额，并在订单汇总中展示。然后点击 **Build** 以创建该功能的第一个版本。

## 步骤 3：调试一个失败的边界情况

构建完成后，有一个测试始终失败：叠加百分比折扣和固定金额折扣时，计算出的总金额不正确。对同一组折扣，只是应用顺序不同，就会得到不同的结果。

使用调试模式，或让 agent 调查根本原因：

> **Agent example: 调试根本原因**
>
> The discount stacking test is failing. The same discounts produce different totals depending on application order. Find out why and fix the ordering of discount application in `src/services/DiscountService.ts`. Show me the before and after values at each step.

**Cursor 的响应：**

- Read `src/services/DiscountService.ts`
- Thought for 3s
- Edited `DiscountService.ts` (+8, -4)

**根本原因：** 折扣按到达顺序应用，这使得结果取决于接收顺序。先应用固定金额（$40 - $30 = $10，再打 50% 折 = $5）与先应用百分比（$40 × 0.5 = $20，再减 $30 = -$10 → 取 $0）会得到不同的结果。

**修复方案：** 对折扣进行排序，确保始终先应用百分比折扣，再应用固定金额折扣。这确保了百分比是针对全额计算的，且现有的 $0 阈值逻辑能处理折扣超过总额的情况。

```
Before: $40 → -$30 = $10 → ×0.5 = $5    (依赖于顺序)
After:  $40 → ×0.5 = $20 → -$30 → 取 $0  (一致)
```

所有折扣测试现在均已通过。

这体现了[查找和修复 Bug](04-finding-fixing-bugs.md) 章节中的调试原则：不要只修复表面症状，而要找到真正的原因。

## 步骤 4：复查与测试

在推送之前，先做一次自查：

> **Ask mode example: 推送前自我检查**
>
> Review all the changes I've made for the discount code feature. Check for bugs, missing error handling, edge cases, and consistency with existing patterns. What would a code reviewer flag?

**Cursor 的响应：**

- Ran `git diff --staged`
- Searched codebase `discount and pricing patterns`

**Review summary:**
- 4 个文件已更改，142 行新增，3 行删除
- **New:** `DiscountService.ts`, `discountCode.test.ts`, `api/discount/validate/route.ts`
- **Modified:** `PricingService.ts`（在 `calculateTotal` 中添加了折扣步骤）

**查找到的问题：**
1. **/api/discount/validate 缺少频率限制** — 增加 `rateLimit(10, '1m')` 以匹配其他校验接口。
2. **缺少索引** — `discount_codes` 表通过 `code` 列进行查询，但没有建立索引。在大规模使用时会变慢。

**确认无误：** 测试覆盖、错误处理模式、折扣排序逻辑、总价 $0 阈值。

如果你已经配置了 cloud agents，可以启动几个来测试你可能遗漏的边缘情况：例如折扣码中的 Unicode 字符、很大的折扣值，或并发使用折扣码的情况。

## 步骤 5：编写规则

你在本节中学到了一点：在进行折扣计算时，应始终先应用百分比折扣，再应用固定金额折扣，这样无论输入顺序如何，结果都能保持一致。把这点写成一条规则，这样智能体下次就会知道该怎么做：

```markdown
# 定价规则
- 先应用百分比折扣，再应用固定金额折扣
- 订单总额不得低于 $0
- 参见 `src/services/PricingService.ts` 了解标准计算模式
```

至此，一个完整的闭环完成了。你已经完成了探索、规划、构建、测试、调试、评审，并为未来沉淀了知识。

## 课程总结

你已经完成了折扣码功能。一个队友请你下周添加礼品卡功能。你的第一步应该是什么？

- ❌ 立刻开始编写礼品卡功能代码，因为结账流程很相似。
- ✅ **先让 Agent 探索礼品卡会如何与现有的折扣和支付系统交互，然后再制定计划。**
- ❌ 复制折扣码功能的实现，然后修改为礼品卡功能。

最好的学习方式，就是把你在这里学到的内容应用起来。

工具和模型在快速演进，具体实践会随时间改变，但这项核心能力——学会如何与编码智能体协作——会持续复利增长。它正在成为软件工程中最重要的技能之一。

通过不断挑战编码智能体的能力边界，你就能在新工具和新模型发布时，最大程度发挥它们的价值。祝你好运，并保持好奇心！
