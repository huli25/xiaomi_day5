# Workflow · 开发工作流与接口设计（S1）

> **交付链第 4 棒** · 上游：`docs/options/options.md`（团队已确认 **方案一 S1**）  
> 版本：v1.0 · 2026-07-16  
> 技术锁定：React 18 + Vite + TS / Node.js + Express + TS · 规则模板拆解 · 服务端规则准出 · 内存车态  
> 每项任务含：**做什么 / 输出什么 / 怎么验证**。按功能流水线拆分，前后端分开列出。

**Delivery progress**

```text
- [x] 1 Flow-map
- [x] 2 PRD
- [x] 3 Design（四方案）
- [x] Options（确认 S1）
- [x] 4 Workflow（本文件）
- [ ] 5 Implement
- [ ] 6 Tests
- [ ] 7 Gates / reflection
```

---

## 1. 流水线总览（谁先谁后）

```text
① 共享类型与契约（前后端对齐本文件 §4）
        ↓
② 后端：车态 Store → 拆解 → 规则 → 编排 → 执行 → 路由
        ↓  （可用 curl/Apifox 先测通）
③ 前端：API Client → 调试/身份 → 提交编排 → 拆解&三态展示 → 确认 → 时间线
        ↓
④ 联调 Demo：C3 正例 / C1 模糊 / C2 访客拒绝 / H2 再查
```

| 阶段 | 后端 | 前端 | 联调门槛 |
| --- | --- | --- | --- |
| M1 车态+调试 | BE-T1 | FE-T1, FE-T2 | GET/PUT vehicle 通 |
| M2 拆解+规则+编排 | BE-T2～T5 | — | POST sessions 返回 tasks+steps |
| M3 确认+再查+执行 | BE-T6, BE-T7 | FE-T3～T5 | confirm 后再查生效 |
| M4 时间线+一正一反 | BE-T8 | FE-T6, FE-T7 | R1–R10 可演示 |

---

## 2. 目录约定（S1）

```text
SRC/
  frontend/          # 前端同学负责
    package.json
    src/
      api/
      components/
      types.ts       # 可与后端共享拷贝或后续抽 packages/shared
      App.tsx
  backend/           # 后端同学负责
    package.json
    src/
      index.ts
      types.ts
      store/
      services/
      routes/
      data/templates.json
```

**职责红线：**
- 前端：**不得**本地计算或改写 `decision` / 伪造 `success`
- 后端：拆解服务 **不得** 输出 `decision`；唯有规则引擎给出三态

---

## 3. 后端模块设计与任务

### 3.1 模块一览

| 模块 | 路径建议 | 职责 |
| --- | --- | --- |
| 类型 | `src/types.ts` | Role / VehicleState / Task / Step / Session / Event |
| 车态 Store | `src/store/vehicle.ts` | 权威车态读写与校验（I5） |
| 会话 Store | `src/store/sessions.ts` | session Map；单活跃 awaiting（I3） |
| 拆解 | `src/services/disassemble.ts` + `data/templates.json` | 文本→tasks（含 ambiguity、dependsOn）；**无 decision** |
| 规则引擎 | `src/services/rules.ts` | 权限→车况→模糊/冲突→`execute\|confirm\|reject` |
| 编排 | `src/services/orchestrate.ts` | 建会话、逐步判定、状态机 |
| 执行 | `src/services/execute.ts` | 模拟执行、写回、部分失败 blocked（I7）、再查 |
| 路由 | `src/routes/*.ts` | 对外 HTTP |

### 3.2 后端任务清单

#### BE-T1 · 车态 Store + 校验

| 项 | 内容 |
| --- | --- |
| 做什么 | 实现内存车态；校验 speedKmh 0–200、gear∈{P,R,N,D}、batteryPct 0–100、fault bool；越界拒绝 |
| 输出什么 | `getVehicle()` / `setVehicle(state)`；非法抛错或返回错误码 |
| 怎么验证 | 单测或脚本：合法写入成功；speed=999 / gear=X → 失败 |

#### BE-T2 · 拆解（规则模板）

