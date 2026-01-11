# Claude Code 工作流 v2.0

一套优化 Claude Code CLI 工作效率的规则和工具配置。

## 功能特性

- **任务分级系统**：自动判断任务复杂度（Level 0/1/2），选择合适的执行流程
- **循环执行机制**：Level 2 复杂任务使用 ralph-loop 自动迭代直到完成
- **自动查文档**：通过 Context7 MCP 主动获取第三方库最新文档
- **PR 审查工具**：多角度代码审查（安全、类型设计、测试覆盖等）
- **代码简化工具**：长会话后自动清理和简化代码
- **项目上下文管理**：持久化项目配置和已验证/已证伪的方案

## 安装步骤

### 1. 添加插件市场

```bash
claude plugin marketplace add anthropics/claude-code
claude plugin marketplace add anthropics/claude-plugins-official
```

### 2. 安装插件

```bash
claude plugin install ralph-wiggum@claude-code-plugins
claude plugin install security-guidance@claude-code-plugins
claude plugin install pr-review-toolkit@claude-plugins-official
claude plugin install code-simplifier@claude-plugins-official
```

### 3. 配置 MCP 服务

```bash
claude mcp add -s user context7 npx @upstash/context7-mcp@latest
```

### 4. 验证 MCP 配置

```bash
claude mcp list
```

### 5. 复制配置文件

将本仓库的文件复制到你的 Claude 配置目录：

```
# Windows
C:\Users\<用户名>\.claude\CLAUDE.md
C:\Users\<用户名>\.claude\commands\init.md
C:\Users\<用户名>\.claude\commands\riper.md

# Mac/Linux
~/.claude/CLAUDE.md
~/.claude/commands/init.md
~/.claude/commands/riper.md
```

## 文件结构

```
.
├── CLAUDE.md                    # 核心工作流规则（全局）
├── README.md                    # 本说明文件
└── .claude/
    └── commands/
        ├── init.md              # /init 命令 - 初始化会话
        └── riper.md             # /riper 命令 - 激活工程协议
```

## 使用方法

### 启动会话

在任何项目目录中执行：

```
/init
```

这会：
1. 检测项目技术栈
2. 检查是否有 project-context.md
3. 报告系统环境和就绪状态

### 任务分级

| Level | 特征 | 流程 |
|-------|------|------|
| 0 | 单文件、<10行、无逻辑变化 | 直接执行 |
| 1 | 多文件、有逻辑变化、边界明确 | 声明计划 → 执行 → 验证 |
| 2 | 跨模块、架构变化 | 完整规划 → 用户确认 → ralph-loop |

### 激活工程协议

需要深度分析和多角色思维时：

```
/riper
```

## 核心规则

1. **先读后做** - 修改文件前必须先读取
2. **先查后建** - 创建前检查是否已存在
3. **验证关闭** - 每次修改后验证结果
4. **失败不重复** - 不再尝试已证伪的方案
5. **只改要求的** - 不"顺便优化"其他代码

## License

MIT
