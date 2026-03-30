---
name: create-owl-frontend-module
description: 使用 Vue3 (owl-ui/owl-admin-ui) 创建新的前端子系统或页面。当用户要求创建新的前端子应用、在管理前端添加模块、编写 Vue 页面或实现前端功能时使用。
---

# 按 owl 体系创建高质量前端模块

本 Skill 与全局规则 **`coding-standards.mdc`** 配合使用。**完整模板与验证步骤**在 `owl-ui/docs/` 和 `owl-admin-ui/docs/` 中，动手前先读对应文档。

## 两条路径（先区分再动手）

| 路径 | 含义 | 前端操作 |
|------|------|----------|
| **A：新建独立子系统** | 新业务线、新仓库/新包 | **新 npm 包**（如某业务 `-ui`），在宿主里 `createFlexAdmin({ subsystems: [yourSubsystem] })` 注册 |
| **B：扩展 owl-admin** | 在现有后台里加功能 | 在 owl-admin-ui 仓库内加 views/xxx + api，菜单 path/component 用本包 `viewModulesPathPrefix`（如 `/system`） |

- **路径 A**：前端是**独立包**，有自己的 `defineSubsystem({ name, viewModulesPathPrefix, viewModules, menuContributions })`，后端菜单的 path/component 前缀与该包的 `viewModulesPathPrefix` 一致（例如 `/cms`），**不要**在 owl-admin-ui 里加页面。
- **路径 B**：前端只在 **owl-admin-ui** 里加页面，`viewModulesPathPrefix` 固定为 `/system`，后端菜单 path/component 为 `/system/xxx/index` 等形式。

## 工作流：先读 docs 再动手

**路径 A — 新建独立子系统**  
先读 `owl-ui/docs/01-architecture-and-bootstrap.md`、`owl-ui/docs/03-subsystem-contract.md`、`owl-ui/docs/08-minimal-subsystem-template.md`；新包需导出 `defineSubsystem` 并在宿主 `createFlexAdmin({ subsystems })` 中注册。

**路径 B — 在 owl-admin 中新增模块**  
先读 `owl-admin-ui/docs/02-feature-folder-pattern.md`、`owl-admin-ui/docs/03-routing-menu-view-contract.md`、`owl-admin-ui/docs/07-canonical-examples-and-ai-guardrails.md`。

## 前端分层与接线顺序

### 路径 A — 新前端子系统包

1. 包结构：`package.json`、`src/index.ts`（`defineSubsystem`：name、viewModulesPathPrefix、viewModules、routes、menuContributions）、`src/routes/index.ts`（可选）、`src/api/`、`src/views/`（扁平模块目录，如 `src/views/issue/`、`src/views/task/`）。
2. **api**：在 `src/api/` 下直接输出模块接口文件（如 `src/api/issue.ts`、`src/api/task.ts`），`http.request` 的 URL、method、params/data 与后端一致。
3. **标准 CRUD 强制五文件**（见下文通用示例）：`types.ts`、`useXList.ts`、`columns.tsx`、`XxxForm.vue`、`index.vue`。
4. **宿主**：安装该包并在 `createFlexAdmin({ subsystems: [yourSubsystem] })` 中注册；后端菜单 path/component 前缀与包内 `viewModulesPathPrefix` 一致。
5. **路径约束**：路径 A 下，独立子系统默认禁止再套额外业务前缀目录（例如 `src/views/inspection/issue/`）；只有当子系统内部确实存在多个一级业务域时，才允许新增一层业务域目录。

### 路径 B — owl-admin-ui 内新增 CRUD 页

1. **api**：在 `src/api/` 下新增或扩展模块，`http.request` 与后端一致。
2. **types**：表单/行数据类型、与后端 JSON 字段名一致；列表 Query 类型键名对齐后端 tag，模糊查询见「查询条件字段映射约定」。
3. **useXList**：列表状态、分页、onSearch、调用上面 api。
4. **columns**：表格列定义，含状态等 cellRenderer。
5. **Form.vue**：弹窗表单，接收 `formInline`，暴露 `getRef()` 与 `getFormData()`，父页在 `beforeSure` 中校验再请求。
6. **index.vue**：页面壳、`defineOptions({ name })` 与菜单 name 一致、PureTableBar + pure-table、addDialog + contentRenderer(Form)、beforeSure 内 FormRef.validate 后调 api 再 done()。

**分页列表响应（`router.PageSuccess`）**：`http.request` 解析后的对象与 JSON 一致——`data` 仅为 `{ list: T[] }`，**`total`、`currentPage`、`pageSize` 在根级**，与 `data` 平级。列表赋值用 `res.data.list`，分页用 `res.total` 等；**不要**写成 `data?.total` 或假定 `data` 上同时有 `list` 与 `total`。

