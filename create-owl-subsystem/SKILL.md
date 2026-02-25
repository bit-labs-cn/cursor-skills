---
name: create-owl-subsystem
description: Creates new subsystems or modules with both Go backend (owl/owl-admin) and Vue3 frontend (owl-ui/owl-admin-ui), or either side alone. Use when the user asks to create a new sub-application, add a module to the admin (backend and/or frontend), add frontend pages, do full-stack feature development, or build from the owl/owl-admin/owl-ui/owl-admin-ui template.
---

# 按 owl 体系创建高质量子系统（全栈）

本 Skill 与全局规则 **`coding-standards.mdc`** 配合使用（交付质量、状态枚举、模型、接口契约、菜单 name 等）；**完整模板与验证步骤**在四仓 `docs/` 中，动手前先读对应文档。做全栈时**前后端必须按协调清单对齐**。

**按需深入阅读（与本文件同目录）：**

- owl ServiceProvider 能力与用法 → [provider-reference.md](provider-reference.md)
- 后端 API 黑盒测试流程 → [api-testing-guide.md](api-testing-guide.md)

## 两条路径（先区分再动手）

| 路径 | 含义 | 后端 | 前端 |
|------|------|------|------|
| **A：新建独立子系统** | 新业务线、新仓库/新包，与 owl-admin 平级 | 新 Go 项目（owl 子应用），独立仓库 | **新 npm 包**（如某业务 `-ui`），在宿主里 `createFlexAdmin({ subsystems: [yourSubsystem] })` 注册 |
| **B：扩展 owl-admin** | 在现有后台里加功能 | 在 owl-admin 仓库内加 model/repo/service/handle/route/menu/migrate | 在 owl-admin-ui 仓库内加 views/xxx + api，菜单 path/component 用本包 `viewModulesPathPrefix`（如 `/system`） |

- **路径 A**：前端是**独立包**，有自己的 `defineSubsystem({ name, viewModulesPathPrefix, viewModules, menuContributions })`，后端菜单的 path/component 前缀与该包的 `viewModulesPathPrefix` 一致（例如 `/cms`），**不要**在 owl-admin-ui 里加页面。
- **路径 B**：前端只在 **owl-admin-ui** 里加页面，`viewModulesPathPrefix` 固定为 `/system`，后端菜单 path/component 为 `/system/xxx/index` 等形式。

## 何时使用

- **路径 A**：新建独立子系统（全栈）—— 后端新仓库 + **新前端包**，两端一起设计 API 与菜单。
- **路径 B**：在 owl-admin 中新增模块（全栈）—— 后端 owl-admin 增加 model/repository/service/handle/route/migrate，前端 **owl-admin-ui** 增加 views/xxx + api，并配置后端菜单。
- **仅后端**：只改 owl 或 owl-admin，不涉及前端。
- **仅前端**：路径 A 时只改新前端包；路径 B 时只改 owl-admin-ui（或 owl-ui 宿主），后端接口与菜单已存在。

## 工作流：先读 docs 再动手

**路径 A — 新建独立子系统（全栈）**  
- 后端：先读 `owl/docs/05-create-new-subapp-playbook.md`、`owl/docs/07-minimal-subapp-template.md`，完成后按 `owl/docs/08-startup-and-verification.md` 验证。  
- 前端：先读 `owl-ui/docs/01-architecture-and-bootstrap.md`、`owl-ui/docs/03-subsystem-contract.md`、`owl-ui/docs/08-minimal-subsystem-template.md`；新包需导出 `defineSubsystem` 并在宿主 `createFlexAdmin({ subsystems })` 中注册。

**路径 B — 在 owl-admin 中新增模块（全栈）**  
- 后端：先读 `owl-admin/docs/07-example-notice-module.md`，完成后按 `owl-admin/docs/08-startup-and-verification.md` 验证。  
- 前端：先读 `owl-admin-ui/docs/02-feature-folder-pattern.md`、`owl-admin-ui/docs/03-routing-menu-view-contract.md`、`owl-admin-ui/docs/07-canonical-examples-and-ai-guardrails.md`。

