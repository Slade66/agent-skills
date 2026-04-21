# Proto 规范

本项目所有 `.proto` 文件按本规范编写，新增资源时照抄模板即可。

## 一、目录与文件

| 维度 | 规则 | 示例 |
|---|---|---|
| 根目录 | 固定 `api/` | `api/` |
| 领域目录 | 一组相关资源共享命名空间，全小写 | `esxi/` |
| 版本目录 | 领域级版本号，`v1` / `v2` | `v1/` |
| 文件名 | 资源名，`lower_snake_case.proto` | `esxi_host.proto` |
| 单文件内容 | 1 个 `service` + 该资源的所有 `message` | 同左 |

**扩展原则**：同领域新增资源时，在同目录新建 `.proto` 文件，**不另开目录**。例如：

```
api/esxi/v1/
├── esxi_host.proto
├── esxi_datastore.proto   ← 未来新增
└── esxi_vm.proto          ← 未来新增
```

领域内的 proto 文件超过 30 个时才考虑按子领域拆分顶层目录。

## 二、文件头模板

```proto
syntax = "proto3";

package <domain>.v1;                     // 全小写，点分

import "google/api/annotations.proto";   // 必备：HTTP 注解
import "google/protobuf/empty.proto";    // 可选：返回 Empty 时导入
import "google/protobuf/timestamp.proto"; // 可选：含时间字段时导入

option go_package = "gitea.nt.cn/nantian/bifrost/api/<domain>/v1;v1";
```

- **不写** `java_package` / `java_multiple_files` / `java_outer_classname`（项目纯 Go）
- `go_package` 末尾 `;v1` 指定生成 Go 代码的包别名，统一为 `package v1`
- `import` 按字母序

## 三、命名规范

| 对象 | 规则 | 示例 |
|---|---|---|
| `service` | `CamelCase` + `Service` 后缀 | `EsxiHostService` |
| `rpc` 方法 | `CamelCase`，动宾结构 | `CreateEsxiHost` / `ListEsxiHosts` |
| `message` | `CamelCase`；请求/响应与 rpc 对齐 | `CreateEsxiHostRequest` / `ListEsxiHostsReply` |
| 字段 | `snake_case` | `created_at` / `host` |
| `repeated` 字段（列表） | **团队约定固定用 `list`** | `repeated EsxiHost list` |
| `enum` 类型 | `CamelCase` | `ErrorReason` |
| `enum` 值 | `CAPITALS_WITH_UNDERSCORES`，零值带 `UNSPECIFIED` | `ESXI_HOST_NOT_FOUND` |

> **注**：Google AIP 推荐 repeated 字段用复数名（如 `esxi_hosts`），但为与团队统一响应 envelope 的 `data.list` 对齐，项目内所有列表字段固定叫 `list`。

## 四、HTTP 映射（内网规范）

**全部使用 POST + `body: "*"`，参数全走 body**。URL 格式：

```
/v1/<资源 kebab 复数>/<verb>
```

五个标准 verb：`create` / `get` / `list` / `update` / `delete`。模板：

```proto
rpc XxxResource (XxxResourceRequest) returns (XxxResourceReply) {
  option (google.api.http) = {
    post: "/v1/xxx-resources/<verb>"
    body: "*"
  };
}
```

> **非对外 RESTful**：此约定来自内网"一切走 POST + JSON body"的安全审计策略，偏离 Google AIP，但符合公司基础设施要求。所有新增 rpc 必须遵守。

## 五、资源标识（ID 约定）

- **对内**：数据库自增 `int64 id`，永不暴露到接口
- **对外**：`string id`，值为 UUID（由 biz 层在创建时生成）
- **proto 字段名固定叫 `id`**（不叫 `uuid`）

好处：未来如果把生成策略从 UUID 换成 ULID / Snowflake / 人可读 code，**字段名无需变更**。

## 六、请求与响应 message

### 6.1 请求 message

- **每个 rpc 必须定义独立请求 message**，即使当前无字段（如 `ListEsxiHostsRequest {}`）
- **绝不使用 `google.protobuf.Empty`** 作为请求类型
- 字段顺序：主键（`id`）→ 核心业务字段 → 可选字段

### 6.2 响应 message

| 响应形态 | 规则 | 示例 |
|---|---|---|
| 单对象 | 直接返回资源 message | `CreateEsxiHost` 返回 `EsxiHost` |
| 列表 | `XxxListReply { repeated Item list = 1; }` | `ListEsxiHostsReply` |
| 空响应 | 允许 `google.protobuf.Empty` | `DeleteEsxiHost` |

### 6.3 敏感字段处理

- **响应 message 刻意省略敏感字段**（例如 `EsxiHost` 不包含 `password`）
- **请求 message 可以包含敏感字段**（如 `CreateEsxiHostRequest.password`）
- Update 语义约定：**全字段覆盖**，即请求中的每个字段（包括空串）都会写入数据库。前端若只想改部分字段，需先 Get 再整体提交

