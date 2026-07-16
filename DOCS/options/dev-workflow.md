# AI智能座舱语音任务编排与安全确认系统 - 实现计划报告

> **交付链第 5 棒** · 上游：`DOCS/options/options_hu.md`（方案一决策）
> 版本：v0.1 · 2026-07-16
> **本文基于方案一（React+Express 规则模板）的技术选型，制定完整实现计划。**

---

## 一、技术选型

### 1.1 技术栈总览

| 层级 | 技术 | 说明 |
|------|------|------|
| **前端** | React 18 + Vite + TypeScript | 响应式UI，组件化开发 |
| **后端** | Node.js + Express + TypeScript | 轻量API服务 |
| **接口** | JSON REST (HTTP) | 前后端分离通信 |
| **状态存储** | 服务端进程内存 | 车态/会话/角色管理 |
| **任务拆解** | 本地规则/关键词/正则模板 | 无外部模型依赖 |
| **安全判定** | 服务端规则引擎 | 唯一决策源 |

### 1.2 技术选型理由

| 优势 | 说明 |
|------|------|
| **安全可控** | 纯规则引擎，无外部依赖，决策源唯一 |
| **离线可用** | 完全离线可演示，不依赖网络 |
| **TypeScript统一** | 前后端同一语言栈，类型安全 |
| **快速交付** | 双进程简单架构，配置少 |

### 1.3 项目目录结构

```
SRC/
├── frontend/                    # React前端
│   ├── src/
│   │   ├── components/
│   │   │   ├── disassembly/     # 拆解结果展示组件
│   │   │   │   ├── TaskList.tsx
│   │   │   │   ├── TaskItem.tsx
│   │   │   │   └── AmbiguityTag.tsx
│   │   │   ├── confirm/         # 确认交互组件
│   │   │   │   ├── ConfirmDialog.tsx
│   │   │   │   └── ParameterInput.tsx
│   │   │   ├── timeline/        # 时间线展示
│   │   │   │   ├── Timeline.tsx
│   │   │   │   └── StepResult.tsx
│   │   │   ├── debug/          # 调试面板
│   │   │   │   ├── RoleSwitcher.tsx
│   │   │   │   └── VehicleStateEditor.tsx
│   │   │   └── layout/          # 布局组件
│   │   ├── hooks/              # 自定义Hooks
│   │   ├── services/           # API调用服务
│   │   ├── types/              # TypeScript类型定义
│   │   ├── store/              # 前端状态管理
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── package.json
│   └── vite.config.ts
│
└── backend/                     # Express后端
    ├── src/
    │   ├── orchestrator/        # 任务编排引擎
    │   │   ├── parser.ts         # 指令解析器
    │   │   ├── decomposer.ts     # 任务拆解器
    │   │   └── templates/        # 规则模板
    │   │       ├── intent.yaml   # 意图模板
    │   │       └── params.yaml   # 参数模板
    │   ├── validator/            # 安全校验模块
    │   │   ├── permission.ts     # 权限校验
    │   │   ├── vehicle.ts        # 车况校验
    │   │   └── risk.ts           # 风险判定
    │   ├── interaction/           # 确认交互模块
    │   │   ├── confirmHandler.ts # 确认处理
    │   │   └── responseHandler.ts # 响应处理
    │   ├── executor/             # 执行监控模块
    │   │   ├── executor.ts       # 执行器
    │   │   └── logger.ts        # 日志记录
    │   ├── permission/           # 权限管控模块
    │   │   └── roleManager.ts    # 角色管理
    │   ├── engine/               # 规则引擎核心
    │   │   ├── engine.ts         # 引擎主逻辑
    │   │   └── decision.ts       # 决策判定
    │   ├── state/                # 状态管理
    │   │   ├── vehicleState.ts   # 车态管理
    │   │   ├── sessionState.ts   # 会话状态
    │   │   └── roleState.ts     # 角色状态
    │   ├── routes/               # API路由
    │   │   ├── orchestrate.ts    # 编排路由
    │   │   ├── confirm.ts        # 确认路由
    │   │   ├── debug.ts          # 调试路由
    │   │   └── vehicle.ts        # 车态路由
    │   ├── types/                # 类型定义
    │   ├── config/               # 配置文件
    │   │   ├── roles.yaml        # 角色权限配置
    │   │   └── rules.yaml       # 安全规则配置
    │   └── server.ts            # 服务入口
    └── package.json
```

