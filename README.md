# Oh-My-OpenCode 集成分析

## 项目概述

**oh-my-opencode** 是一个 OpenCode 的增强插件系统，核心理念是：
- 把 LLM agent 当成"开发团队"来管理
- 主 agent（Sisyphus）是 tech lead，其他 agent 是专家成员
- 通过 hooks、agents、skills、MCPs 四层架构实现功能扩展

---

## 核心架构

### 1. Agents（专家团队）

内置 10+ 个专家 agent，每个有独立职责：

| Agent | 模型 | 职责 | 权限 |
|-------|------|------|------|
| **Sisyphus** | Claude Opus 4.5 | 主编排者，任务分解和调度 | 全权限 |
| **oracle** | GPT-5.2 | 架构设计、代码审查、调试 | 只读 |
| **librarian** | OpenCode big-pickle | 文档查询、开源实现搜索 | 只读 |
| **explore** | GPT-5-nano | 快速代码库探索、grep | 只读 |
| **multimodal-looker** | Gemini 3 Flash | 图片/PDF 分析 | 只读 |
| **Prometheus** | Claude Opus 4.5 | 战略规划（面试模式） | 规划 |
| **Metis** | Claude Sonnet 4.5 | 计划顾问（预规划分析） | 规划 |
| **Momus** | Claude Sonnet 4.5 | 计划审查 | 规划 |

**调用方式：**
```
Ask @oracle to review this design
Ask @librarian how is this implemented in React
Ask @explore for all authentication code
```

**后台执行：**
```python
delegate_task(
    agent="explore",
    background=True,
    prompt="Find all API endpoints"
)
# 继续工作...
background_output(task_id="bg_abc123")
```

---

### 2. Skills（专业技能包）

Skills = 专业工作流 + 内嵌 MCP + 详细指令

内置 skills：
- **playwright**: 浏览器自动化（测试、截图、爬虫）
- **git-master**: Git 专家（原子提交、rebase、blame、bisect）
- **frontend-ui-ux**: UI/UX 设计师人格

**Skill 结构：**
```markdown
---
description: Browser automation skill
mcp:
  playwright:
    command: npx
    args: ["-y", "@anthropic-ai/mcp-playwright"]
---

# Skill 指令内容
...
```

---

### 3. Hooks（生命周期钩子）

先纠正一下：**不是**。我前面给你的“核心 Hooks 列表”只是代表性的核心项，**不是整个项目用到的完整列表**。

按源码看，`oh-my-opencode` 里和 Hooks 相关的东西可以分成 3 层：

1. **31 个可配置的内置 hooks / hook flags**（来自 `src/config/schema.ts` 的 `HookNameSchema`）
2. **2 个内部始终启用的辅助 hook**（不在 `disabled_hooks` 里，但实际参与运行）
3. **Claude Code 兼容层**（当前只兼容 5 个官方 Claude hook 事件）

> 这意味着：如果你要把 oh-my-opencode 的行为集成到你自己的 agent 项目里，不能只照着 Claude Code 官方 hooks 文档来做；你还需要实现 oh-my-opencode 自己依赖的宿主事件。

#### 3.1 全量 hook / hook flag 清单

以下是 `src/config/schema.ts` 里可通过 `disabled_hooks` 控制的 **31 个名字**：

- `todo-continuation-enforcer`
- `context-window-monitor`
- `session-recovery`
- `session-notification`
- `comment-checker`
- `grep-output-truncator`
- `tool-output-truncator`
- `directory-agents-injector`
- `directory-readme-injector`
- `empty-task-response-detector`
- `think-mode`
- `anthropic-context-window-limit-recovery`
- `rules-injector`
- `background-notification`
- `auto-update-checker`
- `startup-toast`
- `keyword-detector`
- `agent-usage-reminder`
- `non-interactive-env`
- `interactive-bash-session`
- `thinking-block-validator`
- `ralph-loop`
- `compaction-context-injector`
- `claude-code-hooks`
- `auto-slash-command`
- `edit-error-recovery`
- `delegate-task-retry`
- `prometheus-md-only`
- `sisyphus-junior-notepad`
- `start-work`
- `atlas`

另外还有 **2 个内部辅助 hook**，虽然不在 `disabled_hooks` 列表里，但实际参与运行：

- `question-label-truncator`
- `task-resume-info`

#### 3.2 需要注意的“名字不等于独立 hook”

这个项目里有几个名字会让人误会，得单独标出来：

