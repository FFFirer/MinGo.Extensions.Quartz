# MinGo.Extensions.Quartz — 多仓实现规划总览

基于 Quartz.NET 的**功能增强 + 统一管理 UI**，按三仓组织代码，单向依赖。

## 仓库结构

| 目录（仓库） | 层 | 职责 | 主要产物 |
|---|---|---|---|
| `MinGo.Quartz.SDK/` | L1 + L2 | Quartz.NET 基础补强 + 开放可观测性 + Agent SDK + Shared 抽象 | `MinGo.Quartz`、`MinGo.Quartz.OpenTelemetry`、`MinGo.Quartz.Agent.Abstractions`、`MinGo.Quartz.Agent`（NuGet） |
| `MinGo.Quartz.Platform/` | L3 + L4 | 平台后端（ASP.NET Core Web API）+ 可视化 UI（React+Vite+TS，位于 `ui/` 子目录） | `MinGo.Quartz.Platform`（Web API）+ 前端静态资源 |
| `MinGo.Quartz.Deploy/` | 编排 | 部署/CI/示例宿主/统一版本 | docker-compose、CI 脚本、samples |

> **仓库合并说明**（2026-09）：原 5 仓（L1 `MinGo.Quartz`、L2 `MinGo.Quartz.Agent`、L3 `MinGo.Quartz.Platform`、L4 `MinGo.Quartz.Platform.UI`、Deploy）重组为 3 仓。L1+L2 合并为 `MinGo.Quartz.SDK`（扁平结构，所有包同级）；L3+L4 合并为 `MinGo.Quartz.Platform`（UI 作为 `ui/` 子目录）。原 4 个远程仓保留作为归档，不再更新。

## 依赖方向

```
L4  MinGo.Quartz.Platform/ui/          (React + Vite + TS)
        ▲ OpenAPI 生成 TS 类型 / REST + SSE
L3  MinGo.Quartz.Platform/src/         (ASP.NET Core Web API)
        ▲ NuGet 引用 L2 Abstractions（传递引用 L1 观测模型）
L2  MinGo.Quartz.SDK/src/MinGo.Quartz.Agent*   (Agent SDK + Abstractions)
        ▲ 同仓 ProjectReference L1（观测模型 + 埋点）
L1  MinGo.Quartz.SDK/src/MinGo.Quartz*         (核心 + OpenTelemetry)
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
| `MinGo.Quartz.SDK` (L1 + L2) | ✅ **已实现并验证**：合并原 L1（`MinGo.Quartz` 核心库 + OpenTelemetry 集成，12 项测试）与 L2（Abstractions 契约包 + Agent SDK 组合 L1 观测，51/51 测试）；扁平结构，4 个 NuGet 包（`MinGo.Quartz`、`MinGo.Quartz.OpenTelemetry`、`MinGo.Quartz.Agent.Abstractions`、`MinGo.Quartz.Agent` 1.0.0）已打包；SDK 仓 63/63 测试全绿（见该仓 `PLAN.md`） |
| `MinGo.Quartz.Platform` (L3 + L4) | ✅ **已实现并验证**：合并原 L3（ASP.NET Core Web API + EF Core/PostgreSQL，Agent 注册/心跳/注销、Scheduler 管理、Job CRUD+代理、ExecutionLog 持久化、SSE Activity Feed、批量操作、Token 鉴权、NSwag OpenAPI，29/29 测试）与 L4（React 19 + Vite 8 + TS 6 + Tailwind CSS 3 + TanStack Query/Table，22 组件 + 9 页面，UI 作为 `ui/` 子目录，构建 0 错误）（见该仓 `PLAN.md`） |
| `MinGo.Quartz.Deploy` | ⏳ 待实现 |

## 版本与发布

- NuGet（SDK 仓 L1/L2 + Platform 仓 L3）：每包独立语义化版本；L1 `1.0` 先行，L2 引用 L1（SDK 仓内 ProjectReference），L3 引用 L2 Abstractions（跨仓 NuGet，本地源 `../MinGo.Quartz.SDK/artifacts`）。
- npm（Platform 仓 `ui/`）：独立版本，随 L3 OpenAPI 契约联调。
- Docker：L3 + `ui/` 静态资源合并单镜像；L2 SDK 由业务宿主自行打包。

## 现有资产迁移来源

| 来源 | 去向 |
|---|---|
| `MinGo.Qap.Shared` | → `MinGo.Quartz.Agent.Abstractions` |
| `MinGo.Qap.Agent` | → `MinGo.Quartz.Agent` |
| `MinGo.Qap.Platform` | → `MinGo.Quartz.Platform/src/` |
| `MinGo.Qap.UI` | → `MinGo.Quartz.Platform/ui/` |
| `MinGo.Quartz`（旧） | → L1 观测模型/读取器；EF Core 抽象 + AppAny 表映射 → L1 可选包 `MinGo.Quartz.EFCore.*`；`Result`/`Paged` → L2 Abstractions |
| `MinGo.Quartz.Extensions`/`MinGo.QuartzStartup`（空壳） | 溶解，由本目录承接 |

各层详细规划见各自 `PLAN.md`。
