---
name: create-owl-backend-module
description: Scaffolds high-quality owl SubApp backends with per-resource model/repository/service/handle layers, typed DTOs, validators, redis locks on writes, and errContract business errors. Use when creating a new Go sub-application, adding owl HTTP APIs, or implementing backend features; do not use for quick throwaway prototypes that collapse multiple domains into one gateway.
---

# 按 owl 体系创建高质量后端模块

本 Skill 与全局规则 **`coding-standards.mdc`** 配合使用。**完整模板与验证步骤**统一以 `owl/docs/` 为准，动手前先读对应文档。

**按需深入阅读（与本文件同目录）：**

- owl ServiceProvider 能力与用法 → `provider-reference.md`
- 后端 API 黑盒测试流程 → `api-testing-guide.md`

## 统一路径（先读 docs 再动手）

本 Skill 只面向 **新建独立子系统**：新业务线、新仓库或新包，基于 `owl` 框架独立搭建后端服务。

## 工作流：先读真实源码，再读 docs，最后动手

### 第一步：读真实参考源码（硬性要求）

**在生成任何 service / repository / handle 代码之前**，必须用 Read 工具阅读以下参考文件（不是凭记忆，不是靠文档片段）：

| 层         | 参考文件（相对于 workspace 根目录）                     | 重点关注                                                   |
|-----------|------------------------------------------------|--------------------------------------------------------|
| **service** | `owl-admin/app/service/dict_service.go`        | DTO struct 的 `validate` + `label` tag 写法、BizError 定义、锁、copier |
| **service** | `owl-admin/app/service/role_service.go`        | 多 DTO（Create/Update/Assign）的 tag 风格对照                    |
| **handle**  | `owl-admin/app/handle/v1/dict_handle.go`       | Bind → Service → router.Success/Fail 的接线                 |
| **repository** | `owl-admin/app/repository/dict.go`          | 接口定义 + WithContext + 构造函数返回接口                           |

> **为什么**：文档里的代码片段可能不完整或过时，真实源码才是"唯一事实来源"。
> 阅读后提取出以下模式并在生成代码时严格遵循：
> - struct tag 的完整写法（`json` + `validate` + `label` 三件套）
> - 业务错误码常量 + 构造函数的组织方式
> - service 方法签名与返回值风格
> - handle 层的 Bind / 响应模式

如果目标项目已有同类 service 文件（如 `owl-portal/app/service/` 下已有文件），也应**至少读 1-2 个已有文件**以保持风格一致。

### 第二步：读框架文档

再读 `owl/docs/05-create-new-subapp-playbook.md`、`owl/docs/07-minimal-subapp-template.md`，了解子应用骨架与接线全貌。

### 第三步：生成代码 & 验证

按下方「后端分层与接线顺序」逐层生成，完成后按 `owl/docs/08-startup-and-verification.md` 验证。

## 后端分层与接线顺序（单资源）

1. **model**：`db.BaseModel`、`TableName()`、状态常量与 model 同包；**业务字段**在 `gorm` tag 中须含 `comment:中文简短说明`（与 `size`/`index` 等写在同一 tag 内），与 `owl/docs/07-minimal-subapp-template.md` 示例一致；`gorm:"-"` 等不参与落库的字段可省略。
2. **repository**：接口 + 实现，`WithContext`，构造函数返回接口类型。
3. **service**：Create/UpdateReq、`validate` 标签、**`label:"中文名"` 标签**、写操作用 `redis.LockerFactory` 加锁、`copier.Copy` 到 model，调 `repo.WithContext(ctx)`。凡带 `validate` 规则的字段，**必须**同时加 `label:"中文名"` 标签（用于验证错误的中文翻译），参照 `owl-admin` 现有 DTO 风格。
4. **handle**：实现 `router.Handler`（`ModuleName()`），Bind → Service → `router.Success`/`router.Fail`/`router.PageSuccess`；对外 HTTP 方法需 swagger 注释。
5. **Binds()**：追加 `NewXxxRepository`、`NewXxxService`、`NewXxxHandle`（顺序建议 repo → service → handle）。
6. **route**：`InitApi` 中注入 handle，用 `router.NewRouteInfoBuilder(...)` 注册路由，`.Name("中文").Build()`；需菜单则 `xxxMenu = r.GetMenu()` 并在 `InitMenu()` 挂到父级。
7. **迁移**：`database/auto_migrate_gen.go` 的 `Migrate(db)` 中追加 `&Xxx{}`，`Bootstrap()` 中调用 `database.Migrate(...)`。

