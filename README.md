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

Hooks 是整个系统的"胶水层"，在关键时刻注入逻辑。

#### 核心 Hooks 列表

| Hook | 触发时机 | 作用 |
|------|---------|------|
| **todo-continuation-enforcer** | Agent 停止时 | 强制继续未完成任务（核心特性） |
| **comment-checker** | 代码提交前 | 检查并清理 AI 生成的冗余注释 |
| **context-window-monitor** | 每次对话 | 监控 context 使用率 |
| **session-recovery** | 会话崩溃后 | 自动恢复会话状态 |
| **tool-output-truncator** | 工具输出后 | 截断过长的 grep/LSP 输出 |
| **directory-agents-injector** | 读取文件时 | 自动注入目录下的 AGENTS.md |
| **rules-injector** | 读取文件时 | 根据 glob 模式注入规则 |
| **background-notification** | 后台任务完成 | 通知主 agent |
| **keyword-detector** | 用户输入后 | 检测 `ultrawork`/`ulw` 关键词 |
| **claude-code-hooks** | 各个时机 | 兼容 Claude Code 的 hooks |
| **ralph-loop** | 循环检测 | 防止 agent 陷入死循环 |
| **atlas** | 任务开始时 | 编排系统（Prometheus/Metis/Momus） |

#### Hook 事件类型

Hooks 监听这些事件：
- `userPromptSubmit`: 用户提交 prompt
- `preToolUse`: 工具调用前
- `postToolUse`: 工具调用后
- `stop`: Agent 停止时
- `compaction`: Context 压缩时

#### Hook 实现示例

```typescript
export function createTodoContinuationEnforcer(options) {
  return {
    handler: async (input) => {
      const { event } = input;
      
      if (event.type === 'stop') {
        // 检查是否有未完成任务
        const incompleteTodos = getIncompleteTodos();
        if (incompleteTodos.length > 0) {
          // 注入继续提示
          injectContinuationPrompt();
        }
      }
    }
  };
}
```


---

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