## 查询条件字段映射约定

**前端强约束：**
- 页面 `reactive(form)` 里可以用 `keyword`、`name` 等任意 UI 字段名；**组装 `getList` 的 payload 时**，必须映射为后端 **`json`/`form` tag** 中的键。
- **禁止**仅凭 Go 字段名生成请求键：例如后端为 `NameLike` + `json:"name"` 时，**错误**为 `nameLike`；**正确**为 `name`。
- **时间范围**：若后端用 `Between`，前端应在 `onSearch` 里格式化为一个 tag 键（如 `eventAt: "1700000000,1700086400"`），而不是拆成 `eventAtGte`/`eventAtLte` 两个键。

## 标准 CRUD 禁止写法

- 在 `index.vue` 内定义整段 `columns: TableColumnList = [...]`（应放到 `columns.tsx` 的 `createColumns()`）。
- 在 `index.vue` 内用临时对象/`setup+render` 充当表单组件。
- 在 `addDialog` 的 `contentRenderer` 里用 `h(resolveComponent("el-form"), ...)` 手搓整页表单（应使用独立 `XxxForm.vue`）。
- 把 `reactive(form) + pagination + onSearch + handleSizeChange` 整段写在 `index.vue`（应放到 `useXList.ts`）。
- `beforeSure` 里用 `options.props.formInline` 作为提交体（应 `xxxFormRef.value.getFormData()`，并在 `getRef().validate` 通过后提交）。
- 用单文件 `<script setup lang="tsx">` 替代标准五文件分层（列定义可留在 `columns.tsx` 为 TSX，页面壳用 `lang="ts"`）。

## 页面分类与最小骨架（非标准 CRUD）

| 类型 | 最小文件 |
|------|----------|
| **只读列表**（无弹窗表单） | `index.vue` + `useXList.ts` + `columns.tsx`；行类型可放 `types.ts` |
| **详情页** | `index.vue` + 按需 `useXxxDetail.ts` + `types.ts`；复杂区块拆 `components/` |
| **创建/向导** | `index.vue` + `useXxxWizard.ts` 或步骤子组件；避免与列表混在单文件 |
| **设计器/画布** | 独立入口 `*.vue` + `components/`；列表页仍按标准 CRUD 分层 |
| **仪表盘** | `index.vue`；数据装配复杂时拆 `composables/` |

## 标准 CRUD 通用代码示例（五文件结构）

**`src/views/issue/types.ts`**
```ts
export interface IssueFormData {
  id?: string;
  title: string;
  content: string;
  status: number;
}
```

**`src/views/issue/useIssueList.ts`**
```ts
import { reactive, ref, onMounted, toRaw } from "vue";
import type { PaginationProps } from "@pureadmin/table";
import { issueAPI } from "../../api/issue";

export function useIssueList() {
  const form = reactive({ title: "", status: "" });
  const dataList = ref([]);
  const loading = ref(true);
  const pagination = reactive<PaginationProps>({
    total: 0, pageSize: 10, currentPage: 1, background: true
  });

  async function onSearch() {
    loading.value = true;
    const payload: any = toRaw(form);
    payload.page = pagination.currentPage;
    payload.pageSize = pagination.pageSize;
    const res = await issueAPI.getList(payload);
    dataList.value = res?.data?.list ?? [];
    pagination.total = res?.total ?? 0;
    pagination.pageSize = res?.pageSize ?? pagination.pageSize;
    pagination.currentPage = res?.currentPage ?? pagination.currentPage;
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

  return { form, dataList, loading, pagination, onSearch, resetForm, handleSizeChange, handleCurrentChange };
}
```

**`src/views/issue/columns.tsx`**
```tsx
export function createColumns(): TableColumnList {
  return [
    { label: "ID", prop: "id", width: 80 },
    { label: "标题", prop: "title", minWidth: 160 },
    {
      label: "状态", prop: "status", width: 100,
      cellRenderer: ({ row }) => (
        <el-tag type={row.status === 1 ? "success" : "danger"}>
          {row.status === 1 ? "启用" : "禁用"}
        </el-tag>
      )
    },
    { label: "操作", fixed: "right", width: 160, slot: "operation" }
  ];
}
```