---

## 二、实现计划（前后端并行）

> **核心原则**：接口契约优先，前后端分离并行开发，互不阻塞。

### 阶段0：接口契约定义（P0）

| 步骤 | 交付物 | 输入 | 输出 | 验证方法 | 风险 |
|------|--------|------|------|----------|------|
| 0.1 | API路由定义 | 无 | 所有REST接口签名 | 契约文档Review | 接口设计不合理 |
| 0.2 | 共享类型定义 | 无 | `shared/types.ts` | TypeScript类型检查 | 前后端类型不一致 |
| 0.3 | Mock数据生成 | 类型定义 | mock数据文件 | 前端可独立运行 | Mock与实现不符 |

### 阶段A：后端核心模块（P0）—— 并行开发

#### A.1 基础设施（后端）

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| A.1.1 | `init_backend()` | 无 | Express项目结构 | `cd backend && npm run dev` | Node版本兼容风险 |
| A.1.2 | `setup_types()` | 共享类型 | 后端类型副本 | `npm run type-check` | 类型同步失败 |
| A.1.3 | `config_cors()` | 无 | CORS配置 | 接口联通测试 | 跨域请求失败 |

#### A.2 状态管理层

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| A.2.1 | `VehicleStateManager` | 车态快照 | 内存状态对象 | `POST /api/vehicle` 更新后查询 | 状态同步丢失 |
| A.2.2 | `SessionManager` | 会话ID | 会话上下文 | `POST /api/orchestrate` 创建会话 | 并发会话冲突 |
| A.2.3 | `RoleManager` | role字符串 | 权限配置 | guest/owner切换生效 | 角色切换不生效 |

#### A.3 任务编排引擎

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| A.3.1 | `parseIntent(text)` | 自然语言文本 | 意图结构 | 单元测试「打开车窗」| 模板覆盖不足 |
| A.3.2 | `extractParams(task)` | 意图对象 | 参数列表+模糊标记 | 单元测试「留条缝」| 参数提取错误 |
| A.3.3 | `decomposeTasks(intent)` | 意图结构 | 任务列表 | 单元测试复合指令 | 拆解顺序错误 |
| A.3.4 | `loadTemplates()` | 无 | 模板Map | 模板命中测试 | YAML解析失败 |

#### A.4 安全校验模块

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| A.4.1 | `checkPermission(role, action)` | role+action | boolean | guest访问门锁→false | 权限校验绕过 |
| A.4.2 | `checkVehicleState(state, task)` | 车态+任务 | boolean+原因 | 行驶中开车窗→false | 车况校验遗漏 |
| A.4.3 | `assessRisk(task)` | 任务对象 | riskLevel | 高风险操作识别 | 风险等级误判 |
| A.4.4 | `detectAmbiguity(params)` | 参数列表 | ambiguity列表 | 「留条缝」标记模糊 | 模糊检测失效 |

#### A.5 规则引擎核心

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| A.5.1 | `makeDecision(task, role, state)` | 任务+角色+车态 | decision | 三态输出验证 | 决策逻辑错误 |
| A.5.2 | `executeTask(task)` | 任务对象 | result | 模拟执行成功 | 执行状态不一致 |
| A.5.3 | `handlePartialFailure(tasks, failedIdx)` | 任务列表+失败索引 | 更新后任务列表 | 后续步blocked | 依赖关系处理错误 |
| A.5.4 | `logSession(session)` | 会话对象 | 完整日志 | 日志完整性检查 | 日志缺失 |

### 阶段B：前端核心模块（P0）—— 并行开发

> **独立开发模式**：前端使用Mock数据，不依赖后端服务，可独立完成UI和交互逻辑。