**仅后端**：只读对应仓库的 docs（owl 或 owl-admin）。  
**仅前端**：路径 A 读 owl-ui 的 03/08；路径 B 读 owl-admin-ui 的 02/03/07。

## 全栈协调清单（前后端必须对齐）

| 协调点 | 后端 | 前端 |
|--------|------|------|
| **API 路径** | route 注册路径（如 `/api/v1/notices`） | `http.request` 的 URL 与之一致 |
| **菜单 name** | 菜单返回的 name（如 `SystemNotice`、`CmsArticle`） | 页面 `defineOptions({ name })` 与之一致 |
| **菜单 path/component** | 菜单项的 path 或 component（如 `/system/notice/index` 或 `/cms/article/index`） | 该子系统包的 **viewModulesPathPrefix**（路径 A 由新包定义，如 `/cms`；路径 B 为 owl-admin-ui 的 `/system`）+ 视图相对路径，能匹配到对应 `views/xxx/index.vue` |
| **字段与类型** | 请求/响应的 JSON tag、状态常量 | types.ts 与接口字段名一致；状态展示与后端常量一致（如 1 启用 2 禁用） |
| **分页约定** | 列表接口的 page、pageSize、total、list 字段名与格式 | 前端请求 params 与解析 data 的字段名一致（如 page、pageSize、data.list、data.total） |
| **列表查询参数** | QueryReq 中 `NameLike`、`CodeLike`、`XxxBetween` 等为**语义后缀**；真正参与 HTTP 绑定的键名由 **`json` / `form` tag** 决定 | 前端列表请求的键名与后端 tag **一致**；**禁止**把 Go 字段名机械翻译成请求键（例如后端 `NameLike` + `json:"name"` 时，请求里用 `name`，**不能**用 `nameLike`）；**时间范围**优先用 `XxxBetween` + 逗号分隔值，不要默认生成 `eventAtGte`/`eventAtLte` |

全栈开发时先定好：资源英文名、中文名、API 路径、菜单 name、菜单 path/component（及对应前端的 viewModulesPathPrefix），再按后端 → 前端顺序实现并对照上表自检。

## 查询条件字段映射约定（`db.AppendWhereFromStruct`）

owl 列表查询常把 DTO 交给 `AppendWhereFromStruct` 拼 `WHERE`。此处有两层含义，**生成前端代码时必须分开理解**：

1. **Go 字段名（含后缀）决定查询语义与数据库列**  
   字段名会经 `Cc2Udl` 转为蛇形键后再解析：最后一个 `_` 之后为**操作符**，之前为**列名**。例如 `NameLike` → `name_like` → 列 `name`、`LIKE`。常见后缀：`like`、`gt`、`gte`、`lt`、`lte`、`in`、`notin`、`between`；无 recognized 后缀则按等值 `=` 处理。  
   **时间类列表筛选在本体系中优先用 `XxxBetween`（如 `EventAtBetween`、`CreatedAtBetween`）表示闭区间**，值为**逗号分隔的两个端点**（与 `between` 分支实现一致），**不要**把时间范围默认写成 `EventAtGte` + `EventAtLte` 两个字段，除非后端 QueryReq 明确如此定义。

2. **`json` / `form` tag 决定 HTTP 入参键名**  
   Bind（JSON body 或 query）时，客户端传的键名以 tag 为准。  
   **因此：前端发 `{"name":"街道"}` 与后端 `NameLike string \`json:"name"\`` 对应；语义上的「模糊查询」体现在 Go 字段名 `NameLike` 上，而不是请求 JSON 里的键名。**

**前端强约束：**

- 页面 `reactive(form)` 里可以用 `keyword`、`name` 等任意 UI 字段名；**组装 `getList` 的 payload 时**，必须映射为后端 **`json`/`form` tag** 中的键。
- **禁止**仅凭 Go 字段名生成请求键：例如后端为 `NameLike` + `json:"name"` 时，**错误**为 `nameLike`；**正确**为 `name`。

**对照示例（正例）**