| 名字 | 实际情况 |
|---|---|
| `startup-toast` | 不是独立源码 hook，更像 `auto-update-checker` 的一个子功能开关 |
| `grep-output-truncator` | 当前代码里主要由 `tool-output-truncator` 承担，文档/配置里保留了旧名字兼容 |
| `preemptive-compaction` | 旧名字，已移除；迁移代码会把它过滤掉 |
| `empty-message-sanitizer` | 旧名字，已移除 |

#### 3.3 oh-my-opencode 真正依赖的宿主 hook 点

从 `src/index.ts` 看，oh-my-opencode 不是只靠 Claude 风格 hooks 跑的，它依赖的是 OpenCode 插件宿主的这些 hook 点 / 生命周期：

- `chat.message`
- `tool.execute.before`
- `tool.execute.after`
- `event`（内部再分很多 event.type）
- `experimental.session.compacting`
- `experimental.chat.messages.transform`

也就是说，如果你自己的 agent 框架里**没有这几类宿主事件**，你不能直接“兼容 oh-my-opencode hooks”，你得先把事件总线做出来。

#### 3.4 按宿主事件分组的完整映射

##### A. `chat.message`

这一类基本对应“用户消息刚提交、模型还没真正处理前”的阶段。

| hook / 模块 | 作用 | 更接近 Claude 哪个事件 |
|---|---|---|
| `keyword-detector` | 检测 `ultrawork` / `ulw` / `analyze` 等关键词并改写行为 | `UserPromptSubmit` |
| `claude-code-hooks` | 执行 Claude Code 兼容层里的 `UserPromptSubmit` hooks | `UserPromptSubmit` |
| `auto-slash-command` | 识别 `/xxx` 风格命令并改写 prompt | 无直接官方等价 |
| `start-work` | 给 Sisyphus 注入启动工作流 | 无直接官方等价 |
| `ralph-loop` | 检测并启动 Ralph Loop | 无直接官方等价 |

##### B. `tool.execute.before`

这一类对应“工具调用真正执行前”。这是最接近 Claude `PreToolUse` 的一层。

| hook / 模块 | 作用 | 对应 Claude |
|---|---|---|
| `question-label-truncator`（内部） | 截断过长问题标签 | 无直接官方等价 |
| `claude-code-hooks` | 执行 `PreToolUse` | `PreToolUse` |
| `non-interactive-env` | 处理无 TTY / 非交互环境 | `PreToolUse` 的内部策略化扩展 |
| `comment-checker`（before 半段） | 提前记录上下文，准备检查注释 | `PreToolUse` / 内部前置准备 |
| `directory-agents-injector` | 在文件/目录访问前注入 AGENTS.md | 更接近 `InstructionsLoaded`，但不是官方实现 |
| `directory-readme-injector` | 注入 README.md | 更接近 `InstructionsLoaded`，但不是官方实现 |
| `rules-injector` | 按 glob 注入规则 | 更接近 `InstructionsLoaded`，但不是官方实现 |
| `prometheus-md-only` | 对 planner 的工具使用施加限制 | 无直接官方等价 |
| `sisyphus-junior-notepad` | 给特定 agent 附加行为 | 无直接官方等价 |
| `atlas` | 编排器在工具前做状态干预 | 无直接官方等价 |

##### C. `tool.execute.after`

这一类对应“工具调用成功返回后”。最接近 Claude `PostToolUse`。

| hook / 模块 | 作用 | 对应 Claude |
|---|---|---|
| `claude-code-hooks` | 执行 `PostToolUse` | `PostToolUse` |
| `tool-output-truncator` | 截断工具输出，防 context 爆炸 | `PostToolUse` |
| `context-window-monitor` | 在工具返回后检查 context headroom | `PostToolUse` |
| `comment-checker`（after 半段） | 检查变更是否引入 AI 注释污染 | `PostToolUse` |
| `directory-agents-injector` | 清理/补充目录注入状态 | 内部扩展 |
| `directory-readme-injector` | 清理/补充 README 注入状态 | 内部扩展 |
| `rules-injector` | 清理/补充规则注入状态 | 内部扩展 |
| `empty-task-response-detector` | 检测 task 工具返回空结果 | 无直接官方等价 |
| `agent-usage-reminder` | 在合适时提醒使用专家 agent | `PostToolUse` 风格提醒 |
| `interactive-bash-session` | 跟踪交互式 shell / tmux 会话状态 | 无直接官方等价 |
| `edit-error-recovery` | Edit 工具失败后的恢复逻辑 | 更接近 `PostToolUseFailure` 的内部替代 |
| `delegate-task-retry` | delegate_task 失败时自动重试 | 无直接官方等价 |
| `atlas` | 编排器在工具后更新状态 | 无直接官方等价 |
| `task-resume-info`（内部） | 给 task 连续执行提供恢复信息 | 无直接官方等价 |

