# 审查和测试代码

编码智能体可以生成大量代码，但这也意味着它们可能带来技术债务。快速推进固然很好，但你仍然需要保持高质量标准。无论代码是手写的还是由智能体生成的，你对可以合并进主分支的标准都应该保持一致。

AI 生成的代码看起来可能是正确的，但在细节上可能存在问题。它也许会遵循现有模式、能够编译并通过你编写的测试，但仍然可能漏掉边界情况、存在安全隐患，或者重复实现你代码库中其他地方已经存在的逻辑。

这就是代码审查如此重要的原因。你需要建立合适的流程，来确保代码库的高质量，并在问题进入生产环境之前将其发现。作为工程师，你有责任投入精力，让代码审查真正发挥作用。

## 自我评审

在请别人帮你查看代码之前，你应该先自己审查一遍。

### 观察 Agent 的工作过程

diff 视图会实时显示变更。如果你发现 agent 方向走偏了，点击 **Stop** 或按下 `Cmd Shift Backspace` 以取消并重定向。不必等它完全结束。对于需要大幅调整方向的情况，可以先撤销这些更改，先完善你的计划再重新运行，这一点我们已经在 [开发功能](03-creating-features.md) 中讲过。

### 让 Agent 一次性审查所有更改

在提示中标记 `@Branch`，让 agent 获取当前分支的完整 diff。像「review the changes on this branch」或「what am I working on right now?」这样的请求，可以为 agent 提供更丰富的上下文，并帮助它在多个不同文件中发现问题。

例如，你可以让 agent 审查自己的工作：

> **Ask mode example: 自我审查**
>
> Review the changes I've made to the discount code feature. Look for bugs, missing error handling, and anything that doesn't match our patterns in `src/services/PricingService.ts`

**Cursor 的响应：**

- Ran `git diff --staged`
- Read `src/services/DiscountService.ts`
- Searched codebase `similar discount and pricing patterns`

**Review of discount code changes:**

**潜在问题：**
1. Missing input validation (`DiscountService.ts:23`) — `applyDiscount()` 没有检查 `price` 是否为负数。`PricingService.ts` 中的其他定价函数会在顶部验证这一点。
2. No rate limiting — `/api/discount/validate` 端点可以被无限制调用。攻击者可能暴力破解有效的折扣码。其他认证相关端点使用了 `rateLimit(10, '1m')`。
3. Inconsistent error codes — 过期码返回 `400`，无效码返回 `404`。结账前端期望所有折扣错误都是 `422`。

**确认无误：**
- 百分比折扣在固定金额折扣之前应用（顺序正确）
- 价格最低为 $0，防止负数总额
- 所有折扣类型都有测试覆盖
- 遵循 `PricingService.ts` 中的现有服务模式

### 预判评审者问题

> **Ask mode example: 预判评审者问题**
>
> What questions will reviewers have about these changes? What context should I include in the PR description?

**Cursor 的响应：**

**可能的评审者问题：**
1. "为什么新建一个 DiscountService 而不是添加到 PricingService？"
   - 回答：PricingService 已经超过 600 行；这种方式可以隔离折扣逻辑
   - 添加到 PR 描述中：「为遵循单一职责原则而提取」
2. "集成测试在哪里？"
   - 单元测试已有，但没有 API 级别的测试
   - 考虑添加：POST /api/discount/validate，覆盖有效/过期/无效码
3. "折扣是在税前还是税后应用？"
   - 当前行为：税前（折扣减少小计，然后对折扣后的金额计税）
   - 添加到 PR 描述中，这样评审者就不需要通读代码

**建议的 PR 描述：**

> Adds discount code support to checkout. Users can enter codes for percentage or fixed-amount discounts. Codes validate against expiry dates and usage limits. Discounts apply before tax.
>
> Extracted into DiscountService (separate from PricingService) since discount logic has different validation needs. Integration tests in follow-up PR.

## 为同伴评审做好准备

Agent 一次可以生成大量代码更改，这可能会变成一个包含数百行变更的大提交，任何人都很难审查。

我们建议使用小而语义清晰的提交，并配上明确的描述。每个提交只代表一个逻辑上的更改。人工评审者可以逐个查看提交历史，而不是面对一大块难以消化的代码差异。

这类提交"清理工作"手动完成很繁琐，但 Agent 很擅长处理。例如：

1. 自由地构建功能。在迭代时，不用在意提交规范。
2. 一切都正常工作后，请 Agent 将提交历史重组为便于评审的若干部分。
3. Agent 会重置到 `main`，读取所有更改，并规划一套合理的顺序，创建干净的提交和有描述性的提交信息。
4. 它会校验最终 diff 与最初的更改一致，确保你的更改没有丢失。

使用这个提示创建一个技能，这样你团队里的任何人都可以在完成一个功能后运行 `/rework-commits`：

```markdown
Create a skill file at `.cursor/skills/rework-commits/SKILL.md` with this content:
# Split branch into reviewable commits
Rework a branch into a series of small, logical, and easy-to-review commits.
```

## 智能体评审

对于所有本地更改，打开 Source Control 选项卡并运行 **Agent Review**，将其与你的主分支进行比较。这样可以捕获整个更改集中的问题。

这类似于手动提示智能体评审你的更改。我们精心设计了一个提示词，使其能更有效地为你服务。

## 用于拉取请求的 Bugbot

Bugbot 会与代码托管平台集成，自动审查拉取请求。它是这一类直接在 PR 上给出反馈的工具之一，而且这一类别正在迅速增长。

Bugbot 会在你推送代码时审查 PR。它会读取你改动的完整上下文，包括修改后的代码如何与其余代码库关联，并查找可能进入生产环境的缺陷。不同于主要用于捕捉格式问题的 linter，Bugbot 能发现逻辑错误：空指针异常、竞态条件、缺失的错误处理以及安全问题。

当 Bugbot 发现问题时，它还可以提出修复方案。启用 autofix 后，你可以直接从拉取请求的评论中提交修复。

你还可以通过提供**额外规则**来自定义 Bugbot，我们会在下一节关于自定义 Agent 的内容中进一步介绍。

## 下一步

你已经学会了如何审查和测试 agent 生成的代码。在下一章中，你将学习如何通过规则和技能来自定义 agent 的行为。

---

> 下一篇：[自定义 Agent](06-customizing-agents.md)