| 项 | 内容 |
| --- | --- |
| 做什么 | 关键词/正则匹配：留条缝→window+ambiguity；空调到 N 度→climate；解锁/授权→door/auth；多目标拆多步；未命中→unknown |
| 输出什么 | `Task[]`（id/target/action/params/ambiguity/dependsOn?）；**禁止**带 decision |
| 怎么验证 | 「主驾留条缝」有 ambiguity；「空调开到24度」无 ambiguity；复合句 tasks.length≥2；tasks≤8，超出由编排拒绝（I2） |

#### BE-T3 · 权限规则

| 项 | 内容 |
| --- | --- |
| 做什么 | `guest` + unlock/auth.share → reject；unknown → reject；不根据话术推断身份 |
| 输出什么 | `{ decision, reason }` |
| 怎么验证 | role=guest + 解锁 → reject；role=owner 同句不因权限 reject（仍可因风险 confirm） |

#### BE-T4 · 车况 / 模糊 / 冲突规则

| 项 | 内容 |
| --- | --- |
| 做什么 | 行驶(D 或 speed>0)：禁解锁、禁车窗全开；ambiguity 非空→**必须 confirm**（清掉不可信精确开度）；冲突动作标 reason |
| 输出什么 | decision + reason |
| 怎么验证 | 留条缝→confirm 且不可 success；行驶+全开窗→reject |

#### BE-T5 · 编排建会话

| 项 | 内容 |
| --- | --- |
| 做什么 | 校验 text 1–200、role 枚举；读车态快照；拆解+逐步 rules；若仅有 execute/reject 可立即执行可执行步；有 confirm→`awaiting_confirm`；已有 awaiting→409 |
| 输出什么 | 完整 `Session`（tasks/steps/events/status） |
| 怎么验证 | POST /api/sessions 返回结构完整；空文本 400；第二活跃会话 409 |

#### BE-T6 · 确认 / 取消 + 再查

| 项 | 内容 |
| --- | --- |
| 做什么 | confirm：合并 params → 用请求 **role + 最新车态** 再跑 rules → 通过则 execute；cancel→cancelled；再查失败→blocked |
| 输出什么 | 更新后 Session；events 含 rechecked |
| 怎么验证 | H2：confirm 前改 guest → blocked；取消后无 success |

#### BE-T7 · 模拟执行 + 部分失败

| 项 | 内容 |
| --- | --- |
| 做什么 | execute 步写回模拟字段；任一步 failed/blocked 后后续全部 blocked（顺序依赖 I7） |
| 输出什么 | step.result；更新 vehicle |
| 怎么验证 | 两步任务中第一步失败 → 第二步 blocked |

#### BE-T8 · HTTP 路由挂载

| 项 | 内容 |
| --- | --- |
| 做什么 | 实现 §4 全部 API；统一错误 JSON；CORS 对前端端口开放 |
| 输出什么 | 可 `npm run dev` 的后端服务（如 :3001） |
| 怎么验证 | curl/Apifox 跑通 §5 联调脚本 |

---

## 4. 前端模块设计与任务

### 4.1 模块一览

| 模块 | 路径建议 | 职责 |
| --- | --- | --- |
| 类型 | `src/types.ts` | 与后端一致的 DTO |
| API Client | `src/api/client.ts` | 封装 §4 五个接口 |
| 身份 | `components/RoleSwitcher` | owner/guest 显式选择 |
| 指令 | `components/CommandInput` | 输入与长度提示 |
| 调试 | `components/DebugPanel` | 车速/挡位/电量/故障 → PUT vehicle |
| 拆解展示 | `components/TaskList` | tasks + ambiguity 标记 |
| 判定展示 | `components/StepDecisionList` | 三色三态 + reason |
| 确认 | `components/ConfirmPanel` | 确认/取消/补参 |
| 时间线 | `components/Timeline` | 全链路事件 |
| 页面壳 | `App.tsx` | 布局与 UI 状态机 |

### 4.2 前端任务清单

#### FE-T1 · 工程脚手架 + API Client

| 项 | 内容 |
| --- | --- |
| 做什么 | Vite React TS 工程；`client.ts` 封装 vehicle/sessions API；错误提示 |
| 输出什么 | 可启动前端（如 :5173）；API 指向后端 baseURL |
| 怎么验证 | 调用 GET vehicle 能拿到 JSON（后端 M1 就绪后） |