##### D. `event`

这是 oh-my-opencode 很关键的一层。很多“看起来像 hook”的功能，其实并不是挂在 Claude 式的 5 个 hook 上，而是监听内部 runtime event。

常见 `event.type` 包括：

- `session.created`
- `session.updated`
- `session.deleted`
- `session.error`
- `message.created`
- `message.updated`
- `tool.execute.before`
- `tool.execute.after`

依赖这层的模块包括：

| hook / 模块 | 依赖的内部事件 | 作用 | 对应 Claude |
|---|---|---|---|
| `auto-update-checker` | `session.created` | 启动时检查更新 / toast | 最接近 `SessionStart` |
| `background-notification` | 后台任务管理事件 | 后台 agent 完成时通知 | 最接近 `Notification`，但不是 Claude 官方实现 |
| `session-notification` | `session.created/updated`、`message.created/updated`、`tool.execute.*`、`session.deleted` | 会话空闲/恢复时发系统通知 | 最接近 `Notification` / `TeammateIdle`，但不是官方实现 |
| `todo-continuation-enforcer` | `session.error`、`message.updated`、`tool.execute.*`、`session.deleted` | 任务未完成时自动续跑 | 最接近 `Stop` + `StopFailure` 的组合替代 |
| `context-window-monitor` | event 总线 + 工具后 | 监控 token headroom | 无直接官方等价 |
| `directory-agents-injector` | `session.deleted` 等 | 清理目录注入缓存 | 无直接官方等价 |
| `directory-readme-injector` | `session.deleted` 等 | 清理 README 注入缓存 | 无直接官方等价 |
| `rules-injector` | `session.deleted` 等 | 清理规则缓存 | 更接近 `InstructionsLoaded` 配套缓存 |
| `think-mode` | `session.deleted` 等 | 管理 thinking mode 状态 | 无直接官方等价 |
| `anthropic-context-window-limit-recovery` | session 级错误 / 停止相关 | 自动恢复超上下文错误 | 最接近 `StopFailure` 的恢复策略 |
| `agent-usage-reminder` | `session.deleted` 等 | 维护提醒状态 | 无直接官方等价 |
| `interactive-bash-session` | `session.deleted` 等 | 清理交互式 bash 状态 | 无直接官方等价 |
| `ralph-loop` | `session.error`、`session.deleted` | 管理 loop 生命周期 | 无直接官方等价 |
| `atlas` | `session.error`、`message.updated`、`session.deleted`、`tool.execute.*` | 主编排器状态机 | 无直接官方等价 |
| `session-recovery`（核心插件逻辑中手动调用） | `session.error` | 会话异常恢复 | 最接近 `StopFailure` |
| `claude-code-hooks`（部分状态管理） | `session.error`、`session.deleted` | 为 Claude Stop/interrupt 兼容层维护状态 | `Stop` / `StopFailure` 配套 |

##### E. `experimental.session.compacting`

| hook / 模块 | 作用 | 对应 Claude |
|---|---|---|
| `claude-code-hooks` | 执行 Claude `PreCompact` hooks | `PreCompact` |
| `compaction-context-injector` | 在压缩前补上下文 | `PreCompact` 的内部增强 |

##### F. `experimental.chat.messages.transform`

| hook / 模块 | 作用 | 对应 Claude |
|---|---|---|
| `thinking-block-validator` | 校验 `<thinking>` block，防 API 错误 | 无直接官方等价 |
| `context injector`（功能模块，不在 disabled_hooks） | 调整 message 注入上下文 | 无直接官方等价 |

#### 3.5 和 Claude Code 官方 hooks 的逐项对比

Claude 官方文档里的 hook 事件比 oh-my-opencode 当前兼容层要大得多。**oh-my-opencode 当前真正显式兼容的只有 5 个 Claude 官方事件**：

- `UserPromptSubmit`
- `PreToolUse`
- `PostToolUse`
- `Stop`
- `PreCompact`

下面是逐项对比：

