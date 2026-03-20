# 开发新功能

现在你已经理解了代码库，可以开始构建一些新的功能了。

要用 agent 交付功能，关键在于把工作拆分为 agent 能自行校验的步骤。为每个重要功能先制定一个计划，然后设置好合适的护栏，让 agent 能够自己发现并修复错误。

## 从计划开始

Agent 可以在你开始写代码之前，帮你搞清楚要构建什么。

在写代码之前，其实有很多决策要先搞清楚。如果你有一个功能想法，可能希望先做一个简单版本，之后再迭代；或者需要考虑一些具体的设计取舍。

你可以在写代码之前使用编码 agent 帮你梳理这些决策。借助 Cursor 的**计划模式（Plan mode）**，agent 会分析你的代码库、向你提出澄清问题，并生成一个你可以编辑和修改的分步计划。

### 澄清问题

当你向计划模式提交提示词时，agent 会先向你提问，以先弄清楚需求：

> **Questions**
>
> 通知偏好应该存储在哪里？
> 1. 数据库（UserPreferences 表）
> 2. localStorage（仅客户端）
> 3. 数据库 + localStorage 缓存

### 生成结构化计划

在你回答后，agent 会生成一个结构化的计划，包含它在构建功能过程中可以审查和验证的里程碑。这个计划是可编辑的，如果有哪里不对，你可以直接改：

> **Plans > notification-preferences.md [Build]**
>
> **Notification preferences**
>
> **Overview**
> 在用户设置中添加通知偏好页面。用户可以按类别（营销、产品更新、安全提醒）切换邮件、推送和站内通知。偏好存储在数据库中，并使用乐观 UI 更新。
>
> **Approach**
> - 遵循 `src/pages/Settings.tsx` 中现有的设置页布局
> - 使用 `UserPreferences` 表中的 JSONB 列来保存通知配置
> - 复用 UI 库中已有的 Toggle 组件
>
> **5 Tasks**
> - ✅ 向 UserPreferences 表添加 `notification_preferences` 列
> - ○ 创建 NotificationPreferences 组件，遵循 `Settings.tsx` 的模式
> - ○ 在 `app/api/user/notifications/route.ts` 添加 API 路由
> - ○ 使用现有的 `useOptimistic` hook 接好乐观更新
> - ○ 为偏好切换和 API 校验添加测试

Cursor 通过将较大的请求拆分为更小、可独立验证的步骤，让计划真正有用。在每一步中，agent 都可以衡量自己的进度，确认该步已成功完成，然后继续往下做。

## 何时该重来

有时 agent 构建的东西偏离了预期。与其尝试通过后续提示来修修补补，不如回到最初的计划。撤销这些更改，把计划写得更具体，然后再执行一次。

例如，如果你遗漏了一条关键的设计说明，计划可能就会构建出错误的东西。从计划开始重来听起来有点反直觉，但往往比在错误方向上打补丁更快。

## 使用 Agent 的测试驱动开发（TDD）

1. **先写测试**。让 agent 根据期望的输入和输出编写测试。要明确说明你在做 TDD，这样它就不会为尚不存在的代码创建 mock 函数。
2. **确认测试失败**。让 agent 运行测试并验证测试确实失败。此时你并不是在编写功能代码。
3. **提交测试**。当你对测试覆盖率和质量都满意时，提交它们。这会锁定你的需求，agent 需要基于这些需求来实现功能。
4. **让 agent 编写代码**。告诉它在不修改测试的前提下让所有测试通过。它会持续迭代直到全部通过。
5. **提交代码**。审查输出，确认行为符合预期，然后提交。

### TDD 第一步：先写测试

```
Write tests for a discountCode() function that:
- Returns the discounted price when given a valid code
- Throws InvalidCodeError for expired codes
- Applies fixed-amount discounts correctly (e.g., "10OFF" = $10 off)
- Never returns a negative price (floor at $0)

Follow the test patterns in `src/__tests__/pricing.test.ts`.
DO NOT write the function yet.
```

### TDD 第二步：编写代码让测试通过

一旦测试已经提交，就让 agent 编写代码。要明确说明：它应该在不修改测试的前提下让所有测试通过。

```
Make all tests in `src/__tests__/discountCode.test.ts` pass.
Follow the service patterns in `src/services/PricingService.ts`.
DO NOT modify the tests.
```

### 为什么 TDD 与 Agent 配合效果好？

因为 agent 可以运行测试、查看失败结果、调整代码并重试。每次运行测试都会给它具体的反馈。如果没有测试，它就无法知道自己所做的代码更改是否有效。

这种方式对后端代码尤其有价值，因为你没办法通过看界面来验证正确性。你在测试中描述预期行为的过程，其实就是在为 agent 提供一份完美的规格说明。

## 从设计到代码

智能体可以处理并理解图像。你可以将截图或模型图直接粘贴到提示输入框中，智能体会根据你的图像还原对应的设计。

适用于以下场景：

- **Mockups**：粘贴线框图或 Figma 导出文件，让智能体构建对应组件
- **Visual debugging**：截取异常的 UI 状态，让智能体帮你排查问题
- **Iteration**：对当前结果截图，并描述需要更改的内容

你还可以连接 **Figma MCP server**，让智能体直接从你的 Figma 文件中提取 design tokens、变量和组件规范。

**Integrated browser** 让你可以在智能体进行更改时预览结果。借助这个浏览器，智能体可以在页面间导航、截图，并验证自己的视觉输出，这样你就不需要再手动把截图发回给智能体了。

## 常见失败模式：未经验证就开始构建

快速开发功能时，最大的风险是跳过验证。Agent 可以很快生成大量代码，但只有速度而没有正确性，后续只会让工作量增加。

下面是一些可以帮助 Agent 验证自己工作的具体方式：

- 针对逻辑和行为的**测试**
- 针对结构正确性的**类型检查**
- 用于强制执行代码风格和模式的**代码规范检查工具（linter）**
- 用于获取 UI 变更反馈的**浏览器工具或 MCP 服务器**

如果 Agent 无法验证自己的输出，你最终会花更多时间在修正错误上。

## 下一步

你已经上线了一个功能。但软件总会有 Bug，而且有些非常棘手。在下一章中，你将学习如何系统化地使用 Agents 来查找和修复 Bug。

---

> 下一篇：[查找并修复 Bug](04-finding-fixing-bugs.md)
