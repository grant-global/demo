# Gemini CLI 项目架构深度分析

> 本文档为 Gemini CLI 项目的完整技术分析,帮助开发者深入理解项目结构和核心逻辑,为后续贡献代码做好准备。

## 目录

1. [项目概述](#项目概述)
2. [技术栈](#技术栈)
3. [项目结构](#项目结构)
4. [核心包分析](#核心包分析)
5. [主体逻辑流程](#主体逻辑流程)
6. [关键设计模式](#关键设计模式)
7. [开发指南](#开发指南)

---

## 项目概述

**Gemini CLI** 是 Google 开发的开源 AI 命令行工具,直接在终端中提供 Gemini AI 模型的访问能力。

### 主要特性

- **免费额度**: 使用个人 Google 账号每分钟 60 次请求,每天 1,000 次请求
- **强大的 Gemini 2.5 Pro**: 支持 100 万 token 上下文窗口
- **内置工具**: Google 搜索接地、文件操作、Shell 命令、Web 抓取
- **可扩展性**: 支持 MCP (Model Context Protocol) 自定义集成
- **终端优先**: 为命令行开发者设计的交互式体验
- **开源**: Apache 2.0 许可证

### 核心功能

1. **代码理解与生成**: 查询和编辑大型代码库
2. **自动化集成**: 支持脚本中的非交互模式
3. **高级能力**: 内置 Google 搜索、对话检查点、自定义上下文文件(GEMINI.md)
4. **GitHub 集成**: 通过 Gemini CLI GitHub Action 实现 PR 审查、Issue 分类等

---

## 技术栈

### 核心技术

```json
{
  "运行时": "Node.js >= 20.0.0",
  "语言": "TypeScript 5.3.3",
  "模块系统": "ESM (type: module)",
  "包管理器": "npm workspaces",
  "构建工具": "esbuild 0.25.0",
  "测试框架": "Vitest 3.1.1+",
  "UI框架": "React 19.1.0 + Ink 6.2.3 (终端 UI)"
}
```

### 主要依赖库

#### AI 和 API 相关
- `@google/genai`: 1.16.0 - Google Gemini API SDK
- `@modelcontextprotocol/sdk`: ^1.15.1 - MCP 协议支持
- `google-auth-library`: ^9.11.0 - Google 认证

#### 文件和系统操作
- `glob`: ^10.4.5 - 文件模式匹配
- `fdir`: ^6.4.6 - 快速目录扫描
- `@joshua.litt/get-ripgrep`: ^0.0.2 - ripgrep 集成
- `simple-git`: ^3.28.0 - Git 操作
- `ignore`: ^7.0.0 - .gitignore 解析
- `picomatch`: ^4.0.1 - 路径匹配

#### 终端和 UI
- `ink`: ^6.2.3 - React 终端渲染
- `@xterm/headless`: 5.5.0 - 终端模拟
- `@lydell/node-pty`: 1.1.0 - 伪终端支持
- `highlight.js`: ^11.11.1 - 代码高亮
- `marked`: ^15.0.12 - Markdown 解析

#### 工具和辅助
- `shell-quote`: ^1.8.3 - Shell 命令解析
- `undici`: ^7.10.0 - HTTP 客户端
- `fzf`: ^0.5.2 - 模糊查找
- `zod`: ^3.23.8 - Schema 验证
- `ajv`: ^8.17.1 - JSON Schema 验证

#### 监控和日志
- `@google-cloud/logging`: ^11.2.1 - Cloud Logging
- `@opentelemetry/sdk-node`: ^0.203.0 - OpenTelemetry 集成

---

## 项目结构

### Monorepo 工作区结构

```
gemini-cli/
├── packages/
│   ├── core/              # 核心业务逻辑和 API 交互
│   ├── cli/               # 用户界面和命令行处理
│   ├── a2a-server/        # Agent-to-Agent 服务器
│   ├── test-utils/        # 测试工具包
│   └── vscode-ide-companion/  # VS Code 扩展
├── integration-tests/     # 集成测试
├── scripts/               # 构建和工具脚本
├── docs/                  # 项目文档
├── bundle/                # 打包输出 (生成)
└── package.json           # 根包配置
```

### 包依赖关系

```
@google/gemini-cli (cli)
    ↓ depends on
@google/gemini-cli-core (core)
    ↓ uses
各种工具和服务
```

---

## 核心包分析

### 1. Core 包 (`packages/core/`)

**职责**: 后端逻辑,与 Gemini API 交互,工具执行编排

#### 目录结构

```
packages/core/src/
├── core/                  # 核心聊天和会话管理
│   ├── geminiChat.ts     # GeminiChat 类 - 管理对话会话
│   ├── turn.ts           # Turn 类 - 管理单个 AI 循环轮次
│   ├── client.ts         # API 客户端封装
│   ├── prompts.ts        # 提示词构建
│   └── coreToolScheduler.ts  # 工具调度器
├── tools/                 # 工具实现
│   ├── tools.ts          # 工具基类和接口定义
│   ├── tool-registry.ts  # 工具注册表
│   ├── read-file.ts      # 文件读取工具
│   ├── write-file.ts     # 文件写入工具
│   ├── edit.ts           # 文件编辑工具
│   ├── shell.ts          # Shell 命令执行
│   ├── web-fetch.ts      # Web 抓取
│   ├── web-search.ts     # Web 搜索
│   ├── grep.ts / ripGrep.ts  # 搜索工具
│   ├── glob.ts           # 文件匹配
│   ├── mcp-client.ts     # MCP 客户端
│   └── ...
├── config/                # 配置管理
│   ├── config.ts         # 主配置类
│   └── storage.ts        # 持久化存储
├── services/              # 服务层
│   ├── fileSystemService.ts
│   ├── gitService.ts
│   ├── shellExecutionService.ts
│   └── chatRecordingService.ts
├── utils/                 # 工具函数
│   ├── filesearch/       # 文件搜索
│   ├── errors.ts         # 错误处理
│   ├── retry.ts          # 重试逻辑
│   └── ...
├── mcp/                   # MCP 协议支持
├── ide/                   # IDE 集成
├── telemetry/             # 遥测和分析
├── code_assist/           # Code Assist 认证
└── index.ts               # 包导出
```

#### 核心类详解

##### 1.1 GeminiChat (`core/geminiChat.ts`)

**作用**: 管理与 Gemini API 的对话会话

```typescript
class GeminiChat {
  // 会话历史
  private history: Content[] = [];

  // 发送消息并返回流式响应
  async sendMessageStream(
    model: string,
    params: SendMessageParameters,
    prompt_id: string,
  ): Promise<AsyncGenerator<StreamEvent>>

  // 获取会话历史(策展或完整)
  getHistory(curated: boolean = false): Content[]

  // 设置工具
  setTools(tools: Tool[]): void

  // 处理流式响应
  private async *processStreamResponse(
    model: string,
    streamResponse: AsyncGenerator<GenerateContentResponse>,
  ): AsyncGenerator<GenerateContentResponse>
}
```

**关键特性**:
- 维护用户和模型之间的完整对话历史
- 支持流式响应处理
- 自动处理无效内容重试(最多 2 次尝试)
- 集成 `ChatRecordingService` 记录对话

##### 1.2 Turn (`core/turn.ts`)

**作用**: 管理单个 Agentic 循环轮次

```typescript
class Turn {
  readonly pendingToolCalls: ToolCallRequestInfo[] = [];

  async *run(
    model: string,
    req: PartListUnion,
    signal: AbortSignal,
  ): AsyncGenerator<ServerGeminiStreamEvent>
}
```

**事件类型** (`GeminiEventType`):
- `Content` - 模型生成的文本内容
- `Thought` - 模型思考过程
- `ToolCallRequest` - 工具调用请求
- `ToolCallResponse` - 工具调用响应
- `Finished` - 轮次完成
- `Error` - 错误事件
- `Retry` - 重试事件
- 等等...

##### 1.3 Tool 系统 (`tools/tools.ts`)

**核心接口**:

```typescript
// 工具调用实例
interface ToolInvocation<TParams, TResult> {
  params: TParams;
  getDescription(): string;
  toolLocations(): ToolLocation[];
  shouldConfirmExecute(signal: AbortSignal): Promise<ToolCallConfirmationDetails | false>;
  execute(signal: AbortSignal, updateOutput?, config?): Promise<TResult>;
}

// 工具构建器
interface ToolBuilder<TParams, TResult> {
  name: string;
  displayName: string;
  description: string;
  kind: Kind;  // Read, Edit, Delete, Execute, etc.
  schema: FunctionDeclaration;
  isOutputMarkdown: boolean;
  canUpdateOutput: boolean;
  build(params: TParams): ToolInvocation<TParams, TResult>;
}

// 声明式工具基类
abstract class DeclarativeTool<TParams, TResult> implements ToolBuilder {
  validateToolParams(params: TParams): string | null;
  abstract build(params: TParams): ToolInvocation<TParams, TResult>;
  async buildAndExecute(...): Promise<TResult>;
}
```

**工具分类** (`Kind`):
- `Read` - 读取操作(文件读取)
- `Edit` - 编辑操作(文件修改)
- `Delete` - 删除操作
- `Execute` - 执行操作(Shell 命令)
- `Search` - 搜索操作
- `Fetch` - 抓取操作
- 等等...

**内置工具**:
1. **文件系统**:
   - `ReadFileTool` - 读取文件内容
   - `WriteFileTool` - 写入文件
   - `EditFileTool` - 编辑文件(diff-based)
   - `ListDirectoryTool` - 列出目录
   - `GlobTool` - 文件模式匹配

2. **搜索**:
   - `RipGrepTool` - 使用 ripgrep 搜索
   - `GrepTool` - 正则表达式搜索

3. **执行**:
   - `ShellTool` - 执行 Shell 命令

4. **网络**:
   - `WebFetchTool` - 抓取 URL 内容
   - `WebSearchTool` - Google 搜索

5. **其他**:
   - `MemoryTool` - 记忆存储
   - `WriteTodosTool` - Todo 列表管理
   - `MCPTool` - MCP 服务器工具包装器

##### 1.4 Config (`config/config.ts`)

**作用**: 集中配置管理

```typescript
class Config {
  // 获取工具注册表
  getToolRegistry(): ToolRegistry;

  // 获取内容生成器
  getContentGenerator(): ContentGenerator;

  // 配置相关
  getWorkingDirectory(): string;
  getModel(): string;

  // 状态管理
  isInFallbackMode(): boolean;
  getQuotaErrorOccurred(): boolean;
}
```

---

### 2. CLI 包 (`packages/cli/`)

**职责**: 用户界面,输入处理,输出渲染

#### 目录结构

```
packages/cli/src/
├── ui/                    # React UI 组件 (Ink)
│   ├── commands/         # 命令实现
│   ├── editors/          # 编辑器集成
│   ├── components/       # UI 组件
│   └── utils/            # UI 工具函数
├── config/                # CLI 配置
│   ├── settings.ts       # 用户设置
│   ├── auth.ts           # 认证管理
│   ├── extension.ts      # 扩展管理
│   ├── policy.ts         # 策略引擎
│   └── trustedFolders.ts # 受信任文件夹
├── services/              # CLI 服务
│   ├── CommandService.ts # 命令服务
│   ├── BuiltinCommandLoader.ts
│   └── FileCommandLoader.ts
├── commands/              # 命令实现
│   ├── extensions/       # 扩展命令
│   └── mcp/              # MCP 命令
├── core/                  # CLI 核心
│   ├── initializer.ts    # 初始化
│   ├── auth.ts           # CLI 认证
│   └── theme.ts          # 主题管理
├── nonInteractiveCli.ts   # 非交互模式
└── index.ts               # 入口点
```

#### 关键组件

##### 2.1 UI 架构 (React + Ink)

使用 **Ink** 库将 React 组件渲染到终端:

```typescript
// 示例: 主 UI 组件
function App() {
  return (
    <Box flexDirection="column">
      <Header />
      <ChatHistory messages={messages} />
      <InputPrompt />
      <StatusBar />
    </Box>
  );
}
```

##### 2.2 命令系统

**内置命令**:
- `/help` - 帮助信息
- `/chat` - 开始新对话
- `/clear` - 清除历史
- `/checkpoint` - 保存/恢复检查点
- `/settings` - 配置设置
- `/extensions` - 管理扩展
- `/mcp` - MCP 服务器管理
- 等等...

**自定义命令**: 通过 `.gemini/commands/` 目录定义

##### 2.3 认证流程

支持多种认证方式:
1. **OAuth (Login with Google)** - 默认推荐
2. **API Key** - `GEMINI_API_KEY`
3. **Vertex AI** - `GOOGLE_API_KEY` + `GOOGLE_GENAI_USE_VERTEXAI=true`

---

### 3. A2A Server 包 (`packages/a2a-server/`)

**职责**: Agent-to-Agent 协议服务器,支持多 Agent 协作

#### 结构

```
packages/a2a-server/src/
├── agent/                 # Agent 执行器
├── http/                  # HTTP 服务器
├── config/                # 配置
├── persistence/           # 持久化(GCS)
└── utils/                 # 工具函数
```

---

### 4. VSCode IDE Companion (`packages/vscode-ide-companion/`)

**职责**: VS Code 扩展,提供 IDE 集成

**功能**:
- Diff 编辑器集成
- 文件操作确认
- MCP 服务器支持

---

## 主体逻辑流程

### 完整交互流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户输入 (CLI)                              │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             CLI Package: 输入处理和验证                            │
│  - 解析命令/提示词                                                  │
│  - 加载上下文 (GEMINI.md, @file 注入)                             │
│  - 检查认证状态                                                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             Core Package: 请求准备                                │
│  - 构建完整提示词 (prompt + history + tools)                       │
│  - 配置生成参数                                                    │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             发送到 Gemini API                                     │
│  - 调用 ContentGenerator.generateContentStream()                │
│  - 处理重试和错误                                                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             流式响应处理 (Turn.run)                               │
│  Loop:                                                           │
│    1. 接收 chunk                                                 │
│    2. 解析内容/思考/工具调用                                        │
│    3. Yield 事件到 CLI                                            │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
  ┌─────────────┐   ┌─────────────────────────────┐
  │  文本内容    │   │    工具调用请求               │
  │  直接显示    │   │  1. 确认 (如需要)             │
  │             │   │  2. 执行工具                 │
  └─────────────┘   │  3. 收集结果                 │
                    │  4. 发送回 Gemini             │
                    └─────────────┬───────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────────┐
                    │    工具执行循环                │
                    │  - 可能触发多个工具            │
                    │  - 迭代直到完成               │
                    └─────────────┬────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             最终响应                                              │
│  - 模型生成最终文本                                                │
│  - FinishReason 事件                                             │
└─────────────────┬───────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│             CLI 显示结果                                          │
│  - Markdown 渲染                                                 │
│  - 代码高亮                                                       │
│  - 更新历史                                                       │
└─────────────────────────────────────────────────────────────────┘
```

### 工具调用详细流程

```typescript
// 1. 模型请求工具调用
{
  functionCalls: [
    { name: "read_file", args: { file_path: "/path/to/file.ts" } }
  ]
}

// 2. Core 接收并验证
const tool = toolRegistry.getTool("read_file");
const invocation = tool.build(args); // 验证参数

// 3. 检查是否需要确认
const confirmDetails = await invocation.shouldConfirmExecute(signal);
if (confirmDetails) {
  // 发送确认事件到 CLI
  yield { type: 'tool_call_confirmation', value: confirmDetails };
  // 等待用户确认...
}

// 4. 执行工具
const result = await invocation.execute(signal, updateOutput);

// 5. 构建响应
const response = {
  callId: request.callId,
  responseParts: [{ functionResponse: { name, response: result.llmContent } }],
  resultDisplay: result.returnDisplay,
  error: result.error
};

// 6. 发送回 Gemini
chat.addHistory({
  role: 'user',
  parts: [{ functionResponse: ... }]
});
```

---

## 关键设计模式

### 1. Builder 模式 (工具系统)

**问题**: 需要灵活地验证和构建工具调用

**解决方案**: 分离验证和执行

```typescript
// 工具定义
class ReadFileTool extends BaseDeclarativeTool {
  // 验证逻辑
  validateToolParamValues(params) {
    if (!params.file_path) return "file_path is required";
    return null;
  }

  // 创建调用实例
  createInvocation(params) {
    return new ReadFileInvocation(params);
  }
}

// 使用
const tool = new ReadFileTool();
const invocation = tool.build({ file_path: "foo.ts" }); // 可能抛出验证错误
const result = await invocation.execute(signal);
```

### 2. Event Stream 模式 (异步生成器)

**问题**: 需要处理长时间运行的 AI 响应和工具调用

**解决方案**: 使用 `AsyncGenerator` 流式传递事件

```typescript
async function* processAIResponse() {
  yield { type: 'content', value: 'Thinking...' };
  yield { type: 'thought', value: { subject: '...', description: '...' } };
  yield { type: 'tool_call_request', value: { name: 'read_file', ... } };
  // ... 执行工具 ...
  yield { type: 'tool_call_response', value: { result: ... } };
  yield { type: 'content', value: 'Based on the file...' };
  yield { type: 'finished', value: { reason: 'STOP' } };
}
```

### 3. Registry 模式 (工具注册)

**问题**: 动态管理可用工具集

**解决方案**: `ToolRegistry` 集中管理

```typescript
class ToolRegistry {
  private tools = new Map<string, AnyDeclarativeTool>();

  registerTool(tool: AnyDeclarativeTool) {
    this.tools.set(tool.name, tool);
  }

  getTool(name: string): AnyDeclarativeTool | undefined {
    return this.tools.get(name);
  }

  getAllTools(): AnyDeclarativeTool[] {
    return Array.from(this.tools.values());
  }
}
```

### 4. Confirmation Bus (消息总线)

**问题**: CLI 和 Core 需要双向通信(工具确认)

**解决方案**: `MessageBus` 发布-订阅模式

```typescript
// Core 发布确认请求
messageBus.publish({
  type: 'TOOL_CONFIRMATION_REQUEST',
  toolCall: { name: 'shell', args: { command: 'rm -rf /' } },
  correlationId: uuid()
});

// CLI 订阅并响应
messageBus.subscribe('TOOL_CONFIRMATION_RESPONSE', (response) => {
  if (response.correlationId === uuid) {
    // 处理用户决定
  }
});
```

### 5. Strategy 模式 (认证)

**问题**: 支持多种认证方式

**解决方案**: 不同的 `ContentGenerator` 实现

```typescript
// OAuth
const generator = new CodeAssistContentGenerator(config);

// API Key
const generator = new GenAIContentGenerator(apiKey);

// Vertex AI
const generator = new VertexAIContentGenerator(credentials);
```

---

## 开发指南

### 环境设置

```bash
# 1. 克隆仓库
git clone https://github.com/google-gemini/gemini-cli.git
cd gemini-cli

# 2. 安装依赖
npm install

# 3. 构建项目
npm run build

# 4. 运行开发版本
npm run start

# 5. 运行测试
npm test
```

### 开发工作流

#### 完整验证 (推荐在提交前运行)

```bash
npm run preflight
```

这会执行:
1. `npm run clean` - 清理
2. `npm ci` - 安装依赖
3. `npm run format` - 格式化代码
4. `npm run lint:ci` - Lint 检查
5. `npm run build` - 构建
6. `npm run typecheck` - 类型检查
7. `npm run test:ci` - 运行测试

#### 单独命令

```bash
# 格式化
npm run format

# Lint
npm run lint
npm run lint:fix

# 类型检查
npm run typecheck

# 测试
npm test                              # 所有测试
npm run test:integration:sandbox:none # 集成测试(无沙箱)
npm run test:e2e                      # E2E 测试

# 构建
npm run build                         # 构建所有包
npm run build:packages                # 仅构建工作区
npm run bundle                        # 打包 CLI
```

### 代码规范

#### TypeScript 最佳实践

参考 `GEMINI.md` 中的详细指南:

1. **优先使用纯对象而非类**
   ```typescript
   // ✅ 推荐
   interface User {
     name: string;
     email: string;
   }

   // ❌ 避免
   class User {
     constructor(public name: string, public email: string) {}
   }
   ```

2. **使用 ES 模块封装而非类成员**
   ```typescript
   // ✅ 推荐 - 私有函数
   function helperFunction() { ... }  // 未导出 = 私有

   export function publicAPI() {
     return helperFunction();
   }

   // ❌ 避免
   class MyClass {
     private helperMethod() { ... }
   }
   ```

3. **避免 `any`,优先使用 `unknown`**
   ```typescript
   // ✅ 推荐
   function processValue(value: unknown) {
     if (typeof value === 'string') {
       console.log(value.toUpperCase());
     }
   }

   // ❌ 避免
   function processValue(value: any) {
     console.log(value.toUpperCase()); // 不安全
   }
   ```

4. **类型穷尽检查**
   ```typescript
   function checkExhaustive(value: never): never {
     throw new Error(`Unexpected value: ${value}`);
   }

   switch (type) {
     case 'read': return handleRead();
     case 'write': return handleWrite();
     default: return checkExhaustive(type);
   }
   ```

#### React/Ink 规范

1. **仅使用函数组件和 Hooks**
2. **保持组件纯净,无渲染副作用**
3. **避免过度使用 `useEffect`**
4. **不在 `useEffect` 中 `setState`**(性能问题)
5. **利用 React Compiler,避免手动优化**

### 测试指南

#### 测试结构

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';

describe('MyComponent', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should do something', async () => {
    // Arrange
    const mock = vi.fn().mockResolvedValue('result');

    // Act
    const result = await myFunction(mock);

    // Assert
    expect(result).toBe('expected');
    expect(mock).toHaveBeenCalledWith('args');
  });
});
```

#### 模拟 (Mocking)

```typescript
// ES 模块模拟
vi.mock('module-name', async (importOriginal) => {
  const actual = await importOriginal();
  return {
    ...actual,
    specificFunction: vi.fn(),
  };
});

// Hoisting
const myMock = vi.hoisted(() => vi.fn());

vi.mock('dependency', () => ({
  something: myMock,
}));
```

### 添加新工具

#### 步骤

1. **创建工具类** (`packages/core/src/tools/my-tool.ts`)

```typescript
import { BaseDeclarativeTool, BaseToolInvocation, Kind } from './tools.js';

interface MyToolParams {
  arg1: string;
  arg2?: number;
}

class MyToolInvocation extends BaseToolInvocation<MyToolParams, ToolResult> {
  getDescription(): string {
    return `Executing my tool with ${this.params.arg1}`;
  }

  async execute(signal: AbortSignal): Promise<ToolResult> {
    // 实现逻辑
    return {
      llmContent: `Result: ${this.params.arg1}`,
      returnDisplay: 'Success!',
    };
  }
}

export class MyTool extends BaseDeclarativeTool<MyToolParams, ToolResult> {
  constructor() {
    super(
      'my_tool',              // name
      'My Tool',              // displayName
      'Description of tool',  // description
      Kind.Other,             // kind
      {                       // JSON Schema
        type: 'object',
        properties: {
          arg1: { type: 'string' },
          arg2: { type: 'number' },
        },
        required: ['arg1'],
      }
    );
  }

  protected createInvocation(params: MyToolParams): MyToolInvocation {
    return new MyToolInvocation(params);
  }
}
```

2. **注册工具** (`packages/core/src/tools/tool-registry.ts`)

```typescript
import { MyTool } from './my-tool.js';

export function createDefaultToolRegistry(): ToolRegistry {
  const registry = new ToolRegistry();
  // ... existing tools ...
  registry.registerTool(new MyTool());
  return registry;
}
```

3. **导出** (`packages/core/src/index.ts`)

```typescript
export * from './tools/my-tool.js';
```

4. **编写测试** (`packages/core/src/tools/my-tool.test.ts`)

```typescript
import { describe, it, expect } from 'vitest';
import { MyTool } from './my-tool.js';

describe('MyTool', () => {
  it('should execute successfully', async () => {
    const tool = new MyTool();
    const invocation = tool.build({ arg1: 'test' });
    const result = await invocation.execute(new AbortController().signal);
    expect(result.llmContent).toContain('test');
  });

  it('should validate params', () => {
    const tool = new MyTool();
    expect(() => tool.build({} as any)).toThrow();
  });
});
```

### 添加新命令

在 `.gemini/commands/my-command.md` 或通过扩展系统实现。

### 贡献流程

1. **Fork 仓库**
2. **创建功能分支**: `git checkout -b feature/my-feature`
3. **编写代码和测试**
4. **运行 preflight**: `npm run preflight`
5. **提交**: `git commit -m "feat: add my feature"`
6. **推送**: `git push origin feature/my-feature`
7. **创建 PR**: 参考 `CONTRIBUTING.md`

---

## 参考资源

### 官方文档
- [Gemini CLI 文档](https://geminicli.com/docs/)
- [贡献指南](./CONTRIBUTING.md)
- [架构概述](./docs/architecture.md)
- [Roadmap](./ROADMAP.md)

### API 参考
- [@google/genai SDK](https://github.com/googleapis/js-genai)
- [MCP 协议](https://modelcontextprotocol.io/)
- [Gemini API](https://ai.google.dev/gemini-api/docs)

### 相关工具
- [Vitest 文档](https://vitest.dev/)
- [Ink 文档](https://github.com/vadimdemedes/ink)
- [esbuild 文档](https://esbuild.github.io/)

---

## 总结

Gemini CLI 是一个设计良好的 Monorepo 项目,采用现代 TypeScript 和 React 技术栈,核心架构清晰分离为:

1. **CLI 包** - 用户界面和交互
2. **Core 包** - 业务逻辑和 API 交互
3. **工具系统** - 可扩展的能力插件

关键亮点:
- **模块化设计**: 清晰的包边界和职责
- **类型安全**: 全面的 TypeScript 类型定义
- **测试覆盖**: Vitest 测试框架,集成和单元测试
- **可扩展性**: MCP 协议,自定义工具和命令
- **流式处理**: AsyncGenerator 模式处理实时响应

开始贡献前,建议:
1. 阅读 `CONTRIBUTING.md` 和 `GEMINI.md`
2. 运行 `npm run preflight` 熟悉工作流
3. 查看现有工具和测试代码作为参考
4. 从小的改进或 bug 修复开始

祝你在 Gemini CLI 项目中的贡献之旅顺利! 🚀