| Claude 官方事件 | oh-my-opencode 当前状态 | 你自己的项目若想兼容，需要什么 |
|---|---|---|
| `SessionStart` | **未直接实现 Claude 兼容 hook**；内部最接近 `session.created` | 需要 session 生命周期事件 |
| `UserPromptSubmit` | **已实现**（通过 `chat.message`） | 需要“用户 prompt 提交前”钩子 |
| `PreToolUse` | **已实现**（通过 `tool.execute.before`） | 需要工具调用前钩子 |
| `PermissionRequest` | **未实现** | 如果你有权限弹窗/审批系统，要单独做 |
| `PostToolUse` | **已实现**（通过 `tool.execute.after`） | 需要工具成功后钩子 |
| `PostToolUseFailure` | **未显式实现 Claude 兼容层**；内部有 `edit-error-recovery` 和 `session.error` 风格恢复 | 如果你想完整对齐 Claude，最好单独补一个失败后钩子 |
| `Notification` | **未直接实现 Claude 兼容 hook**；内部有 `background-notification` / `session-notification` | 需要通知事件总线 |
| `SubagentStart` | **未实现 Claude 兼容 hook** | 如果你有子 agent，要补子 agent 生命周期事件 |
| `SubagentStop` | **未实现 Claude 兼容 hook** | 同上 |
| `TaskCreated` | **未实现 Claude 兼容 hook** | 如果你有 task API，要补 task 生命周期事件 |
| `TaskCompleted` | **未实现 Claude 兼容 hook** | 同上 |
| `Stop` | **已实现** | 需要“本轮 assistant 响应结束/即将 idle”事件 |
| `StopFailure` | **未显式实现 Claude 兼容 hook**；内部最接近 `session.error` | 建议补独立失败结束事件 |
| `TeammateIdle` | **未实现 Claude 兼容 hook** | 如果你有 agent team，建议补 |
| `InstructionsLoaded` | **未直接实现 Claude 兼容 hook**；内部近似物是 `directory-agents-injector` / `rules-injector` | 需要“规则/说明文件加载”事件 |
| `ConfigChange` | **未实现** | 需要配置变更事件 |
| `CwdChanged` | **未实现** | 需要工作目录切换事件 |
| `FileChanged` | **未实现** | 需要文件 watch 事件 |
| `WorktreeCreate` | **未实现** | 若支持 worktree，需单独补 |
| `WorktreeRemove` | **未实现** | 同上 |
| `PreCompact` | **已实现**（通过 `experimental.session.compacting`） | 需要 compaction 前事件 |
| `PostCompact` | **未实现** | 如果你做 compaction，建议补 |
| `Elicitation` | **未实现** | 如果你支持 MCP elicitation，要单独做 |
| `ElicitationResult` | **未实现** | 同上 |
| `SessionEnd` | **未直接实现 Claude 兼容 hook**；内部最接近 `session.deleted` | 需要 session 结束事件 |

#### 3.6 结论：你自己的 agent 项目最少要实现哪些 hooks 事件？

如果你的目标是：**先把 oh-my-opencode 里最有价值的能力接进来**，而不是 100% 复刻 Claude Code 官方 hooks，全局上我建议你至少先做这 **6 类宿主事件**：

1. `chat.message`
   - 支撑：`keyword-detector`、`start-work`、`auto-slash-command`
   - 也能映射 Claude `UserPromptSubmit`

2. `tool.execute.before`
   - 支撑：`claude-code-hooks(PreToolUse)`、`directory-agents-injector`、`rules-injector`、`non-interactive-env`

3. `tool.execute.after`
   - 支撑：`tool-output-truncator`、`comment-checker`、`edit-error-recovery`、`delegate-task-retry`
   - 也能映射 Claude `PostToolUse`

4. `event` 总线（至少这些事件类型）
   - `session.created`
   - `session.updated`
   - `session.deleted`
   - `session.error`
   - `message.created`
   - `message.updated`
   - 支撑：`todo-continuation-enforcer`、`session-recovery`、`session-notification`、`atlas`

5. `experimental.session.compacting`
   - 支撑 Claude `PreCompact`
   - 支撑 `compaction-context-injector`

6. `experimental.chat.messages.transform`
   - 支撑 `thinking-block-validator`
   - 这个不是 Claude 官方 hooks，但 oh-my-opencode 自己会用

如果你的目标是：**让用户在你自己的项目里也能直接使用 Claude Code 风格 hooks 配置**，那你至少要补齐下面这 **5 个 Claude 官方事件**，因为 oh-my-opencode 当前兼容层就是围绕它们写的：

