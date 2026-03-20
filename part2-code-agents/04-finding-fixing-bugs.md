# 查找并修复缺陷

随着编码智能体编写的代码越来越多，工程师会花更多时间来审查这些代码并追踪缺陷。即使有编码智能体的帮助，重新巩固高效调试的基础知识，并通过将该过程中的部分环节交给智能体来加速你的工作，依然非常值得。

## 调试基础

无论是人类还是 Agent 来调试，良好的调试都遵循相同的原则：

- **创建可靠的复现。** 如果你无法复现这个 bug，就无法验证修复是否有效。写下触发问题的精确步骤、输入和条件。
- **缩减到最小复现用例。** 去掉所有与 bug 无关的内容。复现越精简，越容易找到根本原因。
- **隔离变量。** 一次只改变一件事。如果你同时改了三处然后 bug 消失了，你就不知道是哪个改动修好了它。
- **提出具体假设。** 想出几种可能的根本原因。"bug 大概在支付代码里"太模糊了。"bug 出现是因为 `calculateTotal()` 没有处理负折扣"就足够具体，可以进行测试。
- **为代码加监测。** 在你怀疑出问题的输入和输出位置添加日志。对比预期值和实际看到的值。
- **用测试防止回归。** 一旦找到并修复了 bug，就写一个本可以捕获它的测试。这样可以防止同样的 bug 再次出现。

## 两种调试方式

### 快速解决简单错误

对于报错信息清晰或原因简单明了的缺陷，智能体往往可以直接定位并修复问题。粘贴错误信息，补充说明它在什么情况下发生，然后交给智能体处理。

> **Agent example: 堆栈追踪调试**
>
> This test is failing:
> `TypeError: Cannot read properties of undefined (reading 'profile')`
> `at getProfile (src/services/UserService.ts:45)`
> `at UserController.show (src/controllers/UserController.ts:23)`
>
> The error happens when a user created before we added the profile onboarding flow tries to view their profile. Find the root cause and fix it.

**Cursor 的响应：**

- 读取 `src/services/UserService.ts`
- 读取 `src/controllers/UserController.ts`
- 思考 2 秒
- 使用 Grep 查找 `prisma.user.findUnique`
- 修改 `AuthMiddleware.ts` +3 -1

**根本原因：** 在 `AuthMiddleware.ts:45` 中的 `findUser` 查询没有包含 profile 关联：`const user = await prisma.user.findUnique({ where: { id } })`。

**修复方案：** 在查询中添加了 `include: { profile: true }`。

**为什么这只影响老用户：** 新用户会经过 `ProfileOnboarding` 流程，该流程会立即创建 profile 记录。老用户没有 profile 记录，导致 `user.profile` 为 undefined。

当错误信息里已经暴露出问题原因时，这种方式效果很好。智能体可以读取堆栈追踪，定位对应代码并进行修补。不过，这种方式并不总是有效，你可能需要采用更系统的方法来查找根本原因。

## 调试模式：先看证据

对于更棘手的 bug，调试模式（Debug Mode）采用了不同的方法。它不会先猜测修复方案，而是先收集运行时证据。

调试模式遵循如下步骤：

1. **生成假设**，判断可能出了什么问题。
2. **为代码加监测**，添加有针对性的日志。
3. **要求你复现 bug**，同时收集数据。
4. **分析日志**，找出根本原因。
5. **进行有针对性的修复**，依据你的证据来处理。

> **Debug mode example: 间歇性故障**
>
> 有些用户的结账会间歇性失败。没有一致的错误信息。有时订单能成功提交，有时会静默失败，用户只会看到一个空白的确认页面。

调试模式可以帮助你找出并修复最棘手的 bug。它将我们前面讲过的基础方法交给 Agent，让 Agent 成为高效的调试器，把原本需要你手动完成的排查过程自动化。

## 将运行时数据引入 Agent 反馈回路

### 从问题入手

你并不总是需要日志或性能分析工具才能开始排查问题。直接向 agent 提问，它会分析你的代码中的常见问题。

> **Agent example: 从代码入手进行性能调试**
>
> 订单历史页面在订单很多的用户那里需要 4 秒才能加载。为什么这么慢？

**Cursor 的响应：**

- 读取 `src/app/api/orders/route.ts`
- 思考 2 秒
- 读取 `src/services/OrderService.ts`
- 修改 `OrderService.ts` +2 -4

**根本原因：** 在 `getOrdersForUser()` 中存在 N+1 查询。该函数获取所有订单，然后循环遍历每个订单并在单独的查询中加载其明细。一个拥有 50 个订单的用户会触发 51 次数据库往返。

**修复方案：** 将循环替换为 Prisma 的 `include: { items: true }`，这会使用 SQL JOIN 在单个查询中加载订单及其明细。

