# 自定义 Agent

了解如何使用用于静态上下文的规则和用于动态能力的技能来塑造智能体行为。

即使不做任何自定义，Agent 也会尝试通过阅读你的代码来融入。不过，它们不了解你团队的代码偏好、你喜欢的库以及你项目特有的工作流。Cursor 提供了两层自定义方式：**规则（Rules）** 和 **技能（Skills）**。

## 规则：静态上下文

Rules 是位于 `.cursor/rules` 目录下的 Markdown 文件（或在 `cursor settings > General > Rules for AI` 中定义的全局设置）。这些规则在每次对话中都会被包含在内，因此它们非常适合共享团队的规范。

一份好的规则文件应当简短、具体。下面是一个例子：

```markdown
# Commands
- `npm run build`: Build the project
- `npm run typecheck`: Run the typechecker
- `npm run test`: Run tests (prefer single test files for speed)

# Code style
- Use ES modules (import/export), not CommonJS (require)
- Destructure imports: `import { foo } from 'bar'`
- See `components/Button.tsx` for canonical component structure

# Workflow
- Before submitting a PR, always run `npm run lint`.
```

规则在每次对话中都会包含在内。它们非常适合：

- **风格准则**（如使用哪种 Lint 工具或格式化程序）
- **目录结构约定**（如 `/elements` 存放原子组件，`/modules` 存放业务逻辑）
- **项目级知识**（如内部库的用法）

### 规则中应避免的内容

- **经常更改的信息**：如果某些内容每周都在变化，通过规则来同步会很繁琐。
- **单个任务的特定指令**：如果你只想在接下来的几分钟内让 agent 按照某种方式行事，直接在对话中说明即可。
- **非常大的数据集**：这会让上下文变得臃肿。

## Skills：动态上下文

技能帮助 Agent 为其能力学习新工作流。与规则不同，技能是在执行相应操作时按需调用的。

例如，你的团队可能有一个自动创建 PR 的流程：

```markdown
# Create a pull request
description: Gather changes, run tests, and open a PR via GitHub CLI.

1. Review changed files and summarize them
2. Run `npm test` to ensure everything passes
3. Run `gh pr create --title "<Summary>" --body "<Detailed Description>"`
```

这种流程更适合作为**技能**，因为只有当你准备提交代码时它才是相关的。

适合作为技能的其他工作流：

- `/fix-issue <id>`：搜索问题记录并尝试修复
- `/review`：在提交前审查当前更改
- `/update-deps`：检查过时的依赖并运行更新

### 选择正确的方式

如果你希望 Agent **始终知道**某些内容，请使用**规则**。
如果你希望 Agent 学会**执行特定任务**，请使用**技能**。

## MCP：连接到外部工具

模型上下文协议（Model Context Protocol, MCP）赋予了 Agent 与外部系统交互的能力。你可以将 Agent 连接到你的数据库、线性看板（Linear board）或内部 API。这使得 Agent 能够执行如"查看未完成的 Jira 任务"或"查询 Postgres 表结构"等任务。

### 作为 Agent 能力的 CLI 工具

Agent 本身也是一名终端大师。你可以简单地告诉它："你已安装了 AWS CLI。当你需要列出 S3 桶时，请使用 `aws s3 ls`。"通过赋予它对相关 CLI 工具的访问权限，它的能力将得到指数级的增强。

## 前后对比

下面是实际自定义时的效果示例。假设有一个团队使用 Next.js、Tailwind 和 Vitest：

**添加规则前：** Agent 使用 `jest` 进行测试（因为它在训练数据中更常见）、用 CSS Modules 创建组件，并且将 API 路由随意放在各个位置。

**添加三条规则之后：**

```text
- 测试使用 Vitest，而非 Jest。参考 `src/__tests__/example.test.ts` 了解模式。
- 使用 Tailwind 实用类进行样式设置。不使用 CSS modules 或 styled-components。
- API 路由放在 `app/api/[resource]/route.ts` 中，遵循现有模式。
```

现在，Agent 默认会遵守团队的约定。再也不用在每次对话中反复纠正同样的错误。

## 常见失败模式：过度设计规则

你可能会想为所有事情都写规则。克制这种冲动。过多的规则会占用不必要的上下文，还可能让 agent 感到困惑。

让你的规则保持精简且高质量。它们应该是你团队持续维护和更新的共享资产。如果只是偶尔才需要某些内容，把它放到 skill 里，而不是写成规则。

## 下一步

你已经根据团队的工作方式自定义了 agent。在最后一章中，你将通过一个端到端示例把所有内容串联起来，运用你在本课程中学到的全部内容。

---

> 下一篇：[综合运用](07-putting-it-together.md)
