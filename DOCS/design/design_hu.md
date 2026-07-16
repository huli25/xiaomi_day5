# AI智能座舱语音任务编排与安全确认系统 - 技术设计方案

## 文档信息

| 项目 | 内容 |
|------|------|
| 文档版本 | v1.0 |
| 创建日期 | 2026-07-16 |
| 维护责任人 | 项目团队 |
| 基于文档 | DOCS/diagnosis/product-prd.md |

---

## 1. 项目概述

### 1.1 项目背景

根据产品 PRD，本项目需要实现一个 AI 智能座舱语音任务编排与安全确认系统。系统核心功能包括：

- 复合指令结构化拆解
- 权限校验与车况校验
- 三态判定（execute/confirm/reject）
- 六步闭环执行流程
- 完整的日志追溯

### 1.2 设计约束

根据 PRD 的 Non-goals，本方案需遵循以下约束：

- **不做**：语音识别、自然语言生成、车辆硬件控制、机器学习模型训练
- **MVP 范围**：单车单系统、文本输入、模拟执行、简单身份切换
- **安全要求**：AI 无最终执行权、必须调试手段、越权零容忍

---

## 2. 项目方案总览

本章节提供四种技术方案，适用于不同场景和团队规模。所有方案均满足 PRD 的功能要求和安全约束。

---

## 3. 方案一：轻量级单体架构

### 3.1 技术栈

| 层级 | 技术选型 | 版本要求 | 说明 |
|------|---------|---------|------|
| 前端框架 | React | 18.x | 函数式组件 + Hooks |
| 前端构建 | Vite | 5.x | 快速热更新 |
| 前端语言 | TypeScript | 5.x | 类型安全 |
| UI 组件库 | Ant Design | 5.x | 企业级组件 |
| 状态管理 | Zustand | 4.x | 轻量级状态管理 |
| 后端框架 | Express | 4.x | 轻量 Node.js 框架 |
| 后端语言 | TypeScript | 5.x | 与前端统一语言栈 |
| 数据库 | better-sqlite3 | 9.x | 同步 SQLite，性能好 |
| ORM | Prisma | 5.x | 类型安全的数据库访问 |
| 日志 | Winston | 3.x | 结构化日志 |
| 验证 | Zod | 3.x | 运行时类型验证 |
| 测试 | Vitest | 1.x | 快速单元测试 |

### 3.2 系统架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            轻量级单体架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         前端 (React + Vite)                          │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │   │
│  │  │ 指令输入组件 │ │ 拆解展示组件 │ │ 确认交互组件 │ │ 调试面板    │   │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │   │
│  │                              │                                       │   │
│  │                      Zustand 状态管理                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                               REST API                                      │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     后端 (Express + TypeScript)                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                      路由层 (Router)                          │   │   │
│  │  │  /api/orchestrate  /api/execute  /api/debug  /api/session    │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │   │
│  │  │ 编排引擎   │ │ 安全校验   │ │ 执行监控   │ │ 权限管控   │       │   │
│  │  │ Orchestrat │ │ Validator  │ │ Executor   │ │ Permission │       │   │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │   │
│  │                              │                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    业务规则层 (Rules)                         │   │   │
│  │  │  permission-rules.ts  risk-rules.ts  vehicle-rules.ts        │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    数据层 (Prisma + SQLite)                   │   │   │
│  │  │  Session  Task  ExecutionLog  VehicleSnapshot                │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 目录结构