**影响：** 对于拥有 50 个订单的用户，数据库查询从 51 次大幅下降到 1 次。响应时间应从 ~800ms 优化到 ~15ms。

### 向它提供运行时证据

当仅靠代码分析还不够时，给 Agent 提供真实数据。将终端输出、查询日志或其他数据粘贴到对话中。Agent 可以利用这些数据更有效地找出根本原因。

例如，如果你在排查一个很慢的数据库查询，可以对一个慢的 Postgres 查询运行 `EXPLAIN ANALYZE`，粘贴输出结果，Agent 就能把问题追踪回你的表结构：

> **Agent example: EXPLAIN ANALYZE 调试**
>
> 这个查询在生产环境中耗时 1.2 秒。下面是 `EXPLAIN ANALYZE` 的输出：
> ```
> Seq Scan on orders (cost=0.00..45892.00 rows=47 width=244)
>   (actual time=0.423..1206.5ms)
>   Filter: (user_id = 'usr_abc123')
>   Rows Removed by Filter: 2341856
> Planning Time: 0.089 ms
> Execution Time: 1203.298 ms
> ```
> 找出为什么它在执行顺序扫描，并修复它。

**Cursor 的响应：**

- **修复方案：** 通过新的 Prisma 迁移在 `orders.user_id` 上添加了 B-tree 索引。Postgres 将使用索引扫描而不是读取全表。
- **预期改进：** 对 47 行进行索引扫描，而不是对 230 万行进行顺序扫描。查询时间应从 ~1,200ms 下降到 5ms 以下。

同样的模式也适用于应用日志、性能分析数据，或者构建与测试输出。

### 使用浏览器进行前端调试

对于前端问题，Cursor 的集成浏览器能让 agent 直接访问你的 Web 应用。它可以读取控制台日志、检查网络请求并观察 DOM，而无需你手动复制任何内容。

让 agent 打开某个页面、复现问题，并检查控制台中的错误或 Network（网络）面板中的慢请求。agent 能看到你在开发者工具（DevTools）里看到的内容，并可以将问题追溯到你的源代码。

### 使用 MCP 连接监控工具

MCP servers 会为你的 agent 提供新能力，并将它连接到生产环境的可观测性工具。无需再手动粘贴数据，agent 会按需自动拉取所需数据。

> **Agent example: 基于 MCP 的调试**
>
> 自从昨天的部署之后，我们的结账错误突然激增。你能从 Sentry 拉取详细信息并检查 Datadog 日志，帮我看看发生了什么吗？

**Cursor 的响应：**

- 调用 Sentry MCP `get_issue_details(issue_id)`
- 思考 2 秒
- 调用 Datadog MCP `search_logs(service:checkout)`
- 读取 `src/services/PaymentService.ts`
- 修改 `PaymentService.ts` +4 -1

**根本原因：** 昨天的部署更改了产品目录 API 的响应。一些项目现在针对免费项目返回 `price: null` 而不是 `price: 0`。`processPayment()` 中的 `reduce()` 将 `null` 加到总和中，产生 `NaN`，导致支付网关拒绝交易。

**修复方案：** 在 reduce 回调中添加了验证，以捕获价格缺失的项目。同时开启了一个后续任务来修复目录 API，使其针对免费项目返回 `0` 而不是 `null`。

适合调试的 MCP 服务器包括：

- **Sentry**：错误详情、堆栈追踪，以及 breadcrumbs
- **Datadog**：生产环境日志和 APM 追踪
- **数据库**：查询生产数据，用于验证假设
- **Linear 或 GitHub Issues**：将缺陷报告和复现步骤拉入对话

你可以设置工作流，让生产应用上的监控告警自动触发 agent 调查。如果错误率飙升，就会在 Linear 中创建工单，而 agent 会在有人介入之前就开始诊断问题。

## 常见失败模式：接受你没弄懂的修复方案

如果你不理解这个修复，就无法验证它是否正确。Agent 可能加了一个空值检查，让错误消失了，但底层的数据不一致问题仍然存在。

当 Agent 提出修复方案时，要不断追问直到你理解为止。为什么这个值是 null？是什么变化导致了这种情况？这是根本原因，还是我们只是在掩盖症状？正如我们在基础课程中提到的，Agent 可能会编造听起来似乎合理的解释（幻觉）。你需要形成你自己的理解，并根据排查中得到的数据来确认 Agent 是否真正找到了根本原因。

## 下一步

现在你已经了解了调试，以及编码 Agent 如何加快你的排查过程。现在你需要确保这些更改是正确的，并且不会引入新的问题。在下一章中，你将学习如何进行代码评审和系统化测试。

---

> 下一篇：[审查和测试代码](05-reviewing-testing.md)