| 后端 Go 字段（示意） | tag | 含义 | 前端 payload 键与值 |
|----------------------|-----|------|------------------------|
| `NameLike` | `json:"name"` | 对列 `name` 做 `LIKE` | `name`: 字符串 |
| `CodeLike` | `json:"code"` | 对列 `code` 做 `LIKE` | `code`: 字符串 |
| `EventAtBetween` | `json:"eventAt"` | 对列 `event_at` 做**闭区间** `>=` 且 `<=` | `eventAt`: **一个字符串**，两段用英文逗号拼接，如 `"1700000000,1700086400"`（具体类型/格式以后端约定为准，常见为 Unix 秒或 RFC 字符串，但必须能 `Split(,)` 成两段） |

`between` 在实现上要求**单个字段**承载 `左端点,右端点`；前端若用 `el-date-picker` 的 `datetimerange`，应在 `onSearch` 里格式化为上述**一个** tag 键，而不是拆成 `eventAtGte`/`eventAtLte` 两个键（除非后端未使用 `Between` 且明确为两个 tag）。

**反例**

- 错误：`{"page":1,"pageSize":10,"nameLike":"街道","codeLike":"街道"}`（键名按 Go 字段名直译，若后端 tag 为 `name`/`code` 则绑定不到或语义不一致）。
- 正确：`{"page":1,"pageSize":10,"name":"街道","code":"街道"}`（与 `json:"name"`、`json:"code"` 一致）。

生成 `types.ts` 中「列表查询参数」接口时：字段名应使用**与后端 JSON 一致的 camelCase 键**（如 `name?`、`code?`），并在注释中标明「对应后端 `NameLike`/`CodeLike`」，避免与 Go 字段名混淆。

## 后端分层与接线顺序（单资源）

1. **model**：`db.BaseModel`、`TableName()`、状态常量与 model 同包。  
2. **repository**：接口 + 实现，`WithContext`，构造函数返回接口类型。  
3. **service**：Create/UpdateReq、`validate` 标签、写操作用 `redis.LockerFactory` 加锁、`copier.Copy` 到 model，调 `repo.WithContext(ctx)`。  
4. **handle**：实现 `router.Handler`（`ModuleName()`），Bind → Service → `router.Success`/`router.Fail`/`router.PageSuccess`；对外 HTTP 方法需 swagger 注释。  
5. **Binds()**：追加 `NewXxxRepository`、`NewXxxService`、`NewXxxHandle`（顺序建议 repo → service → handle）。  
6. **route**：`InitApi` 中注入 handle，用 `router.NewRouteInfoBuilder(...)` 注册路由，`.Name("中文").Build()`；需菜单则 `xxxMenu = r.GetMenu()` 并在 `InitMenu()` 挂到父级。  
7. **迁移**：`database/auto_migrate_gen.go` 的 `Migrate(db)` 中追加 `&Xxx{}`，`Bootstrap()` 中调用 `database.Migrate(...)`。

> **禁止**：Handle 直接注入 Repository。即使是最简单的 CRUD，也必须经过 Service 层。  
> Service 是放校验、copier、锁、事件的唯一位置；Handle 只做 Bind + 调 Service + 返回响应。

### 后端列表 QueryReq 的 `json` / `form` tag（与 `AppendWhereFromStruct` 配套）

生成 `RetrieveXxxReq` 等列表查询 DTO 时：

- **操作符只写在 Go 字段名里**（如 `NameLike`、`CodeLike`、`EventAtBetween`），**`json`/`form` tag 不要照搬 Go 后缀**。对外键名应表示**业务列/含义**，不带 `Like`/`Gte`/`Between` 等实现后缀。  
  - 正例：`NameLike string \`json:"name"\``、`CodeLike string \`json:"code"\``、`CreatedAtBetween string \`json:"createdAt"\``。  
  - 反例：`NameLike string \`json:"nameLike"\``（易误导前端与 Bind 键名，且与「列名 + 操作符在 Go 侧解析」的约定不一致）。