```
src/
├── frontend/                          # 前端应用
│   ├── src/
│   │   ├── components/
│   │   │   ├── InputPanel/           # 指令输入面板
│   │   │   │   ├── index.tsx
│   │   │   │   └── styles.css
│   │   │   ├── DisassemblyView/      # 拆解结果展示
│   │   │   │   ├── index.tsx
│   │   │   │   ├── TaskList.tsx
│   │   │   │   └── AmbiguityTag.tsx
│   │   │   ├── ConfirmDialog/        # 确认对话框
│   │   │   │   ├── index.tsx
│   │   │   │   ├── ParamFixPanel.tsx
│   │   │   │   └── ConfirmActions.tsx
│   │   │   ├── TimelineView/         # 执行时间线
│   │   │   │   ├── index.tsx
│   │   │   │   └── StepCard.tsx
│   │   │   ├── DebugPanel/           # 调试面板
│   │   │   │   ├── RoleSwitcher.tsx
│   │   │   │   ├── VehicleStateEditor.tsx
│   │   │   │   └── DebugHistory.tsx
│   │   │   └── Layout/
│   │   │       ├── MainLayout.tsx
│   │   │       └── Header.tsx
│   │   ├── store/
│   │   │   ├── useSessionStore.ts    # 会话状态
│   │   │   ├── useDebugStore.ts      # 调试状态
│   │   │   └── useVehicleStore.ts    # 车况状态
│   │   ├── api/
│   │   │   ├── orchestrate.ts        # 编排接口
│   │   │   ├── execute.ts            # 执行接口
│   │   │   └── debug.ts              # 调试接口
│   │   ├── types/
│   │   │   └── index.ts              # 共享类型定义
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── index.html
│   ├── vite.config.ts
│   ├── tsconfig.json
│   └── package.json
├── backend/                           # 后端应用
│   ├── src/
│   │   ├── routes/
│   │   │   ├── index.ts              # 路由入口
│   │   │   ├── orchestrate.ts        # 编排路由
│   │   │   ├── execute.ts            # 执行路由
│   │   │   ├── session.ts            # 会话路由
│   │   │   └── debug.ts              # 调试路由
│   │   ├── services/
│   │   │   ├── orchestrator/
│   │   │   │   ├── index.ts          # 编排服务入口
│   │   │   │   ├── Disassembler.ts   # 指令拆解
│   │   │   │   ├── TaskPlanner.ts    # 任务规划
│   │   │   │   └── TaskScheduler.ts  # 任务调度
│   │   │   ├── validator/
│   │   │   │   ├── index.ts          # 校验服务入口
│   │   │   │   ├── PermissionCheck.ts    # 权限校验
│   │   │   │   ├── VehicleStateCheck.ts  # 车况校验
│   │   │   │   └── RiskAssess.ts         # 风险评估
│   │   │   ├── executor/
│   │   │   │   ├── index.ts          # 执行服务入口
│   │   │   │   ├── TaskExecutor.ts   # 任务执行
│   │   │   │   ├── StateTracker.ts   # 状态跟踪
│   │   │   │   └── FailureHandler.ts # 失败处理
│   │   │   ├── interaction/
│   │   │   │   ├── index.ts          # 交互服务入口
│   │   │   │   ├── ConfirmManager.ts # 确认管理
│   │   │   │   └── ParamResolver.ts  # 参数解析
│   │   │   └── permission/
│   │   │       ├── index.ts          # 权限服务入口
│   │   │       ├── RoleManager.ts    # 角色管理
│   │   │       └── BoundaryControl.ts # 边界控制
│   │   ├── rules/
│   │   │   ├── permission-rules.ts   # 权限规则
│   │   │   ├── risk-rules.ts         # 风险规则
│   │   │   ├── vehicle-rules.ts      # 车况规则
│   │   │   └── ambiguity-rules.ts    # 模糊性规则
│   │   ├── models/
│   │   │   ├── Session.ts
│   │   │   ├── Task.ts
│   │   │   ├── ExecutionLog.ts
│   │   │   └── VehicleSnapshot.ts
│   │   ├── types/
│   │   │   └── index.ts
│   │   ├── utils/
│   │   │   ├── logger.ts
│   │   │   └── validator.ts
│   │   ├── db/
│   │   │   ├── index.ts
│   │   │   └── schema.prisma
│   │   ├── app.ts
│   │   └── server.ts
│   ├── tsconfig.json
│   ├── package.json
│   └── prisma/
│       └── schema.prisma
└── shared/                            # 共享类型（可选）
    └── types/
        └── index.ts
```

### 3.4 核心模块设计

#### 3.4.1 编排引擎（Orchestrator）

```typescript
// 编排引擎接口
interface IOrchestrator {
  disassemble(input: VoiceCommand): DisassemblyResult;
  plan(tasks: Task[]): PlanResult;
  schedule(plan: PlanResult): ScheduleResult;
}
```

**职责：**
- 接收自然语言指令
- 结构化拆解为任务列表
- 识别模糊参数和依赖关系
- 输出拆解结果供安全校验

#### 3.4.2 安全校验（Validator）

```typescript
// 校验器接口
interface IValidator {
  checkPermission(role: Role, action: Action): PermissionResult;
  checkVehicleState(state: VehicleState, action: Action): VehicleResult;
  assessRisk(action: Action, params: Params): RiskLevel;
  makeDecision(task: Task, context: Context): Decision;
}
```

**职责：**
- 权限校验（owner/guest）
- 车况校验（速度、挡位、电量、故障）
- 风险等级判定（高/中/低）
- 做出三态判定（execute/confirm/reject）

#### 3.4.3 执行监控（Executor）