#### FE-T2 · 身份切换 + 调试面板

| 项 | 内容 |
| --- | --- |
| 做什么 | RoleSwitcher；DebugPanel 表单写入车态；展示当前车态 |
| 输出什么 | 可切换 owner/guest；可改四车态字段 |
| 怎么验证 | R9：改车态后刷新/再读一致；不根据话术改 role（R4a） |

#### FE-T3 · 指令提交 + 拆解展示

| 项 | 内容 |
| --- | --- |
| 做什么 | 提交 `{text, role}`；展示 TaskList（含 ambiguity）；空/超长前端拦截（I1） |
| 输出什么 | 提交后可见结构化拆解 |
| 怎么验证 | R1：留条缝可见模糊标记 |

#### FE-T4 · 三态判定展示

| 项 | 内容 |
| --- | --- |
| 做什么 | 按 steps 渲染 execute/confirm/reject 色标与 reason；禁止全部标成待确认 |
| 输出什么 | StepDecisionList |
| 怎么验证 | R2/R10：正例+反例同 Demo 出现不同色态 |

#### FE-T5 · 确认交互

| 项 | 内容 |
| --- | --- |
| 做什么 | ConfirmPanel：展示原因与将执行内容；确认带当前 role+可选 params；取消 |
| 输出什么 | 调用 POST confirm；刷新 session |
| 怎么验证 | R3/R5：确认前无 success；取消无 success |

#### FE-T6 · 时间线

| 项 | 内容 |
| --- | --- |
| 做什么 | 展示 events：原文、拆解、判定、确认、再查、结果、role、车态 |
| 输出什么 | Timeline 组件 |
| 怎么验证 | R8：可指认「模糊未直执 / 访客拒绝 / 再查阻止」 |

#### FE-T7 · UI 状态机与错误态

| 项 | 内容 |
| --- | --- |
| 做什么 | idle/loading/planned/awaiting_confirm/done/error；处理 409/400 文案 |
| 输出什么 | App 级状态切换 |
| 怎么验证 | 待确认时再提交编排有提示（I3） |

---

## 5. 前后端接口设计

**约定：**
- Base URL 开发：`http://localhost:3001`（后端）、前端 Vite proxy 可选
- `Content-Type: application/json`
- 成功：2xx + JSON body
- 失败：`{ "error": string, "code"?: string, "field"?: string }`

### 5.1 共享数据类型

```ts
type Role = 'owner' | 'guest'
type Decision = 'execute' | 'confirm' | 'reject'
type StepResult = 'pending' | 'success' | 'blocked' | 'failed' | 'cancelled'
type SessionStatus =
  | 'planned'
  | 'awaiting_confirm'
  | 'completed'
  | 'rejected'
  | 'cancelled'

interface VehicleState {
  speedKmh: number      // 0..200
  gear: 'P' | 'R' | 'N' | 'D'
  batteryPct: number    // 0..100
  fault: boolean
  windowPercent?: number
  climateTemp?: number
}

interface Task {
  id: string
  target: 'window' | 'climate' | 'door' | 'auth' | 'unknown'
  action: string
  params: Record<string, unknown>
  ambiguity: string[]
  dependsOn?: string[]
}

interface Step {
  id: string
  taskId: string
  decision: Decision
  reason: string
  result: StepResult
  recheck?: {
    at: string
    role: Role
    ok: boolean
    reason?: string
  }
}

interface TimelineEvent {
  at: string // ISO
  type:
    | 'input'
    | 'disassembled'
    | 'decided'
    | 'confirm_requested'
    | 'user_confirmed'
    | 'user_cancelled'
    | 'rechecked'
    | 'executed'
    | 'vehicle_updated'
  payload: unknown
}

interface Session {
  id: string
  text: string
  roleAtStart: Role
  vehicleSnapshot: VehicleState
  tasks: Task[]
  steps: Step[]
  status: SessionStatus
  events: TimelineEvent[]
}
```

### 5.2 API 一览