## 七、时间字段

- 类型统一 `google.protobuf.Timestamp`
- 标准字段名：`created_at` / `updated_at`；业务时间按 `<动作>_at` 命名
- **存储与传输一律 UTC**，字段注释中明确标注
- 生成的 Go 类型为 `*timestamppb.Timestamp`，service 层需显式转换

## 八、错误码定义

每个 domain 在 `api/<domain>/v1/error_reason.proto` 集中登记业务错误码枚举：

```proto
// api/esxi/v1/error_reason.proto
syntax = "proto3";

package esxi.v1;

option go_package = "gitea.nt.cn/nantian/bifrost/api/esxi/v1;v1";

enum ErrorReason {
  ESXI_UNSPECIFIED        = 0;   // 占位零值，不使用
  ESXI_HOST_NOT_FOUND     = 1;
  ESXI_HOST_NAME_CONFLICT = 2;
  // 新增条目追加在末尾，不改已有编号
}
```

**约定**：

- 零值 `<DOMAIN>_UNSPECIFIED = 0` 占位
- 命名：`<DOMAIN>_<RESOURCE>_<CONDITION>`，全大写
- 只追加、不改编号、不删（兼容性）
- 不引入 `protoc-gen-go-errors`；biz 层手动用 `errors.NotFound(v1.ErrorReason_XXX.String(), msg)` 组装
- HTTP 状态码由 biz 层选择 Kratos 构造器决定（`NotFound` = 404 / `Conflict` = 409 / `BadRequest` = 400 / ...）

详细使用方式见 `CRUD_CONVENTIONS.md` 第六节。

## 九、响应 Envelope（transport 层，不是 proto 层）

**proto 保持干净**，不引入 `code` / `message` 字段。外层包装由 `internal/server/http_encoder.go` 的 `responseEncoder` 实现：

- **成功响应**：`{"code": 0, "message": "ok", "data": <业务 reply>}`
- **失败响应**：沿用 Kratos 默认 ErrorEncoder，形如 `{"code": <http>, "reason": <...>, "message": <...>, "metadata": {}}`

proto 文件**不关心也不表达** envelope。

## 十、注释规范

**所有层级都必须有注释**，粒度要求：

| 对象 | 注释应写什么 |
|---|---|
| `service` | 领域职责 + 全局约束（如"全 POST"） |
| `rpc` | 该操作做什么 + 特殊语义（如"Update 全字段覆盖"） |
| `message`（实体） | 对外语义 + 哪些字段**不在**响应里（安全声明） |
| `message`（请求/响应） | 用途说明 |
| 字段 | 业务含义 + 边界条件（非空、长度、空串含义、单位、时区等） |

## 十一、版本演进

- 当前所有 proto 均为 `v1`
- **破坏兼容才升 v2**：删字段、语义变更、消息拆分
- 兼容性改动（加字段、加 rpc）直接在 v1 进行
- 新增 v2 时新建 `api/<domain>/v2/` 目录，v1 保留过渡期

## 十二、新增资源 Checklist

1. 在 `api/<domain>/v1/` 下新建 `<resource_snake>.proto`
2. 按模板写文件头：`syntax` → `package` → `imports`（字母序）→ `option go_package`
3. 定义 `service`：5 个 rpc + 全 POST + `/v1/<kebab>/<verb>` 路径
4. 定义资源 message：字段用 `snake_case`，时间用 `Timestamp`，主键叫 `id`，敏感字段刻意省略
5. 为每个 rpc 定义独立 Request，必要时定义 Reply
6. 按第十节要求**补齐所有层级的注释**
7. 执行 `make api` 生成 `.pb.go` / `_http.pb.go` / `_grpc.pb.go`

## 十三、完整参考

当前项目的规范实现：`api/esxi/v1/esxi_host.proto`。新增资源时以此为蓝本复制修改。

### 最小可用骨架

```proto
syntax = "proto3";

package <domain>.v1;

import "google/api/annotations.proto";

option go_package = "gitea.nt.cn/nantian/bifrost/api/<domain>/v1;v1";

// <Resource>Service 管理 xxx。
// 按内网要求：所有接口一律走 POST，参数全部放 body。
service <Resource>Service {
  // 创建一条 xxx。
  rpc Create<Resource> (Create<Resource>Request) returns (<Resource>) {
    option (google.api.http) = {
      post: "/v1/<resource-kebab>/create"
      body: "*"
    };
  }
  // ...（其他 4 个 verb 同理）
}

// <Resource> 是响应实体。
message <Resource> {
  // 对外主键，UUID 字符串。
  string id = 1;
  // ...
}

// Create<Resource>Request 是 Create<Resource> 的请求体。
message Create<Resource>Request {
  // ...
}

// ...（Get/Update/Delete/List 的请求与响应 message）
```

---

**维护者注**：本规范随项目演进持续更新。对现状的任何偏离都应在对应 proto 的注释中写明原因。