```typescript
// 执行器接口
interface IExecutor {
  execute(task: Task): ExecutionResult;
  trackState(taskId: string): StateUpdate;
  handleFailure(taskId: string, error: Error): FailureResult;
}
```

**职责：**
- 模拟执行任务
- 跟踪执行状态
- 处理失败和异常
- 写回车态变更

#### 3.4.4 确认交互（Interaction）

```typescript
// 交互管理器接口
interface IInteractionManager {
  requestConfirm(task: Task, reason: string): ConfirmRequest;
  resolveAmbiguity(taskId: string, params: Params): ResolutionResult;
  cancelConfirm(sessionId: string, taskId: string): CancelResult;
}
```

**职责：**
- 发起确认请求
- 处理参数补全
- 管理确认超时
- 记录用户选择

### 3.5 数据库设计

```prisma
// Prisma Schema
model Session {
  id          String   @id @default(uuid())
  role        Role     @map("role")
  input       String
  createdAt   DateTime @default(now())
  completedAt DateTime?
  
  tasks       Task[]
  snapshots   VehicleSnapshot[]
}

model Task {
  id              String      @id @default(uuid())
  sessionId       String      @map("session_id")
  step            Int
  object          String
  action          String
  params          Json
  ambiguity       Json?
  decision        Decision
  reason          String?
  result          Result?
  executedAt      DateTime?
  
  session         Session     @relation(fields: [sessionId], references: [id])
  
  @@map("tasks")
}

model VehicleSnapshot {
  id          String   @id @default(uuid())
  sessionId   String   @map("session_id")
  type        String   // 'initial' | 'pre_execute' | 'post_execute'
  speedKmh    Int      @map("speed_kmh")
  gear        Gear
  batteryPct  Int      @map("battery_pct")
  fault       Boolean
  createdAt   DateTime @default(now())
  
  session     Session  @relation(fields: [sessionId], references: [id])
  
  @@map("vehicle_snapshots")
}

enum Role {
  owner
  guest
}

enum Gear {
  P
  R
  N
  D
}

enum Decision {
  execute
  confirm
  reject
}

enum Result {
  success
  blocked
  failed
  cancelled
}
```

### 3.6 API 设计

| 端点 | 方法 | 描述 |
|------|------|------|
| POST | /api/orchestrate | 提交指令并获取拆解结果 |
| POST | /api/execute/:taskId | 执行单个任务 |
| POST | /api/confirm/:taskId | 确认并继续执行 |
| POST | /api/cancel/:taskId | 取消确认 |
| POST | /api/debug/role | 更新调试角色 |
| POST | /api/debug/vehicle | 更新调试车态 |
| GET | /api/session/:id | 获取会话详情 |
| GET | /api/session/:id/timeline | 获取时间线数据 |

### 3.7 优缺点分析

| 优点 | 缺点 |
|------|------|
| 开发速度快，快速出 MVP | 单点故障风险 |
| 架构简单，易于理解和维护 | 扩展性有限 |
| 前后端 TypeScript 统一 | 数据库扩展受限于 SQLite |
| 调试方便，无需容器环境 | 并发能力有限 |
| 适合小团队（2-3人） | 难以支撑大规模用户 |

### 3.8 适用场景

- **MVP 原型演示**：两天 hackathon 或快速验证
- **小团队开发**：2-3 人团队
- **演示环境**：不需要高可用性
- **学习目的**：理解系统架构

### 3.9 预计工时

| 模块 | 工时 | 说明 |
|------|------|------|
| 前端基础搭建 | 4h | React + Vite + TypeScript |
| 前端组件开发 | 8h | 5 个核心组件 |
| 后端基础搭建 | 3h | Express + Prisma |
| 编排引擎 | 4h | 拆解、规划、调度 |
| 安全校验模块 | 6h | 权限、车况、风险 |
| 执行监控模块 | 3h | 执行、跟踪、异常 |
| 确认交互模块 | 3h | 确认、补参、取消 |
| 调试能力 | 2h | 身份切换、车态修改 |
| 单元测试 | 4h | 核心模块测试 |
| 集成调试 | 3h | 端到端联调 |
| **总计** | **40h** | |

---

## 4. 方案二：微服务架构

### 4.1 技术栈