| # | 方法 | 路径 | 调用方 | 说明 |
| --- | --- | --- | --- | --- |
| A1 | GET | `/api/vehicle` | 前端调试区 | 读当前车态 |
| A2 | PUT | `/api/vehicle` | 前端调试区 | 写车态（全量四字段） |
| A3 | POST | `/api/sessions` | 前端提交编排 | 创建会话并返回拆解+判定 |
| A4 | POST | `/api/sessions/:id/confirm` | 前端确认面板 | 确认/取消 + 再查 |
| A5 | GET | `/api/sessions/:id` | 前端刷新/时间线 | 拉全量会话 |

---

### A1 `GET /api/vehicle`

**响应 200**

```json
{
  "speedKmh": 0,
  "gear": "P",
  "batteryPct": 80,
  "fault": false
}
```

---

### A2 `PUT /api/vehicle`

**请求**

```json
{
  "speedKmh": 60,
  "gear": "D",
  "batteryPct": 80,
  "fault": false
}
```

| 状态 | 含义 |
| --- | --- |
| 200 | 返回更新后车态 |
| 400 | 字段越界/类型错误，`field` 指出字段 |

---

### A3 `POST /api/sessions`

**请求**

```json
{
  "text": "主驾留条缝",
  "role": "owner"
}
```

**响应 201：** 完整 `Session`（含 tasks、steps、events、status）

| 状态 | 含义 |
| --- | --- |
| 201 | 创建成功 |
| 400 | text 空/超长、role 非法 |
| 409 | 已存在 `awaiting_confirm` 会话 |

**后端行为摘要：** 读车态快照 → 拆解 → 逐步 rules → 无 confirm 则可立即模拟执行可执行步 → 写 events。

---

### A4 `POST /api/sessions/:id/confirm`

**请求**

```json
{
  "stepId": "step-1",
  "action": "confirm",
  "role": "owner",
  "params": { "percent": 15 }
}
```

| 字段 | 说明 |
| --- | --- |
| action | `confirm` \| `cancel` |
| role | **确认当下**身份（再查用此值，I10） |
| params | 可选补参，合并进对应 task |

**响应 200：** 更新后 `Session`

| 状态 | 含义 |
| --- | --- |
| 200 | 成功 |
| 400 | 参数非法 / 会话状态不允许确认 |
| 404 | session 不存在 |

**后端行为摘要：** cancel→cancelled；confirm→再查(role+最新车态)→execute 或 blocked→I7 传播。

---

### A5 `GET /api/sessions/:id`

**响应 200：** 完整 `Session`  
**404：** 不存在

---

## 6. 联调验收脚本（对齐 PRD）

| 步骤 | 操作 | 期望 | 对应 |
| --- | --- | --- | --- |
| 1 | PUT vehicle 静止；role=owner；POST「空调开到24度」 | execute→success | C3, R10 正 |
| 2 | POST「主驾留条缝」 | confirm；确认前无 success | C1, R3 |
| 3 | role=guest；POST「解锁车门」 | reject；日志 role=guest | C2, R4 |
| 4 | owner 触发需 confirm；改 guest；再 confirm | blocked + recheck | H2, R6 |
| 5 | 复合两步且第一步失败 | 后续 blocked | R7 |
| 6 | 打开 Timeline | 可回看全文链路 | R8 |

---

## 7. 任务依赖与并行建议

```text
BE-T1 ──► BE-T2 ──► BE-T3/T4 ──► BE-T5 ──► BE-T6/T7 ──► BE-T8
                │
FE-T1 ──────────┴──（等 A1/A2）► FE-T2
FE-T1 ──（等 A3）► FE-T3 → FE-T4 → FE-T5（等 A4）→ FE-T6 → FE-T7
```

- 后端可先独立用 curl 做完 M2  
- 前端 FE-T1/T2 可与 BE-T1 并行  
- **禁止**前端用 mock decision 冒充联调通过

---

## 8. 交给下游

| 去向 | 内容 |
| --- | --- |
| 实现 | 按 BE-T* / FE-T* 开工；接口以 §5 为准 |
| 测试 | 用 §6 脚本扩成 validation 用例 |
| decision | 若改 API 字段，先改本文件再改代码 |

---

## 9. 修订记录

| 版本 | 日期 | 说明 |
| --- | --- | --- |
| v1.0 | 2026-07-16 | 基于 options 确认的 S1；前后端模块+任务；完整 REST 契约 |