- `UserPromptSubmit`
- `PreToolUse`
- `PostToolUse`
- `Stop`
- `PreCompact`

#### 3.7 你真正需要实现的：宿主事件 → hooks 覆盖表

如果你是从工程实现角度看，这里最重要的不是记 31 个名字，而是明确：

> **一个宿主事件，会扇出覆盖多个 hooks**。

你自己的 agent 项目里，建议按下面这个映射来设计。

##### A. 用户提交信息后（可命名为 `onUserPromptSubmit` / `chat.message`）

**会覆盖的 hooks：**
- `keyword-detector`
- `claude-code-hooks(UserPromptSubmit)`
- `auto-slash-command`
- `start-work`
- `ralph-loop`

**作用：**
- 识别 `ultrawork` / `ulw` 关键词
- 执行 Claude 风格的 `UserPromptSubmit`
- 识别 `/command` 风格命令
- 启动特定工作流

##### B. 工具调用前（可命名为 `onBeforeToolUse` / `tool.execute.before`）

**会覆盖的 hooks：**
- `question-label-truncator`（内部辅助）
- `claude-code-hooks(PreToolUse)`
- `non-interactive-env`
- `comment-checker`（前置阶段）
- `directory-agents-injector`
- `directory-readme-injector`
- `rules-injector`
- `prometheus-md-only`
- `sisyphus-junior-notepad`
- `atlas`

**作用：**
- Claude 风格 `PreToolUse`
- 在读文件/读目录前注入规则和上下文
- 对特定 agent 或特定工具调用加限制
- 在真正执行工具前做宿主级预处理

##### C. 工具调用后（可命名为 `onAfterToolUse` / `tool.execute.after`）

**会覆盖的 hooks：**
- `claude-code-hooks(PostToolUse)`
- `tool-output-truncator`
- `context-window-monitor`
- `comment-checker`（后置阶段）
- `directory-agents-injector`
- `directory-readme-injector`
- `rules-injector`
- `empty-task-response-detector`
- `agent-usage-reminder`
- `interactive-bash-session`
- `edit-error-recovery`
- `delegate-task-retry`
- `atlas`
- `task-resume-info`（内部辅助）

**作用：**
- Claude 风格 `PostToolUse`
- 截断工具输出，控制上下文体积
- 检查编辑结果、注释污染、空 task 返回
- 失败自动恢复 / 自动重试 / 状态推进

##### D. 本轮响应结束 / 即将 idle（可命名为 `onTurnStop`）

**会覆盖的 hooks：**
- `claude-code-hooks(Stop)`
- `todo-continuation-enforcer`
- `session-notification`

**作用：**
- Claude 风格 `Stop`
- 任务没做完时自动续跑
- agent 空闲时通知用户

> 这个事件很关键。没有它，`todo-continuation-enforcer` 基本跑不起来。

##### E. 会话开始 / 恢复（可命名为 `onSessionStart`）

**会覆盖的 hooks / 行为：**
- `auto-update-checker`
- `session-notification`（初始化）
- `atlas`（初始化）

**作用：**
- 初始化 session 级状态
- 启动时检查更新和提示
- 初始化编排器状态

##### F. 会话异常 / 本轮失败（可命名为 `onSessionError`）

**会覆盖的 hooks：**
- `session-recovery`
- `anthropic-context-window-limit-recovery`
- `todo-continuation-enforcer`
- `atlas`
- `ralph-loop`
- `claude-code-hooks`（Stop/interrupt 兼容状态）

**作用：**
- 做异常恢复
- 上下文窗口超限时恢复
- 避免错误导致任务半路烂尾

##### G. 会话结束（可命名为 `onSessionEnd` / `session.deleted`）

**会覆盖的 hooks / 状态清理：**
- `session-notification`
- `directory-agents-injector`
- `directory-readme-injector`
- `rules-injector`
- `think-mode`
- `interactive-bash-session`
- `ralph-loop`
- `atlas`
- `todo-continuation-enforcer`
- `agent-usage-reminder`

**作用：**
- 清理缓存
- 清理 session 级状态
- 清理 loop / tmux / 通知状态

##### H. 压缩前（可命名为 `onPreCompact` / `experimental.session.compacting`）

**会覆盖的 hooks：**
- `claude-code-hooks(PreCompact)`
- `compaction-context-injector`

