# OpenCode 技能加载机制详解

## 问题背景

在使用 OpenCode 时，当用户明确使用某个技能（通过 `/指令` 或直接调用），技能文件 `SKILL.md` 的内容是如何发送给大模型的？是插入到用户提示词前面，还是作为系统提示词？

## OpenCode 的技能处理机制

### 核心机制：按需加载（On-Demand Loading）

OpenCode **不是**直接把 SKILL.md 内容插入用户提示词前面，而是采用**按需加载**机制：

1. **技能发现阶段**：OpenCode 在 `skill` 工具的描述中列出所有可用技能的 `name` 和 `description`（仅元数据），以 `<available_skills>` XML 格式呈现给大模型

2. **按需加载**：大模型看到可用技能列表后，**自主决定**是否调用 `skill` 工具来加载完整内容：
   ```
   skill({ name: "git-release" })
   ```

3. **内容注入**：当代理调用 skill 工具后，SKILL.md 的完整内容才会被加载到上下文中

## 发送给大模型的消息结构

### 系统提示词 (System Prompt) 包含：

1. **Agent Prompt**
   - Build/Plan 等内置 agent 的默认 prompt
   - 或自定义 agent 的 prompt（在 `opencode.json` 或 agent markdown 文件中定义）

2. **Rules（规则）**
   - `AGENTS.md` 文件内容（项目根目录）
   - 或 `CLAUDE.md`（兼容 Claude Code）
   - 全局规则：`~/.config/opencode/AGENTS.md`

3. **Instructions（指令）**
   - `opencode.json` 中 `instructions` 字段引用的文件内容
   - 如 `CONTRIBUTING.md`、`docs/guidelines.md` 等

4. **Tools 定义**
   - 所有可用工具的描述（包括 `skill` 工具）
   - `<available_skills>` 列表（仅包含技能的 name 和 description，**不包含完整内容**）

### 用户提示词 (User Prompt) 包含：

#### 情况 1：使用 `/指令`（自定义命令）

当用户输入 `/test` 这样的命令时：

```
用户提示词 = 命令的 template 内容 + 用户额外输入（如果有）
```

例如命令定义为：
```markdown
---
description: Run tests with coverage
---
Run the full test suite with coverage report and show any failures.
Focus on the failing tests and suggest fixes.
```

发送给大模型的用户提示词就是这段 template 内容。

#### 情况 2：直接调用技能（通过 skill 工具）

**关键区别**：技能内容**不是**插入到用户提示词前面，而是：

1. 大模型看到 `<available_skills>` 列表（只有 name + description）
2. 大模型**自主决定**调用 `skill({ name: "xxx" })` 工具
3. **工具调用结果**（SKILL.md 完整内容）作为**工具响应**注入到对话上下文中
4. 这个内容既不在系统提示词，也不在用户提示词，而是作为**工具调用轮次**的一部分

## 实际案例分析

### 案例：使用技能优化 URL

用户输入：
```
使用这个技能，优化一下我的这个 url https://www.leaflypackaging.com/why-leafly.html
```

这种情况的消息结构是：

| 部分 | 内容 |
|------|------|
| **系统提示词** | Agent prompt + AGENTS.md + instructions + tools 定义（含 available_skills 列表） |
| **用户提示词** | "使用这个技能，优化一下我的这个 url https://www.leaflypackaging.com/why-leafly.html" |
| **工具调用** | 大模型识别到需要技能 → 调用 `skill({ name: "page-cro" })` |
| **工具响应** | SKILL.md 完整内容被注入上下文 |

## Claude Code 的处理方式

Claude Code 采用**类似的按需加载机制**：

- 技能文件同样通过 `skill` 工具进行发现
- 代理看到可用技能列表后，根据需要主动调用加载
- 加载的内容作为**上下文**提供给模型（既不是纯用户提示词，也不是系统提示词，而是作为工具调用的结果注入到对话上下文中）

## 关键区别对比

| 特性 | 直接插入提示词 | OpenCode/Claude Code 实际方式 |
|------|--------------|---------------------------|
| 加载时机 | 每次对话都加载 | 按需加载（代理决定） |
| 上下文占用 | 始终占用 | 仅在使用时占用 |
| 灵活性 | 低 | 高（代理可自主选择） |

## 总结

- **系统提示词** = Agent 角色定义 + 项目规则 + 全局指令 + 工具定义
- **用户提示词** = 用户的实际输入（或命令的 template）
- **技能内容** = 作为工具调用结果注入，**不是**拼接到用户提示词前面

OpenCode 和 Claude Code 都采用这种**按需加载的工具调用机制**，而非简单地把 SKILL.md 内容前置到用户提示词中。

---

*文档创建日期：2026-04-15*
*基于 OpenCode 官方文档分析*
