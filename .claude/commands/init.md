---
description: 初始化会话，加载规则和项目上下文
---

执行以下初始化步骤：

## 1. 检测项目上下文

检查当前目录是否有 `project-context.md`：

- **如果没有**，询问用户：
  ```
  检测到这是新项目（没有 project-context.md）。
  要初始化项目配置吗？我会：
  1. 创建 project-context.md
  2. 检测技术栈并建议安装 LSP
  3. 创建项目级 .claude/settings.json

  是/否？
  ```

- **如果有**，读取并加载项目上下文

## 2. 检测技术栈

检测以下文件判断技术栈：
- tsconfig.json → TypeScript 项目，建议 typescript-lsp
- pyproject.toml / requirements.txt → Python 项目，建议 pyright-lsp
- go.mod → Go 项目，建议 gopls-lsp
- Cargo.toml → Rust 项目，建议 rust-analyzer-lsp
- package.json → Node.js 项目

## 3. 检测系统环境

识别当前操作系统（Windows/Mac/Linux）以使用正确的命令。

## 4. 报告状态

简要报告：
```
【初始化完成】
━━━━━━━━━━━━━━━━━━━━━━━━
系统环境：[Windows/Mac/Linux]
工作目录：[当前目录]
技术栈：[检测到的技术栈]
项目上下文：[已加载/需要创建]
━━━━━━━━━━━━━━━━━━━━━━━━
已就绪，等待任务指派。
```