| 层级 | 技术选型 | 版本要求 | 说明 |
|------|---------|---------|------|
| 前端框架 | React | 18.x | SPA 应用 |
| 前端构建 | Vite | 5.x | 快速构建 |
| 前端语言 | TypeScript | 5.x | 类型安全 |
| UI 组件库 | Ant Design | 5.x | 企业级组件 |
| 状态管理 | Zustand + React Query | 最新 | 服务端状态+客户端状态 |
| API 网关 | Express / NestJS | 4.x / 10.x | 统一入口 |
| 编排服务 | Node.js / Go | 20.x / 1.21 | 任务编排逻辑 |
| 校验服务 | Node.js / Go | 20.x / 1.21 | 安全校验逻辑 |
| 执行服务 | Node.js / Go | 20.x / 1.21 | 任务执行逻辑 |
| 交互服务 | Node.js / Go | 20.x / 1.21 | 确认交互逻辑 |
| 消息队列 | RabbitMQ | 3.12 | 服务间异步通信 |
| 主数据库 | PostgreSQL | 15.x | 结构化数据 |
| 缓存 | Redis | 7.x | 会话缓存、实时状态 |
| 容器化 | Docker | 24.x | 容器化部署 |
| 编排工具 | Docker Compose | 2.x | 本地开发 |
| 日志 | ELK Stack | 8.x | 日志收集分析 |
| 测试 | Vitest / Jest | 1.x / 29.x | 单元测试 |

### 4.2 系统架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            微服务架构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         负载均衡层 (Nginx)                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      API Gateway (Express/NestJS)                    │   │
│  │  • 路由分发    • 认证鉴权    • 限流熔断    • 请求聚合                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│          │            │            │            │                           │
│          ▼            ▼            ▼            ▼                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                    │
│  │ Orchestr │  │ Validat  │  │ Execute  │  │ Interact │                    │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │                    │
│  │ :3001    │  │ :3002    │  │ :3003    │  │ :3004    │                    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘                    │
│       │            │            │            │                            │
│       └────────────┴─────┬──────┴────────────┘                            │
│                          │                                                 │
│                    ┌─────┴─────┐                                           │
│                    │  RabbitMQ │  ← 消息队列（异步解耦）                      │
│                    └─────┬─────┘                                           │
│                          │                                                 │
│       ┌──────────────────┼──────────────────┐                              │
│       ▼                  ▼                  ▼                              │
│  ┌─────────┐       ┌─────────┐       ┌─────────┐                           │
│  │PostgreSQL│       │  Redis  │       │   ELK   │                           │
│  │  主数据库 │       │  缓存   │       │  日志   │                           │
│  └─────────┘       └─────────┘       └─────────┘                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 服务设计

#### 4.3.1 API Gateway

**职责：**
- 统一入口，路由分发
- 身份认证（简化版：角色传递）
- 请求限流
- 日志记录

```yaml
# docker-compose.yml (Gateway)
api-gateway:
  image: nginx:alpine
  ports:
    - "8080:80"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

#### 4.3.2 Orchestrator Service

**职责：**
- 接收语音指令
- 拆解为任务列表
- 返回拆解结果

```yaml
# docker-compose.yml (Orchestrator)
orchestrator:
  build: ./services/orchestrator
  ports:
    - "3001:3001"
  environment:
    - NODE_ENV=production
    - REDIS_HOST=redis
    - RABBITMQ_HOST=rabbitmq
  depends_on:
    - redis
    - rabbitmq
```

#### 4.3.3 Validator Service

**职责：**
- 权限校验
- 车况校验
- 风险评估
- 三态判定

#### 4.3.4 Executor Service

**职责：**
- 任务执行
- 状态跟踪
- 失败处理
- 车态更新

#### 4.3.5 Interaction Service

**职责：**
- 确认请求管理
- 参数补全
- WebSocket 推送（可选）

### 4.4 消息队列设计

```typescript
// RabbitMQ 交换机和队列设计
interface QueueConfig {
  exchange: {
    name: 'cabin.system';
    type: 'topic';
  };
  queues: {
    orchestrate: 'orchestrate.result';
    validate: 'validate.request';
    execute: 'execute.command';
    interaction: 'interaction.notify';
  };
}
```

### 4.5 服务间通信

```typescript
// 服务间通信示例 (使用 RPC 或 HTTP)
interface IServiceClient {
  // Orchestrator -> Validator
  validate(task: Task, context: Context): Promise<ValidationResult>;
  
  // Validator -> Executor
  execute(task: Task): Promise<ExecutionResult>;
  
  // Executor -> Interaction
  requestConfirm(taskId: string, reason: string): Promise<void>;
  