- **时间范围**：优先 `EventAtBetween string \`json:"eventAt"\``（或业务约定的单列名），值为逗号分隔两端点；**不要**默认写成 `EventAtGte`/`EventAtLte` 两个字段及 `json:"eventAtGte"`/`json:"eventAtLte"`，除非产品明确要求拆开。
- **与精确条件同屏时避免 json 键冲突**：若已有 `Province string \`json:"province"\``（等值），则 `ProvinceLike` 的 tag **不能**再用 `"province"`，应使用可区分的键（如 `provinceFuzzy`）；`AppendWhereFromStruct` 仍只读 **Go 字段名**，语义不变。

## 前端分层与接线顺序

### 路径 A — 新前端子系统包

1. 包结构：`package.json`、`src/index.ts`（`defineSubsystem`：name、viewModulesPathPrefix、viewModules、routes、menuContributions）、`src/routes/index.ts`（可选）、`src/api/`、`src/views/<模块>/`。  
2. **api**：在 `src/api/` 下新增模块，`http.request` 的 URL、method、params/data 与后端一致。  
3. **标准 CRUD 强制五文件**（见下文通用示例）：`types.ts`、`useXList.ts`、`columns.tsx`、`XxxForm.vue`、`index.vue`。  
4. **宿主**：安装该包并在 `createFlexAdmin({ subsystems: [yourSubsystem] })` 中注册；后端菜单 path/component 前缀与包内 `viewModulesPathPrefix` 一致。

原则性长文档仍以 `owl-ui/docs/03-subsystem-contract.md`、`owl-ui/docs/08-minimal-subsystem-template.md` 为准；**生成代码时以本 Skill 下列通用示例为唯一结构样板**，不要依赖某个业务仓库目录作为“照抄来源”。

### 路径 A：标准 CRUD 禁止写法

- 在 `index.vue` 内定义整段 `columns: TableColumnList = [...]`（应放到 `columns.tsx` 的 `createColumns()`）。  
- 在 `index.vue` 内用临时对象/`setup+render` 充当表单组件。  
- 在 `addDialog` 的 `contentRenderer` 里用 `h(resolveComponent("el-form"), ...)` 手搓整页表单（应使用独立 `XxxForm.vue`）。  
- 把 `reactive(form) + pagination + onSearch + handleSizeChange` 整段写在 `index.vue`（应放到 `useXList.ts`）。  
- `beforeSure` 里用 `options.props.formInline` 作为提交体（应 `xxxFormRef.value.getFormData()`，并在 `getRef().validate` 通过后提交）。  
- 用单文件 `<script setup lang="tsx">` 替代标准五文件分层（列定义可留在 `columns.tsx` 为 TSX，页面壳用 `lang="ts"`）。

### 路径 A：页面分类与最小骨架（非标准 CRUD）

| 类型 | 最小文件 |
|------|----------|
| **只读列表**（无弹窗表单） | `index.vue` + `useXList.ts` + `columns.tsx`；行类型可放 `types.ts` |
| **详情页** | `index.vue` + 按需 `useXxxDetail.ts` + `types.ts`；复杂区块拆 `components/` |
| **创建/向导** | `index.vue` + `useXxxWizard.ts` 或步骤子组件；避免与列表混在单文件 |
| **设计器/画布** | 独立入口 `*.vue` + `components/`；列表页仍按标准 CRUD 分层 |
| **仪表盘** | `index.vue`；数据装配复杂时拆 `composables/` |

列表 + 侧栏（如主表 + 从表/成员）：主表 CRUD 仍用五文件；侧栏逻辑可 `useXxxSidePanel.ts` 或保留在 `index.vue` 但**不得**把主表表单再内联进页面壳。

### 路径 A：标准 CRUD 通用代码示例（替换命名与字段即可）

以下示例中资源名为 **Article**，请替换为你的模块名、API、`defineOptions({ name })` 与菜单 name。  
列表搜索若走后端 `AppendWhereFromStruct`，**payload 键名必须与后端 `json`/`form` tag 一致**，不得用 Go 字段名（如 `NameLike`）直译成 `nameLike`；详见上文「查询条件字段映射约定」。

**`src/views/article/types.ts`**

