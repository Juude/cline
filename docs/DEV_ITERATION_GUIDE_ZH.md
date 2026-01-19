# Cline 开发迭代加速指南

本文档旨在为 Cline 开发者提供一套经过验证的最佳实践，帮助你优化开发环境配置，实现毫秒级的增量编译与热重载，从而大幅提升开发效率。

## 1. 核心开发模式概览

在开始之前，了解 Cline 的项目结构对于选择正确的开发模式至关重要。Cline 主要由三部分组成：
1. **Extension Host (Backend)**: 运行在 VS Code 扩展进程中的 TypeScript 代码。
2. **Webview UI (Frontend)**: 基于 React + Vite 的用户界面。
3. **CLI/Standalone**: 独立运行的命令行工具和核心逻辑。

针对不同的修改目标，应采用不同的迭代策略：

| 修改内容 | 推荐模式 | 关键命令 | 预期反馈速度 |
| :--- | :--- | :--- | :--- |
| **UI 样式/组件** | Vite HMR | `npm run dev:webview` + F5 Launch | < 100ms (热重载) |
| **扩展业务逻辑** | Watch Mode + Reload | `npm run watch` + Reload Window | ~1-2s (无需重启调试器) |
| **核心算法/CLI** | Standalone Watch | `node scripts/dev-cli-watch.mjs` | ~500ms (增量构建) |

---

## 2. 环境准备与初次构建

为了确保后续的快速迭代，请首先完成标准的环境初始化：

```bash
# 1. 安装所有依赖 (包括根目录和 webview-ui)
npm run install:all

# 2. 生成 Protocol Buffer 定义 (这是构建的前提)
npm run protos
```

---

## 3. UI 开发加速 (Webview HMR)

这是开发体验提升最明显的环节。Cline 的架构允许在 VS Code 宿主中通过 Webview 加载本地 Vite 开发服务器的内容，从而实现**真正的热模块替换 (HMR)**。

### 🚀 极速开发流程

1. **启动 UI 开发服务器（后台运行）**
   在终端中运行：
   ```bash
   npm run dev:webview
   ```
   *注意：该命令会启动 Vite 服务器（默认端口 25463），并生成 `.vite-port` 文件供扩展读取。*

2. **启动扩展宿主**
   在 VS Code 中按 `F5` 启动 "Run Extension (local)"。

3. **实时修改**
   现在，当你修改 `webview-ui/src` 下的任何 React 组件或 CSS 时，VS Code 中的 Cline 界面会**立即更新**，无需刷新页面，状态（State）也会保留。

### 🔧 原理说明
`WebviewProvider.ts` 会检测本地开发服务器是否运行。如果运行中，它会注入连接到 localhost 的脚本，而不是加载编译后的 `dist` 文件。

---

## 4. 扩展逻辑开发加速

对于 `src/` 目录下的后端逻辑修改，虽然无法像 UI 那样热重载，但我们可以通过"热重载窗口"来避免重启整个调试会话。

### 🚀 高效工作流

1. **开启全量监听**
   ```bash
   npm run watch
   ```
   这个命令会并行运行 `watch:esbuild` (后端增量编译) 和 `dev:webview`。`esbuild` 的增量编译通常在 100ms 内完成。

2. **使用 "Reload Window"**
   当你在 `npm run watch` 运行时修改了 `src/*.ts` 文件，终端会显示 `[watch] build finished`。
   此时，不要停止调试器！只需在扩展宿主窗口中：
   -按下 `Cmd+R` (macOS) 或 `Ctrl+R` (Windows/Linux)
   -或者运行命令面板 `Developer: Reload Window`

   扩展会重新加载最新的 JS 代码，整个过程仅需 1-2 秒。

### ⚡️ 进阶技巧：不启动调试器运行 (No-Debugger Mode)

调试器 (Debugger) 会带来显著的性能开销。如果你主要依赖 `console.log` 或界面反馈进行开发，建议**不带调试器启动**：

- 使用快捷键 **`Ctrl + F5`** (默认) 启动扩展。
- 或者在运行面板点击 "Run Without Debugging"。

**优势：** 启动速度快 3-5 倍，且界面操作更流畅。

---

## 5. 核心逻辑与 CLI 开发 (Standalone Mode)

如果你正在开发不依赖 VS Code API 的核心功能（如提示词生成、API 调用逻辑、文件操作等），使用 Standalone 模式是最高效的。

### 🚀 命令行极速迭代

项目提供了一个专门的脚本用于 CLI 和核心库的快速开发：

```bash
node scripts/dev-cli-watch.mjs
```

**该脚本的功能：**
- **智能监听**: 使用 `chokidar` 监听 Go 和 TypeScript 文件。
- **增量编译**: 仅重编译变动的部分（Go 二进制或 TS 代码）。
- **自动重启**: 编译完成后自动重启 CLI 实例。
- **反馈闭环**: 可以在终端直接看到输出结果，完全脱离 IDE UI 的干扰。

---

## 6. 测试驱动开发 (TDD) 加速

运行完整的集成测试 (`npm run test`) 非常耗时且需要启动 GUI。在日常开发中，应优先使用单元测试。

### 🎯 单元测试策略

1. **运行单个测试文件**
   不要运行所有测试。使用 `mocha` 直接运行你正在工作的文件：
   ```bash
   npx mocha -r ts-node/register src/your/file.test.ts
   ```
   或者在 VS Code `launch.json` 中选择 **"Debug Current Test File"** 配置，然后在编辑器中打开测试文件并按 F5。

2. **监听模式**
   如果你在频繁修改某个模块，可以使用 watch 模式：
   ```bash
   npm run watch-tests
   ```

---

## 7. 常见问题排查

**Q: 修改了 UI 代码但没有热更新？**
- 检查 `npm run dev:webview` 是否正在运行。
- 检查 VS Code 扩展宿主控制台是否有 CSP (Content Security Policy) 报错。
- 确保 `.vite-port` 文件已生成。

**Q: Reload Window 后新代码未生效？**
- 检查 `npm run watch` 终端是否有报错。
- 确认 `dist/extension.js` 文件的时间戳已更新。

**Q: 为什么首次构建这么慢？**
- `npm run protos` 需要编译 Go 和 TS 的 Protocol Buffers，这是必要的耗时步骤。后续的增量编译会跳过此步骤，除非 `.proto` 文件发生变化。

---

*这份文档基于 Cline v3.37.1 源码分析得出。*