  // 各服务 -> Redis
  cacheSession(sessionId: string, data: Session): Promise<void>;
}
```

### 4.6 优缺点分析

| 优点 | 缺点 |
|------|------|
| 服务独立部署和扩展 | 架构复杂，开发成本高 |
| 故障隔离，单服务不影响全局 | 需要服务治理（监控、追踪） |
| 技术栈可独立选择 | 数据库事务处理复杂 |
| 支持高并发 | 需要容器环境 |
| 适合大型团队 | 调试相对复杂 |

### 4.7 适用场景

- **生产环境**：需要高可用性
- **大型团队**：5 人以上
- **长期项目**：需要维护和扩展
- **云原生部署**：K8s 环境

### 4.8 预计工时

| 模块 | 工时 | 说明 |
|------|------|------|
| 基础设施搭建 | 8h | Docker、K8s、CI/CD |
| API Gateway | 4h | 路由、认证、限流 |
| 各微服务开发 | 20h | 5 个服务各 4h |
| 消息队列集成 | 4h | RabbitMQ 配置 |
| 数据库设计 | 4h | PostgreSQL 建模 |
| 服务治理 | 6h | 监控、日志、追踪 |
| 集成测试 | 6h | 服务间集成 |
| **总计** | **52h** | |

---

## 5. 方案三：全栈 TypeScript 方案

### 5.1 技术栈

| 层级 | 技术选型 | 版本要求 | 说明 |
|------|---------|---------|------|
| 前端框架 | Next.js | 14.x | App Router，支持 SSR |
| 前端语言 | TypeScript | 5.x | 严格模式 |
| UI 组件库 | Tailwind CSS + Radix | 最新 | 原子化 CSS |
| 状态管理 | React Query + Zustand | 最新 | 服务端+客户端状态 |
| 后端框架 | tRPC + Zod | 10.x / 3.x | 端到端类型安全 |
| ORM | Prisma | 5.x | 类型安全数据库访问 |
| 数据库 | SQLite (MVP) / PostgreSQL | 3.x / 15.x | 可切换 |
| 部署 | Vercel / Docker | 最新 | 边缘部署支持 |
| 测试 | Vitest | 1.x | 单元测试 |
| E2E 测试 | Playwright | 1.x | 端到端测试 |

### 5.2 核心特性：tRPC 端到端类型安全

```typescript
// server/routers/orchestrate.ts
import { z } from 'zod';
import { router, publicProcedure } from '../trpc';

export const orchestrateRouter = router({
  // 提交指令
  submit: publicProcedure
    .input(z.object({
      input: z.string().min(1).max(200),
      role: z.enum(['owner', 'guest']),
    }))
    .mutation(async ({ input, ctx }) => {
      const result = await ctx.orchestrator.disassemble(input);
      return result;
    }),

  // 执行任务
  execute: publicProcedure
    .input(z.object({
      sessionId: z.string().uuid(),
      taskId: z.string().uuid(),
    }))
    .mutation(async ({ input, ctx }) => {
      return ctx.executor.execute(input.taskId);
    }),

  // 确认任务
  confirm: publicProcedure
    .input(z.object({
      sessionId: z.string().uuid(),
      taskId: z.string().uuid(),
      params: z.record(z.any()).optional(),
    }))
    .mutation(async ({ input, ctx }) => {
      return ctx.interaction.confirm(input.taskId, input.params);
    }),
});
```

### 5.3 系统架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         全栈 TypeScript 方案                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    Next.js 应用 (Monorepo)                           │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                      前端层 (React)                           │   │   │
│  │  │  • 页面组件    • UI 组件    • 状态管理                        │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │                              ▼                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    tRPC 客户端                               │   │   │
│  │  │  • 类型安全的 API 调用    • 自动推断返回类型                   │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │                              ▼                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    tRPC 服务端                               │   │   │
│  │  │  • 路由定义    • 输入验证 (Zod)    • 业务逻辑                  │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐ │   │
│  │  │ 编排服务     │ │ 校验服务     │ │ 执行服务     │ │ 交互服务   │ │   │
│  │  │ Orchestrator │ │ Validator    │ │ Executor     │ │Interaction │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └────────────┘ │   │
│  │                              │                                       │   │
│  │                              ▼                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    Prisma ORM                                │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │                              ▼                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    SQLite / PostgreSQL                       │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 目录结构

```
src/
├── app/                              # Next.js App Router
│   ├── (main)/                       # 主布局
│   │   ├── page.tsx                  # 首页
│   │   ├── orchestrate/
│   │   │   └── page.tsx              # 编排页面
│   │   └── debug/
│   │       └── page.tsx              # 调试页面
│   ├── api/
│   │   └── trpc/[trpc]/route.ts      # tRPC API 路由
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── ui/                           # 基础 UI 组件
│   ├── InputPanel/
│   ├── DisassemblyView/
│   ├── ConfirmDialog/
│   ├── TimelineView/
│   └── DebugPanel/
├── server/
│   ├── routers/                      # tRPC 路由
│   │   ├── orchestrate.ts
│   │   ├── execute.ts
│   │   ├── session.ts
│   │   └── debug.ts
│   ├── services/                     # 业务服务
│   │   ├── orchestrator/
│   │   ├── validator/
│   │   ├── executor/
│   │   └── interaction/
│   ├── rules/                        # 业务规则
│   └── trpc.ts                       # tRPC 初始化
├── lib/
│   ├── prisma.ts                     # Prisma 客户端
│   └── utils.ts
├── prisma/
│   └── schema.prisma
└── types/
    └── index.ts