#### B.1 基础设施（前端）

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| B.1.1 | `init_frontend()` | 无 | React项目结构 | `cd frontend && npm run dev` | Vite版本兼容风险 |
| B.1.2 | `setup_types()` | 共享类型 | 前端类型副本 | `npm run type-check` | 类型同步失败 |
| B.1.3 | `setup_mock_api()` | Mock数据 | Mock API服务 | 前端独立运行 | Mock与实现不符 |

#### B.2 主流程UI

| 步骤 | 组件/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| B.2.1 | `CommandInput` | 用户文本 | 提交事件 | 输入「打开车窗」 | 输入验证缺失 |
| B.2.2 | `TaskList` | 任务列表 | 可视化列表 | 展示拆解结果 | 列表渲染错误 |
| B.2.3 | `AmbiguityTag` | 模糊标记 | 高亮展示 | 「留条缝」可见标记 | 标记样式不一致 |

#### B.3 确认交互

| 步骤 | 组件/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| B.3.1 | `ConfirmDialog` | decision+原因 | 确认/取消事件 | 高风险操作弹窗 | 弹窗阻断失效 |
| B.3.2 | `ParameterInput` | ambiguous字段 | 补全参数 | 「留条缝」补参 | 参数格式错误 |
| B.3.3 | `DecisionBadge` | decision | 视觉区分 | 三态颜色区分 | 视觉区分不明显 |

#### B.4 调试面板

| 步骤 | 组件/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| B.4.1 | `RoleSwitcher` | owner/guest | 切换角色 | guest拒绝门锁 | 切换不生效 |
| B.4.2 | `VehicleStateEditor` | 车态字段 | 更新请求 | 改speed后编排生效 | 车态更新丢失 |
| B.4.3 | `StateSnapshot` | 无 | 当前快照展示 | 展示role+车态 | 快照显示错误 |

### 阶段C：联调与闭环（P1）

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| C.1 | `replace_mock_with_real()` | Mock服务 | 真实API调用 | 联调测试 | 接口不兼容 |
| C.2 | `orchestrate(text, role, state)` | 文本+角色+车态 | 编排结果 | 完整闭环测试 | 流程中断 |
| C.3 | `handleConfirm(sessionId, response)` | 会话ID+响应 | 更新结果 | 确认后继续执行 | 确认响应丢失 |
| C.4 | `recheckState(sessionId)` | 会话ID | 重新校验 | 改车态后确认再查 | 再查逻辑错误 |
| C.5 | `finalizeSession(sessionId)` | 会话ID | 终态记录 | 完整日志归档 | 日志不完整 |

### 阶段D：增强与优化（P2）

| 步骤 | 函数/模块名 | 输入 | 输出 | 验证命令 | 风险 |
|------|-------------|------|------|----------|------|
| D.1 | `TimelineView` | 会话对象 | 时间线UI | 逐步结果展示 | 样式不一致 |
| D.2 | `exportLog(sessionId)` | 会话ID | 导出JSON | 日志导出验证 | 导出格式错误 |
| D.3 | `expandTemplates()` | 新意图 | 更新模板 | 模板覆盖扩展 | 新模板冲突 |

---

## 三、风险汇总

### 3.1 高风险（需优先应对）

| 风险等级 | 风险描述 | 影响方案 | 发生概率 | 应对策略 | 责任人 |
|----------|----------|----------|----------|----------|--------|
| 🔴 高 | 规则模板覆盖不足，口语指令无法拆解 | 方案一 | 高 | 建立测试用例集，覆盖主流表达；每周迭代规则库 | 全员 |
| 🔴 高 | 模糊参数被直执（R3违反） | 方案一 | 中 | 规则引擎强制confirm，无参数不得execute | 后端 |
| 🔴 高 | 访客身份被话术推断而非显式role | 全部 | 低 | 规则引擎强制使用前端role，忽略话术身份 | 后端 |

### 3.2 中风险（需监控）

