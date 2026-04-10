# MCP 工作流程详解

## 一、配置文件的加载

项目中有多个 MCP 配置文件：

| 文件 | 用途 |
|------|------|
| `opencode.json` | OpenCode 的配置文件，定义了 `chrome-devtools` MCP server |
| `.cursor/mcp.json` | Cursor 编辑器的 MCP 配置 |
| `.claude/settings.local.json` | Claude Desktop 的本地权限配置 |

**OpenCode 启动时**会读取 `opencode.json`，发现里面配置了 MCP server：

```json
{
  "mcp": {
    "chrome-devtools": {
      "type": "local",
      "command": ["npx", "-y", "chrome-devtools-mcp@latest"]
    }
  }
}
```

## 二、完整交互流程

### 1. 启动阶段

```
OpenCode 启动
    ↓
读取 opencode.json
    ↓
发现 MCP 配置 → 启动 chrome-devtools-mcp 进程
    ↓
通过 STDIO 建立 JSON-RPC 连接
    ↓
MCP Server 返回可用工具列表（tools/list）
    ↓
┌───────────────────────────────────────────────────────────┐
│ chrome-devtools-mcp 提供的工具：                            │
│ • chrome-devtools_navigate_page                            │
│ • chrome-devtools_click                                    │
│ • chrome-devtools_take_screenshot                          │
│ • chrome-devtools_fill                                     │
│ • chrome-devtools_take_snapshot                            │
│ • ... 等等 20+ 个工具                                       │
└───────────────────────────────────────────────────────────┘
```

### 2. 用户输入阶段

```
用户说："打开浏览器，打开百度，并搜索美伊战争最新情况"
    ↓
OpenCode 将你的消息 + 系统提示词 + 可用工具列表
一起发送给大模型（如 Claude/GPT）
```

### 3. 大模型决策阶段

```
大模型收到：
┌───────────────────────────────────────────────────────────┐
│ System Prompt:                                            │
│ "你是一个编程助手，可以使用以下工具帮助用户..."               │
│                                                           │
│ Available Tools:                                          │
│ - chrome-devtools_navigate_page: 导航到指定URL              │
│ - chrome-devtools_fill: 填写表单                           │
│ - chrome-devtools_click: 点击元素                          │
│ - chrome-devtools_take_screenshot: 截图                    │
│ - bash: 执行命令行                                         │
│ - ...                                                     │
│                                                           │
│ User Message:                                             │
│ "打开浏览器，打开百度，并搜索美伊战争最新情况"                │
└───────────────────────────────────────────────────────────┘

大模型分析：
"用户要打开浏览器 → 使用 chrome-devtools_navigate_page"
"要打开百度 → URL 是 https://www.baidu.com"
"要搜索 → 需要填写搜索框 + 点击搜索按钮"

大模型输出工具调用请求：
┌───────────────────────────────────────────────────────────┐
│ {                                                          │
│   "tool": "chrome-devtools_navigate_page",                 │
│   "arguments": { "url": "https://www.baidu.com" }          │
│ }                                                          │
└───────────────────────────────────────────────────────────┘
```

### 4. 工具执行阶段

```
OpenCode 收到大模型的工具调用请求
    ↓
通过 JSON-RPC 发送给 chrome-devtools-mcp
    ↓
MCP Server 执行实际操作：
- 启动 Chrome 浏览器（或连接已有实例）
- 导航到 https://www.baidu.com
    ↓
返回执行结果给 OpenCode
    ↓
OpenCode 将结果返回给大模型
```

### 5. 循环迭代阶段

```
大模型收到执行结果 → 继续决策下一步
    ↓
"页面已打开，现在需要填写搜索框"
    ↓
调用 chrome-devtools_take_snapshot 获取页面结构
    ↓
分析快照，找到搜索框的 uid
    ↓
调用 chrome-devtools_fill 填写"美伊战争"
    ↓
调用 chrome-devtools_click 点击搜索按钮
    ↓
调用 chrome-devtools_take_screenshot 截图展示结果
    ↓
最终用自然语言总结给用户
```

