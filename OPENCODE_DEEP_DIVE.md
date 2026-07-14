# OpenCode 项目深度解读文档（含二开示例）

> 本文档对 [opencode](https://opencode.ai) 项目进行全面深度解读，包含精确文件路径、源码分析与二次开发示例代码。

---

## 目录

1. [项目概览](#1-项目概览)
2. [整体架构](#2-整体架构)
3. [技术栈](#3-技术栈)
4. [Monorepo 包结构](#4-monorepo-包结构)
5. [核心包深度解析](#5-核心包深度解析)
   - 5.1 [packages/core — 业务核心](#51-packagescore--业务核心)
   - 5.2 [packages/llm — LLM 集成层](#52-packagesllm--llm-集成层)
   - 5.3 [packages/server — HTTP API 服务](#53-packagesserver--http-api-服务)
   - 5.4 [packages/opencode — CLI 主入口](#54-packagesopencode--cli-主入口)
   - 5.5 [packages/tui — 终端 UI](#55-packagestui--终端-ui)
   - 5.6 [packages/app — Web UI](#56-packagesapp--web-ui)
   - 5.7 [packages/plugin — 插件 SDK](#57-packagesplugin--插件-sdk)
   - 5.8 [packages/client — 生成客户端](#58-packagesclient--生成客户端)
   - 5.9 [packages/sdk-next — 嵌入式 SDK](#59-packagessdk-next--嵌入式-sdk)
6. [数据库结构](#6-数据库结构)
7. [Session 生命周期与数据流](#7-session-生命周期与数据流)
8. [System Context 系统](#8-system-context-系统)
9. [工具系统（Tool System）](#9-工具系统tool-system)
10. [权限系统](#10-权限系统)
11. [LLM Provider 集成架构](#11-llm-provider-集成架构)
12. [配置系统](#12-配置系统)
13. [事件系统](#13-事件系统)
14. [自动压缩（Compaction）](#14-自动压缩compaction)
15. [二次开发指南（二开示例）](#15-二次开发指南二开示例)
    - 15.1 [开发自定义工具](#151-开发自定义工具)
    - 15.2 [注入自定义 System Context](#152-注入自定义-system-context)
    - 15.3 [开发 Plugin](#153-开发-plugin)
    - 15.4 [接入自定义 LLM Provider](#154-接入自定义-llm-provider)
    - 15.5 [嵌入式 SDK 使用](#155-嵌入式-sdk-使用)
    - 15.6 [CLI 自动化脚本](#156-cli-自动化脚本)
    - 15.7 [添加 CLI 子命令](#157-添加-cli-子命令)
    - 15.8 [监听事件流](#158-监听事件流)
16. [开发规范与贡献指南](#16-开发规范与贡献指南)

---

## 1. 项目概览

**OpenCode** 是一个开源 AI 编程 Agent，提供多种使用方式：

| 界面方式 | 说明 | 启动命令 |
|----------|------|---------|
| TUI（终端 UI）| SolidJS + OpenTUI，终端内全功能运行 | `opencode [directory]` |
| Web UI | 浏览器访问 | `opencode web` |
| Desktop App | Electron 封装（Beta） | 独立安装包 |
| Headless API | HTTP 服务器模式 | `opencode serve` |
| Embedded SDK | 作为库嵌入其他应用 | `import from "@opencode-ai/sdk-next"` |

**安装：**
```bash
curl -fsSL https://opencode.ai/install | bash
npm i -g opencode-ai@latest
brew install anomalyco/tap/opencode
```

---

## 2. 整体架构

```
┌────────────────────────────────────────────────────────────────┐
│                        用户界面层                               │
│  packages/tui (Terminal)  │  packages/app (Web/Desktop)        │
└───────────────────────────────┬────────────────────────────────┘
                                │ HTTP / In-Memory Transport
┌───────────────────────────────▼────────────────────────────────┐
│              API 服务层 (packages/server)                       │
│  packages/server/src/routes.ts     ← 路由创建入口               │
│  packages/server/src/api.ts        ← HttpApi 定义              │
│  packages/server/src/handlers/     ← 各资源处理器               │
│  packages/server/src/middleware/   ← Auth/Authorization/Schema  │
└───────────────────────────────┬────────────────────────────────┘
                                │ Effect Services
┌───────────────────────────────▼────────────────────────────────┐
│              核心业务层 (packages/core)                         │
│  src/session.ts            ← Session 服务主接口                 │
│  src/session/runner/       ← Provider Turn 执行器               │
│  src/catalog.ts            ← Provider/Model 目录                │
│  src/config.ts             ← 配置加载                           │
│  src/tool/                 ← 工具注册与执行                      │
│  src/system-context/       ← 系统上下文代数                      │
│  src/permission.ts         ← 权限规则评估                        │
│  src/event.ts              ← 事件溯源引擎                        │
│  src/database/             ← SQLite + Drizzle ORM               │
└───────────────────────────────┬────────────────────────────────┘
                                │
┌───────────────────────────────▼────────────────────────────────┐
│              LLM 集成层 (packages/llm)                          │
│  src/llm.ts                ← 主入口（stream/generate/generateObject）│
│  src/protocols/            ← 各 Provider 协议适配器               │
│  src/providers/            ← Provider 配置工厂                   │
│  src/route/                ← Route/Auth/Endpoint/Transport       │
└────────────────────────────────────────────────────────────────┘
```

**关键依赖方向（严格单向）：**
```
packages/schema
  ↓
packages/core / packages/protocol / packages/llm
  ↓
packages/server
  ↓
packages/client
  ↓
packages/app / packages/tui / packages/desktop
```

`packages/sdk-next` 是例外，它横跨 Client + Core + Server（进程内嵌入模式）。

---

## 3. 技术栈

| 类别 | 技术 | 版本 | 关键文件 |
|------|------|------|---------|
| **运行时** | Bun | 1.3.14+ | `bunfig.toml` |
| **语言** | TypeScript | 5.8.2 | `tsconfig.json` |
| **效应系统** | Effect-TS | 4.0.0-beta.83 | `package.json` catalog |
| **HTTP 框架** | Hono | 4.10.7 | `packages/server/src/` |
| **UI 框架** | SolidJS | 1.9.10 | `packages/app/`, `packages/tui/` |
| **终端 UI** | OpenTUI (@opentui/*) | 0.4.3 | `packages/tui/src/app.tsx` |
| **ORM** | Drizzle ORM | 1.0.0-rc.2 | `packages/core/src/database/` |
| **AI SDK** | Vercel AI SDK (ai) | 6.0.168 | `packages/llm/src/` |
| **语法高亮** | Shiki | 4.2.0 | `packages/app/` |
| **构建** | Turbo | 2.10.2 | `turbo.json` |
| **数据库** | SQLite | - | `packages/core/src/database/schema.gen.ts` |
| **Lint** | oxlint | 1.60.0 | `.oxlintrc.json` |
| **部署** | SST v4 | 4.13.1 | `infra/`, `sst.config.ts` |

---

## 4. Monorepo 包结构

```
f:\code\opencode\
├── packages/
│   ├── core/                   # 核心业务逻辑（最重要）
│   │   ├── src/session.ts      # Session 服务主接口
│   │   ├── src/catalog.ts      # Provider/Model 目录
│   │   ├── src/config.ts       # 配置加载
│   │   ├── src/tool/           # 工具系统
│   │   ├── src/system-context/ # 系统上下文
│   │   ├── src/permission.ts   # 权限系统
│   │   ├── src/event.ts        # 事件溯源
│   │   └── src/database/       # 数据持久化
│   ├── llm/                    # LLM 集成抽象层
│   │   ├── src/llm.ts          # 主入口
│   │   ├── src/protocols/      # 协议适配器
│   │   ├── src/providers/      # Provider 定义
│   │   └── src/route/          # Route/Auth/Transport
│   ├── server/                 # HTTP API 服务器
│   │   ├── src/routes.ts       # 路由创建
│   │   ├── src/api.ts          # HttpApi 定义
│   │   ├── src/handlers/       # 请求处理器
│   │   └── src/middleware/     # 中间件
│   ├── opencode/               # CLI 主入口
│   │   ├── src/index.ts        # CLI 入口（yargs）
│   │   └── src/cli/cmd/        # 各子命令
│   ├── tui/                    # 终端 UI
│   │   └── src/app.tsx         # TUI 主应用
│   ├── app/                    # Web UI（共享）
│   │   └── src/app.tsx         # Web 主应用
│   ├── session-ui/             # 会话 UI 组件
│   ├── plugin/                 # 插件 SDK
│   │   └── src/index.ts        # Hooks 接口定义
│   ├── client/                 # 生成的客户端 SDK
│   ├── protocol/               # HTTP API 路由定义（无实现）
│   ├── schema/                 # 共享类型定义（最底层）
│   ├── sdk-next/               # 嵌入式 SDK
│   ├── desktop/                # Electron 桌面应用
│   ├── console/                # 管理控制台
│   └── ui/                     # 共享 UI 原语
├── infra/                      # SST 基础设施定义
├── specs/v2/                   # V2 架构设计文档
├── .opencode/                  # OpenCode 自身的配置
│   └── opencode.jsonc          # 项目配置示例
└── package.json                # Monorepo 根配置
```

---

## 5. 核心包深度解析

### 5.1 packages/core — 业务核心

**根路径：** `f:\code\opencode\packages\core\src\`

#### 完整目录树

```
src/
├── session.ts                  ← Session 服务主接口（约 487 行）
├── session/
│   ├── schema.ts               ← Session ID、Info 类型
│   ├── message.ts              ← Message 类型（user/assistant/tool）
│   ├── prompt.ts               ← Prompt 数据结构
│   ├── input.ts                ← Prompt 持久化接收（Inbox）
│   ├── execution.ts            ← 执行协调器接口（约 35 行）
│   ├── execution/
│   │   └── local.ts            ← 本地进程内执行实现
│   ├── runner/
│   │   ├── index.ts            ← SessionRunner 接口定义
│   │   └── model.ts            ← 模型解析器（catalog → LLM Model）
│   ├── projector.ts            ← 事件投影器（重建 Session 状态）
│   ├── store.ts                ← Session 状态内存缓存
│   ├── event.ts                ← Session 事件类型定义
│   ├── revert.ts               ← 撤销/恢复逻辑
│   ├── compaction.ts           ← 上下文压缩逻辑
│   ├── sql.ts                  ← Drizzle 表定义
│   └── error.ts                ← Session 错误类型
├── catalog.ts                  ← Provider/Model 目录（约 302 行）
├── config.ts                   ← 配置加载主入口（约 228 行）
├── config/
│   ├── agent.ts                ← Agent 配置结构
│   ├── mcp.ts                  ← MCP 服务器配置
│   ├── provider.ts             ← Provider 配置结构
│   ├── compaction.ts           ← 压缩配置
│   ├── formatter.ts            ← 格式化器配置
│   ├── lsp.ts                  ← LSP 配置
│   ├── attachments.ts          ← 附件处理配置
│   ├── tool-output.ts          ← 工具输出限制配置
│   ├── watcher.ts              ← 文件监视配置
│   ├── reference.ts            ← 外部参考配置
│   ├── command.ts              ← 自定义命令配置
│   └── plugin.ts               ← 插件配置
├── system-context/
│   ├── index.ts                ← System Context 代数（约 321 行）
│   ├── registry.ts             ← Location-scoped 注册表（约 50 行）
│   └── builtins.ts             ← 内置源（环境/日期）注册
├── tool/
│   ├── tool.ts                 ← Tool.make 工厂函数（约 163 行）
│   ├── tools.ts                ← Tools.Service 注册接口
│   ├── registry.ts             ← 工具注册表（约 148 行）
│   ├── application-tools.ts    ← 进程全局工具池（约 58 行）
│   ├── bash.ts                 ← Bash 工具实现（约 208 行）
│   ├── read.ts                 ← 文件读取工具（约 118 行）
│   ├── read-filesystem.ts      ← 文件系统读写抽象（约 200+ 行）
│   ├── edit.ts                 ← 文件编辑工具
│   ├── write.ts                ← 文件写入工具
│   ├── glob.ts                 ← 文件搜索工具
│   ├── grep.ts                 ← 代码搜索工具
│   ├── webfetch.ts             ← 网络请求工具
│   ├── websearch.ts            ← 网络搜索工具
│   ├── apply-patch.ts          ← Patch 应用工具
│   ├── question.ts             ← 交互式提问工具
│   ├── skill.ts                ← Skill 调用工具
│   ├── todowrite.ts            ← TODO 管理工具
│   └── builtins.ts             ← 汇总所有内置工具
├── tool-output-store.ts        ← 工具输出边界管理（约 212 行）
├── agent.ts                    ← Agent 服务（约 112 行）
├── permission.ts               ← 权限服务（约 311 行）
├── event.ts                    ← 事件溯源引擎（约 639 行）
├── database/
│   ├── database.ts             ← Drizzle ORM 连接
│   ├── schema.gen.ts           ← 生成的数据库 Schema
│   └── migration/              ← 数据库迁移文件
├── location.ts                 ← Location（工作目录）服务（约 40 行）
├── location-service-map.ts     ← Location → Service 映射
├── location-mutation.ts        ← 文件路径解析与外部目录检测
├── model.ts                    ← Model 引用结构
├── provider.ts                 ← Provider 元数据接口
├── project.ts                  ← 项目信息（根目录、VCS）
├── workspace.ts                ← Workspace 管理
├── credential.ts               ← 凭证存储
├── integration.ts              ← 外部集成（OAuth 等）
├── snapshot.ts                 ← 文件系统快照（undo/revert）
├── fs-util.ts                  ← 文件系统工具集
├── global.ts                   ← 全局路径（~/.local/share/opencode）
├── policy.ts                   ← OPA 风格策略评估
└── state.ts                    ← 不可变状态管理（草稿模式）
```

#### Session 服务完整接口

**文件：** `packages/core/src/session.ts`

```typescript
// packages/core/src/session.ts 第 113-180 行
export interface Interface {
  // 列出 Session（支持按目录/项目/Workspace 过滤、分页、搜索）
  readonly list: (input?: ListInput) => Effect.Effect<SessionSchema.Info[]>
  
  // 创建新 Session
  readonly create: (input: CreateInput) => Effect.Effect<SessionSchema.Info>
  
  // 获取单个 Session（不存在则 NotFoundError）
  readonly get: (sessionID: SessionSchema.ID) => Effect.Effect<SessionSchema.Info, NotFoundError>
  
  // 分页获取消息历史
  readonly messages: (input: {
    sessionID: SessionSchema.ID
    limit?: number
    order?: "asc" | "desc"
    cursor?: { id: SessionMessage.ID; direction: "previous" | "next" }
  }) => Effect.Effect<SessionMessage.Message[], NotFoundError | MessageDecodeError>
  
  // 获取单条消息
  readonly message: (input: { sessionID; messageID }) => Effect.Effect<SessionMessage.Message | undefined>
  
  // 获取 Session 上下文消息
  readonly context: (sessionID) => Effect.Effect<SessionMessage.Message[], ...>
  
  // SSE 订阅 durable 事件流（支持从指定序列号之后重播）
  readonly events: (input: { sessionID; after?: number }) => Stream.Stream<DurableEvent, NotFoundError>
  
  // 有限分页历史（REST 友好）
  readonly history: (input: { sessionID; after?: number; limit: number }) 
    => Effect.Effect<{ events: ReadonlyArray<DurableEvent>; hasMore: boolean }, NotFoundError>
  
  // 切换 Agent
  readonly switchAgent: (input: { sessionID; agent: string }) => Effect.Effect<void, NotFoundError>
  
  // 切换 Model
  readonly switchModel: (input: { sessionID; model: ModelV2.Ref }) => Effect.Effect<void, NotFoundError>
  
  // 发送 Prompt（核心操作）
  // delivery: "steer"（默认）= 在下一个安全边界插入
  // delivery: "queue" = 排队等待当前 drain 完成
  // resume: false = 只接收不执行
  readonly prompt: (input: {
    id?: SessionMessage.ID       // 可选的幂等 ID
    sessionID: SessionSchema.ID
    prompt: PromptInput.Prompt   // { text, files?, agents? }
    delivery?: SessionInput.Delivery  // "steer" | "queue"
    resume?: boolean             // false = 只持久化，不触发执行
  }) => Effect.Effect<SessionInput.Admitted, NotFoundError | PromptConflictError>
  
  // 触发压缩
  readonly compact: (input: CompactInput) => Effect.Effect<void, NotFoundError | OperationUnavailableError>
  
  // 等待 Session 空闲
  readonly wait: (id: SessionSchema.ID) => Effect.Effect<void, ...>
  
  // 当前进程活跃的 Session 集合（快照）
  readonly active: Effect.Effect<ReadonlySet<SessionSchema.ID>>
  
  // 恢复执行（用于从已有历史继续）
  readonly resume: (sessionID) => Effect.Effect<void, ...>
  
  // 中断当前执行
  readonly interrupt: (sessionID) => Effect.Effect<void>
  
  // 撤销/恢复操作
  readonly revert: {
    readonly stage: (input: { sessionID; messageID; files?: boolean }) => Effect.Effect<Revert.State, ...>
    readonly clear: (sessionID) => Effect.Effect<void, ...>
    readonly commit: (sessionID) => Effect.Effect<void, ...>
  }
}
```

#### Agent 服务接口

**文件：** `packages/core/src/agent.ts`

```typescript
// 内置 Agent：build（默认，全权限）、plan（只读）、general（通用子 Agent）
export const defaultID = ID.make("build")

export interface Interface extends State.Transformable<Draft> {
  get(id: ID): Effect<Info | undefined>        // 获取特定 Agent
  default(): Effect<Info | undefined>           // 获取默认 Agent
  resolve(id?: ID | string): Effect<Info | undefined>  // 有 ID 则查找，无则返回默认
  select(id?: ID | string): Effect<Selection>   // 总是返回有效选择（fallback 到 build）
  all(): Effect<Info[]>                         // 列出所有可见 Agent
}

// Draft 接口用于 State 热更新
export type Draft = {
  list: () => readonly Info[]
  get: (id: ID) => Info | undefined
  default: (id: ID | undefined) => void
  update: (id: ID, fn: (agent: DeepMutable<Info>) => void) => void
  remove: (id: ID) => void
}
```

#### Catalog 接口

**文件：** `packages/core/src/catalog.ts`

`model.small()` 的选择算法（权重：成本 80% + 发布时间 20%）：
```typescript
// packages/core/src/catalog.ts 第 249-286 行
const SMALL_MODEL_RE = /\b(nano|flash|lite|mini|haiku|small|fast)\b/

// 候选：最近 18 个月内发布、有成本数据、支持文本输入输出的模型
// 优先选名称匹配 SMALL_MODEL_RE 的，否则从所有候选中选成本最低的
const pick = (items) => Array.sortWith(
  item => (item.cost / maxCost) * 0.8 + (item.age / maxAge) * 0.2,
  Order.Number
)
```

#### Tool Output Store

**文件：** `packages/core/src/tool-output-store.ts`

```typescript
// 默认限制
export const MAX_LINES = 2_000
export const MAX_BYTES = 50 * 1024  // 50 KB
export const RETENTION = Duration.days(7)
export const MANAGED_DIRECTORY = "tool-output"  // ~/.local/share/opencode/tool-output/

// 超出处理：保留头尾各 50%，完整内容存到临时文件
// 模型看到：
// "... output truncated; full content saved to /path/to/tool_01JXXXX ..."
```

---

### 5.2 packages/llm — LLM 集成层

**根路径：** `f:\code\opencode\packages\llm\src\`

#### 目录结构

```
src/
├── llm.ts                      ← 主入口：stream/generate/generateObject（约 187 行）
├── tool.ts                     ← LLM Tool 类型系统（约 254 行）
├── schema.ts                   ← LLMRequest/LLMResponse/LLMEvent 类型
├── route/
│   ├── client.ts               ← Route/RouteBody 接口（约 80 行）
│   ├── auth.ts                 ← Auth 类型（bearer/header/none）
│   ├── auth-options.ts         ← AuthOptions.bearer() 工厂
│   ├── endpoint.ts             ← Endpoint（baseURL/path/query）
│   ├── transport.ts            ← HttpTransport/WebSocketExecutor
│   ├── executor.ts             ← RequestExecutor
│   ├── framing.ts              ← SSE 帧解析
│   └── protocol.ts             ← 协议注册表
├── protocols/
│   ├── openai-chat.ts          ← OpenAI Chat Completions（约 100+ 行）
│   ├── openai-responses.ts     ← OpenAI Responses API
│   ├── anthropic-messages.ts   ← Anthropic Messages API
│   ├── gemini.ts               ← Google Gemini API
│   ├── bedrock-converse.ts     ← Amazon Bedrock Converse
│   ├── openai-compatible.ts    ← 通用 OpenAI 兼容端点
│   ├── openai-compatible-chat.ts
│   └── shared.ts               ← 共享工具函数（ProviderShared）
└── providers/
    ├── openai.ts               ← OpenAI Provider 工厂（约 64 行）
    ├── anthropic.ts
    ├── google.ts
    ├── azure.ts
    ├── bedrock.ts
    ├── openrouter.ts
    ├── xai.ts
    ├── github-copilot.ts
    └── cloudflare.ts
```

#### 核心 API

**文件：** `packages/llm/src/llm.ts`

```typescript
// 流式调用（主要用于 Session Runner）
export const stream: (request: LLMRequest) => Stream<LLMEvent, LLMError>

// 一次性调用（等待完整响应）
export const generate: (request: LLMRequest) => Effect<LLMResponse, LLMError>

// 结构化输出（通过强制工具调用实现，所有 Provider 行为一致）
// 支持两种模式：
//   1. schema: EffectSchema<T>  → 编译时类型安全
//   2. jsonSchema: JsonSchema   → 运行时 JSON Schema
export function generateObject<S extends ToolSchema>(
  options: { schema: S } & RequestBase
): Effect<GenerateObjectResponse<Schema.Type<S>>, LLMError>

export function generateObject(
  options: { jsonSchema: JsonSchema } & RequestBase
): Effect<GenerateObjectResponse<unknown>, LLMError>
```

#### OpenAI Provider 工厂模式（二开参考）

**文件：** `packages/llm/src/providers/openai.ts`

```typescript
// 标准 Provider 工厂模式（二开自定义 Provider 参考此结构）
export const configure = (input: Config = {}) => {
  // 1. 将配置应用到各协议路由
  const responsesRoute = configuredRoute(OpenAIResponses.route, input)
  const chatRoute = configuredRoute(OpenAIChat.route, input)
  
  // 2. model 工厂函数：接受 modelID，返回 Model 实例
  const responses = (id: string | ModelID) =>
    responsesRoute
      .with(withOpenAIOptions(id, modelDefaults, { textVerbosity: true }))
      .model({ id })

  return {
    id,            // ProviderID
    model: responses,   // 默认使用 responses API
    responses,
    chat,
    configure,     // 支持继续配置
  }
}

export const provider = configure()  // 使用默认配置的 Provider 实例
export const model = provider.model  // 快捷访问
```

---

### 5.3 packages/server — HTTP API 服务

**根路径：** `f:\code\opencode\packages\server\src\`

#### 路由创建

**文件：** `packages/server/src/routes.ts`

```typescript
// 两种创建路由的方式
export function createRoutes(password?: string) // 网络模式（可选密码保护）
export function createEmbeddedRoutes()           // 嵌入模式（无 TCP 监听）

// 应用服务层（所有 Session 相关的基础服务）
const applicationServices = LayerNode.group([
  Database.node,           // SQLite 数据库
  EventV2.node,            // 事件溯源引擎
  httpClient,              // HTTP 客户端（用于 Provider 调用）
  ToolOutputStore.cleanupNode,  // 工具输出文件定期清理
  SessionV2.node,          // Session 服务
  PermissionSaved.node,    // 已保存权限规则
  PtyTicket.node,          // PTY 分配
  Credential.node,         // 凭证存储
  PtyEnvironment.node,     // PTY 环境变量
  LocationServiceMap.node, // Location → Service 映射
])
```

#### 处理器结构

**目录：** `packages/server/src/handlers/`

```
handlers/
├── session.ts     ← Session CRUD、prompt、switchAgent/Model、revert、compact
├── message.ts     ← 消息操作
├── provider.ts    ← Provider 目录查询
├── model.ts       ← 模型信息
├── credential.ts  ← 凭证管理（API Key 存储）
├── permission.ts  ← 权限请求/回复
├── pty.ts         ← PTY 分配（用于终端工具）
├── event.ts       ← 全局 SSE 事件订阅
├── fs.ts          ← 文件系统操作
├── integration.ts ← OAuth 等集成
├── command.ts     ← Slash 命令执行
└── session/       ← Session 子操作（history、events 等）
```

#### 关键 API 端点（来自 `packages/server/src/handlers/session.ts`）

```typescript
// Session 处理器注册方式（HttpApiBuilder.group）
export const SessionHandler = HttpApiBuilder.group(Api, "server.session", (handlers) =>
  Effect.gen(function* () {
    const session = yield* SessionV2.Service  // 注入 Session 服务

    return handlers
      .handle("session.list", ...)
      .handle("session.create", ...)
      .handle("session.prompt", ...)    // 关键：发送 Prompt
      .handle("session.switchAgent", ...)
      .handle("session.switchModel", ...)
      .handle("session.revert.stage", ...)
      // ... 更多
  })
)
```

#### 中间件链

```
请求 → ServerAuth（Basic Auth 验证）
     → authorizationLayer（Location 级授权）
     → schemaErrorLayer（Schema 错误格式化）
     → locationLayer（解析 x-opencode-directory 头）
     → sessionLocationLayer（从 Session ID 解析 Location）
     → handlers（实际业务处理）
```

---

### 5.4 packages/opencode — CLI 主入口

**根路径：** `f:\code\opencode\packages\opencode\src\`

#### CLI 入口

**文件：** `packages/opencode/src/index.ts`

```typescript
// yargs CLI 框架，所有命令在此注册
const cli = yargs(args)
  .command(TuiThreadCommand)   // 默认命令：启动 TUI
  .command(ServeCommand)       // opencode serve
  .command(WebCommand)         // opencode web
  .command(RunCommand)         // opencode run <message>
  .command(DebugCommand)       // opencode debug
  .command(SessionCommand)     // opencode session list/info/messages/interrupt
  .command(ProvidersCommand)   // opencode provider
  .command(ModelsCommand)      // opencode models
  .command(McpCommand)         // opencode mcp
  .command(PluginCommand)      // opencode plugin install/remove/list
  .command(GithubCommand)      // opencode github
  .command(PrCommand)          // opencode pr
  .command(ExportCommand)      // opencode export
  .command(ImportCommand)      // opencode import
  .command(AttachCommand)      // opencode attach <url>
  .command(ConsoleCommand)     // opencode account
  .command(UpgradeCommand)     // opencode upgrade
  .command(DbCommand)          // opencode db
```

#### Serve 命令实现

**文件：** `packages/opencode/src/cli/cmd/serve.ts`

```typescript
export const ServeCommand = effectCmd({
  command: "serve",
  describe: "starts a headless opencode server",
  instance: false,  // 服务器模式下不需要 InstanceContext
  handler: Effect.fn("Cli.serve")(function* (args) {
    const { Server } = yield* Effect.promise(() => import("../../server/server"))
    const opts = yield* resolveNetworkOptions(args)
    const server = yield* Effect.promise(() => Server.listen(opts))
    console.log(`opencode server listening on http://${server.hostname}:${server.port}`)
    yield* Effect.never  // 永远等待（直到 Ctrl+C）
  }),
})
```

#### TUI 启动流程

**文件：** `packages/opencode/src/cli/cmd/tui.ts`

TUI 使用 Worker 线程架构：
```
主进程（packages/opencode/src/cli/cmd/tui.ts）
  │  创建 Worker
  ▼
Worker 线程（packages/opencode/src/cli/tui/worker.ts）
  │  运行 opencode HTTP 服务器
  │  + 处理 TUI 渲染
  ▼
主进程通过 Rpc.client 与 Worker 通信
  - client.call("fetch", ...) → 转发 HTTP 请求给 Worker
  - client.on("global.event", ...) → 接收 SSE 事件
```

---

### 5.5 packages/tui — 终端 UI

**根路径：** `f:\code\opencode\packages\tui\src\`

**文件：** `packages/tui/src/app.tsx`（约 1135 行，主应用）

#### Context Provider 层次（完整）

```
<ArgsProvider>                          ← CLI 参数
  <TuiPathsProvider>                    ← 文件路径配置
    <TuiStartupProvider>                ← 启动状态
      <TuiTerminalEnvironmentProvider>  ← 终端环境（Win32 兼容）
        <SDKProvider>                   ← OpenCode 客户端 SDK
          <LocalProvider>               ← 本地状态
            <ProjectProvider>           ← 项目信息
              <ThemeProvider>           ← 主题
                <KVProvider>            ← 本地键值存储
                  <PluginRuntimeProvider> ← 插件运行时
                    <OpencodeKeymapProvider> ← 键盘绑定
                      <DialogProvider>  ← 对话框
                      <ToastProvider>   ← 通知
                      <Routes>          ← 路由
                        <Home />        ← 首页（Session 列表）
                        <Session />     ← Session 视图（主界面）
```

#### 全局键位绑定

```typescript
// packages/tui/src/app.tsx 第 92-100 行
const appGlobalBindingCommands = [
  "session.list",          // 打开 Session 列表对话框
  "session.new",           // 新建 Session
  "session.quick_switch.1", // 快速切换到 Session 1-6
  // ...
]
```

---

### 5.6 packages/app — Web UI

**根路径：** `f:\code\opencode\packages\app\src\`

**文件：** `packages/app/src/app.tsx`（主应用，Router + Providers）

#### 路由定义

```typescript
// 三个主页面
<Route path="/" component={...} />           // 首页
<Route path="/session/:id" component={...} /> // Session 视图
<Route path="/session/new" component={...} /> // 新建 Session
```

---

### 5.7 packages/plugin — 插件 SDK

**根路径：** `f:\code\opencode\packages\plugin\src\`

**文件：** `packages/plugin/src/index.ts`（约 336 行，完整 Hooks 接口）

#### 插件类型定义

```typescript
// 插件函数签名
export type Plugin = (input: PluginInput, options?: PluginOptions) => Promise<Hooks>

// 插件输入（可访问的能力）
export type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>  // OpenCode SDK 客户端
  project: Project                                  // 当前项目信息
  directory: string                                 // 工作目录
  worktree: string                                  // 项目根目录
  experimental_workspace: {
    register(type: string, adapter: WorkspaceAdapter): void
  }
  serverUrl: URL
  $: BunShell                                       // Bun Shell 接口
}

// 完整 Hooks 接口（所有可扩展点）
export interface Hooks {
  dispose?: () => Promise<void>                     // 插件卸载时清理
  event?: (input: { event: Event }) => Promise<void> // 监听全局事件
  config?: (input: Config) => Promise<void>          // 修改 SDK 配置
  tool?: { [key: string]: ToolDefinition }           // 注册自定义工具
  auth?: AuthHook                                    // 添加认证提供者
  provider?: ProviderHook                            // 添加/修改 Provider
  "chat.message"?: (input, output) => Promise<void>  // 消息钩子
  "chat.params"?: (input, output) => Promise<void>   // 修改 LLM 参数
  "chat.headers"?: (input, output) => Promise<void>  // 修改请求头
  "permission.ask"?: (input, output) => Promise<void>// 拦截权限请求
  "tool.execute.before"?: (input, output) => Promise<void> // 工具执行前
  "tool.execute.after"?: (input, output) => Promise<void>  // 工具执行后
  "tool.definition"?: (input, output) => Promise<void>     // 修改工具定义
  "shell.env"?: (input, output) => Promise<void>     // 修改 shell 环境变量
  "experimental.session.compacting"?: (input, output) => Promise<void>  // 自定义压缩提示
  "experimental.chat.system.transform"?: (input, output) => Promise<void>
}
```

---

### 5.8 packages/client — 生成客户端

**根路径：** `f:\code\opencode\packages\client\src\`

从 Server HttpApi 自动生成，提供两种风格：

```typescript
// Promise 风格（零 Effect 依赖，适合普通 JS/TS 项目）
import { createOpencodeClient } from "@opencode-ai/client"
const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })
const { data: sessions } = await client.sessions.list()

// Effect 风格（完整类型安全，适合 Effect-TS 项目）
import { client } from "@opencode-ai/client/effect"
const sessions = yield* client.sessions.list()
```

**生成命令：**
```bash
# 修改 packages/protocol 或 packages/server 的 HttpApi 后，重新生成
bun run generate  # 在 packages/client 目录下执行
```

---

### 5.9 packages/sdk-next — 嵌入式 SDK

进程内嵌入，不开 TCP 监听，通过内存传输调用同一套 HttpRouter：

```typescript
import { OpenCode } from "@opencode-ai/sdk-next"

// 创建嵌入式实例（带 Scope，关闭时自动清理所有资源）
const opencode = await OpenCode.create({
  directory: "/path/to/project",
  // 可选配置
})

const session = await opencode.sessions.create({})
await opencode.sessions.prompt({
  sessionID: session.data.id,
  prompt: { text: "帮我重构这段代码" },
})

// 订阅事件
for await (const event of (await opencode.events.subscribe()).stream) {
  if (event.type === "session.status" && event.properties.status.type === "idle") break
  if (event.type === "message.part.updated") {
    const part = event.properties.part
    if (part.type === "text" && part.time?.end) console.log(part.text)
  }
}

await opencode.close()  // 释放所有资源
```

---

## 6. 数据库结构

**文件：** `packages/core/src/database/schema.gen.ts`

SQLite 数据库路径：`~/.local/share/opencode/opencode.db`

#### 核心表

```sql
-- 工作区（工作台）
CREATE TABLE workspace (
  id text PRIMARY KEY,
  type text NOT NULL,         -- "local" | "remote" | ...
  name text NOT NULL,
  branch text,
  directory text,
  project_id text NOT NULL
);

-- 项目（VCS 根目录）
CREATE TABLE project (
  id text PRIMARY KEY,
  worktree text NOT NULL,     -- 项目根目录（git worktree）
  vcs text,                   -- "git" | null
  name text,
  time_created integer NOT NULL,
  time_updated integer NOT NULL
);

-- Session（对话会话）
CREATE TABLE session (
  id text PRIMARY KEY,
  project_id text NOT NULL,
  workspace_id text,
  parent_id text,             -- 用于 fork
  slug text NOT NULL,
  directory text NOT NULL,    -- 会话工作目录
  title text NOT NULL,
  version text NOT NULL,      -- opencode 版本
  cost real DEFAULT 0,        -- 累计花费（$）
  tokens_input integer DEFAULT 0,
  tokens_output integer DEFAULT 0
);

-- Session 消息（对话消息）
CREATE TABLE session_message (
  id text PRIMARY KEY,
  session_id text NOT NULL,
  type text NOT NULL,         -- 消息类型
  seq integer NOT NULL,       -- 有序序列号（事件溯源）
  data text NOT NULL          -- JSON 序列化消息内容
);

-- Session 输入 Inbox（Prompt 持久化接收）
CREATE TABLE session_input (
  id text PRIMARY KEY,
  session_id text NOT NULL,
  prompt text NOT NULL,
  delivery text NOT NULL,     -- "steer" | "queue"
  admitted_seq integer NOT NULL, -- 接收时的事件序列号
  promoted_seq integer,          -- 提升为可见消息时的序列号（null=待处理）
  time_created integer NOT NULL
);

-- Context Epoch（上下文纪元，用于 provider cache）
CREATE TABLE session_context_epoch (
  session_id text PRIMARY KEY,
  baseline text NOT NULL,     -- 完整 Baseline System Context 文本
  snapshot text NOT NULL,     -- JSON 快照（用于增量比较）
  baseline_seq integer NOT NULL  -- 建立此 Epoch 时的事件序列号
);

-- 事件序列（聚合根序列号追踪）
CREATE TABLE event_sequence (
  aggregate_id text PRIMARY KEY, -- Session ID 等
  seq integer NOT NULL,
  owner_id text                  -- 进程 ID（用于分布式锁）
);

-- 持久化事件（事件溯源存储）
CREATE TABLE event (
  id text PRIMARY KEY,
  aggregate_id text NOT NULL,
  seq integer NOT NULL,
  type text NOT NULL,            -- 事件类型
  data text NOT NULL             -- JSON 序列化事件数据
);

-- 已保存权限规则（"始终允许"持久化）
CREATE TABLE permission (
  id text PRIMARY KEY,
  project_id text NOT NULL,
  action text NOT NULL,          -- 工具名称
  resource text NOT NULL,        -- 匹配资源（支持通配符）
  time_created integer NOT NULL
);

-- 凭证存储（API Key 等）
CREATE TABLE credential (
  id text PRIMARY KEY,
  integration_id text,
  label text NOT NULL,
  value text NOT NULL,           -- 加密存储的凭证值
  active integer
);
```

---

## 7. Session 生命周期与数据流

### 用户发送 Prompt 的完整链路

```
用户 → sessions.prompt({sessionID, prompt, delivery?})
         │
         ├─ [1] 验证 Session 存在
         │   位置：packages/core/src/session.ts SessionV2.prompt
         │
         ├─ [2] 持久化接收（Durable Admission）
         │   写入 session_input 表（id、prompt、delivery、admitted_seq）
         │   发布 PromptAdmitted durable event
         │   位置：packages/core/src/session/input.ts SessionInput.admit
         │
         └─ [3] 触发执行（Advisory Wake）
             调用 SessionExecution.wake(sessionID)
             位置：packages/core/src/session/execution.ts
                    ↓
             SessionRunCoordinator（进程全局，序列化同一 Session 的执行）
                    ↓
             通过 LocationServiceMap.get(session.location) 获取 Location 服务
                    ↓
             SessionRunner.run({ sessionID, force: false })
             位置：packages/core/src/session/runner/index.ts

── Session Runner 内部流程 ──────────────────────────────────────────

[4] Context Epoch 初始化（首次 Provider Turn 前）
    │  SystemContextRegistry.load()（并发观察所有注册源）
    │  位置：packages/core/src/system-context/registry.ts
    │
    ├─ 有不可用源 → 阻塞，Prompt 保持 pending 可重试
    └─ 全部可用 → SystemContext.initialize(ctx) 生成 Baseline
        写入 session_context_epoch 表

[5] Prompt Promotion（安全边界）
    │  从 session_input 中取出 pending 的 Prompt
    │  steer: 立即提升为可见 session_message
    └─ queue: 等当前 drain 结束后才提升

[6] Context 协调（Safe Provider-Turn Boundary）
    │  重新观察所有 Context Sources
    │  与 session_context_epoch.snapshot 比较
    └─ 有变化 → 生成 Mid-Conversation System Message（持久化）

[7] 组装 LLMRequest
    │  SystemContext Baseline（作为系统提示）
    │  历史消息（session_message 表）
    │  工具定义（经权限过滤的 ToolRegistry.materialize()）
    └─ GenerationOptions（来自 Catalog 模型参数）

[8] llm.stream(request)（一次 Provider Turn）
    位置：packages/llm/src/llm.ts
    │
    ├─ 流式处理助手文本
    ├─ 流式处理工具调用
    └─ 工具执行（eager，立即执行，不等 provider-stream 结束）：
        ├─ 权限检查 PermissionV2.assert()
        ├─ 执行 tool.execute(input, context)
        ├─ ToolOutputStore.bound()（输出大小限制）
        └─ 持久化结果（durable settlement）

[9] 历史重新加载 + 决定是否继续
    ├─ 还有工具调用 → 回到 [7]
    ├─ 还有 steer 输入 → 回到 [5]
    └─ 空闲 → Session Drain 结束
```

### 事件类型（持久化 vs Live-only）

**持久化事件（Durable）：** 写入 event 表，可重放，跨进程恢复

| 事件类型 | 说明 |
|---------|------|
| `session.next.created.1` | Session 创建 |
| `session.next.prompted.1` | Prompt 被提升为可见消息 |
| `session.next.input.admitted.1` | Prompt 被接收到 Inbox |
| `session.next.assistant.started.1` | 助手开始回复 |
| `session.next.tool.call.1` | 工具调用 |
| `session.next.tool.result.1` | 工具结果 |
| `session.next.agent.switched.1` | Agent 切换 |
| `session.next.model.switched.1` | 模型切换 |
| `session.next.compaction.ended.1` | 压缩完成 |

**Live-only 事件：** 只用于实时渲染，不持久化

| 事件类型 | 说明 |
|---------|------|
| `session.next.assistant.text.delta.1` | 文本增量 |
| `session.next.reasoning.delta.1` | 思考过程增量 |
| `session.next.compaction.started.1` | 压缩开始（进度） |

---

## 8. System Context 系统

System Context 是呈现给模型的"特权信息"，独立于对话历史，每次 Provider Turn 前重新评估。

**关键文件：**
- 代数定义：`packages/core/src/system-context/index.ts`
- 注册表：`packages/core/src/system-context/registry.ts`
- 内置源：`packages/core/src/system-context/builtins.ts`

### Source 接口（完整定义）

```typescript
// packages/core/src/system-context/index.ts 第 32-39 行
interface Source<A> {
  readonly key: Key         // 命名空间化 key，格式：`namespace/name`
                            // 例："core/environment"、"core/date"
  readonly codec: Schema.Codec<A, Schema.Json, never, never>
                            // JSON 编解码（用于快照比较）
  readonly load: Effect.Effect<A | Unavailable>
                            // 加载当前值（返回 unavailable = 临时不可用）
  readonly baseline: (current: A) => string
                            // 首次渲染（非空字符串，否则抛出）
  readonly update: (previous: A, current: A) => string
                            // 值变化时渲染
  readonly removed?: (previous: A) => string
                            // 源被移除时渲染（动态源必须提供）
}
```

### 内置 Context Sources

**文件：** `packages/core/src/system-context/builtins.ts`

```typescript
// 实际源码示例
const context = SystemContext.combine([
  // 源 1：环境信息（工作目录、平台等）
  SystemContext.make({
    key: SystemContext.Key.make("core/environment"),
    codec: Schema.toCodecJson(Schema.String),
    load: Effect.succeed(environment),  // 从 location.directory 等构建
    baseline: (env) => `Here is some useful information about the environment:\n${env}`,
    update: (_prev, env) => `The environment you are running in is now:\n${env}`,
  }),
  
  // 源 2：日期
  SystemContext.make({
    key: SystemContext.Key.make("core/date"),
    codec: Schema.toCodecJson(Schema.String),
    load: DateTime.nowAsDate.pipe(Effect.map(date => date.toDateString())),
    baseline: (date) => `Today's date: ${date}`,
    update: (_prev, date) => `Today's date is now: ${date}`,
  }),
])
```

### 协调流程

```typescript
// reconcile 返回以下四种结果之一
type ReconcileResult =
  | { _tag: "Unchanged" }          // 无变化
  | { _tag: "Updated"; text: string; snapshot: Snapshot }  // 有更新，含新文本
  | { _tag: "ReplacementReady"; generation: Generation }   // 需重建 Baseline
  | { _tag: "ReplacementBlocked" }  // 重建被阻塞（有不可用源）
```

---

## 9. 工具系统（Tool System）

**关键文件：**
- 工具定义：`packages/core/src/tool/tool.ts`
- 工具注册：`packages/core/src/tool/tools.ts`
- 工具注册表：`packages/core/src/tool/registry.ts`
- 进程级工具池：`packages/core/src/tool/application-tools.ts`
- Bash 工具示例：`packages/core/src/tool/bash.ts`

### Tool.make 工厂函数（完整类型）

```typescript
// packages/core/src/tool/tool.ts 第 71-131 行
type Config<Input, Output, Structured = Output> = {
  readonly description: string       // 工具描述（展示给 LLM）
  readonly input: Input              // 输入 Schema（Effect Schema）
  readonly output: Output            // 完整输出 Schema
  readonly structured?: Structured   // 精简的结构化输出 Schema（可选）
  readonly toStructuredOutput?: (input: { input, output }) => Structured  // 转换逻辑
  readonly execute: (
    input: Schema.Type<Input>,
    context: Tool.Context,           // { sessionID, agent, assistantMessageID, toolCallID }
  ) => Effect.Effect<Schema.Type<Output>, ToolFailure>
  readonly toModelOutput?: (input: { input, output }) => ReadonlyArray<Content>
  // Content = { type: "text", text: string } | { type: "file", data: string, mime: string }
}

// 权限装饰
const readTool = Tool.withPermission(baseTool, "read")
// → 此工具在执行时会触发 PermissionV2.assert({ action: "read", ... })
```

### 工具执行上下文

```typescript
// packages/core/src/tool/tool.ts 第 9-14 行
export interface Context {
  readonly sessionID: SessionSchema.ID
  readonly agent: AgentV2.ID
  readonly assistantMessageID: SessionMessage.ID
  readonly toolCallID: string
}
```

### 工具注册层次

```
ApplicationTools（进程全局，packages/core/src/tool/application-tools.ts）
    ├─ 注册一次，所有 Location 可见
    └─ 存储 Map<string, { identity: object; tool: AnyTool }>

ToolRegistry（Location-scoped，packages/core/src/tool/registry.ts）
    ├─ 继承 ApplicationTools 的所有工具
    ├─ 叠加 Location 级动态注册（MCP 工具、插件工具）
    └─ materialize(permissions) → { definitions, settle }
       过滤被权限完全禁用的工具，返回模型可用的工具列表
```

### Bash 工具完整实现分析

**文件：** `packages/core/src/tool/bash.ts`（208 行）

```typescript
// Layer.effectDiscard 在 Layer 初始化时注册工具
const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const tools = yield* Tools.Service      // 获取工具注册服务
    const permission = yield* PermissionV2.Service

    yield* tools.register({
      [name]: Tool.make({
        description: "Execute one shell command...",
        input: Input,     // { command, workdir?, timeout? }
        output: Output,   // { output, exit?, truncated, timeout? }
        structured: StructuredOutput,  // 只含 { exit?, truncated, timeout? }
        
        // 精简结构化输出（LLM 不需要看全部 output 文本）
        toStructuredOutput: ({ output }) => ({
          truncated: output.truncated,
          ...(output.exit !== undefined ? { exit: output.exit } : {}),
        }),
        
        // 模型输出：命令原始输出 + 退出信息
        toModelOutput: ({ output }) => [
          { type: "text", text: output.output },
          { type: "text", text: `Command exited with code ${output.exit}.` },
        ],
        
        execute: (input, context) => Effect.gen(function* () {
          // 1. 解析工作目录（检测外部目录）
          const target = yield* mutation.resolve({ path: input.workdir ?? ".", kind: "directory" })
          
          // 2. 如果是外部目录，请求 external_directory 权限
          if (target.externalDirectory)
            yield* permission.assert({ action: "external_directory", ... })
          
          // 3. 请求命令执行权限（支持"始终允许"持久化）
          yield* permission.assert({
            action: name,           // "bash"
            resources: [input.command],
            save: [input.command],  // 支持"始终允许此命令"
            sessionID: context.sessionID,
            agent: context.agent,
            source: { type: "tool", messageID: ..., callID: ... },
          })
          
          // 4. 执行命令
          const result = yield* appProcess.run(command, { ... })
          return { exit: result.exitCode, output: result.output, truncated: ... }
        }).pipe(
          Effect.mapError(() => new ToolFailure({ message: `Unable to execute command: ${input.command}` }))
        ),
      }),
    }).pipe(Effect.orDie)
  }),
)
```

---

## 10. 权限系统

**文件：** `packages/core/src/permission.ts`（311 行）

### 权限评估逻辑

```typescript
// packages/core/src/permission.ts 第 76-86 行
export function evaluate(action: string, resource: string, ...rulesets: Permission.Ruleset[]): Permission.Rule {
  return (
    rulesets
      .flat()
      // findLast：最后匹配的规则生效（规则列表越靠后优先级越高）
      .findLast((rule) =>
        Wildcard.match(action, rule.action) &&
        Wildcard.match(resource, rule.resource)
      ) ?? {
      action,
      resource: "*",
      effect: "ask",  // 默认：询问用户
    }
  )
}
```

### 权限请求生命周期

```
工具调用 → permission.assert(input)
    │
    ├─ 评估已配置规则（agent.permissions）
    ├─ 合并已保存的 "always" 规则（permission 表）
    │
    ├─ effect = "deny"  → BlockedError（工具调用失败）
    ├─ effect = "allow" → 立即执行（不中断）
    └─ effect = "ask"   → 创建 pending 请求，发布 Event.Asked
                          Deferred 等待用户回复
                            │
                            用户通过 PermissionV2.reply()
                            ├─ "once"   → 本次允许
                            ├─ "always" → 本次允许 + 写入 permission 表永久保存
                            │             同时自动批准其他等待相同规则的请求
                            └─ "reject" → 拒绝（+ 拒绝所有同 Session 的待处理请求）
```

---

## 11. LLM Provider 集成架构

**关键文件：**
- 主入口：`packages/llm/src/llm.ts`
- OpenAI Provider：`packages/llm/src/providers/openai.ts`
- Route 接口：`packages/llm/src/route/client.ts`
- 协议适配器：`packages/llm/src/protocols/`

### 模型解析流程

**文件：** `packages/core/src/session/runner/model.ts`

```typescript
// 支持三种 API 类型（来自 catalog）
export const fromCatalogModel = (model: ModelV2.Info, credential?: Credential.Value) => {
  // 1. @ai-sdk/openai → 使用 OpenAI Responses 路由
  if (resolved.api.type === "aisdk" && resolved.api.package === "@ai-sdk/openai")
    return Effect.succeed(
      withDefaults(resolved, OpenAIResponses.route)
        .with({ auth: Auth.bearer(key) })
        .model({ id: resolved.api.id })
    )
  
  // 2. @ai-sdk/anthropic → 使用 Anthropic Messages 路由
  if (resolved.api.type === "aisdk" && resolved.api.package === "@ai-sdk/anthropic")
    return Effect.succeed(
      withDefaults(resolved, AnthropicMessages.route)
        .with({ auth: Auth.header("x-api-key", key) })
        .model({ id: resolved.api.id })
    )
  
  // 3. @ai-sdk/openai-compatible + url → 使用 OpenAI Compatible Chat 路由
  if (resolved.api.type === "aisdk" && resolved.api.package === "@ai-sdk/openai-compatible" && url)
    return Effect.succeed(
      withDefaults(resolved, OpenAICompatibleChat.route)
        .with({ auth: Auth.bearer(key) })
        .model({ id: resolved.api.id })
    )
  
  return Effect.fail(new UnsupportedApiError(...))
}
```

---

## 12. 配置系统

**文件：** `packages/core/src/config.ts`

### 配置发现顺序（优先级从低到高）

```
1. 全局配置：~/.local/share/opencode/opencode.jsonc
2. 项目祖先目录的 .opencode/opencode.jsonc（从外到内）
3. 项目中直接的 opencode.json / opencode.jsonc
4. 更深层 .opencode 目录（越近越高优先级）
```

### 完整配置字段参考

```jsonc
// ~/.opencode/opencode.jsonc 或 项目根 opencode.jsonc
{
  "$schema": "https://opencode.ai/config.json",
  
  // 基础
  "shell": "/bin/zsh",
  "model": "anthropic/claude-opus-4-5",
  "default_agent": "build",
  "autoupdate": true,         // true | false | "notify"
  "share": "manual",          // "manual" | "auto" | "disabled"
  "username": "developer",
  "snapshots": true,          // 启用文件系统快照（undo/revert）
  
  // Agent 覆盖/自定义
  "agents": {
    "build": {                // 覆盖内置 build agent
      "model": "openai/gpt-4o",
      "prompt": "You are an expert TypeScript developer.",
      "permissions": [
        { "effect": "allow", "action": "bash", "resource": "npm test" },
        { "effect": "deny", "action": "bash", "resource": "rm -rf *" }
      ]
    },
    "reviewer": {             // 自定义 Agent
      "model": "anthropic/claude-3-5-haiku-20241022",
      "description": "代码审查专家",
      "prompt": "You are a strict code reviewer focusing on security.",
      "permissions": [
        { "effect": "deny", "action": "write", "resource": "*" },
        { "effect": "deny", "action": "edit", "resource": "*" },
        { "effect": "allow", "action": "*", "resource": "*" }
      ]
    }
  },
  
  // 全局权限规则（作用于所有 Agent）
  "permissions": [
    { "effect": "allow", "action": "bash", "resource": "git *" },
    { "effect": "ask",   "action": "bash", "resource": "*" }
  ],
  
  // Provider 配置
  "providers": {
    "openai": {
      "apiKey": "${OPENAI_API_KEY}",
      "models": {
        "o3-mini": { "enabled": true }
      }
    },
    "local-llm": {            // 自定义 OpenAI 兼容 Provider
      "name": "Local LLM",
      "env": ["LOCAL_LLM_API_KEY"],
      "options": {
        "baseURL": "http://localhost:11434/v1",
        "headers": { "Authorization": "Bearer ${LOCAL_LLM_API_KEY}" }
      },
      "models": {
        "llama3.2": {
          "name": "Llama 3.2",
          "api": { "id": "llama3.2" }
        }
      }
    }
  },
  
  // MCP 服务器
  "mcp": {
    "filesystem": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    },
    "database": {
      "type": "local",
      "command": "my-db-mcp-server",
      "env": { "DB_URL": "postgresql://..." }
    }
  },
  
  // 工具输出限制
  "tool_output": {
    "max_lines": 2000,
    "max_bytes": 51200
  },
  
  // 压缩配置
  "compaction": {
    "keep_tokens": 8000,   // 压缩后保留最近多少 token 的历史
    "buffer": 2000          // 预留输出 headroom（token）
  },
  
  // 额外指令来源（自动加载到系统提示）
  "instructions": [
    "./CONTRIBUTING.md",
    ".cursor/rules/*.md",
    "https://example.com/coding-standards.md"
  ],
  
  // 外部参考（可通过 @alias 引用）
  "references": {
    "effect": {
      "repository": "github.com/Effect-TS/effect",
      "description": "Effect-TS 文档"
    },
    "ui-lib": {
      "path": "../shared-ui",
      "description": "共享 UI 组件库"
    }
  },
  
  // Skill 发现路径
  "skills": ["./opencode-skills"],
  
  // 格式化器
  "formatter": {
    "prettier": { "disabled": false },
    "my-formatter": {
      "command": ["./scripts/format", "$FILE"],
      "extensions": [".myext"]
    }
  },
  
  // LSP 配置
  "lsp": {
    "typescript": { "disabled": false },
    "my-lsp": {
      "command": ["my-language-server", "--stdio"],
      "extensions": [".myext"]
    }
  },
  
  // 插件
  "plugins": [
    "@company/opencode-audit-plugin",
    {
      "package": "@company/opencode-telemetry",
      "options": { "endpoint": "https://telemetry.example.com" }
    }
  ],
  
  // 文件监视忽略
  "watcher": {
    "ignore": ["node_modules/**", ".git/**", "dist/**", "*.log"]
  }
}
```

---

## 13. 事件系统

**文件：** `packages/core/src/event.ts`（639 行）

### 事件服务接口

```typescript
// packages/core/src/event.ts 第 126-148 行
export interface Interface {
  // 发布事件（持久化 + 通知订阅者）
  publish<D extends Definition>(
    definition: D,
    data: Data<D>,
    options?: PublishOptions
  ): Effect.Effect<Payload<D>>
  
  // 订阅特定类型事件（实时流）
  subscribe<D extends Definition>(definition: D): Stream.Stream<Payload<D>>
  
  // 订阅所有事件（实时流）
  all(): Stream.Stream<Payload>
  
  // 聚合根的 durable 事件流（历史重播 + 实时追踪）
  durable(input: { aggregateID: string; after?: number }): Stream.Stream<Payload>
  
  // 注册投影处理器（每个新事件触发）
  project<D extends Definition>(definition: D, projector: Subscriber<D>): Effect.Effect<void>
  
  // 从序列化事件重放（用于数据迁移/集群同步）
  replay(event: SerializedEvent, options?): Effect.Effect<void>
}
```

---

## 14. 自动压缩（Compaction）

### 触发条件

```
每次 Provider Turn 前计算：
  estimatedTokens = 历史消息 token 数 + System Context token 数
  reserved = max(model.outputLimit, config.compaction.buffer)
  
  if estimatedTokens > model.contextWindow - reserved:
    → 触发压缩（需要有至少一个完整的历史 turn 可以压缩）
  
  if provider 返回 context overflow 错误 AND 没有已持久化输出:
    → 触发溢出压缩（尝试一次，失败则返回给模型报错）
```

### 压缩事件流

```
session.next.compaction.started.1  ← 持久化，标识本次压缩尝试

    LLM 调用生成 rolling summary...（live-only 进度事件）

session.next.compaction.ended.1    ← 持久化，包含：
  { summary: "...", recent: "..." }

下次 Provider Turn：
  检测到 compaction.ended → 渲染全新 Context Epoch Baseline
  [新 Baseline System Context]
  [Rolling Summary]
  [最近 N token 的完整历史]
  [新的消息...]
```

---

## 15. 二次开发指南（二开示例）

### 15.1 开发自定义工具

**相关文件：**
- 工具定义：`packages/core/src/tool/tool.ts`（Tool.make）
- 参考示例：`packages/core/src/tool/bash.ts`（完整实现）
- 注册接口：`packages/core/src/tool/tools.ts`（Tools.Service）

#### 方式一：通过插件系统注册工具（推荐，无需修改源码）

```typescript
// 文件：my-plugin/src/index.ts
import { tool } from "@opencode-ai/plugin"
import { z } from "zod"

// 旧版插件工具 API（使用 Zod Schema）
// 文件参考：packages/plugin/src/tool.ts
export function myPluginFn(input, options) {
  return {
    // 工具定义
    tool: {
      "database-query": tool({
        description: "Execute a SQL query against the project database",
        args: {
          query: z.string().describe("SQL query to execute"),
          limit: z.number().optional().default(100).describe("Maximum rows to return"),
        },
        async execute({ query, limit }, context) {
          // context: { sessionID, messageID, agent, directory, worktree, abort, metadata, ask }
          
          // 可以使用 context.ask 请求权限
          await context.ask({
            permission: "database",
            patterns: [query],
            always: [],
            metadata: { query },
          })
          
          // 实际执行查询
          const results = await executeQuery(query, limit)
          return {
            title: `Query: ${query.slice(0, 50)}`,
            output: JSON.stringify(results, null, 2),
            metadata: { rowCount: results.length },
          }
        },
      }),
    },
  }
}
```

#### 方式二：在 Core 中直接添加工具（需修改源码，适合贡献到项目）

**参考文件：** `packages/core/src/tool/bash.ts` 的完整结构

```typescript
// 新建文件：packages/core/src/tool/my-tool.ts
export * as MyTool from "./my-tool"

import { ToolFailure } from "@opencode-ai/llm"
import { Effect, Layer, Schema } from "effect"
import { makeLocationNode } from "../effect/app-node"
import { PermissionV2 } from "../permission"
import { ToolRegistry } from "./registry"
import { Tool } from "./tool"
import { Tools } from "./tools"

export const name = "my_tool"

// 1. 定义输入 Schema（Effect Schema，自动验证）
const Input = Schema.Struct({
  action: Schema.String.annotate({
    description: "The action to perform",
  }),
  target: Schema.String.pipe(Schema.optional).annotate({
    description: "Target of the action (optional)",
  }),
})

// 2. 定义完整输出 Schema
const Output = Schema.Struct({
  success: Schema.Boolean,
  result: Schema.String,
  details: Schema.String.pipe(Schema.optional),
})

// 3. 精简结构化输出（只暴露关键元数据给模型，减少 token）
const StructuredOutput = Schema.Struct({
  success: Schema.Boolean,
})

// 4. 创建注册 Layer
const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const tools = yield* Tools.Service
    const permission = yield* PermissionV2.Service

    yield* tools.register({
      [name]: Tool.make({
        description: "Perform a custom action with permission check",
        input: Input,
        output: Output,
        structured: StructuredOutput,
        
        // 精简输出（给 LLM 看的结构化摘要）
        toStructuredOutput: ({ output }) => ({
          success: output.success,
        }),
        
        // 模型输出（展示给 LLM 的文本）
        toModelOutput: ({ output }) => [
          { type: "text", text: output.result },
          ...(output.details ? [{ type: "text", text: output.details }] : []),
        ],
        
        execute: (input, context) =>
          Effect.gen(function* () {
            // 权限检查（会触发用户确认对话框）
            yield* permission.assert({
              action: name,
              resources: [input.action],
              save: [input.action],     // 支持"始终允许"
              sessionID: context.sessionID,
              agent: context.agent,
              source: {
                type: "tool",
                messageID: context.assistantMessageID,
                callID: context.toolCallID,
              },
            })
            
            // 实际业务逻辑
            const result = yield* Effect.tryPromise({
              try: () => doSomething(input.action, input.target),
              catch: (error) => new ToolFailure({ message: String(error) }),
            })
            
            return {
              success: true,
              result: `Action '${input.action}' completed successfully`,
              details: JSON.stringify(result),
            }
          }).pipe(
            Effect.mapError((error) => new ToolFailure({ message: `Failed: ${error}` })),
          ),
      }),
    }).pipe(Effect.orDie)
  }),
)

// 5. 导出 node（用于依赖注入）
export const node = makeLocationNode({
  name: "tool/my-tool",
  layer,
  deps: [ToolRegistry.node, PermissionV2.node],  // 声明依赖
})
```

然后在工具汇总文件中注册：

```typescript
// 修改：packages/core/src/tool/builtins.ts
// 添加：
import { MyTool } from "./my-tool"
// ...在 builtins 数组中添加 MyTool.node
```

---

### 15.2 注入自定义 System Context

**相关文件：**
- 内置示例：`packages/core/src/system-context/builtins.ts`
- 代数定义：`packages/core/src/system-context/index.ts`
- 注册表：`packages/core/src/system-context/registry.ts`

```typescript
// 新建文件：packages/core/src/system-context/my-context.ts
import { DateTime, Effect, Layer, Schema } from "effect"
import { makeLocationNode } from "../effect/app-node"
import { SystemContext } from "./index"
import { SystemContextRegistry } from "./registry"
import { Location } from "../location"

// 1. 定义数据类型
type GitStatus = {
  branch: string
  dirty: boolean
  uncommittedFiles: string[]
}

// 2. 创建注册 Layer
const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const location = yield* Location.Service
    const registry = yield* SystemContextRegistry.Service
    
    // 3. 创建 SystemContext 源
    const gitStatusContext = SystemContext.make<GitStatus>({
      // key 格式：lowercase namespace/name，支持 [a-z0-9._-/]
      key: SystemContext.Key.make("my-extension/git-status"),
      
      // JSON 编解码（用于快照比较，检测变化）
      codec: Schema.toCodecJson(
        Schema.Struct({
          branch: Schema.String,
          dirty: Schema.Boolean,
          uncommittedFiles: Schema.Array(Schema.String),
        })
      ),
      
      // 加载当前值（每次 Provider Turn 前调用）
      load: Effect.tryPromise({
        try: async () => {
          const { execSync } = await import("child_process")
          const branch = execSync("git rev-parse --abbrev-ref HEAD", {
            cwd: location.directory,
          }).toString().trim()
          
          const statusOutput = execSync("git status --porcelain", {
            cwd: location.directory,
          }).toString()
          
          const uncommittedFiles = statusOutput
            .split("\n")
            .filter(line => line.trim())
            .map(line => line.slice(3))
          
          return { branch, dirty: uncommittedFiles.length > 0, uncommittedFiles }
        },
        // 如果 git 不可用，返回 unavailable（不阻塞执行）
        catch: () => SystemContext.unavailable,
      }),
      
      // 首次注入到模型（当此 Source 首次被提交时）
      baseline: (status) => [
        `Git Status:`,
        `  Branch: ${status.branch}`,
        `  Dirty: ${status.dirty}`,
        status.uncommittedFiles.length > 0
          ? `  Uncommitted files:\n${status.uncommittedFiles.map(f => `    - ${f}`).join("\n")}`
          : "  No uncommitted changes",
      ].join("\n"),
      
      // 当值变化时注入（如分支切换或文件修改）
      update: (previous, current) => [
        `Git status has changed:`,
        `  Branch: ${previous.branch} → ${current.branch}`,
        `  Dirty: ${previous.dirty} → ${current.dirty}`,
        current.uncommittedFiles.length > 0
          ? `  Current uncommitted files:\n${current.uncommittedFiles.map(f => `    - ${f}`).join("\n")}`
          : "  No uncommitted changes now",
      ].join("\n"),
      
      // 当此 Source 被移除时注入（动态源必须提供）
      removed: () => "Git status information is no longer available.",
    })
    
    // 4. 注册到 SystemContextRegistry（Scope 关闭时自动注销）
    yield* registry.register({
      key: SystemContext.Key.make("my-extension/git-status"),
      load: Effect.succeed(gitStatusContext),
    })
  }),
)

export const node = makeLocationNode({
  name: "my-extension/git-status-context",
  layer,
  deps: [Location.node, SystemContextRegistry.node],
})
```

---

### 15.3 开发 Plugin

**相关文件：**
- 插件接口：`packages/plugin/src/index.ts`（完整 Hooks 定义）
- 旧版工具 API：`packages/plugin/src/tool.ts`
- Shell 接口：`packages/plugin/src/shell.ts`

#### 完整插件示例

```typescript
// 文件：my-opencode-plugin/src/index.ts
import type { Plugin, Hooks } from "@opencode-ai/plugin"
import { tool } from "@opencode-ai/plugin"
import { z } from "zod"

const myPlugin: Plugin = async (input, options) => {
  const { client, project, directory, $ } = input
  
  // 初始化插件资源（如数据库连接、API 客户端等）
  const apiClient = new MyApiClient(options?.apiKey as string)
  
  const hooks: Hooks = {
    // === 自定义工具 ===
    tool: {
      // 工具 1：调用外部 API
      "external-api-call": tool({
        description: "Make a call to the external API with a prompt",
        args: {
          endpoint: z.string().describe("API endpoint path"),
          payload: z.string().describe("JSON payload as a string"),
        },
        async execute({ endpoint, payload }, context) {
          // context.ask 请求权限
          await context.ask({
            permission: "external-api",
            patterns: [`${endpoint}:*`],
            always: [`${endpoint}:read`],
            metadata: { endpoint },
          })
          
          const result = await apiClient.call(endpoint, JSON.parse(payload))
          return {
            title: `API call to ${endpoint}`,
            output: JSON.stringify(result, null, 2),
            metadata: { status: result.status },
          }
        },
      }),
      
      // 工具 2：Shell 操作（使用 Bun Shell）
      "run-tests": tool({
        description: "Run the test suite for the current project",
        args: {
          pattern: z.string().optional().describe("Test pattern filter"),
        },
        async execute({ pattern }, context) {
          const cmd = pattern ? `npm test -- ${pattern}` : "npm test"
          
          await context.ask({
            permission: "bash",
            patterns: [cmd],
            always: [cmd],
            metadata: {},
          })
          
          const result = await $`npm test ${pattern ? ["--", pattern] : []}`.cwd(directory).quiet().nothrow()
          
          return {
            title: `Test Results`,
            output: result.stdout.toString(),
            metadata: {
              exitCode: result.exitCode,
              passed: result.exitCode === 0,
            },
          }
        },
      }),
    },
    
    // === 事件监听 ===
    event: async ({ event }) => {
      // 监听所有事件（用于日志、遥测等）
      if (event.type === "session.created") {
        console.log(`[Plugin] New session created: ${event.properties?.id}`)
        // 可以调用外部系统记录
      }
      if (event.type === "session.idle") {
        // Session 执行完成
        const sessionID = event.properties?.sessionID
        // 发送通知到 Slack/钉钉等
      }
    },
    
    // === 修改 LLM 参数 ===
    "chat.params": async (input, output) => {
      // input: { sessionID, agent, model, provider, message }
      // output: { temperature, topP, topK, maxOutputTokens, options }
      
      // 针对特定 Provider 调整参数
      if (input.provider.info.id === "anthropic") {
        output.temperature = 0.7  // 修改温度
      }
      
      // 添加自定义选项
      output.options["custom-header"] = "my-value"
    },
    
    // === 修改请求头 ===
    "chat.headers": async (input, output) => {
      // 添加自定义认证头
      output.headers["X-Request-ID"] = crypto.randomUUID()
      output.headers["X-Plugin-Version"] = "1.0.0"
    },
    
    // === 权限请求拦截 ===
    "permission.ask": async (permission, output) => {
      // permission: { id, sessionID, action, resources, ... }
      // 可以自动批准某些操作
      if (permission.action === "read" && permission.resources.every(r => r.endsWith(".md"))) {
        output.status = "allow"  // 自动允许读取 Markdown 文件
      }
    },
    
    // === 工具执行前后钩子 ===
    "tool.execute.before": async (input, output) => {
      // input: { tool: string, sessionID: string, callID: string }
      // output: { args: any } - 可以修改工具参数！
      
      if (input.tool === "bash") {
        // 记录所有 bash 命令
        console.log(`[Audit] Bash: ${output.args.command}`)
      }
    },
    
    "tool.execute.after": async (input, output) => {
      // 工具执行后钩子，可以后处理输出
      if (input.tool === "bash" && !output.metadata?.timeout) {
        // 自定义输出后处理
      }
    },
    
    // === Shell 环境变量 ===
    "shell.env": async (input, output) => {
      // input: { cwd: string, sessionID?: string, callID?: string }
      // output: { env: Record<string, string> }
      output.env["MY_PLUGIN_VAR"] = "my-value"
      output.env["CUSTOM_PATH"] = `/opt/my-tools:${output.env.PATH ?? ""}`
    },
    
    // === 自定义压缩提示 ===
    "experimental.session.compacting": async (input, output) => {
      // 添加额外上下文到压缩提示
      output.context.push("Focus on preserving code structure decisions and architectural choices.")
    },
    
    // === 修改工具定义（改变模型看到的工具描述）===
    "tool.definition": async ({ toolID }, output) => {
      if (toolID === "bash") {
        output.description += "\n\nNote: All commands are audited for security compliance."
      }
    },
    
    // === 清理资源 ===
    dispose: async () => {
      await apiClient.close()
      console.log("[Plugin] Cleaned up resources")
    },
  }
  
  return hooks
}

export default myPlugin
// 或者命名导出：export const server = myPlugin
```

#### 插件配置（opencode.jsonc）

```jsonc
{
  "plugins": [
    // 方式1：直接包名
    "my-opencode-plugin",
    // 方式2：带选项
    {
      "package": "my-opencode-plugin",
      "options": {
        "apiKey": "${MY_API_KEY}",
        "endpoint": "https://api.example.com"
      }
    }
  ]
}
```

#### 插件 package.json

```json
{
  "name": "my-opencode-plugin",
  "main": "./src/index.js",
  "exports": {
    ".": "./src/index.js"
  }
}
```

---

### 15.4 接入自定义 LLM Provider

**相关文件：**
- OpenAI Provider 参考：`packages/llm/src/providers/openai.ts`
- Route 接口：`packages/llm/src/route/client.ts`

通过配置文件接入 OpenAI 兼容端点（无需修改源码）：

```jsonc
// opencode.jsonc
{
  "providers": {
    "my-local-llm": {
      "name": "My Local LLM (Ollama)",
      "env": ["OLLAMA_API_KEY"],
      "options": {
        "baseURL": "http://localhost:11434/v1",
        "headers": {}
      },
      "models": {
        "llama3.2": {
          "name": "Llama 3.2 3B",
          "api": {
            "type": "aisdk",
            "package": "@ai-sdk/openai-compatible",
            "id": "llama3.2",
            "url": "http://localhost:11434/v1"
          },
          "limit": {
            "context": 128000,
            "output": 8192
          },
          "capabilities": {
            "input": ["text"],
            "output": ["text"]
          }
        },
        "codellama": {
          "name": "Code Llama 13B",
          "api": {
            "type": "aisdk",
            "package": "@ai-sdk/openai-compatible",
            "id": "codellama",
            "url": "http://localhost:11434/v1"
          }
        }
      }
    },
    
    "azure-custom": {
      "name": "Azure OpenAI（自定义部署）",
      "env": ["AZURE_OPENAI_API_KEY"],
      "options": {
        "baseURL": "https://my-deployment.openai.azure.com/openai/deployments",
        "headers": {
          "api-key": "${AZURE_OPENAI_API_KEY}"
        }
      },
      "models": {
        "gpt-4o-deploy": {
          "name": "GPT-4o（Azure 部署）",
          "api": {
            "type": "aisdk",
            "package": "@ai-sdk/openai-compatible",
            "id": "gpt-4o",
            "url": "https://my-deployment.openai.azure.com/openai/deployments/gpt-4o/chat/completions?api-version=2024-02-01"
          }
        }
      }
    }
  }
}
```

如需完全自定义 Provider 行为（非 OpenAI 兼容），可以通过插件的 `provider` 钩子：

```typescript
// packages/plugin/src/index.ts 第 214-217 行
export type ProviderHook = {
  id: string                   // Provider ID
  models?: (provider: ProviderV2, ctx: ProviderHookContext) => Promise<Record<string, ModelV2>>
  // 动态返回模型列表（如从 API 获取最新模型列表）
}

// 插件中使用
const hooks: Hooks = {
  provider: {
    id: "my-provider",
    models: async (provider, ctx) => {
      // 动态从 API 获取最新模型列表
      const response = await fetch("https://api.my-provider.com/models", {
        headers: { Authorization: `Bearer ${ctx.auth?.access}` },
      })
      const { models } = await response.json()
      return Object.fromEntries(models.map(m => [m.id, {
        id: m.id,
        name: m.name,
        contextLength: m.context_length,
        // ...其他字段
      }]))
    },
  },
}
```

---

### 15.5 嵌入式 SDK 使用

**相关文件：** `packages/sdk-next/`

将 OpenCode 嵌入到你自己的 Bun/Node 应用中，完全进程内，无 HTTP 开销：

```typescript
// my-app/src/opencode-integration.ts
import { OpenCode } from "@opencode-ai/sdk-next"

// 场景1：单次任务自动化
async function analyzeCode(directory: string, query: string): Promise<string> {
  // 创建嵌入式实例（Scope 关闭时自动清理）
  const opencode = await OpenCode.create({
    directory,
    // 可以传入配置覆盖
    config: {
      model: "anthropic/claude-opus-4-5",
      permissions: [
        { effect: "allow", action: "*", resource: "*" }  // 全自动模式
      ],
    },
  })
  
  try {
    // 创建 Session
    const { data: session } = await opencode.sessions.create({})
    const sessionID = session.id
    
    // 订阅事件（必须在 prompt 前订阅，避免丢失事件）
    const { stream: events } = await opencode.events.subscribe()
    
    // 发送 Prompt
    await opencode.sessions.prompt({
      sessionID,
      prompt: { text: query },
    })
    
    // 收集响应
    let fullResponse = ""
    
    for await (const event of events) {
      if (event.type === "message.part.updated") {
        const part = event.properties.part
        if (part.sessionID !== sessionID) continue
        
        // 收集文本输出
        if (part.type === "text" && part.time?.end) {
          fullResponse += part.text
        }
      }
      
      // Session 空闲 = 执行完成
      if (
        event.type === "session.status" &&
        event.properties.sessionID === sessionID &&
        event.properties.status.type === "idle"
      ) break
    }
    
    return fullResponse
  } finally {
    await opencode.close()
  }
}

// 使用示例
const analysis = await analyzeCode("/path/to/project", "分析这个项目的架构，给出改进建议")
console.log(analysis)

// 场景2：批量处理多个任务
async function batchProcess(directory: string, tasks: string[]): Promise<string[]> {
  const opencode = await OpenCode.create({ directory })
  
  try {
    const results: string[] = []
    
    for (const task of tasks) {
      const { data: session } = await opencode.sessions.create({})
      const sessionID = session.id
      
      const { stream: events } = await opencode.events.subscribe()
      
      await opencode.sessions.prompt({
        sessionID,
        prompt: { text: task },
        delivery: "steer",  // 立即执行
      })
      
      let result = ""
      for await (const event of events) {
        if (event.type === "message.part.updated") {
          const part = event.properties.part
          if (part.sessionID === sessionID && part.type === "text" && part.time?.end) {
            result += part.text
          }
        }
        if (event.type === "session.status" &&
            event.properties.sessionID === sessionID &&
            event.properties.status.type === "idle") break
      }
      
      results.push(result)
    }
    
    return results
  } finally {
    await opencode.close()
  }
}

// 场景3：带文件附件的任务
async function reviewFile(directory: string, filePath: string): Promise<string> {
  const opencode = await OpenCode.create({ directory })
  const fs = await import("fs")
  
  try {
    const { data: session } = await opencode.sessions.create({})
    
    const fileContent = fs.readFileSync(filePath, "utf-8")
    const base64Content = Buffer.from(fileContent).toString("base64")
    
    await opencode.sessions.prompt({
      sessionID: session.id,
      prompt: {
        text: "请审查以下代码文件并给出改进建议：",
        files: [{
          name: filePath,
          uri: `data:text/plain;base64,${base64Content}`,
        }],
      },
    })
    
    // ... 等待结果
  } finally {
    await opencode.close()
  }
}
```

---

### 15.6 CLI 自动化脚本

**相关文件：** `packages/opencode/src/cli/cmd/run.ts`（约 900 行）

使用 `opencode run` 命令进行非交互式自动化：

```bash
# 基本使用
opencode run "重构 src/utils.ts，提取重复代码到独立函数"

# 连接远程服务器
opencode run --attach http://my-server:4096 --password secret "分析代码质量"

# 指定 Agent 和模型
opencode run --agent plan --model anthropic/claude-opus-4-5 "给出重构方案"

# 附件文件
opencode run --file ./spec.md "根据规范文件实现功能"

# JSON 格式输出（用于程序解析）
opencode run --format json "列出项目所有 TODO" | jq '.text'

# 继续上次 Session
opencode run --continue "继续之前的任务"

# Fork 并继续
opencode run --session <sessionID> --fork "在此基础上添加测试"

# 全自动模式（自动批准所有权限）
opencode run --auto "运行所有测试并修复失败的用例"

# 指定工作目录
opencode run --dir ./backend "重构后端 API"
```

#### 程序化使用 Run 命令的事件流

```typescript
// 自动化脚本示例（使用 SDK 而非 CLI）
import { createOpencodeClient } from "@opencode-ai/client"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  headers: { Authorization: `Basic ${btoa("opencode:password")}` },
})

async function runAutomatedTask(task: string) {
  // 1. 创建 Session
  const { data: session } = await client.sessions.create({})
  const sessionID = session.id

  // 2. 订阅事件流（必须在 prompt 前）
  const { stream: events } = await client.events.subscribe()

  // 3. 发送 Prompt
  await client.sessions.prompt({
    sessionID,
    prompt: { text: task },
  })

  // 4. 处理事件
  for await (const event of events) {
    switch (event.type) {
      case "message.part.updated": {
        const part = event.properties.part
        if (part.sessionID !== sessionID) continue
        
        if (part.type === "text" && part.time?.end) {
          process.stdout.write(part.text)
        }
        if (part.type === "tool" && part.state.status === "completed") {
          console.error(`\n[Tool] ${part.tool}: ${JSON.stringify(part.state.output).slice(0, 100)}`)
        }
        break
      }
      
      case "permission.asked": {
        const perm = event.properties
        if (perm.sessionID !== sessionID) continue
        // 自动批准（慎用！）
        await client.permissions.reply({
          requestID: perm.id,
          reply: "once",
        })
        break
      }
      
      case "session.error": {
        if (event.properties.sessionID !== sessionID) continue
        console.error("Error:", event.properties.error)
        return
      }
      
      case "session.status": {
        if (event.properties.sessionID !== sessionID) continue
        if (event.properties.status.type === "idle") {
          console.log("\n[Done]")
          return
        }
        break
      }
    }
  }
}

await runAutomatedTask("分析 src 目录中所有 TypeScript 文件，找出潜在的安全问题")
```

---

### 15.7 添加 CLI 子命令

**相关文件：**
- 命令入口：`packages/opencode/src/index.ts`
- Serve 命令参考：`packages/opencode/src/cli/cmd/serve.ts`
- 命令工具：`packages/opencode/src/cli/cmd/cmd.ts`

```typescript
// 新建文件：packages/opencode/src/cli/cmd/my-command.ts
import { cmd } from "./cmd"
import { Effect } from "effect"

export const MyCommand = cmd({
  command: "my-command [target]",
  describe: "My custom command description",
  
  builder: (yargs) =>
    yargs
      .positional("target", {
        type: "string",
        describe: "Target to process",
        default: ".",
      })
      .option("output", {
        type: "string",
        alias: ["o"],
        describe: "Output format",
        choices: ["json", "text"],
        default: "text",
      })
      .option("verbose", {
        type: "boolean",
        alias: ["v"],
        default: false,
      }),
  
  // handler 可以是普通 async 函数或 Effect 函数
  handler: async (args) => {
    // 访问 opencode SDK
    const { createOpencodeClient } = await import("@opencode-ai/client")
    
    // 对于本地运行，可以使用嵌入式模式
    // 这里简化为直接使用 CLI 工具
    console.log(`Processing target: ${args.target}`)
    
    if (args.verbose) {
      console.log("Verbose mode enabled")
    }
    
    // 实际业务逻辑
    const result = await processTarget(args.target)
    
    if (args.output === "json") {
      console.log(JSON.stringify(result, null, 2))
    } else {
      console.log(formatResult(result))
    }
  },
})

// 或使用 Effect 风格（可以注入 Effect 服务）
export const MyEffectCommand = cmd({
  command: "my-effect-command",
  handler: Effect.fn("Cli.myEffectCommand")(function* (args) {
    // 使用 Effect 服务
    const { Config } = yield* Effect.promise(() => import("@opencode-ai/core/config"))
    const config = yield* Config.Service
    const entries = yield* config.entries()
    
    console.log("Config entries:", entries.length)
    yield* Effect.never  // 如果需要持续运行
  }),
})
```

在 `packages/opencode/src/index.ts` 中注册：

```typescript
// 添加导入
import { MyCommand } from "./cli/cmd/my-command"

// 在 .command() 链中添加
const cli = yargs(args)
  // ... 现有命令
  .command(MyCommand)  // 添加新命令
```

---

### 15.8 监听事件流

**相关文件：** `packages/core/src/event.ts`

#### 服务端监听（Effect 风格）

```typescript
// 在 Core 层内部监听事件
import { EventV2 } from "@opencode-ai/core/event"
import { Effect, Layer } from "effect"

// 创建一个监听 Session 创建事件的 Layer
const auditLayer = Layer.effectDiscard(
  Effect.gen(function* () {
    const events = yield* EventV2.Service
    
    // 订阅特定类型事件
    // 注意：subscribe 返回 Stream，需要 fork 运行
    yield* events.subscribe(SessionEvent.Created).pipe(
      Stream.runForEach((event) =>
        Effect.sync(() => {
          console.log(`[Audit] Session created: ${event.data.sessionID}`)
          // 可以调用外部系统
        })
      ),
      Effect.forkScoped,  // 在 Scope 内 fork，Scope 关闭时自动取消
    )
    
    // 监听所有事件（用于调试）
    yield* events.all().pipe(
      Stream.filter(event => event.type.startsWith("session.")),
      Stream.runForEach((event) =>
        Effect.sync(() => console.log(`[Debug] Event: ${event.type}`))
      ),
      Effect.forkScoped,
    )
  })
)
```

#### 客户端监听（SDK 风格）

```typescript
// 通过 HTTP 客户端订阅 SSE 事件流
const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

// 全局实例级事件（live-only，不可重播）
const { stream: globalEvents } = await client.events.subscribe()
for await (const event of globalEvents) {
  console.log(event.type, event.properties)
  // 如果断开重连，需要重新订阅（无法回放丢失的事件）
}

// Session 级持久化事件流（可从指定序列号重播）
const { stream: sessionEvents } = await client.sessions.events({
  sessionID: "01JXXXX",
  after: 0,  // 从头开始，或传入上次已处理的序列号
})
for await (const event of sessionEvents) {
  console.log(`[seq:${event.durable?.seq}] ${event.type}`)
  // 记录 event.durable.seq，断线重连时从此处继续
  lastSeq = event.durable?.seq ?? lastSeq
}

// 分页获取历史事件（REST 风格，适合一次性查询）
let after = undefined
while (true) {
  const { data: history } = await client.sessions.history({
    sessionID: "01JXXXX",
    after,
    limit: 50,
  })
  
  for (const event of history.events) {
    console.log(event)
  }
  
  if (!history.hasMore) break
  after = history.events.at(-1)?.durable?.seq
}
```

---

## 16. 开发规范与贡献指南

**相关文件：** `AGENTS.md`、`CONTRIBUTING.md`

### 分支与提交规范

```
分支命名：最多三个单词，短横线分隔
  ✓ session-recovery
  ✓ fix-scroll-state
  ✓ add-custom-tool
  ✗ feat/add-new-provider（不用斜杠前缀）

Commit 格式：type(scope): summary
  类型：feat | fix | docs | chore | refactor | test
  范围（可选）：core | opencode | tui | app | desktop | sdk | plugin
  示例：feat(core): add git-status system context source
```

### Effect 代码规范

```typescript
// ✓ 先绑定服务，再调用方法
const config = yield* Config.Service
const entries = yield* config.entries()

// ✗ 嵌套 yield*（不可读，性能也差）
const entries = yield* (yield* Config.Service).entries()

// ✓ 使用 Effect.fn 为追踪添加名称
const myOperation = Effect.fn("MyModule.operation")(function* (input) {
  // ...
})

// ✓ Layer.effectDiscard 用于副作用（如注册工具、订阅事件）
const layer = Layer.effectDiscard(
  Effect.gen(function* () {
    const tools = yield* Tools.Service
    yield* tools.register({ ... }).pipe(Effect.orDie)
  })
)
```

### 导入规范

```typescript
// ✓ 通过命名空间访问（不 re-export 内部细节）
import { SessionV2 } from "@opencode-ai/core/session"
SessionV2.Service

// ✗ 星号导入
import * as Session from "@opencode-ai/core/session"

// ✗ 别名导入
import { Service as SessionService } from "@opencode-ai/core/session"

// ✓ 动态导入（用于重型模块，延迟加载）
const { Server } = yield* Effect.promise(() => import("../../server/server"))
```

### 数据库 Schema（Drizzle）

```typescript
// ✓ snake_case 列名（无需手动指定字符串别名）
const table = sqliteTable("my_table", {
  id: text().primaryKey(),
  session_id: text().notNull(),
  time_created: integer().notNull(),
})

// ✗ 驼峰命名
const table = sqliteTable("my_table", {
  sessionId: text("session_id").notNull(),
})
```

### 构建与测试命令

```bash
# 安装依赖
bun install

# 开发模式（TUI）
bun dev

# 开发 Web UI（需要先启动 server）
bun dev serve
bun run --cwd packages/app dev  # http://localhost:5173

# 开发桌面应用
bun dev:desktop

# 类型检查（全量）
bun typecheck

# 在特定包中运行类型检查
bun --cwd packages/core typecheck

# 构建可执行文件
./packages/opencode/script/build.ts --single
# 生成位置：./packages/opencode/dist/opencode-<platform>/bin/opencode

# 重新生成 SDK（修改 Protocol/Server HttpApi 后）
bun run generate  # 在 packages/client 目录执行

# Lint
bun lint

# 调试（分离 Server + TUI）
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096
# 然后：
opencode attach http://localhost:4096
```

---

*本文档基于 opencode 项目源代码（`f:\code\opencode\`）全面分析生成，所有代码示例均基于实际源码提取，文件路径均已验证。*