| 风险等级 | 风险描述 | 影响方案 | 发生概率 | 应对策略 | 责任人 |
|----------|----------|----------|----------|----------|--------|
| 🟡 中 | 行驶状态拦截失效 | 全部 | 低 | 车况校验作为安全判定前置条件 | 后端 |
| 🟡 中 | 确认等待中身份变更导致权限绕过 | 全部 | 低 | 再查须使用确认当下的role | 全栈 |
| 🟡 中 | 调试手段被用于绕过安全规则 | 全部 | 低 | 调试改快照后必须触发重新编排 | 全栈 |
| 🟡 中 | 前后端联调复杂度 | 方案一 | 中 | 接口优先开发，契约测试 | 全栈 |

### 3.3 低风险（可接受）

| 风险等级 | 风险描述 | 影响方案 | 发生概率 | 应对策略 | 责任人 |
|----------|----------|----------|----------|----------|--------|
| 🟢 低 | 部分失败时后续步骤状态不一致 | 全部 | 低 | 统一使用顺序依赖策略，失败后后续步blocked | 后端 |
| 🟢 低 | 日志记录不完整，无法追溯 | 全部 | 低 | 统一日志格式，包含所有必填字段 | 后端 |
| 🟢 低 | npm包安全漏洞 | 方案一、二、三 | 中 | 定期`npm audit` + 依赖锁定 | DevOps |

---

## 四、依赖关系

### 4.1 任务依赖图（并行开发模式）

```
════════════════════════════════════════════════════════════
                    并行开发启动线
════════════════════════════════════════════════════════════

         ┌─────────────────────────────────┐
         │ 阶段0：接口契约定义              │
         │                                 │
         │  0.1 API路由定义                │
         │  0.2 共享类型定义               │
         │  0.3 Mock数据生成               │
         └───────────────┬─────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
═════════════════════════   ══════════════════════════════════
阶段A：后端核心模块        阶段B：前端核心模块（Mock模式）
═════════════════════════   ══════════════════════════════════

┌───────────────────┐     ┌───────────────────┐
│ A.1 基础设施(后端) │     │ B.1 基础设施(前端) │
│     │              │     │     │              │
│     ▼              │     │     ▼              │
│ A.2 状态管理层     │     │ B.2 主流程UI       │
│     │              │     │     │              │
│     ▼              │     │     ▼              │
│ A.3 任务编排引擎   │     │ B.3 确认交互       │
│     │              │     │     │              │
│     ▼              │     │     ▼              │
│ A.4 安全校验模块   │     │ B.4 调试面板       │
│     │              │     │     │              │
│     ▼              │     │     │              │
│ A.5 规则引擎核心   │     │ 【前端可独立开发】 │
└─────────┬──────────┘     └───────────────────┘
          │
          ▼
════════════════════════════════════════════════════════════
         前后端联调阶段（阶段C）
════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────┐
│ C.1 替换Mock为真实API                        │
│ C.2 orchestrate() 联调                      │
│ C.3 handleConfirm() 联调                    │
│ C.4 recheckState() 联调                     │
│ C.5 finalizeSession() 联调                  │
└────────────────────┬────────────────────────┘
                     │
════════════════════════════════════════════════════════════

P2 增强优化（可延后）
════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────┐
│ D.1 TimelineView 时间线                     │
│ D.2 exportLog 日志导出                      │
│ D.3 expandTemplates 模板扩展                │
└─────────────────────────────────────────────┘
```

### 4.2 里程碑计划

| 里程碑 | 完成条件 | 对应阶段 | 预计工时 | 验收标准 |
|--------|----------|----------|----------|----------|
| **M0: 契约确立** | 阶段0 完成 | P0 | 2h | 类型定义Review通过，前后端无歧义 |
| **M1: 后端可用** | 阶段A 完成 | P0 | 10h | 后端API全部可用，核心逻辑跑通 |
| **M2: 前端可用** | 阶段B 完成（Mock模式） | P0 | 9h | UI交互完整，可独立演示 |
| **M3: 联调完成** | 阶段C 完成 | P1 | 4h | 真实API联调通过，闭环演示成功 |
| **M4: 增强优化** | 阶段D 完成 | P2 | 3h | 体验优化，模板扩展 |

### 4.3 验收检查清单

