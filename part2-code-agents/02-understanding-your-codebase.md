# 理解你的代码库

软件工程师最重要的工作之一，就是为代码库构建清晰的心智模型，并深入理解系统是如何运作的。随着项目规模增长，找到代码会变得越来越困难。

过去，要在大型代码库中查找内容，往往需要记住正则表达式模式，或者学习专门的工具来进行高效搜索。借助智能体，你可以用自然语言描述要找的内容，再由智能体调用工具为你定位。

Cursor 为智能体提供了专门的搜索工具，理解这些工具的工作方式，将帮助你提出更好的问题，并获得更准确的结果。

## Agentic 搜索

查找某段代码最精确的方式，是搜索精确匹配的内容，无论是函数名、变量名，还是其他代码片段。Agent 可以使用 `grep`（用于查找精确字符串的工具），并且能够创建更复杂的正则表达式模式或单词边界匹配。基于 `grep` 也出现了一些改进工具，比如 `ripgrep`，可以进行递归搜索。

这两个工具都非常好用。不过，Cursor 通过 **Instant Grep** 进一步增强了 `grep`，在大型代码库上执行 Agentic 搜索时，相比 `ripgrep` 能显著提速。

## 语义搜索

除了精确匹配，智能体还可以利用**语义搜索**来寻找在概念上相关的代码，即便你没有用到确切的关键词。语义搜索是通过理解你问题的意图（"含义"），然后将其与代码库中含义相似的部分进行匹配。

理解何时使用这两种搜索方式是提出好问题的关键。我们来看一些例子。

### 示例 1：定向搜索

在第一个例子中，代理从 `grep` 开始，因为提示请求的是具体的内容。它会搜索 `import.*PaymentService`，以找到每个引用该服务的文件。

> **Ask mode example: 定向搜索**
>
> 查找所有从我们的 `PaymentService` 导入的文件，并展示它们是如何处理 `PaymentFailedError` 的。

**Cursor 的响应：**

- Grepped `import.*PaymentService`
- Grepped `PaymentFailedError`
- Read `src/api/routes/payments.ts`（捕获错误并返回 402 及错误码）
- Read `src/api/routes/subscriptions.ts`（捕获错误，记录日志，重试一次后失败）
- Read `src/workers/BillingWorker.ts`（捕获错误，加入死信队列等待人工审核）
- Read `src/services/OrderService.ts`（让错误冒泡到路由处理程序）
- Read `src/services/RefundService.ts`（**注意**：未捕获该错误——潜在 Bug：退款失败会静默崩溃）

### 示例 2：广泛探索

当你在探索不熟悉的领域时，就要把问题问得更宽泛一些。在下一个例子中，我们提出了一个关于应用如何处理支付失败的一般性问题。注意第一个工具调用是 "Search codebase"。这是代理在使用语义搜索来找到相关文件，然后再用 `grep` 去补全细节。

> **Ask mode example: 广泛探索**
>
> 我们的应用是如何处理支付失败的？请从结账表单开始，一步步讲解错误流程直到用户看到的错误信息。

**Cursor 的响应：**

- **Searched codebase** for "payment failure handling flow"
- Found `CheckoutForm.tsx`, `useCheckout.ts`, and `error-mapping.ts`
- Read `src/components/CheckoutForm.tsx`
- Grepped for `submitPayment`
- Read `src/hooks/useCheckout.ts`
- ...（沿着 API 和错误处理程序追踪完整流程）

## Explore 子代理

Agent 还可以创建子代理来更高效地完成任务。内置的 **Explore 子代理**（Explore Subagent）可以帮助你搜索整个代码库。Explore 子代理在独立于父 agent 的上下文窗口中运行，并使用更快的模型，因此它可以执行大量并行搜索，而不会让主对话变得臃肿。

你不必手动调用它。agent 会在认为相关时自动使用。不过，如果你愿意，也可以直接请求使用这个子代理。

正如我们在基础课程中提到的，理解并关注你的上下文占用非常重要。如果你在代码库中搜索很多文件，就会生成大量上下文。子代理可以通过只返回它们的关键信息、让主对话保持聚焦，从而显著优化你的上下文管理。

## 架构图

对于大型或不熟悉的代码库，你可以让 Agent 生成架构图，比如 **Mermaid 图表**，帮助你可视化你的代码库。

> **Ask mode example: 架构图**
>
> Create a Mermaid diagram showing the data flow for our payment system,
> including the checkout form, API routes, payment service, and Stripe integration.

**Cursor 的响应：**

- Searched codebase: payment system architecture
- Read `src/pages/Checkout.tsx`
- Read `src/services/PaymentService.ts`
- Read `src/api/routes/payments.ts`
- Grepped `stripe.*webhook`
- Read `src/api/webhooks/stripe.ts`
- 生成 Mermaid 图表：`Checkout Form → API Route → Payment Service → Stripe API`

这些图对于新人上手、文档编写和设计评审都很有用。它们也可以揭示架构问题，比如某个服务依赖了太多其他服务，或者数据流经了一条出乎意料的路径。

## 常见失败模式：尚未理解就开始修改

一个常见错误是，在还没搞清楚现有代码的情况下就让 agent 修改代码。agent 可能会在已有工具函数的前提下再创建一个新的，或者使用与现有代码库不一致的模式。

在请求修改之前，先让 agent 帮你了解情况：

> **Ask mode example: 先探索再修改**
>
> 在进行任何修改之前，先向我展示我们现有的表单校验是如何工作的。
> 我们使用了哪些模式？通用的校验器在哪里？

## 下一步

你已经可以查找并理解代码了。在下一节中，我们将学习如何开发新功能。

---

> 下一篇：[开发新功能](03-creating-features.md)