## 三、关键技术细节

### 1. MCP 协议（JSON-RPC 2.0 over STDIO）

```
OpenCode (Client)                    MCP Server (chrome-devtools-mcp)
      │                                         │
      │  initialize()                           │
      │────────────────────────────────────────>│
      │                                         │
      │  tools/list()                           │
      │────────────────────────────────────────>│
      │  [{name: "navigate_page", ...}, ...]    │
      │<────────────────────────────────────────│
      │                                         │
      │  tools/call({name: "navigate_page",     │
      │              arguments: {...}})         │
      │────────────────────────────────────────>│
      │                                         │
      │  {content: [...], isError: false}       │
      │<────────────────────────────────────────│
      │                                         │
```

### 2. 为什么大模型"知道"使用 chrome-devtools？

**不是大模型"知道"，而是被"告知"的！**

OpenCode 在每次请求大模型时，会动态注入 **可用工具列表**：

```json
{
  "messages": [
    {
      "role": "system",
      "content": "你是编程助手。你可以使用以下工具：\n\n" +
                 "### chrome-devtools 工具集\n" +
                 "- chrome-devtools_navigate_page: 导航到URL\n" +
                 "- chrome-devtools_click: 点击元素\n" +
                 "- chrome-devtools_fill: 填写表单\n" +
                 "...\n\n" +
                 "### bash 工具\n" +
                 "- bash: 执行 shell 命令\n" +
                 "..."
    },
    {
      "role": "user",
      "content": "打开浏览器，打开百度，并搜索美伊战争最新情况"
    }
  ],
  "tools": [
    {
      "name": "chrome-devtools_navigate_page",
      "description": "Navigate to a URL",
      "input_schema": {...}
    },
    ...
  ]
}
```

大模型看到这个工具列表后，根据 **工具名称** 和 **描述** 判断哪个工具适合当前任务。

### 3. MCP Server 启动过程

```javascript
const { spawn } = require('child_process');

// 启动 MCP Server 进程
const mcpProcess = spawn('npx', ['-y', 'chrome-devtools-mcp@latest'], {
  stdio: ['pipe', 'pipe', 'pipe']  // stdin, stdout, stderr
});

// 通过 stdin 发送 JSON-RPC 请求
mcpProcess.stdin.write(JSON.stringify({
  jsonrpc: '2.0',
  id: 1,
  method: 'initialize',
  params: {...}
}) + '\n');

// 通过 stdout 接收响应
mcpProcess.stdout.on('data', (data) => {
  const response = JSON.parse(data);
  // 处理响应...
});
```

## 四、npx 缓存 vs 全局安装

| 特性 | `npm install -g` | `npx` |
|------|------------------|-------|
| 安装位置 | `node_global/node_modules` | `node_cache/_npx/<hash>/` |
| 持久性 | 永久安装，直到手动卸载 | 缓存形式，可能被清理 |
| 命令可用 | 直接运行 `chrome-devtools-mcp` | 必须加 `npx` 前缀 |
| 版本管理 | 固定版本 | 每次可指定不同版本 |

**MCP 使用 npx 的优势**：
1. 无需手动安装，首次运行自动下载
2. 版本灵活，可随时切换 `@latest` 或指定版本
3. 配置简单，跨平台兼容性好

## 五、总结

| 环节 | 关键机制 |
|------|----------|
| **配置发现** | OpenCode 读取 `opencode.json` 中的 `mcp` 配置 |
| **进程启动** | 通过 `npx -y chrome-devtools-mcp@latest` 启动子进程 |
| **工具注册** | MCP Server 通过 `tools/list` 返回可用工具 |
| **模型决策** | 大模型根据工具名称+描述选择合适工具 |
| **工具执行** | OpenCode 通过 JSON-RPC 调用 MCP Server 执行操作 |
| **结果返回** | 执行结果返回给大模型，继续下一轮决策 |

**核心要点**：大模型本身不知道 chrome-devtools 的存在，是 OpenCode 在每次请求时把工具列表"喂"给大模型，大模型根据工具描述做出选择。