**作用：**
- Claude 风格 `PreCompact`
- 在压缩前补关键上下文

##### I. 消息发送前 transform（可命名为 `transformMessages`）

**会覆盖的 hooks / 模块：**
- `thinking-block-validator`
- `context injector`（内部功能模块）

**作用：**
- 在消息真正发给模型前做最终修正
- 防止 `<thinking>` block 导致 API 错误

##### J. 内部 runtime event bus（必须有）

除了上面这些显式 hook 点，你自己的宿主最好还要有一个事件总线，至少能发这些事件：

- `session.created`
- `session.updated`
- `session.deleted`
- `session.error`
- `message.created`
- `message.updated`
- `tool.execute.before`
- `tool.execute.after`

因为 `todo-continuation-enforcer`、`session-notification`、`atlas`、`session-recovery` 这些关键能力，并不只是靠 Claude 风格 hooks 跑，而是同时依赖内部 runtime event。

##### K. 一张最短总结表

| 你自己的宿主事件 | 能覆盖的主要 hooks |
|---|---|
| 用户提交信息后 | `keyword-detector`, `claude-code-hooks(UserPromptSubmit)`, `auto-slash-command`, `start-work`, `ralph-loop` |
| 工具调用前 | `claude-code-hooks(PreToolUse)`, `non-interactive-env`, `comment-checker(pre)`, `directory-agents-injector`, `directory-readme-injector`, `rules-injector`, `prometheus-md-only`, `sisyphus-junior-notepad`, `atlas` |
| 工具调用后 | `claude-code-hooks(PostToolUse)`, `tool-output-truncator`, `context-window-monitor`, `comment-checker(post)`, `empty-task-response-detector`, `agent-usage-reminder`, `interactive-bash-session`, `edit-error-recovery`, `delegate-task-retry`, `atlas` |
| 本轮停止 / idle | `claude-code-hooks(Stop)`, `todo-continuation-enforcer`, `session-notification` |
| 会话开始 / 恢复 | `auto-update-checker`, `session-notification(init)`, `atlas(init)` |
| 会话异常 | `session-recovery`, `anthropic-context-window-limit-recovery`, `todo-continuation-enforcer`, `atlas`, `ralph-loop` |
| 会话结束 | 各种状态清理类 hooks |
| 压缩前 | `claude-code-hooks(PreCompact)`, `compaction-context-injector` |
| messages transform | `thinking-block-validator` |

#### 3.8 推荐实现顺序

我建议按这个顺序来，不容易把自己写死：

1. **先做宿主级事件**：`chat.message` / `tool.execute.before` / `tool.execute.after` / `event`
2. **再做 session 级状态事件**：`session.created` / `session.deleted` / `session.error` / `message.updated`
3. **再补 Claude 兼容层**：`UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `Stop` / `PreCompact`
4. **最后补增强事件**：`PostToolUseFailure` / `SessionStart` / `SessionEnd` / `InstructionsLoaded` / `PostCompact`

一句话总结：

- **如果你只想跑 oh-my-opencode 的核心价值**：先实现它依赖的宿主事件
- **如果你还想吃 Claude Code hooks 生态**：再把官方事件名和输入输出协议兼容上
- **今天这个项目本身并没有完整实现 Claude 官方全部 hook 生命周期**，它只覆盖了其中最关键的 5 个

#### 3.9 对照依据

这部分对照主要基于以下源码/文档：

- `oh-my-opencode/src/config/schema.ts`：可配置 hook 名单（`HookNameSchema`）
- `oh-my-opencode/src/index.ts`：宿主 hook 点注册（`chat.message` / `tool.execute.before` / `tool.execute.after` / `event` / `experimental.*`）
- `oh-my-opencode/src/hooks/AGENTS.md`：hooks 模块概览
- `oh-my-opencode/src/hooks/claude-code-hooks/AGENTS.md`：Claude 兼容层说明
- Claude Code 官方 Hooks 文档：<https://code.claude.com/docs/en/hooks>

### 4. MCPs（Model Context Protocol）

内置 MCP 服务器：

| MCP | 功能 | 用途 |
|-----|------|------|
| **websearch (Exa)** | 实时网络搜索 | 查找最新信息 |
| **context7** | 官方文档查询 | 查询任何库/框架文档 |
| **grep_app** | GitHub 代码搜索 | 查找开源实现示例 |

Skills 可以内嵌自己的 MCP：
```yaml
mcp:
  playwright:
    command: npx
    args: ["@playwright/mcp@latest"]