| 检查项 | 验收条件 | 验证方法 |
|--------|----------|----------|
| R1: 指令拆解 | 提交「打开车窗」后UI出现tasks列表 | 功能测试 |
| R2: 三态判定 | 每步有decision+reason，含两类以上 | 样例测试 |
| R3: 模糊不直执 | 「留条缝」无confirm前无success | 边界测试 |
| R4: 权限拒绝 | guest+门锁→reject，日志含role=guest | 安全测试 |
| R5: 确认交互 | confirm态有确认/取消，取消后无success | 交互测试 |
| R6: 执行前再查 | 改车态后编排结果变化 | 调试测试 |
| R7: 部分失败 | 失败后后续步blocked | 依赖测试 |
| R8: 可追溯 | 可展开完整日志 | 日志测试 |
| R9: 调试能力 | 切换身份/车态后下次编排生效 | 调试测试 |
| R10: 一正一反 | 至少一次直执+一次直拒 | Demo测试 |

---

## 五、接口设计

### 5.1 API 路由总览

| 方法 | 路径 | 描述 | 输入 | 输出 |
|------|------|------|------|------|
| POST | `/api/orchestrate` | 提交编排请求 | `{text, role, vehicleState}` | `{sessionId, tasks, decisions}` |
| POST | `/api/confirm` | 处理确认响应 | `{sessionId, taskId, action, params?}` | `{result, updatedTasks}` |
| POST | `/api/cancel` | 取消会话 | `{sessionId}` | `{cancelled, session}` |
| GET | `/api/session/:id` | 获取会话详情 | sessionId | `{session, tasks, logs}` |
| GET | `/api/state` | 获取当前状态 | - | `{vehicleState, role}` |
| POST | `/api/vehicle` | 更新车态 | `VehicleState` | `{updated}` |
| POST | `/api/role` | 切换角色 | `{role}` | `{updated}` |

### 5.2 错误码设计

#### 5.2.1 错误响应格式

所有接口的错误响应统一使用以下 JSON 结构：

```json
{
  "error": "错误描述信息（供开发者/调试用）",
  "code": "错误码",
  "field": "触发错误的字段名（可选）"
}
```

#### 5.2.2 错误码分类

