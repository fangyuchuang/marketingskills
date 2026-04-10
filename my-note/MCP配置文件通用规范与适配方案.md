# MCP 配置文件通用规范与适配方案

## 一、背景与现状

目前 **MCP（Model Context Protocol）协议本身并没有规定统一的配置文件命名和路径**。每个宿主应用（Host）都使用自己定义的配置文件格式和名称，导致在不同 AI Agent 工具之间切换时，需要重复配置或维护多份文件。

### 主流 AI Agent 的 MCP 配置对比

| 工具 | 配置文件路径 | 格式特点 | 是否支持自定义 |
|------|-------------|----------|----------------|
| **OpenCode** | `opencode.json` | 自定义结构，包含 `$schema`, `permission`, `mcp` 等字段 | ❌ 固定文件名 |
| **Cursor** | `.cursor/mcp.json` | 标准 `mcpServers` 对象 | ❌ 固定路径 |
| **Cline** | `.cline/mcp.json` | 标准 `mcpServers` 对象 | ❌ 固定路径 |
| **Claude Desktop** | `%APPDATA%/Claude/claude_desktop_config.json` | 全局配置，非项目级 | ❌ 固定路径 |
| **Windsurf** | 设置界面 / `.windsurfrules` | GUI 配置为主 | ❌ 无标准文件 |
| **GitHub Copilot** | VS Code 设置 | 无本地 MCP 配置文件 | - |

> **核心痛点**：像 `agents.md` 和 `.agents/` 目录已经是行业通用的 Agent 提示词规范，但 **MCP 工具配置至今没有通用标准**。

---

## 二、为什么没有统一标准？

MCP 协议（Model Context Protocol）目前只定义了 **运行时通信协议**（JSON-RPC 2.0 over STDIO/SSE），规范了：
- 客户端与服务端如何握手（`initialize`）
- 如何交换工具列表（`tools/list`）
- 如何调用工具（`tools/call`）

但 **没有规定**：
- 配置文件应该叫什么名字
- 配置文件应该放在项目根目录还是隐藏目录
- 配置文件的 JSON 结构应该长什么样

因此，各工具厂商只能各自为战，定义自己的配置加载逻辑。

---

## 三、通用解决方案（推荐）

### 方案 1：`mcp.json` 主配置 + 自动同步脚本 ⭐（推荐）

通过维护一份通用的 `mcp.json`，编写轻量脚本自动转换为各工具所需的格式。

#### 1. 目录结构
```
project/
├── mcp.json                    # 🟢 主配置（通用格式，手动维护）
├── scripts/
│   └── sync-mcp-config.js      # 🔧 同步脚本（自动生成）
├── opencode.json               # 📄 自动生成（OpenCode 专用）
├── .cursor/
│   └── mcp.json                # 📄 自动生成（Cursor 专用）
└── .cline/
    └── mcp.json                # 📄 自动生成（Cline 专用）
```

#### 2. 主配置 `mcp.json`
采用最通用的 `mcpServers` 结构，兼容大多数工具：
```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token_here"
      }
    }
  }
}
```

#### 3. 同步脚本 `scripts/sync-mcp-config.js`
```javascript
const fs = require('fs');
const path = require('path');

const mainConfigPath = path.join(__dirname, '..', 'mcp.json');
const mainConfig = JSON.parse(fs.readFileSync(mainConfigPath, 'utf8'));

console.log('🔄 开始同步 MCP 配置...\n');

// 1. 生成 Cursor / Cline 配置（格式完全一致）
const cursorConfig = { mcpServers: mainConfig.mcpServers };
const clineConfig = { mcpServers: mainConfig.mcpServers };

fs.mkdirSync(path.join(__dirname, '..', '.cursor'), { recursive: true });
fs.writeFileSync(
  path.join(__dirname, '..', '.cursor', 'mcp.json'),
  JSON.stringify(cursorConfig, null, 2)
);
console.log('✅ .cursor/mcp.json 已更新');

fs.mkdirSync(path.join(__dirname, '..', '.cline'), { recursive: true });
fs.writeFileSync(
  path.join(__dirname, '..', '.cline', 'mcp.json'),
  JSON.stringify(clineConfig, null, 2)
);
console.log('✅ .cline/mcp.json 已更新');

// 2. 生成 OpenCode 配置（格式特殊，需转换）
const opencodeConfig = {
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "skill": { "*": "allow" },
    "file": { "read": "allow", "write": "allow" },
    "shell": "allow"
  },
  "mcp": {}
};

for (const [name, server] of Object.entries(mainConfig.mcpServers)) {
  opencodeConfig.mcp[name] = {
    type: "local",
    command: [server.command, ...server.args]
  };
}

fs.writeFileSync(
  path.join(__dirname, '..', 'opencode.json'),
  JSON.stringify(opencodeConfig, null, '\t')
);
console.log('✅ opencode.json 已更新\n');

console.log('🎉 所有 MCP 配置同步完成！');
```

#### 4. 使用方法
在 `package.json` 中添加快捷命令：
```json
{
  "scripts": {
    "sync:mcp": "node scripts/sync-mcp-config.js"
  }
}
```
每次修改 `mcp.json` 后，只需运行：
```bash
npm run sync:mcp
```

---

### 方案 2：使用 `.mcp.json` 隐藏文件

虽然官方未规定，但社区逐渐倾向于使用 `.mcp.json` 作为通用配置文件名。
- **优点**：符合 Unix 隐藏文件惯例，不干扰项目文件列表。
- **缺点**：仍需各工具手动配置读取路径，自动化程度低。

---

### 方案 3：环境变量注入

部分工具支持通过环境变量指定 MCP 配置路径：
```bash
export MCP_CONFIG_PATH=/path/to/mcp.json
```
- **优点**：无需生成多份文件。
- **缺点**：兼容性差，仅少数工具支持。

---

## 四、方案对比与建议

| 方案 | 维护成本 | 兼容性 | 自动化程度 | 推荐指数 |
|------|----------|--------|------------|----------|
| **方案 1：主配置 + 同步脚本** | 低（只改 1 个文件） | 高（覆盖主流工具） | 高（一键同步） | ⭐⭐⭐⭐⭐ |
| 方案 2：`.mcp.json` | 中 | 中 | 低 | ⭐⭐ |
| 方案 3：环境变量 | 低 | 低 | 中 | ⭐⭐ |

### 为什么推荐方案 1？
1. **单一数据源**：只需维护 `mcp.json`，避免多文件不一致。
2. **扩展性强**：新增工具支持时，只需在脚本中添加一个转换逻辑。
3. **团队协作友好**：可将 `mcp.json` 提交到 Git，生成的配置文件加入 `.gitignore`（或保留以便开箱即用）。
4. **符合工程化思维**：类似 `tailwind.config.js` 或 `eslint.config.js` 的衍生配置生成模式。

---

## 五、总结

| 环节 | 关键结论 |
|------|----------|
| **现状** | 各 AI Agent 工具 MCP 配置文件名/路径不统一，无官方标准 |
| **原因** | MCP 协议仅定义运行时通信，未规定配置规范 |
| **最佳实践** | 使用 `mcp.json` 作为主配置 + 脚本自动同步到各工具 |
| **核心优势** | 一次编写，多处运行；降低维护成本；便于团队协作 |

> 💡 **提示**：随着 MCP 生态发展，未来可能会出现官方统一的配置标准（如 `mcp.config.json`）。在此之前，采用 **主配置 + 转换脚本** 是最稳妥的工程化方案。
