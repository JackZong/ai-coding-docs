# 与 Agents 协作

如今开发者正借助 agents 来写**大量**代码。与其手动输入每一行，他们会与 agent 对话，让它来帮他们写代码。

在 [Foundations](../part1-ai-fundamentals/06-agents.md) 课程中，我们了解了 agents 背后的核心行为机制：为它们提供强大的工具，并让它们在一个循环中自主运行。在本课程中，我们将介绍在构建软件时如何高效地与 coding agents 协作的策略。

## 什么是 Agent 框架？

Cursor 是一款内置编码 agent 的 AI 编辑器。这个 agent 运行在一个称为"框架"的环境中，它由三部分组成：

- **Instructions**：用于引导行为的 system 提示词和规则
- **Tools**：文件编辑、代码库搜索、终端执行等能力
- **Model**：你为任务选择的 agent 模型

编码 agent 会根据你设定的目标来帮助你完成任务。每个框架的行为都会略有不同，具体取决于它的调优方式以及你使用的模型。

有些模型经过训练后会更频繁地调用 shell 命令，另一些则需要更明确的指令。借助 Cursor，我们的目标是支持所有前沿模型，并尽可能优化框架，包括它们可以使用的工具。

## 编写高效的提示词

你开始与 agents 协作时，第一步入口就是你给它们的提示词。来看两种不同的方式：

### 模糊提示词（Vague Prompt）

```
Add a user settings page
```

在这种情况下，agent 几乎什么都得自己猜：你想要什么布局、用哪些组件、采用什么样式方案，等等。有时这样也能奏效，但很多时候你需要更具体地说明你的意图。

### 约束提示词（Constrained Prompt）

再来看一个更详细的提示词，它引用了你代码库中已有的模式：

```
Add a user settings page.

Look at the existing profile page in `src/app/profile/page.tsx` for our
layout pattern. Use the same form components from `src/components/ui/Form.tsx`.

Settings should include:
• Display name (text input)
• Email notifications (toggle)
• Theme preference (dropdown: light, dark, system)

Store settings using our existing `useUserPreferences` hook. Follow the same
API route pattern as `src/app/api/user/profile/route.ts`.
```

第二个提示词要好得多，因为它给了 agent 以你的代码库为基础的明确指示：现有文件、组件，以及清晰的范围。agent 会遵循你已有的模式，而不是自己发明新的做法。

> **交互思考：哪种方式能让 agents 获得最佳结果？**
>
> - ❌ 用笼统的方式描述你想要什么，让 agent 自己搞清楚细节。
> - ✅ **引用具体文件、现有模式，并明确界定范围边界。**
> - ❌ 把项目中的每个文件都包含进来，以提供最大限度的上下文。
> - ❌ 为 agent 编写伪代码，让它翻译成真实代码。

## 管理你的上下文

当你与 agent 协作时，你的对话会不断累积上下文：消息、工具调用、文件内容等等。这个上下文就是 agent 的工作记忆，而它是有限的。

要留意 agent 正在使用多少上下文。当你切换到新任务时，或者发现 agent 开始出错时，就开启一段新的对话。

如果你仍在处理同一个功能，而且 agent 从之前的消息中已经积累了有用的上下文，那么继续当前对话是值得的。但如果 agent 一直在兜圈子，即使你还在这个功能处理中，也应当重新开始。你可以引用旧对话，让 agent 读取聊天记录。

### 引用过去的对话

```
Continue the auth refactor from "Red auth refactor". I've addressed the
review feedback on the JWT expiry handling. Now update the refresh token
rotation to invalidate old tokens on use.
```

最新的模型已经非常擅长帮你查找上下文了。当配备了像语义搜索这样的工具时，agent 可以按需拉取相关文件。如果你知道确切的文件，就标记它。否则，就给 agent 一个大致描述，让它自己找到合适的文件。

## 常见失败模式：需求不断膨胀

最常见的错误之一，是在没有充分规划的情况下，一次性要求改动范围过大。你可能会发现智能代理在做无关的修改、编辑你并不想改动的文件，或者逐渐失去重点。

如果你看到这种情况，先停下来，想一想是否可以把任务拆分成更小的部分。如果你在规划一个较大的功能，我们会在 [开发新功能](03-creating-features.md) 中讲解如何编写可靠的方案和规格说明。一旦你有了这个方案并开始第一次对话，之后通过一系列更小、更聚焦的后续对话迭代，往往会更快。

## 下一步

你已经了解了代理框架的工作原理、如何思考并编写高效的提示词，以及如何管理上下文。在下一章中，我们将更深入地介绍代理如何搜索你的代码库，并帮助你更好地理解你正在处理的代码和架构。

---

> 下一篇：[理解你的代码库](02-understanding-your-codebase.md)