| 错误码 | 说明 | HTTP状态码 | 触发场景 | 示例 |
|--------|------|------------|----------|------|
| **安全规则类 (SEC_)** | | | | |
| `SEC_VEHICLE_MOVING` | 车辆处于行驶状态，禁止该操作 | 403 | 行驶中(D档或speed>0)执行解锁/开车窗等 | `{"error":"行驶状态下禁止执行门锁操作","code":"SEC_VEHICLE_MOVING","field":"gear"}` |
| `SEC_FAULT_ACTIVE` | 车辆故障中，禁止敏感操作 | 403 | fault=true时执行空调调节等 | `{"error":"车辆故障中，禁止执行操作","code":"SEC_FAULT_ACTIVE","field":"fault"}` |
| `SEC_SPEED_TOO_HIGH` | 车速过高，禁止该操作 | 403 | speed>120时执行开窗等 | `{"error":"当前车速过高，请先减速","code":"SEC_SPEED_TOO_HIGH","field":"speedKmh"}` |
| `SEC_RISK_LEVEL_HIGH` | 操作风险等级过高 | 403 | 高风险操作未经确认 | `{"error":"该操作风险较高，需要确认","code":"SEC_RISK_LEVEL_HIGH"}` |
| **权限规则类 (PER_)** | | | | |
| `PER_GUEST_FORBIDDEN` | 访客身份无权限执行该操作 | 403 | guest角色执行门锁/授权相关操作 | `{"error":"访客身份无权执行门锁操作","code":"PER_GUEST_FORBIDDEN","field":"role"}` |
| `PER_ROLE_REQUIRED` | 缺少必需的角色标识 | 400 | role字段为空或非法 | `{"error":"role字段缺失或无效","code":"PER_ROLE_REQUIRED","field":"role"}` |
| `PER_CONFIRM_REQUIRED` | 操作需要先确认 | 400 | 执行需confirm的步骤 | `{"error":"该操作需要先确认","code":"PER_CONFIRM_REQUIRED"}` |
| **车态不匹配类 (STA_)** | | | | |
| `STA_GEAR_INVALID` | 挡位状态不允许该操作 | 400 | N档执行行驶相关操作 | `{"error":"当前挡位不允许该操作","code":"STA_GEAR_INVALID","field":"gear"}` |
| `STA_BATTERY_LOW` | 电量过低，禁止该操作 | 400 | batteryPct<10时执行高耗电操作 | `{"error":"电量过低，无法执行该操作","code":"STA_BATTERY_LOW","field":"batteryPct"}` |
| `STA_CONFLICT` | 任务之间存在冲突 | 400 | 同时执行矛盾的操作（如开窗和关窗） | `{"error":"任务操作存在冲突","code":"STA_CONFLICT"}` |
| `STA_AMBIGUITY` | 指令存在歧义需要澄清 | 400 | 模糊参数未补全 | `{"error":"指令存在歧义，请补充参数","code":"STA_AMBIGUITY","field":"params"}` |
| **会话状态类 (SES_)** | | | | |
| `SES_ALREADY_AWAITING` | 已有待确认会话，无法新建 | 409 | 存在awaiting_confirm状态会话时再次提交 | `{"error":"存在待确认的会话，请先处理","code":"SES_ALREADY_AWAITING"}` |
| `SES_NOT_FOUND` | 会话不存在 | 404 | 访问不存在的sessionId | `{"error":"会话不存在","code":"SES_NOT_FOUND","field":"sessionId"}` |
| `SES_STATUS_INVALID` | 会话状态不允许该操作 | 400 | 已完成的会话再次confirm | `{"error":"会话已完成，无法执行该操作","code":"SES_STATUS_INVALID"}` |
| `SES_ALREADY_CANCELLED` | 会话已取消 | 400 | 取消已取消的会话 | `{"error":"会话已取消","code":"SES_ALREADY_CANCELLED"}` |
| **系统错误类 (SYS_)** | | | | |
| `SYS_TIMEOUT` | 请求处理超时 | 504 | 服务端处理超时 | `{"error":"处理超时，请重试","code":"SYS_TIMEOUT"}` |
| `SYS_INTERNAL` | 系统内部错误 | 500 | 未预期的异常 | `{"error":"系统内部错误","code":"SYS_INTERNAL"}` |
| `SYS_UNAVAILABLE` | 服务不可用 | 503 | 后端服务未启动 | `{"error":"服务暂不可用","code":"SYS_UNAVAILABLE"}` |
| **输入验证类 (VAL_)** | | | | |
| `VAL_TEXT_EMPTY` | 指令文本为空 | 400 | text字段为空 | `{"error":"指令文本不能为空","code":"VAL_TEXT_EMPTY","field":"text"}` |
| `VAL_TEXT_TOO_LONG` | 指令文本过长 | 400 | text长度超过200字符 | `{"error":"指令文本过长，最大200字符","code":"VAL_TEXT_TOO_LONG","field":"text"}` |
| `VAL_FIELD_MISSING` | 必需字段缺失 | 400 | 缺少必需字段 | `{"error":"缺少必需字段","code":"VAL_FIELD_MISSING","field":"xxx"}` |
| `VAL_FIELD_TYPE` | 字段类型错误 | 400 | 字段类型不匹配 | `{"error":"字段类型错误","code":"VAL_FIELD_TYPE","field":"xxx"}` |
| `VAL_FIELD_RANGE` | 字段值超出范围 | 400 | speedKmh不在0-200范围 | `{"error":"字段值超出允许范围","code":"VAL_FIELD_RANGE","field":"xxx"}` |

#### 5.2.3 错误码使用规则

1. **安全规则类 (SEC_*)**: 返回 HTTP 403，禁止操作
2. **权限规则类 (PER_*)**: 返回 HTTP 403（安全原因）或 400（格式问题）
3. **车态不匹配类 (STA_*)**: 返回 HTTP 400，附带 `field` 指出问题字段
4. **会话状态类 (SES_*)**: 
   - `SES_NOT_FOUND`: HTTP 404
   - `SES_ALREADY_AWAITING`: HTTP 409（冲突）
   - 其他: HTTP 400