```ts
export interface ArticleFormData {
  id?: string;
  title: string;
  content: string;
  status: number;
}
```

**`src/views/article/useArticleList.ts`**

```ts
import { reactive, ref, onMounted, toRaw } from "vue";
import type { PaginationProps } from "@pureadmin/table";
import { articleAPI } from "../../api/article";

export function useArticleList() {
  const form = reactive({ title: "", status: "" });
  const dataList = ref([]);
  const loading = ref(true);
  const pagination = reactive<PaginationProps>({
    total: 0,
    pageSize: 10,
    currentPage: 1,
    background: true
  });

  async function onSearch() {
    loading.value = true;
    const payload: any = toRaw(form);
    payload.page = pagination.currentPage;
    payload.pageSize = pagination.pageSize;
    const { data } = await articleAPI.getList(payload);
    dataList.value = data?.list ?? [];
    pagination.total = data?.total ?? 0;
    loading.value = false;
  }

  function resetForm(formEl: any) {
    formEl?.resetFields();
    onSearch();
  }

  function handleSizeChange(val: number) {
    pagination.pageSize = val;
    pagination.currentPage = 1;
    onSearch();
  }

  function handleCurrentChange(val: number) {
    pagination.currentPage = val;
    onSearch();
  }

  onMounted(() => onSearch());

  return {
    form,
    dataList,
    loading,
    pagination,
    onSearch,
    resetForm,
    handleSizeChange,
    handleCurrentChange
  };
}
```

**`src/views/article/columns.tsx`**

```tsx
export function createColumns(): TableColumnList {
  return [
    { label: "ID", prop: "id", width: 80 },
    { label: "标题", prop: "title", minWidth: 160 },
    {
      label: "状态",
      prop: "status",
      width: 100,
      cellRenderer: ({ row }) => (
        <el-tag type={row.status === 1 ? "success" : "danger"}>
          {row.status === 1 ? "启用" : "禁用"}
        </el-tag>
      )
    },
    { label: "创建时间", prop: "createdAt", minWidth: 160 },
    { label: "操作", fixed: "right", width: 160, slot: "operation" }
  ];
}
```

**`src/views/article/ArticleForm.vue`**

```vue
<script setup lang="ts">
import { ref } from "vue";
import type { ArticleFormData } from "./types";

const props = defineProps<{ formInline: ArticleFormData }>();
const ruleFormRef = ref();
const newFormInline = ref<ArticleFormData>({
  id: props.formInline?.id,
  title: props.formInline?.title ?? "",
  content: props.formInline?.content ?? "",
  status: props.formInline?.status ?? 1
});

const rules = {
  title: [{ required: true, message: "请输入标题", trigger: "blur" }]
};

function getRef() {
  return ruleFormRef.value;
}
defineExpose({ getRef, getFormData: () => newFormInline.value });
</script>

<template>
  <el-form ref="ruleFormRef" :model="newFormInline" :rules="rules" label-width="80px">
    <el-form-item label="标题" prop="title">
      <el-input v-model="newFormInline.title" placeholder="请输入标题" clearable />
    </el-form-item>
    <el-form-item label="内容" prop="content">
      <el-input v-model="newFormInline.content" type="textarea" placeholder="请输入内容" />
    </el-form-item>
    <el-form-item label="状态" prop="status">
      <el-radio-group v-model="newFormInline.status">
        <el-radio :value="1">启用</el-radio>
        <el-radio :value="2">禁用</el-radio>
      </el-radio-group>
    </el-form-item>
  </el-form>
</template>
```

**`src/views/article/index.vue`**