```

---

## 集成到你的 Agent 项目

### 方案 A：完整集成（推荐用于新项目）

如果你的项目是基于 OpenCode 或兼容架构：

1. **直接安装 oh-my-opencode**
   ```bash
   bunx oh-my-opencode install
   ```

2. **配置文件位置**
   - 项目级：`.opencode/oh-my-opencode.json`
   - 用户级：`~/.config/opencode/oh-my-opencode.json`

3. **最小配置**
   ```json
   {
     "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/master/assets/oh-my-opencode.schema.json",
     "agents": {
       "oracle": { "model": "openai/gpt-5.2" }
     }
   }
   ```


---

### 方案 B：选择性集成（推荐用于现有项目）

如果你只想要部分功能，可以抽取核心模块。

#### 1. 只要 Hooks 系统

**核心文件：**
- `src/hooks/index.ts` - Hook 导出
- `src/hooks/todo-continuation-enforcer.ts` - 任务持续执行
- `src/hooks/comment-checker/` - 注释检查
- `src/features/hook-message-injector.ts` - 消息注入机制

**集成步骤：**
```typescript
// 1. 实现 Hook 接口
interface Hook {
  handler: (input: { event: { type: string } }) => Promise<void>;
}

// 2. 注册 hooks
const hooks = [
  createTodoContinuationEnforcer(),
  createCommentCheckerHooks(),
];

// 3. 在关键时刻触发
async function onAgentStop() {
  for (const hook of hooks) {
    await hook.handler({ event: { type: 'stop' } });
  }
}
```

**需要实现的接口：**
- 事件系统：`userPromptSubmit`, `preToolUse`, `postToolUse`, `stop`
- 消息存储：读写 agent 对话历史
- Todo 管理：读写任务列表


#### 2. 只要 Agent 编排系统

**核心文件：**
- `src/features/background-agent.ts` - 后台 agent 管理
- `src/tools/delegate-task.ts` - 任务委托
- `src/tools/call-omo-agent.ts` - Agent 调用

**集成步骤：**
```typescript
// 1. 定义 Agent 配置
const agents = {
  oracle: {
    model: "openai/gpt-5.2",
    permission: { edit: "deny", bash: "deny" }
  },
  librarian: {
    model: "anthropic/claude-sonnet-4-5",
    permission: { edit: "deny" }
  }
};

// 2. 实现委托接口
async function delegateTask(options: {
  agent: string;
  prompt: string;
  background?: boolean;
}) {
  const agentConfig = agents[options.agent];
  // 启动子 agent 会话
  // 返回结果或 task_id
}
```


#### 3. 只要 Skills 系统

**核心文件：**
- `src/features/skill-loader.ts` - Skill 加载器
- Skills 目录结构：
  ```
  .claude/skills/
  └── my-skill/
      └── SKILL.md
  ```

**Skill 格式：**
```markdown
---
description: My custom skill
mcp:
  my-server:
    command: npx
    args: ["my-mcp-server"]
---

# Skill Instructions

When user asks for X, do Y...
```

**集成步骤：**
- 扫描 `.claude/skills/` 目录
- 解析 YAML frontmatter
- 启动内嵌的 MCP 服务器
- 将 skill 指令注入 system prompt


---

## 核心实现要点

### 1. Todo Continuation Enforcer（最核心）

这是 oh-my-opencode 的灵魂功能，确保 agent 不会半途而废。

**工作原理：**
1. Agent 停止时检查 todo 列表
2. 如果有未完成任务，倒计时 2 秒
3. 自动注入继续提示：
   ```
   Incomplete tasks remain in your todo list. 
   Continue working on the next pending task.
   - Proceed without asking for permission
   - Mark each task complete when finished
   - Do not stop until all tasks are done
   ```

**关键代码位置：**
- `src/hooks/todo-continuation-enforcer.ts`
- 依赖 todo 存储：`~/.claude/todos/`


### 2. Comment Checker

防止 AI 生成过多注释，保持代码简洁。

**工作原理：**
- 在 `postToolUse` 时检查代码变更
- 如果新增注释过多，要求 agent 解释或删除
- 确保生成的代码像人类写的

**关键代码：**
- `src/hooks/comment-checker/`

### 3. Background Agent

并行执行多个 agent，提高效率。

**工作原理：**
```typescript
// 启动后台任务
const taskId = await delegateTask({
  agent: "explore",
  background: true,
  prompt: "Find all API routes"
});