5. **系统错误类 (SYS_*)**: 
   - `SYS_TIMEOUT`: HTTP 504
   - `SYS_INTERNAL`: HTTP 500
   - `SYS_UNAVAILABLE`: HTTP 503
6. **输入验证类 (VAL_*)**: 返回 HTTP 400，附带 `field` 指出问题字段

#### 5.2.4 前端错误处理建议

```typescript
// 错误码到用户提示的映射
const errorMessages: Record<string, string> = {
  'SEC_VEHICLE_MOVING': '车辆正在行驶，请先停车',
  'SEC_FAULT_ACTIVE': '车辆故障中，请先处理',
  'SEC_SPEED_TOO_HIGH': '车速过快，请先减速',
  'SEC_RISK_LEVEL_HIGH': '该操作需要确认',
  'PER_GUEST_FORBIDDEN': '访客身份无权执行该操作',
  'PER_ROLE_REQUIRED': '请选择您的身份',
  'PER_CONFIRM_REQUIRED': '请先确认操作',
  'STA_GEAR_INVALID': '当前挡位不支持该操作',
  'STA_BATTERY_LOW': '电量不足，无法执行该操作',
  'STA_CONFLICT': '操作存在冲突，请重新描述',
  'STA_AMBIGUITY': '请明确操作参数',
  'SES_ALREADY_AWAITING': '已有待确认的操作，请先处理',
  'SES_NOT_FOUND': '会话不存在',
  'SES_STATUS_INVALID': '会话状态不支持该操作',
  'SYS_TIMEOUT': '网络超时，请重试',
  'SYS_INTERNAL': '系统异常，请稍后重试',
  'SYS_UNAVAILABLE': '服务暂时不可用',
  'VAL_TEXT_EMPTY': '请输入指令',
  'VAL_TEXT_TOO_LONG': '指令过长，请精简',
  'VAL_FIELD_MISSING': '信息不完整',
  'VAL_FIELD_TYPE': '信息格式错误',
  'VAL_FIELD_RANGE': '数值超出范围',
};
```

### 5.3 核心数据类型

```typescript
// 角色
type Role = 'owner' | 'guest';

// 车态
interface VehicleState {
  speedKmh: number;      // 0-200
  gear: 'P' | 'R' | 'N' | 'D';
  batteryPct: number;    // 0-100
  fault: boolean;
}

// 任务
interface Task {
  id: string;
  object: string;        // 车窗/空调/门锁等
  action: string;        // 打开/关闭/调节等
  params: Record<string, any>;
  ambiguity: string[];   // 模糊参数列表
  dependencies: string[]; // 依赖任务ID
}

// 判定
type Decision = 'execute' | 'confirm' | 'reject';
type Result = 'success' | 'blocked' | 'failed' | 'cancelled';

// 会话
interface Session {
  id: string;
  text: string;
  role: Role;
  vehicleState: VehicleState;
  tasks: Task[];
  decisions: Map<string, Decision>;
  results: Map<string, Result>;
  logs: LogEntry[];
  createdAt: Date;
  updatedAt: Date;
}
```

---

## 六、环境配置

### 6.1 前端环境

```bash
# 端口：5173（默认Vite）
# 环境变量：.env.local

VITE_API_BASE=http://localhost:3000
```

### 6.2 后端环境

```bash
# 端口：3000
# 环境变量：.env

PORT=3000
NODE_ENV=development
LOG_LEVEL=info
```

### 6.3 启动命令

```bash
# 开发模式（并行启动）
cd SRC/frontend && npm run dev  # 端口5173
cd SRC/backend && npm run dev    # 端口3000

# 验证命令
curl http://localhost:3000/api/state  # 检查后端状态
open http://localhost:5173             # 打开前端
```

---

## 修订记录

| 版本 | 日期 | 说明 |
|------|------|------|
| v0.1 | 2026-07-16 | 初始版本：基于方案一（React+Express规则模板）的完整实现计划 |
| v0.2 | 2026-07-16 | 重构实现计划：接口契约优先，前后端并行开发模式 |

---

**文档状态**：草稿
**下一步**：进入SRC/目录开始实现阶段0（接口契约定义）
