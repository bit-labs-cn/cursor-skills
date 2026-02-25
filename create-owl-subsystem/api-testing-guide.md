# API 黑盒测试

代码开发完成后，对后端子系统进行独立启动并执行接口黑盒测试。子系统独立运行时**无鉴权中间件**（`AccessAuthorized` 仅用于菜单元数据），所有接口均可直接访问，无需 auth token。

## 前提条件

- 后端已通过 `go run main.go`（在子系统项目根目录）启动
- 数据库服务可连接（按 `conf/database.yaml` 配置）
- 默认监听端口见 `conf/router.yaml`（通常为 8080）
- 先访问 `/health` 确认服务存活

## 测试工具

在子系统项目根目录下创建 `tests/` 目录，编写 Go 测试文件（`*_test.go`）。使用标准库 `net/http` 发送真实 HTTP 请求到已启动的服务，用 `encoding/json` 解析响应并做断言。测试文件按模块拆分（如 `tag_test.go`、`article_test.go`），共用 `helper_test.go` 中的请求与断言辅助函数。测试通过 `go test ./tests/ -v -count=1` 运行（`-count=1` 禁用缓存）。测试函数按 `TestXxx_01_Create`、`TestXxx_02_List` 编号以控制执行顺序。

## 响应格式速查

| 类型 | 格式 |
|------|------|
| **成功** | `{"code":"","msg":"操作成功","success":true,"data":<data或null>}` |
| **分页成功** | `{"code":"","msg":"操作成功","success":true,"data":{"list":[...]},"total":N,"currentPage":N,"pageSize":N}` |
| **校验失败** | HTTP 400，`{"code":"","msg":"<校验错误>","success":false}` |
| **业务错误** | HTTP 200，`{"code":"<bizCode>","msg":"<业务错误>","success":false}` |

## 测试顺序原则

按资源依赖拓扑排列，无依赖资源先测，有依赖的后测。例如：Tag/Category（无依赖） → Article（依赖 Tag/Category） → Document（依赖 Article）。

## 标准 CRUD 测试流程（每个资源）

对每个资源按以下顺序执行，记录每步创建的 ID 供后续步骤使用：

1. **创建（POST）**：发送完整有效 JSON，断言 HTTP 200 + `success: true`
2. **列表（GET + 分页参数）**：断言 `data.list` 为数组、`total >= 1`
3. **详情（GET /:id）**：用创建返回或列表获取的 ID，断言 `data` 包含所有字段
4. **更新（PUT /:id）**：修改一个字段，断言 `success: true`；再查详情确认变更生效
5. **特殊操作**：如有状态变更（`PUT /:id/status`）、发布（`PUT /:id/publish`）等，逐一测试
6. **负面用例 - 校验失败**：缺少必填字段或字段值不合法，断言 HTTP 400 + `success: false`
7. **负面用例 - 不存在的资源**：用不存在的 ID（如 999999）查详情或删除，断言 `success: false`
8. **删除（DELETE /:id）**：断言 `success: true`；再查列表确认已消失

## 关联资源测试

若资源间有关联（如文档收录文章），在基础 CRUD 之后追加：

1. **绑定关联**（POST）：断言 `success: true`
2. **查询关联列表**（GET）：断言返回数组包含已绑定项
3. **排序**（PUT sort）：如有，断言 `success: true`
4. **解绑关联**（DELETE）：断言 `success: true`；再查列表确认已移除

## 测试数据清理

测试完成后，按依赖拓扑**逆序**删除测试数据（先删有依赖的，再删无依赖的），保持数据库干净。

## 断言规则

每个请求必须检查：

- HTTP 状态码（200 或 400）
- `success` 字段值（true/false）
- 正面用例的 `msg` 应为 `"操作成功"`
- 列表接口的 `total`、`currentPage`、`pageSize` 字段存在且合理
- 详情接口的 `data` 包含模型定义的所有字段

## 测试结果汇总

测试完成后输出汇总表：

| 模块 | 接口 | 方法 | 路径 | 结果 |
|------|------|------|------|------|
| Tag | 创建标签 | POST | /api/v1/cms/tags | PASS/FAIL |
| ... | ... | ... | ... | ... |
