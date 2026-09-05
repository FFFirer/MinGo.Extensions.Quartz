# MinGo.Extensions.Quartz — 多仓实现规划总览

基于 Quartz.NET 的**功能增强 + 统一管理 UI**，按四层拆分代码仓库，单向依赖。

## 仓库结构

| 目录（仓库） | 层 | 职责 | 主要产物 |
|---|---|---|---|
| `MinGo.Quartz/` | L1 | Quartz.NET 基础补强 + 开放可观测性 | `MinGo.Quartz`、`MinGo.Quartz.OpenTelemetry`（NuGet） |
| `MinGo.Quartz.Agent/` | L2 | Agent SDK + Shared 抽象 | `MinGo.Quartz.Agent.Abstractions`、`MinGo.Quartz.Agent`（NuGet） |
| `MinGo.Quartz.Platform/` | L3 | 平台后端（前后端分离之后端） | `MinGo.Quartz.Platform`（ASP.NET Core Web API） |
| `MinGo.Quartz.Platform.UI/` | L4 | 可视化 UI | React+Vite+TS 前端（npm 包） |
| `MinGo.Quartz.Deploy/` | 编排 | 部署/CI/示例宿主/统一版本 | docker-compose、CI 脚本、samples |

## 依赖方向

```
L4  MinGo.Quartz.Platform.UI
        ▲ OpenAPI 生成 TS 类型 / REST + SSE
L3  MinGo.Quartz.Platform
        ▲ 引用 L2 Abstractions（传递引用 L1 观测模型）
L2  MinGo.Quartz.Agent
        ▲ 引用 L1（观测模型 + 埋点）
L1  MinGo.Quartz
        ▲ 仅依赖 Quartz.NET + OpenTelemetry.Api
L0  Quartz.NET（第三方，不建仓库）
```

禁止反向依赖；L3 不依赖 L2 SDK 实现、不依赖 L1 埋点实现，只复用契约与观测 DTO。

## 命名约定

- 仓库名/包名/命名空间统一 `MinGo.Quartz.*`（L2 内部 `MinGo.Quartz.Agent.*`，L3 `MinGo.Quartz.Platform.*`）。
- 目标框架：L1/L2 包 `net8.0`（最大化复用，如需可多目标 `net8.0;net10.0`）；L3 `net10.0`；L4 Node 20+。

## 实现顺序（阶段门禁）

1. **阶段 0 — 骨架**：各仓补齐 slnx / Directory.Build.props / .gitignore / CI。
2. **阶段 1 — L1**：观测模型 + 读取器 + OpenTelemetry 埋点（独立可发布）。
3. **阶段 2 — L2**：Abstractions 先行 → Agent SDK 迁移（组合 L1）。
4. **阶段 3 — L3**：Platform 后端迁移 + ExecutionLog 持久化 + SSE。
5. **阶段 4 — L4**：UI 迁移 + OpenAPI 类型生成。
6. **阶段 5 — Deploy**：docker-compose、CI/CD、契约测试、版本矩阵。

每阶段完成后跑通本层测试 + 下游契约测试，再进入下一层。

## 实现状态

| 仓库 | 状态 |
|---|---|
| `MinGo.Quartz` (L1) | ✅ **已实现并验证**：核心库 + OpenTelemetry 集成 + 12 项测试全绿 + `MinGo.Quartz` 1.0.0 / `MinGo.Quartz.OpenTelemetry` 1.0.0 已打包（见该仓 `PLAN.md` §7） |
| `MinGo.Quartz.Agent` (L2) | ✅ **已实现并验证**：Abstractions 契约包 + Agent SDK 迁移并组合 L1 观测（自动挂监听器）；M3 补强（Misfire 闭环/并发语义化/Trigger 级操作）；51/51 测试全绿；Sample.Agent ↔ PlatformStub 注册/心跳/上报 E2E 通过；`MinGo.Quartz.Agent.Abstractions` 1.0.0 / `MinGo.Quartz.Agent` 1.0.0 已打包（见该仓 `PLAN.md` §6） |
| `MinGo.Quartz.Platform` (L3) | ⏳ 待实现 |
| `MinGo.Quartz.Platform.UI` (L4) | ⏳ 待实现 |
| `MinGo.Quartz.Deploy` | ⏳ 待实现 |

## 版本与发布

- NuGet（L1/L2/L3）：每包独立语义化版本；L1 `1.0` 先行，L2 引用 L1，L3 引用 L2。
- npm（L4）：独立版本，随 L3 OpenAPI 契约联调。
- Docker：L3 + L4 静态资源合并单镜像；L2 SDK 由业务宿主自行打包。

## 现有资产迁移来源

| 来源 | 去向 |
|---|---|
| `MinGo.Qap.Shared` | → `MinGo.Quartz.Agent.Abstractions` |
| `MinGo.Qap.Agent` | → `MinGo.Quartz.Agent` |
| `MinGo.Qap.Platform` | → `MinGo.Quartz.Platform` |
| `MinGo.Qap.UI` | → `MinGo.Quartz.Platform.UI` |
| `MinGo.Quartz`（旧） | → L1 观测模型/读取器；EF Core 抽象 + AppAny 表映射 → L1 可选包 `MinGo.Quartz.EFCore.*`；`Result`/`Paged` → L2 Abstractions |
| `MinGo.Quartz.Extensions`/`MinGo.QuartzStartup`（空壳） | 溶解，由本目录承接 |

各层详细规划见各自 `PLAN.md`。