**`src/views/issue/IssueForm.vue`**
```vue
<script setup lang="ts">
import { ref } from "vue";
import type { IssueFormData } from "./types";

const props = defineProps<{ formInline: IssueFormData }>();
const ruleFormRef = ref();
const newFormInline = ref<IssueFormData>({
  id: props.formInline?.id,
  title: props.formInline?.title ?? "",
  content: props.formInline?.content ?? "",
  status: props.formInline?.status ?? 1
});

const rules = { title: [{ required: true, message: "请输入标题", trigger: "blur" }] };

function getRef() { return ruleFormRef.value; }
defineExpose({ getRef, getFormData: () => newFormInline.value });
</script>

<template>
  <el-form ref="ruleFormRef" :model="newFormInline" :rules="rules" label-width="80px">
    <el-form-item label="标题" prop="title">
      <el-input v-model="newFormInline.title" placeholder="请输入标题" clearable />
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

**`src/views/issue/index.vue`**
```vue
<script setup lang="ts">
import { ref, h } from "vue";
import { useIssueList } from "./useIssueList";
import { createColumns } from "./columns";
import IssueForm from "./IssueForm.vue";
import type { IssueFormData } from "./types";
import { issueAPI } from "../../api/issue";
import { addDialog } from "@bit-labs.cn/owl-ui/components/ReDialog";
import { PureTableBar } from "@bit-labs.cn/owl-ui/components/RePureTableBar";

defineOptions({ name: "InspectionIssue" }); // 必须与后端菜单 name 一致

const formRef = ref();
const issueFormRef = ref();
const { form, loading, dataList, pagination, onSearch, resetForm, handleSizeChange, handleCurrentChange } = useIssueList();
const columns = createColumns();

function openDialog(title = "新增", row?: IssueFormData) {
  addDialog({
    title: `${title}问题`,
    props: {
      formInline: { id: row?.id, title: row?.title ?? "", content: row?.content ?? "", status: row?.status ?? 1 }
    },
    width: "500px",
    contentRenderer: ({ options }) => h(IssueForm, { ref: issueFormRef, formInline: options.props.formInline }),
    beforeSure: done => {
      const FormRef = issueFormRef.value.getRef();
      const curData = issueFormRef.value.getFormData() as IssueFormData;
      FormRef.validate((valid: boolean) => {
        if (valid) {
          const api = curData.id ? issueAPI.update(curData.id, curData) : issueAPI.create(curData);
          api.then(() => { done(); onSearch(); });
        }
      });
    }
  });
}

function handleDelete(row: { id: string }) {
  issueAPI.remove(row.id).then(() => onSearch());
}
</script>

<template>
  <div class="main">
    <el-form ref="formRef" :inline="true" :model="form" class="search-form">
      <el-form-item label="标题" prop="title">
        <el-input v-model="form.title" placeholder="标题" clearable />
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
      :data="dataList" :columns="columns" :pagination="pagination" :loading="loading"
      @size-change="handleSizeChange" @current-change="handleCurrentChange"
    >
      <template #operation="{ row }">
        <el-button link type="primary" @click="openDialog('编辑', row)">编辑</el-button>
        <el-button link type="danger" @click="handleDelete(row)">删除</el-button>
      </template>
    </pure-table>
  </div>
</template>
```

## 前端快速参考

- **路径 A（新子系统包）**：包入口导出 `defineSubsystem`，宿主 `createFlexAdmin({ subsystems: [sub] })` 注册；菜单 path/component 前缀与包内 `viewModulesPathPrefix` 一致；`src/views/` 使用扁平模块目录（如 `src/views/issue/`），`src/api/` 直接输出模块接口文件（如 `src/api/issue.ts`）；**标准 CRUD 必须用本 Skill 中五文件通用示例的结构**。
- **路径 B（owl-admin-ui 内加页）**：在 owl-admin-ui 的 `src/views/`、`src/api/` 下按文档增加文件；**不要**在 `src/routes/index.ts` 为业务页加静态路由；路由由后端菜单 + 动态注入。
- **通用**：`defineOptions({ name })` 与后端菜单项 name 必须一致；Form 暴露 `getRef()` 与 `getFormData()`，父页在 addDialog 的 `beforeSure` 中 `FormRef.validate` 通过后再请求、再 `done()`。

## 前端自检清单

- [ ] **页面与菜单**：页面 `defineOptions({ name })` 与菜单 name 一致。
- [ ] **表单**：Form 暴露 getRef、`getFormData`，beforeSure 中先 validate 再请求。
- [ ] **API**：api 的 URL/params 与后端一致；列表搜索参数未误用 Go 字段名（如 `nameLike`）代替 tag（如 `name`）。
- [ ] **路径 A 专属**：前端包已导出 `defineSubsystem` 并在宿主中通过 `createFlexAdmin({ subsystems })` 注册；后端菜单 path/component 前缀与该包 `viewModulesPathPrefix` 一致；标准 CRUD 为五文件分层，无内联 columns/内联表单。
- [ ] **路径 B 专属**：未在 owl-admin-ui 的 routes/index.ts 注册业务路由。
- [ ] **验证**：登录后菜单可见、列表/增删改可通。