```

### 5.5 优缺点分析

| 优点 | 缺点 |
|------|------|
| 端到端类型安全，无重复定义 | 学习曲线（tRPC 概念） |
| SSR/SSG 支持，SEO 友好 | 需要熟悉 TypeScript |
| 前后端代码共享类型 | Prisma 性能一般 |
| Vercel 一键部署 | 单体架构限制 |
| 开发体验好 | 不适合大型团队 |

### 5.6 适用场景

- **中型项目**：3-5 人团队
- **类型安全优先**：需要严格类型检查
- **快速迭代**：需要 SSR 和边缘部署
- **学习项目**：TypeScript 最佳实践

### 5.7 预计工时

| 模块 | 工时 | 说明 |
|------|------|------|
| Next.js 基础搭建 | 4h | App Router 配置 |
| tRPC 配置 | 3h | 端到端类型安全 |
| Prisma 配置 | 2h | 数据库建模 |
| 前端组件 | 8h | UI 组件开发 |
| 业务服务 | 12h | 4 个核心服务 |
| 业务规则 | 4h | 规则引擎 |
| 测试 | 4h | 单元+E2E |
| 部署 | 2h | Vercel/Docker |
| **总计** | **39h** | |

---

## 6. 方案四：Python + 前端方案

### 6.1 技术栈

| 层级 | 技术选型 | 版本要求 | 说明 |
|------|---------|---------|------|
| 前端框架 | Vue | 3.x | Composition API |
| 前端构建 | Vite | 5.x | 快速构建 |
| 前端语言 | TypeScript | 5.x | 类型安全 |
| UI 组件库 | Element Plus | 2.x | Vue 3 组件库 |
| 状态管理 | Pinia | 2.x | Vue 状态管理 |
| 后端框架 | FastAPI | 0.104.x | 高性能异步框架 |
| 后端语言 | Python | 3.11+ | 现代 Python |
| 数据验证 | Pydantic | 2.x | 类型验证 |
| 数据库 | SQLite | 3.x | MVP 使用 |
| 文档 | OpenAPI/Swagger | 自动生成 | API 文档 |
| 规则引擎 | 内部实现 | - | Python 规则引擎 |
| 测试 | pytest | 7.x | 单元测试 |
| ASGI 服务器 | Uvicorn | 0.24.x | ASGI 服务器 |

### 6.2 Python 规则引擎优势

```python
# Python 规则引擎实现示例
from typing import Protocol, runtime_checkable
from dataclasses import dataclass
from enum import Enum