```vue
<script setup lang="ts">
import { ref, h } from "vue";
import { useArticleList } from "./useArticleList";
import { createColumns } from "./columns";
import ArticleForm from "./ArticleForm.vue";
import type { ArticleFormData } from "./types";
import { articleAPI } from "../../api/article";
import { addDialog } from "@bit-labs.cn/owl-ui/components/ReDialog";
import { PureTableBar } from "@bit-labs.cn/owl-ui/components/RePureTableBar";

defineOptions({ name: "CmsArticle" });

const formRef = ref();
const articleFormRef = ref();
const { form, loading, dataList, pagination, onSearch, resetForm, handleSizeChange, handleCurrentChange } =
  useArticleList();
const columns = createColumns();

function openDialog(title = "新增", row?: ArticleFormData) {
  addDialog({
    title: `${title}文章`,
    props: {
      formInline: {
        id: row?.id,
        title: row?.title ?? "",
        content: row?.content ?? "",
        status: row?.status ?? 1
      }
    },
    width: "500px",
    contentRenderer: ({ options }) => h(ArticleForm, { ref: articleFormRef, formInline: options.props.formInline }),
    beforeSure: done => {
      const FormRef = articleFormRef.value.getRef();
      const curData = articleFormRef.value.getFormData() as ArticleFormData;
      FormRef.validate((valid: boolean) => {
        if (valid) {
          const api = curData.id ? articleAPI.update(curData.id, curData) : articleAPI.create(curData);
          api.then(() => {
            done();
            onSearch();
          });
        }
      });
    }
  });
}

function handleDelete(row: { id: string }) {
  articleAPI.remove(row.id).then(() => onSearch());
}
</script>

<template>
  <div class="main">
    <el-form ref="formRef" :inline="true" :model="form" class="search-form bg-bg_color w-[99/100] pl-8 pt-[12px]">
      <el-form-item label="标题" prop="title">
        <el-input v-model="form.title" placeholder="标题" clearable class="!w-[180px]" />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="onSearch">搜索</el-button>
        <el-button @click="resetForm(formRef)">重置</el-button>
      </el-form-item>
    </el-form>
    <PureTableBar :columns="columns" @refresh="onSearch">
      <template #buttons>
        <el-button type="primary" @click="openDialog()">新增</el-button>
      </template>
    </PureTableBar>
    <pure-table
      :data="dataList"
      :columns="columns"
      :pagination="pagination"
      :loading="loading"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
    >
      <template #operation="{ row }">
        <el-button link type="primary" @click="openDialog('编辑', row)">编辑</el-button>
        <el-button link type="danger" @click="handleDelete(row)">删除</el-button>
      </template>
    </pure-table>
  </div>
</template>
```

**路径 B — owl-admin-ui 内新增 CRUD 页**  
1. **api**：在 `src/api/` 下新增或扩展模块，`http.request` 与后端一致。  
2. **types**：表单/行数据类型、与后端 JSON 字段名一致；列表 Query 类型键名对齐后端 tag，模糊查询见「查询条件字段映射约定」。  
3. **useXList**：列表状态、分页、onSearch、调用上面 api。  
4. **columns**：表格列定义，含状态等 cellRenderer。  
5. **Form.vue**：弹窗表单，接收 `formInline`，暴露 `getRef()` 与 `getFormData()`，父页在 `beforeSure` 中校验再请求。  
6. **index.vue**：页面壳、`defineOptions({ name })` 与菜单 name 一致、PureTableBar + pure-table、addDialog + contentRenderer(Form)、beforeSure 内 FormRef.validate 后调 api 再 done()。  
7. **后端配菜单**：菜单 name = 页面 name，path/component = `/system/xxx/index`（与 owl-admin-ui 的 viewModulesPathPrefix + views 路径对应）。

路径 B 的目录约定与自检以 `owl-admin-ui/docs/02-feature-folder-pattern.md` 为准；**结构上与上文路径 A 标准 CRUD 五文件一致**，仅前缀与宿主不同。

## 后端快速参考

- **SubApp 契约**：`Name`、`Bootstrap`、`ServiceProviders`、`Menu`、`Commands`、`RegisterRouters`、`Binds`；结构体必须有 `app foundation.Application`。  
- **Provider 选择**：最小 HTTP+DB → `router.RouterServiceProvider`、`db.DBServiceProvider`；加 RBAC → 再加 `permission.GuardProvider` 与 JWT Provider。需要文件上传 → 加 `storage.StorageServiceProvider`。**先查 [provider-reference.md](provider-reference.md)，已有的直接用。**  
- **常见坑**：漏注册 Binds、漏写 route、漏加 Migrate、缺 `app` 字段、Swagger `@Router` 与真实路径不一致、**需要的能力框架已提供却自己重新实现**。

