# Tree-sitter 在项目中的使用说明

本文档详细介绍了本项目如何使用 Tree-sitter 进行代码解析、定义提取以及如何扩展对新语言的支持。

## 1. 概述

Tree-sitter 是一个增量解析系统，用于为编程语言构建具体的语法树。在本项目中，我们主要使用它来：

*   **提取代码定义**：从源代码中提取类、函数、方法等定义，用于构建上下文信息，帮助模型更好地理解代码结构。
*   **轻量级解析**：利用 WASM (WebAssembly) 实现高性能解析，避免了原生 Node.js 绑定的兼容性问题，确保在 VS Code 插件环境中的稳定性。

## 2. 核心架构

代码主要位于 `src/services/tree-sitter/` 目录下。

### 2.1. 文件结构

*   `index.ts`: 核心入口。提供 `parseSourceCodeForDefinitionsTopLevel` 函数，负责协调文件读取、解析和结果格式化。
*   `languageParser.ts`: 负责加载 WASM 语言解析器。它管理 `web-tree-sitter` 的初始化和各语言 WASM 文件的加载。
*   `queries/`: 包含各语言的查询语句 (S-expressions)，用于提取特定的语法节点。

### 2.2. 工作流程

1.  **文件筛选**:
    `index.ts` 中的 `separateFiles` 函数根据文件扩展名筛选出需要解析的文件。为了性能考虑，目前策略是优先处理特定扩展名的文件，并限制解析数量（如前 50 个文件）。

2.  **加载解析器**:
    `languageParser.ts` 中的 `loadRequiredLanguageParsers` 根据筛选出的文件扩展名，动态加载对应的 WASM 解析器 (`tree-sitter-*.wasm`)。
    *   项目依赖 `web-tree-sitter` 作为核心库。
    *   使用 `tree-sitter-wasms` 提供预编译的 WASM 文件，避免了本地编译的复杂性。

3.  **解析与提取**:
    `parseFile` 函数执行以下步骤：
    *   读取目标文件的内容。
    *   使用对应语言的解析器生成 AST (抽象语法树)。
    *   使用预定义的查询语句 (`queries/*.ts`) 在 AST 中查找匹配的节点（主要关注函数定义、类定义等）。
    *   格式化输出：提取定义的名称和所在行，忽略具体实现细节，生成简洁的代码大纲。

## 3. 查询语法 (Query Syntax)

Tree-sitter 使用 S-expressions (S 表达式) 来进行模式匹配。我们在 `src/services/tree-sitter/queries/` 下定义了各语言的查询规则。

### 3.1. 捕获 (Captures)

查询中使用 `@` 符号来标记需要捕获的节点。我们主要关注以下捕获名称：

*   `@name`: 定义的名称（如函数名、类名）。这是生成大纲的关键信息。
*   `@definition.*`: 定义的主体（如 `@definition.function`, `@definition.class`）。

### 3.2. 示例 (JavaScript)

以 `src/services/tree-sitter/queries/javascript.ts` 为例：

```scheme
(
  (comment)* @doc
  .
  (function_declaration
    name: (identifier) @name) @definition.function
  (#strip! @doc "^[\\s\\*/]+|^[\\s\\*/]$")
  (#select-adjacent! @doc @definition.function)
)
```

这段查询匹配了一个函数声明 (`function_declaration`)：
*   `name: (identifier) @name`: 捕获函数名为 `@name`。
*   `@definition.function`: 捕获整个函数声明节点。
*   同时处理了关联的注释 (`@doc`)，虽然目前主要使用的是 `@name`。

## 4. 如何添加新语言支持

如果您需要让插件支持解析新的编程语言，请按照以下步骤操作：

### 第一步：确认 WASM 支持

确保 `tree-sitter-wasms` 包中包含该语言的 WASM 文件，或者您可以手动编译该语言的 Tree-sitter 解析器为 WASM 格式。

### 第二步：添加文件扩展名支持

在 `src/services/tree-sitter/index.ts` 的 `separateFiles` 函数中，将新语言的扩展名添加到 `extensions` 数组中。

```typescript
const extensions = [
    // ... 现有语言
    "new_lang", // 添加新语言扩展名 (例如 "rs", "go" 等)
].map((e) => `.${e}`)
```

### 第三步：加载解析器

在 `src/services/tree-sitter/languageParser.ts` 的 `loadRequiredLanguageParsers` 函数中，添加新语言的 `case` 分支。

```typescript
case "new_lang_ext": // 文件扩展名
    language = await loadLanguage("new_language_name") // 对应 tree-sitter-new_language_name.wasm
    query = language.query(newLangQuery) // 下一步创建的查询对象
    break
```

### 第四步：编写查询语句

1.  在 `src/services/tree-sitter/queries/` 目录下创建新文件，例如 `new-lang.ts`。
2.  编写该语言的 Tree-sitter 查询语句。您需要了解该语言在 Tree-sitter 中的语法树结构（可以使用 Tree-sitter 的 playground 或 CLI 工具查看）。
3.  确保捕获 `@name` 以便提取器能识别定义名称。
4.  在 `src/services/tree-sitter/queries/index.ts` 中导出该查询。

```typescript
export { default as newLangQuery } from "./new-lang"
```

### 第五步：验证

运行相关测试或在开发环境中打开该语言的文件，检查是否能正确提取定义名称并生成代码大纲。