> **禁止**：Handle 直接注入 Repository。即使是最简单的 CRUD，也必须经过 Service 层。  
> Service 是放校验、copier、锁、事件的唯一位置；Handle 只做 Bind + 调 Service + 返回响应。

## 文件与模块边界（硬约束）

- **model**：按业务域拆分为多个文件（如 `site_config.go`、`navigation.go`）。允许同一文件内放强相关的多个类型（例如 `Product` 与 `ProductCategory`），**禁止**把所有实体堆进单个 `models.go`。
- **repository**：每个资源一套 `xxx_repository.go`，包含 **接口 + 实现 + `WithContext`**，构造函数返回接口类型。**禁止** `NewModel(module string)`、`switch module`、反射拼装列表等“总线式”仓储。
- **service**：每个资源一套 `xxx_service.go`，含强类型 `CreateXxxReq` / `UpdateXxxReq` / `RetrieveXxxReq`（及模块特有方法）。**禁止**用 `map[string]any` 作为对外 HTTP 入参载体替代上述结构体。
- **handle**：每个资源一套 `xxx_handle.go`（或 `v1/xxx_handle.go`），`ShouldBindJSON` / `ShouldBindQuery` 绑定到 service 的 Req。**禁止**单文件 `PortalHandle` 聚合全站 CRUD。
- **route**：可在 `route/api.go` 集中注册，但每条路由必须调用**对应资源**的 Handle 方法；**禁止**用 `ctx.Set("module", ...)` + 通用 `Create/Update` 承载核心业务逻辑。

### 反例（一律不允许）

- `PortalRepository` / `PortalService` / `PortalHandle` 多模块网关。
- `reflect` + `switch module` 的通用 `Retrieve`。
- 列表查询用 `keyword` 字符串在仓储层 `switch module` 决定查哪一列（应落在各资源 `RetrieveXxxReq` + 各 repo 的查询闭包或 `AppendWhereFromStruct`）。

## 交付验收（生成后自检）

- [ ] 每个对外资源具备独立 `repository` 接口文件、`service` 文件、`handle` 文件。
- [ ] **model**：落库业务字段的 `gorm` tag 均含 `comment:中文简短说明`（与 `owl/docs/07-minimal-subapp-template.md` 示例一致）。
- [ ] **label 标签**：所有 Req 结构体中带 `validate` 规则的字段均已加 `label:"中文名"` 标签（验证错误中文翻译必须）。
- [ ] 写操作 service 使用 `redis.LockerFactory` 加锁（按资源+主键设计 key）。
- [ ] 领域错误使用 `errContract.NewBizError`，并在 service 包内集中定义错误码常量与构造函数（可按资源拆 `errors_xxx.go` 或分节组织）。
- [ ] `Binds()` 注册顺序：各 `NewXxxRepository` → `NewXxxService` → `NewXxxHandle`。

## 业务错误约定

- 生成的 service **必须优先封装业务错误**，不要把“已存在 / 不存在 / 状态不允许 / 重复操作 / 规则不满足”这类领域错误直接写成裸 `errors.New(...)`。
- 每个模块应在 service 包内定义：
  - 业务错误码常量，如 `CodePlanNotFound`、`CodeTaskNotSubmittable`
  - 业务错误构造函数，如 `PlanNotFound()`、`TaskNotSubmittable(msg string)`