// 继续主任务...

// 需要时获取结果
const result = await backgroundOutput(taskId);
```

**关键代码：**
- `src/features/background-agent.ts`


---

## 集成建议

### 如果你的项目是基于 OpenCode

**直接用方案 A**，完整安装 oh-my-opencode。

### 如果你的项目是自研 Agent 框架

**推荐抽取这些模块（按优先级）：**

1. **Todo Continuation Enforcer** ⭐⭐⭐⭐⭐
   - 最核心功能
   - 确保任务完成
   - 需要实现：todo 存储、事件系统

2. **Agent 编排系统** ⭐⭐⭐⭐
   - 多 agent 协作
   - 后台执行
   - 需要实现：子会话管理、消息传递

3. **Comment Checker** ⭐⭐⭐
   - 代码质量控制
   - 需要实现：代码 diff 分析

4. **Skills 系统** ⭐⭐
   - 可选，如果需要插件化
   - 需要实现：YAML 解析、MCP 启动


---

## 需要实现的基础设施

### 如果要集成 Hooks 系统

你的项目需要提供：

1. **事件系统**
   ```typescript
   interface EventSystem {
     on(event: 'userPromptSubmit' | 'preToolUse' | 'postToolUse' | 'stop', 
        handler: Function): void;
   }
   ```

2. **消息存储**
   ```typescript
   interface MessageStorage {
     getMessages(sessionId: string): Message[];
     addMessage(sessionId: string, message: Message): void;
   }
   ```

3. **Todo 管理**
   ```typescript
   interface TodoManager {
     getTodos(sessionId: string): Todo[];
     updateTodo(sessionId: string, todoId: string, status: string): void;
   }
   ```


### 如果要集成 Agent 编排

你的项目需要提供：

1. **子会话管理**
   ```typescript
   interface SessionManager {
     createSession(agentConfig: AgentConfig): string;
     sendMessage(sessionId: string, message: string): Promise<string>;
     getSessionState(sessionId: string): SessionState;
   }
   ```

2. **权限控制**
   ```typescript
   interface PermissionManager {
     checkPermission(agent: string, action: string): 'allow' | 'deny' | 'ask';
   }
   ```

---

## 快速开始示例

### 最小可用集成（只要 Todo Continuation）

```typescript
// 1. 监听 agent 停止事件
agent.on('stop', async () => {
  const todos = getTodos(sessionId);
  const incomplete = todos.filter(t => t.status !== 'completed');
  
  if (incomplete.length > 0) {
    // 2. 注入继续提示
    await agent.injectSystemMessage(`
      You have ${incomplete.length} incomplete tasks.
      Continue working without asking permission.
    `);
    
    // 3. 自动恢复执行
    await agent.resume();
  }
});
```


---

## 关键文件清单

如果你要手动抽取代码，重点看这些文件：

### Hooks 核心
- `src/hooks/index.ts` - 所有 hook 导出
- `src/hooks/todo-continuation-enforcer.ts` - 任务持续（最重要）
- `src/hooks/comment-checker/` - 注释检查
- `src/features/hook-message-injector.ts` - 消息注入机制

### Agent 编排
- `src/features/background-agent.ts` - 后台 agent
- `src/tools/delegate-task.ts` - 任务委托
- `src/tools/call-omo-agent.ts` - Agent 调用

### 配置系统
- `src/shared/config.ts` - 配置加载
- `assets/oh-my-opencode.schema.json` - JSON Schema

---

## 总结

**oh-my-opencode 的核心价值：**
1. **Todo Continuation** - 让 agent 不会半途而废
2. **Multi-Agent** - 专家团队协作，而不是单打独斗
3. **Hooks** - 在关键时刻注入逻辑
4. **Skills** - 插件化扩展能力

**集成难度评估：**
- 完整集成（OpenCode 项目）：⭐ 简单
- 只要 Hooks：⭐⭐ 中等（需要事件系统）
- 只要 Agent 编排：⭐⭐⭐ 中等（需要会话管理）
- 完整移植到自研框架：⭐⭐⭐⭐ 困难（需要重写大量基础设施）

**推荐路径：**
1. 如果你用 OpenCode → 直接装 oh-my-opencode
2. 如果你是自研框架 → 先抽取 Todo Continuation，再逐步加其他功能
3. 如果只是想了解思路 → 重点看 `todo-continuation-enforcer.ts` 和 `background-agent.ts`

