# CLI 与 Standalone 模式的区别

在 Cline 项目中，"Standalone" 和 "CLI" 是两个紧密相关但概念不同的部分。本文旨在阐明它们的定义、关系以及在开发和构建过程中的区别。

## 1. 核心概念

### 1.1. Standalone (独立构建)

**Standalone** 指的是 Cline 核心逻辑 (`src/`) 的一种构建产物。它是一个"无头"（Headless）的 Node.js 应用程序，不包含图形用户界面（WebView），也不依赖于 VS Code 环境运行。

*   **本质**: 一个通过 gRPC 暴露服务的后端逻辑包。
*   **产物**: `dist-standalone/` 目录，其中包含 `cline-core.js`（打包后的核心代码）和必要的原生 Node 模块（如 `better-sqlite3`, `tree-sitter`）。
*   **能力**: 包含了智能体的推理、规划、工具调用等核心能力，但它本身无法直接操作文件系统或显示界面，必须通过 gRPC 连接到一个"宿主"（Host）来获得这些能力。

### 1.2. CLI (命令行客户端)

**CLI** 是一个面向最终用户的产品，它提供了一个终端界面来与 Cline 交互。

*   **本质**: 一个 Go 编写的客户端程序。
*   **产物**: `cli/bin/cline`（主程序）和 `cli/bin/cline-host`（宿主服务）。
*   **角色**:
    *   **作为客户端**: 接收用户输入（Prompt），发送给 Standalone 核心。
    *   **作为宿主**: 启动 `cline-host` 进程，通过 gRPC 为 Standalone 核心提供文件读写、命令执行等系统能力。

## 2. 架构关系

CLI 是 Standalone 的**消费者**和**宿主**。它们之间的关系如下：

```mermaid
graph TD
    User[用户 (Terminal)] -->|输入 Prompt| CLI[CLI Client (Go)]
    CLI -->|启动 & 监控| CoreProcess[Standalone Core (Node.js)]
    CLI -->|启动 & 监控| HostProcess[Host Bridge (Go)]

    CLI <-->|gRPC (指令/状态)| CoreProcess
    CoreProcess <-->|gRPC (文件操作/系统调用)| HostProcess
```

1.  用户启动 CLI。
2.  CLI 启动 Standalone 进程 (`node dist-standalone/cline-core.js`)。
3.  CLI 启动 Host 进程 (`cline-host`)。
4.  CLI 将用户输入转发给 Standalone 核心。
5.  Standalone 核心进行推理，如果需要读写文件，它会向 Host 进程发送请求。

## 3. 构建流程对比

| 特性 | Standalone 构建 | CLI 构建 |
| :--- | :--- | :--- |
| **命令** | `npm run compile-standalone` | `scripts/build-cli.sh` |
| **工具** | `esbuild`, `npm` | `go build` |
| **输入** | `src/**/*.ts` (TypeScript) | `cli/**/*.go` (Go) |
| **输出** | `dist-standalone/` (JS + Node Modules) | `cli/bin/` (Executable Binaries) |
| **依赖** | 依赖 Standalone 构建产物吗？ **否** | 依赖 Standalone 构建产物吗？ **是** (运行时依赖) |

**注意**: CLI 在运行时需要找到 `cline-core.js`。在开发模式下，它会查找 `dist-standalone/` 目录；在生产发布包中，这些文件会被打包在一起。

## 4. 开发工作流

为了方便同时开发核心逻辑（TypeScript）和 CLI 客户端（Go），项目提供了专门的脚本：

*   **脚本**: `scripts/dev-cli-watch.mjs`
*   **命令**: `npm run dev:cli:watch`

这个脚本实现了全栈热重载：
1.  **监听 TypeScript**: 使用 `esbuild` 监听 `src/` 变化，增量构建 Standalone 包。
2.  **监听 Go**: 使用 `chokidar` 监听 `cli/` 变化，重新编译 Go 二进制文件。
3.  **自动重启**: 当任一部分构建完成后，自动杀掉旧进程并重启 CLI 实例，让开发者能立即看到修改效果。

## 5. 总结

*   **Standalone** 是"大脑"，负责思考和规划，是一个 Node.js 库/服务。
*   **CLI** 是"身体"和"嘴巴"，负责与外界（用户和操作系统）交互，是一个 Go 应用程序。
*   CLI 依赖 Standalone 来提供智能能力，Standalone 依赖 CLI (Host) 来执行实际操作。