class RiskLevel(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"

@dataclass
class Task:
    object: str
    action: str
    params: dict
    ambiguity: dict | None = None

@dataclass
class ValidationContext:
    role: str
    vehicle_state: dict
    session_id: str

@runtime_checkable
class Rule(Protocol):
    """规则协议"""
    def applies(self, task: Task, context: ValidationContext) -> bool: ...
    def evaluate(self, task: Task, context: ValidationContext) -> tuple[str, str]: ...

class PermissionRule:
    """权限规则"""
    def applies(self, task: Task, context: ValidationContext) -> bool:
        return task.object in ["门锁", "授权", "钥匙共享"]
    
    def evaluate(self, task: Task, context: ValidationContext) -> tuple[str, str]:
        if context.role == "guest":
            return ("reject", "访客无权限执行此操作")
        return ("continue", "")

class VehicleStateRule:
    """车况规则"""
    def applies(self, task: Task, context: ValidationContext) -> bool:
        return context.vehicle_state.get("speed_kmh", 0) > 0
    
    def evaluate(self, task: Task, context: ValidationContext) -> tuple[str, str]:
        speed = context.vehicle_state.get("speed_kmh", 0)
        if speed > 0 and task.object == "车窗":
            return ("confirm", "车辆正在行驶，是否确认打开车窗？")
        return ("continue", "")

class AmbiguityRule:
    """模糊性规则"""
    def applies(self, task: Task, context: ValidationContext) -> bool:
        return task.ambiguity is not None
    
    def evaluate(self, task: Task, context: ValidationContext) -> tuple[str, str]:
        if task.ambiguity:
            return ("confirm", f"参数模糊，需要确认：{task.ambiguity}")
        return ("continue", "")

class RuleEngine:
    """规则引擎"""
    def __init__(self):
        self.rules: list[Rule] = [
            PermissionRule(),
            VehicleStateRule(),
            AmbiguityRule(),
        ]
    
    def evaluate(self, task: Task, context: ValidationContext) -> dict:
        for rule in self.rules:
            if rule.applies(task, context):
                decision, reason = rule.evaluate(task, context)
                if decision != "continue":
                    return {
                        "decision": decision,
                        "reason": reason,
                        "rule": rule.__class__.__name__
                    }
        return {"decision": "execute", "reason": "", "rule": None}
```

### 6.3 系统架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Python + 前端方案                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    前端 (Vue 3 + Vite + TypeScript)                  │   │
│  │  • 页面组件    • Element Plus    • Pinia 状态管理                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                               REST API                                      │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    后端 (FastAPI + Python 3.11)                      │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                      API 路由层                              │   │   │
│  │  │  /api/orchestrate  /api/execute  /api/debug  /api/session   │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │   │
│  │  │ 编排服务   │ │ 规则引擎   │ │ 执行服务   │ │ 交互服务   │       │   │
│  │  │ Orchestrat │ │ RuleEngine │ │ Executor   │ │Interaction │       │   │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │   │
│  │                              │                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    Pydantic 数据模型                         │   │   │
│  │  │  • 输入验证    • 输出序列化    • 自动文档                     │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                              │                                       │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                    SQLite 数据库                             │   │   │
│  │  │  • SQLAlchemy ORM    • Alembic 迁移    • 会话管理            │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 FastAPI 实现示例

```python
# main.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from typing import Optional
import uvicorn

app = FastAPI(
    title="AI智能座舱语音任务编排系统",
    description="基于 FastAPI 的安全校验与任务编排后端",
    version="1.0.0"
)

# CORS 配置
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Pydantic 模型
class OrchestrateRequest(BaseModel):
    input: str = Field(..., min_length=1, max_length=200)
    role: str = Field(..., pattern="^(owner|guest)$")

class TaskDecision(BaseModel):
    task_id: str
    decision: str  # execute, confirm, reject
    reason: str

# 路由
@app.post("/api/orchestrate")
async def orchestrate(request: OrchestrateRequest):
    """指令编排接口"""
    # 调用编排服务
    result = await orchestrator.disassemble(request.input, request.role)
    return result

@app.post("/api/execute/{task_id}")
async def execute_task(task_id: str, confirmed: bool = False):
    """任务执行接口"""
    result = await executor.execute(task_id, confirmed=confirmed)
    return result

@app.get("/api/session/{session_id}")
async def get_session(session_id: str):
    """获取会话详情"""
    session = await session_service.get(session_id)
    if not session:
        raise HTTPException(status_code=404, detail="Session not found")
    return session

# 健康检查
@app.get("/health")
async def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### 6.5 优缺点分析

| 优点 | 缺点 |
|------|------|
| Python 规则引擎实现自然 | 需要维护两套语言栈 |
| Pydantic 验证强大 | Python GIL 限制并发 |
| FastAPI 性能优秀 | 前端 Vue 生态不如 React |
| AI/ML 扩展方便 | 部署相对复杂 |
| 大量现成库可用 | 类型系统不如 TypeScript |

### 6.6 适用场景

- **AI 扩展需求**：后续可能集成 AI 模型
- **规则复杂**：Python 规则引擎更灵活
- **团队 Python 背景**：熟悉 Python 开发
- **数据处理**：需要大量数据处理

### 6.7 预计工时

| 模块 | 工时 | 说明 |
|------|------|------|
| Vue 基础搭建 | 3h | Vue 3 + Vite + TypeScript |
| FastAPI 基础 | 3h | 框架配置、路由、中间件 |
| Pydantic 模型 | 2h | 数据验证模型 |
| 前端组件 | 8h | Element Plus 组件 |
| 规则引擎 | 6h | Python 规则引擎 |
| 业务服务 | 10h | 编排、校验、执行、交互 |
| 数据库 | 3h | SQLAlchemy + SQLite |
| 测试 | 4h | pytest 单元测试 |
| **总计** | **39h** | |

---

## 7. 方案对比

### 7.1 核心指标对比

| 指标 | 方案一：单体 | 方案二：微服务 | 方案三：全栈 TS | 方案四：Python |
|------|-------------|---------------|----------------|---------------|
| **开发速度** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **类型安全** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **扩展性** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **运维复杂度** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **团队要求** | 低 | 高 | 中 | 中 |
| **适合规模** | 1-3 人 | 5+ 人 | 3-5 人 | 2-4 人 |

### 7.2 成本估算对比

| 成本项 | 方案一 | 方案二 | 方案三 | 方案四 |
|--------|--------|--------|--------|--------|
| **开发工时** | 40h | 52h | 39h | 39h |
| **基础设施** | 低 | 高 | 中 | 低 |
| **运维成本** | 低 | 高 | 中 | 低 |
| **总成本** | 低 | 高 | 中 | 中低 |

### 7.3 推荐选择

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| MVP 原型演示 | 方案一 | 快速开发，两天内出原型 |
| 大型生产项目 | 方案二 | 高可用，可扩展 |
| 类型安全优先 | 方案三 | 端到端类型安全 |
| AI 功能扩展 | 方案四 | Python AI 生态丰富 |

---

## 8. 技术选型建议

### 8.1 短期项目（两周内交付）

**推荐：方案一（轻量级单体架构）**

理由：
- 开发速度最快
- 架构简单，易于调试
- 满足 PRD 所有功能要求
- 适合 hackathon 或快速验证

### 8.2 中期项目（一个月交付）

**推荐：方案三（全栈 TypeScript）**

理由：
- 前后端类型统一
- 开发体验好
- SSR 支持未来可能的 Web 需求
- 代码质量高，易维护

### 8.3 长期项目（生产环境）

**推荐：方案二（微服务架构）**

理由：
- 高可用性
- 服务独立部署和扩展
- 故障隔离
- 适合大规模用户

---

## 9. 安全设计要点

### 9.1 越权拦截（100% 覆盖）

```typescript
// 所有权限校验必须经过统一入口
interface IPermissionGuard {
  check(role: Role, action: Action, object: string): PermissionResult;
}

// 禁止从输入推断身份
const forbiddenPatterns = [
  "我是车主",
  "我是主人", 
  "让我授权",
  // ...
];

function validateInputIntegrity(input: string, role: Role): void {
  for (const pattern of forbiddenPatterns) {
    if (input.includes(pattern) && role === 'guest') {
      // 必须拒绝，不能因为话术内容改变身份
      throw new SecurityError('身份来源必须为显式 role');
    }
  }
}
```

### 9.2 AI 无最终执行权

```typescript
// 模糊参数检测
interface AmbiguityDetector {
  detect(task: Task): Ambiguity | null;
}

// 模糊参数必须 confirm，不能 execute
function makeDecision(task: Task, context: Context): Decision {
  const ambiguity = ambiguityDetector.detect(task);
  
  if (ambiguity) {
    // 【安全关键】模糊参数必须确认，不能直执
    return { decision: 'confirm', reason: `参数模糊: ${ambiguity.detail}` };
  }
  
  // 其他校验...
}
```

### 9.3 执行前再查车态

```typescript
// 确认时必须再查车态
async function confirmAndExecute(taskId: string, context: Context): Promise<Result> {
  // 1. 获取当前车态（再查）
  const currentVehicleState = await vehicleStateService.get();
  
  // 2. 校验车态是否允许执行
  const validation = await validator.validate(taskId, {
    ...context,
    vehicleState: currentVehicleState
  });
  
  if (validation.decision === 'reject') {
    return { result: 'blocked', reason: validation.reason };
  }
  
  // 3. 执行任务
  return executor.execute(taskId);
}
```

---

## 10. 下一步行动

### 10.1 方案确认

请确认选择的技术方案，我们将进入详细设计阶段：

1. **架构设计细化**
2. **接口设计（API Contract）**
3. **数据库详细设计**
4. **组件设计**
5. **测试策略**

### 10.2 交付计划

| 阶段 | 交付物 | 预计时间 |
|------|--------|---------|
| 方案确认 | 选定方案文档 | 第 1 天 |
| 架构设计 | 详细架构图、接口文档 | 第 2 天 |
| 编码实现 | 前端 + 后端代码 | 第 3-10 天 |
| 集成测试 | 端到端测试 | 第 11-12 天 |
| 演示准备 | 演示用例、文档 | 第 13-14 天 |

---

**文档版本**：v1.0
**创建日期**：2026-07-16
**维护责任人**：项目团队