## 前端快速参考

- **路径 A（新子系统包）**：包入口导出 `defineSubsystem`，宿主 `createFlexAdmin({ subsystems: [sub] })` 注册；菜单 path/component 前缀与包内 `viewModulesPathPrefix` 一致；**标准 CRUD 必须用本 Skill 中五文件通用示例的结构**。  
- **路径 B（owl-admin-ui 内加页）**：在 owl-admin-ui 的 `src/views/`、`src/api/` 下按文档增加文件；**不要**在 `src/routes/index.ts` 为业务页加静态路由；路由由后端菜单 + 动态注入。  
- **通用**：`defineOptions({ name })` 与后端菜单项 name 必须一致；Form 暴露 `getRef()` 与 `getFormData()`，父页在 addDialog 的 `beforeSure` 中 `FormRef.validate` 通过后再请求、再 `done()`。  
- **列表查询**：`useXList` 里组装的 `params`/`data` 键名以**后端 QueryReq 的 `json`/`form` tag** 为准；`Like`/`Between` 等仅存在于 Go 字段名上，**不要**生成 `nameLike`、`codeLike` 这类键，除非后端 tag 明确如此命名；**时间范围**优先对应 `XxxBetween` 的**单个** tag（值为 `a,b`），勿默认拆成两个 `*Gte`/`*Lte` 请求键。

## 全栈自检清单

- [ ] **框架能力**：所有功能需求（存储、锁、支付、OCR、消息队列等）已优先查过 [provider-reference.md](provider-reference.md)；凡框架已提供的，均通过 ServiceProvider 注入使用，未重复实现。  
- [ ] **协调**：API 路径、菜单 name、菜单 path/component（与对应前端的 viewModulesPathPrefix 一致）、字段与分页约定前后端一致；列表查询键名与后端 `json`/`form` tag 一致（参见「查询条件字段映射约定」）。  
- [ ] **后端**：Binds 已注册、InitApi 已注册路由、InitMenu 已挂菜单、Migrate 已加 model、SubApp 有 `app` 字段。  
- [ ] **前端**：页面 `defineOptions({ name })` 与菜单 name 一致；Form 暴露 getRef、`getFormData`，beforeSure 中先 validate 再请求；api 的 URL/params 与后端一致；列表搜索参数未误用 Go 字段名（如 `nameLike`）代替 tag（如 `name`）。  
  - **路径 A**：前端包已导出 `defineSubsystem` 并在宿主中通过 `createFlexAdmin({ subsystems })` 注册；后端菜单 path/component 前缀与该包 `viewModulesPathPrefix` 一致；标准 CRUD 为五文件分层，无内联 columns/内联表单。  
  - **路径 B**：未在 owl-admin-ui 的 routes/index.ts 注册业务路由。  
- [ ] **验证**：后端按 `owl/docs/08-*` 或 `owl-admin/docs/08-*` 做启动与接口验证；有前端时登录后菜单可见、列表/增删改可通。  
- [ ] **黑盒测试**（后端）：按 [api-testing-guide.md](api-testing-guide.md) 执行并汇总结果。

## 重要约定

- 子应用均为**独立包/独立项目**，不在 owl-admin 仓库内创建新子应用；入口用 `owl.NewApp(&xxx.SubAppXxx{}).WebShell()`（不是 `.Run()`）。  
- SubApp 与自定义 ServiceProvider 结构体**必须**包含字段 `app foundation.Application`，字段名必须是 `app`。  
- 完整可复制说明在四仓 docs：独立后端子系统 `owl/docs/07-minimal-subapp-template.md`；owl-admin 新模块 `owl-admin/docs/07-example-notice-module.md`；前端子系统契约与独立包模板 `owl-ui/docs/03-subsystem-contract.md`、`owl-ui/docs/08-minimal-subsystem-template.md`；owl-admin-ui 内新页面 `owl-admin-ui/docs/02-feature-folder-pattern.md`。
