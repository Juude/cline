# Cline CLI 客户端架构说明

本文档详细介绍了 Cline 命令行客户端 (CLI) 的架构设计、核心组件以及其与 VS Code 扩展核心逻辑的交互方式。

## 1. 概述

Cline CLI 是一个独立的命令行工具，允许用户在终端中直接使用 Cline 的智能编程能力。它采用了前后端分离的架构，其中：

*   **前端 (CLI)**: 基于 Go 语言编写，负责接收用户输入、渲染界面、管理进程以及与核心服务通信。
*   **后端 (Core)**: 基于 Node.js 运行的 `cline-core`，复用了 VS Code 插件的核心逻辑。
*   **宿主桥接 (Host Bridge)**: 一个 Go 服务，模拟 VS Code 扩展宿主环境的部分功能（如文件系统访问、窗口管理），供 Core 调用。

## 2. 项目结构

CLI 的源代码主要位于 `cli/` 目录下：

```
cli/
├── cmd/
│   ├── cline/          # CLI 主程序入口 (main.go)
│   └── cline-host/     # 宿主桥接服务入口
├── pkg/
│   ├── cli/            # CLI 核心逻辑库
│   │   ├── global/     # 全局状态管理 (Registry, Clients)
│   │   ├── display/    # 终端界面渲染 (TUI)
│   │   └── ...
│   ├── hostbridge/     # 宿主桥接服务实现 (gRPC Server)
│   └── ...
├── proto/              # Protocol Buffers 定义
└── scripts/            # 构建和打包脚本
```

## 3. 核心架构与组件

### 3.1. 进程模型

当用户启动 `cline` 命令时，CLI 会协调启动两个子进程：

1.  **`cline-host` (Go)**:
    *   提供文件系统操作、窗口管理等底层能力。
    *   通过 gRPC 服务暴露这些能力。
    *   充当 VS Code 宿主环境的替代品。

2.  **`cline-core` (Node.js)**:
    *   这是打包后的 JS 应用程序 (源自 `src/`)。
    *   包含了智能体的核心逻辑（规划、推理、工具调用）。
    *   通过 gRPC 连接到 `cline-host` 来执行实际的文件操作。
    *   同时自身也启动一个 gRPC 服务，供 CLI 客户端调用（发送指令、获取状态）。

### 3.2. 通信机制 (gRPC)

组件之间通过 gRPC 进行通信，协议定义在 `proto/` 目录下：

*   **CLI -> Core**: CLI 客户端通过 gRPC 调用 Core 的服务，发送用户输入的 prompt，并流式接收执行进度和结果。
*   **Core -> Host**: Core 在执行任务时（如写文件），通过 gRPC 调用 Host 的服务来完成操作。

### 3.3. 实例管理 (Registry)

CLI 使用 `pkg/cli/global/registry.go` 中的逻辑来管理运行中的 Cline 实例。
*   **SQLite**: 使用 SQLite 数据库来维护实例列表、进程锁和端口分配信息。
*   **自动发现**: CLI 可以发现并连接到现有的后台实例，或者启动新实例。

## 4. 关键流程

### 4.1. 启动流程 (`cmd/cline/main.go`)

1.  **初始化**: 解析命令行参数 (`cobra`)，初始化全局配置。
2.  **实例检查**: 检查是否已有运行中的实例，或者根据参数启动新实例。
3.  **启动服务**:
    *   分配可用端口对 (Core Port, Host Port)。
    *   启动 `cline-host` 进程。
    *   启动 `cline-core` 进程，并传入端口参数。
4.  **连接**: 等待服务启动并自注册到 SQLite。
5.  **交互**: 进入交互模式 (`plan` 或 `act`)，接收用户输入并发送给 Core。

### 4.2. 任务执行

1.  用户输入 prompt。
2.  CLI 将 prompt 发送给 Core。
3.  Core 进行推理，生成计划。
4.  Core 需要执行操作（如读取文件）时，向 Host 发送 gRPC 请求。
5.  Host 执行文件操作并返回结果。
6.  Core 将进度和结果流式返回给 CLI。
7.  CLI 渲染进度条或文本输出。

## 5. 构建流程

CLI 的构建涉及到 Go 和 TypeScript (Node.js) 两部分：

1.  **Core 构建**:
    *   使用 `esbuild` 将 `src/` 下的 TypeScript 代码打包成 `dist-standalone/cline-core.js`。
    *   命令: `npm run compile-standalone`

2.  **CLI 构建**:
    *   使用 `go build` 编译 `cmd/cline` 和 `cmd/cline-host`。
    *   脚本: `scripts/build-cli.sh`

开发者可以通过 `scripts/dev-cli-watch.mjs` 脚本在开发模式下运行，实现增量编译和热重载。