- 推荐使用 `errContract.NewBizError(code, message)` 返回业务错误；让 handle 统一走 `router.Fail(...)`。
- 只有真正的底层异常、第三方失败、未知系统错误才直接向上返回原始 `error`。
- 如果参考示例与本条冲突，以本节为准。

示例：

```go
const (
    CodePlanNotFound = "PLAN_NOT_FOUND"
    CodeTaskLocked   = "TASK_LOCKED"
)

func PlanNotFound() *errContract.BizError {
    return errContract.NewBizError(CodePlanNotFound, "巡检计划不存在")
}

func TaskLocked() *errContract.BizError {
    return errContract.NewBizError(CodeTaskLocked, "任务当前不可操作")
}
```

## 查询条件字段映射约定（`db.AppendWhereFromStruct`）

生成 `RetrieveXxxReq` 等列表查询 DTO 时：

1. **操作符只写在 Go 字段名里**（如 `NameLike`、`CodeLike`、`EventAtBetween`），**`json`/`form` tag 不要照搬 Go 后缀**。对外键名应表示**业务列/含义**，不带 `Like`/`Gte`/`Between` 等实现后缀。
   - 正例：`NameLike string \`json:"name"\``、`CodeLike string \`json:"code"\``、`CreatedAtBetween string \`json:"createdAt"\``。
   - 反例：`NameLike string \`json:"nameLike"\``（易误导前端与 Bind 键名，且与「列名 + 操作符在 Go 侧解析」的约定不一致）。
2. **时间范围**：优先使用 `EventAtBetween` 对应 `json:"eventAt"`（或业务约定的单列名），值为逗号分隔两端点；**不要**默认拆成 `EventAtGte`/`EventAtLte` 和 `json:"eventAtGte"`/`json:"eventAtLte"`，除非产品明确要求拆开。
3. **与精确条件同屏时避免 json 键冲突**：若已有 `Province` 对应 `json:"province"`（等值），则 `ProvinceLike` 的 tag **不能**再用 `"province"`，应使用可区分的键（如 `provinceFuzzy`）；`AppendWhereFromStruct` 仍只读 **Go 字段名**，语义不变。

## 后端快速参考

- **SubApp 契约**：`Name`、`Bootstrap`、`ServiceProviders`、`Menu`、`Commands`、`RegisterRouters`、`Binds`；结构体必须有 `app foundation.Application`。
- **Provider 选择**：最小 HTTP+DB → `router.RouterServiceProvider`、`db.DBServiceProvider`；加 RBAC → 再加 `permission.GuardProvider` 与 JWT Provider。需要文件上传 → 加 `storage.StorageServiceProvider`。**先查 `provider-reference.md`，已有的直接用。**
- **常见坑**：漏注册 Binds、漏写 route、漏加 Migrate、缺 `app` 字段、Swagger `@Router` 与真实路径不一致、**需要的能力框架已提供却自己重新实现**。

## 重要约定

- 子应用均为**独立包/独立项目**，按 `owl` 框架的 SubApp 方式搭建；入口用 `owl.NewApp(&xxx.SubAppXxx{}).WebShell()`（不是 `.Run()`）。
- SubApp 与自定义 ServiceProvider 结构体**必须**包含字段 `app foundation.Application`，字段名必须是 `app`。

## 后端自检清单

- [ ] **参考源码**：生成代码前已用 Read 工具实际阅读了上方「参考文件」表中的 service / handle / repository 源码，而非凭记忆生成。
- [ ] **框架能力**：所有功能需求已优先查过 `provider-reference.md`；凡框架已提供的，均通过 ServiceProvider 注入使用，未重复实现。
- [ ] **Binds与路由**：Binds 已注册、InitApi 已注册路由、InitMenu 已挂菜单。
- [ ] **数据库**：Migrate 已加 model；model 业务字段已写 `gorm` 列注释（`comment:`）。
- [ ] **应用结构**：SubApp 有 `app` 字段。
- [ ] **验证**：按 `owl/docs/08-*` 做启动与接口验证。
- [ ] **黑盒测试**：按 `api-testing-guide.md` 执行并汇总结果。
