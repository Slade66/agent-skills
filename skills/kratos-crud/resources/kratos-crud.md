# CRUD 代码规范

本项目所有资源（`EsxiHost` / `EsxiDatastore` / ...）的 CRUD 代码按本规范组织。规范来源：Kratos 官方 Layout + 项目实际约束。

> proto 生成流程见另一份文档，本规范只讲"生成之后如何写 Go 代码"。

## 一、分层职责

```
api/                    proto 定义 + 生成的 .pb.go        ── 对外契约层
internal/
├── service/            实现 pb server 接口              ── Application 层（DDD）
├── biz/                业务逻辑 + 领域实体 + Repo 接口   ── Domain 层
├── data/               Repo 接口的实现 + DB 细节         ── Infrastructure 层
└── server/             HTTP/gRPC Server 装配             ── Transport 层
```

**核心原则——依赖倒置（DIP）**：

- `biz` **定义** `Repo` 接口
- `data` **实现** `Repo` 接口
- `biz` **只依赖自己定义的接口**，不 `import` `data` 包
- 编译期依赖方向：`service → biz ← data`（`data` 反向依赖 `biz`）
- 运行期通过 wire 注入 `data.NewXxxRepo()` 给 `biz.NewXxxUsecase()`

好处：biz 可单测（mock repo）；DB 换方言或加缓存，biz 一行不改。

## 二、对象分离：严格 DTO / DO / PO 三层

| 对象类型 | 所在层 | Go 类型举例 | 特征 |
|---|---|---|---|
| **DTO** (Data Transfer Object) | service | `*v1.EsxiHost` / `*v1.CreateEsxiHostRequest` | protoc 生成，带 protobuf tag，时间是 `*timestamppb.Timestamp` |
| **DO** (Domain Object) | biz | `biz.EsxiHost` | 手写纯 Go struct，**无任何框架 tag**，时间是 `time.Time`，id 是对外 UUID |
| **PO** (Persistence Object) | data | `data.EsxiHostPO` | 手写 GORM model，带 `gorm:"..."` tag，含内部 `int64 ID`、`uniqueIndex` 等存储细节 |

### 2.1 数据流向

```
客户端 JSON
    │  (Kratos HTTP decoder)
    ▼
*v1.CreateEsxiHostRequest      ← service 层入参
    │  service 层转换：DTO → DO
    ▼
*biz.EsxiHost                   ← biz 层入参，biz 生成 UUID
    │  biz 调 repo.Create(ctx, do)
    ▼
data 层 repo 内部：DO → PO
    │
    ▼
*data.EsxiHostPO                ← GORM 操作的对象
    │  DB.Create(po)
    ▼
数据库行
```

返程（读取）逆向再转一次：`PO → DO → DTO`。

### 2.2 转换函数放哪

- **DTO ↔ DO** 转换函数放在 `internal/service/<resource>.go` 里（私有函数，命名 `toBizXxx` / `toProtoXxx`）
- **DO ↔ PO** 转换函数放在 `internal/data/<resource>.go` 里（私有函数，命名 `toPO` / `toDO`）
- 永远不跨层使用对象（例如 service 不直接用 PO）

## 三、Repo 接口：显式 CRUD

```go
// internal/biz/<resource>.go
type EsxiHostRepo interface {
    Create(ctx context.Context, h *EsxiHost) error
    GetByID(ctx context.Context, id string) (*EsxiHost, error)
    ListAll(ctx context.Context) ([]*EsxiHost, error)
    Update(ctx context.Context, h *EsxiHost) error
    Delete(ctx context.Context, id string) error
}
```

**约定**：

- 方法名就叫 `Create` / `GetByID` / `ListAll` / `Update` / `Delete`，**不用** `Save` / `FindByID` / `Remove` 等同义词
- `Create` / `Update` / `Delete` 返回 `error`，不返回对象
- `GetByID` 返回 `(*EsxiHost, error)`
- `ListAll` 返回 `([]*EsxiHost, error)`
- 未来加分页再加 `ListPaged(ctx, page, size)` 这样的专用方法，**不**给 `ListAll` 加可选参数

## 四、UUID 生成

**在 biz 层 `Create` 流程里生成**：

```go
// internal/biz/esxi_host.go
import "github.com/google/uuid"

func (uc *EsxiHostUsecase) Create(ctx context.Context, h *EsxiHost) (*EsxiHost, error) {
    h.ID = uuid.NewString()        // biz 负责生成
    if err := uc.repo.Create(ctx, h); err != nil {
        return nil, err
    }
    return h, nil
}
```

- data 层的 PO 里 uuid 字段是普通 string，不写 `default:` 或 GORM `BeforeCreate` hook
- 将来想换 ULID / Snowflake，只改 biz 一处

## 五、Update：全字段覆盖 + RowsAffected 判 NotFound

采用"前端传什么就存什么"的语义：**即使空字符串也直接覆盖**。后端不做"空串 = 不改"的合并。

**前端责任**：只想改部分字段时，先 `Get` 拿全量再整体提交。

### 5.1 铁律：PO 必须带 `UpdatedAt` 字段

方案能工作的前提是**每个 PO 都有 `UpdatedAt time.Time` 字段**，字段名**必须**就叫 `UpdatedAt`（首字母大写，GORM 自动识别）：

```go
type EsxiHostPO struct {
    // ... 业务字段
    CreatedAt time.Time
    UpdatedAt time.Time   // ← 必须有，且字段名固定
}
```

**为什么是铁律**：GORM 在 UPDATE 时自动刷新 `UpdatedAt`。这保证每次 UPDATE 对行总有"实际变更"，于是 MySQL 默认模式（`mysql_affected_rows` 只数变更行）下的 `RowsAffected` 也能稳定反映"是否匹配到行"，不会因"业务字段未变"而错报 NotFound。

**忘记加 `UpdatedAt` 的后果**：Update 逻辑在 MySQL 下会静默错判（请求值 = 旧值时被误报 NotFound）。Code Review 必须卡死这一条。

### 5.2 data 层实现

```go
func (r *esxiHostRepo) Update(ctx context.Context, h *biz.EsxiHost) error {
    result := r.data.DB.Model(&EsxiHostPO{}).
        Where("uuid = ?", h.ID).
        Updates(map[string]any{
            "name":     h.Name,
            "host":     h.Host,
            "username": h.Username,
            "password": h.Password,
        })
    if result.Error != nil {
        if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
            return biz.ErrEsxiHostNameConflict
        }
        return result.Error
    }
    if result.RowsAffected == 0 {
        return biz.ErrEsxiHostNotFound
    }
    return nil
}
```

**实现关键点**：

- **用 `Updates(map[string]any{...})` 而不是 `Updates(struct)`**：struct 版本会跳过零值（空串、0、false），达不到"空串也覆盖"的语义；map 版本所有 key 都写入
- **不要在 map 里列 `updated_at`**：GORM 会自动维护，手写反而容易写错
- `RowsAffected == 0` → 没匹配到行 → 翻译成 `biz.ErrEsxiHostNotFound`
- 方案单次往返，性能比"先读后写"好一倍

## 六、错误处理：proto 枚举 + biz 组装 + data 翻译

错误体系分三步：

1. **proto** 定义 `ErrorReason` 枚举，集中登记错误码
2. **biz** 用枚举值组装 `*errors.Error` 变量（带 HTTP code + 默认 message）
3. **data** 捕获底层 error（gorm / 驱动）翻译成 biz 定义的 error

### 6.1 proto：在 `api/<domain>/v1/error_reason.proto` 定义枚举

每个 domain 下放一份 `error_reason.proto`，集中登记该 domain 的所有业务错误码：

```proto
// api/esxi/v1/error_reason.proto
syntax = "proto3";

package esxi.v1;

option go_package = "gitea.nt.cn/nantian/bifrost/api/esxi/v1;v1";

// ErrorReason 集中登记 esxi domain 的所有业务错误码。
// 新增错误码时追加在末尾，不删除已有条目，不改已有编号。
enum ErrorReason {
  // 占位零值，不使用。proto3 枚举必须有零值。
  ESXI_UNSPECIFIED = 0;

  // 指定 id 的 ESXi 连接不存在。
  ESXI_HOST_NOT_FOUND = 1;

  // 创建/更新 ESXi 连接时发生名称重复。
  ESXI_HOST_NAME_CONFLICT = 2;
}
```

**约定**：

- 零值固定 `<DOMAIN>_UNSPECIFIED = 0` 占位，不使用
- 命名：`<DOMAIN>_<RESOURCE>_<CONDITION>`，全大写下划线
- 新增码**只追加**，不改已有编号、不删（兼容性）
- 跑 `make api` 生成 `error_reason.pb.go`，产出 `v1.ErrorReason_ESXI_HOST_NOT_FOUND` 常量

### 6.2 biz：用枚举常量组装 `*errors.Error`

```go
// internal/biz/esxi_host.go
import (
    "github.com/go-kratos/kratos/v2/errors"
    v1 "gitea.nt.cn/nantian/bifrost/api/esxi/v1"
)

var (
    ErrEsxiHostNotFound = errors.NotFound(
        v1.ErrorReason_ESXI_HOST_NOT_FOUND.String(),
        "esxi host not found",
    )
    ErrEsxiHostNameConflict = errors.Conflict(
        v1.ErrorReason_ESXI_HOST_NAME_CONFLICT.String(),
        "esxi host name already exists",
    )
)
```

**组装公式**：`errors.<HttpKind>(<枚举>.String(), <默认消息>)`

| 元素 | 作用 | 规则 |
|---|---|---|
| `errors.NotFound` / `Conflict` / `BadRequest` / ... | 决定 HTTP 状态码（404 / 409 / 400 / ...） | 用 Kratos 内置构造器，**不**手传 code |
| `v1.ErrorReason_XXX.String()` | 前端分类用的机器可读 reason | **必须**用 proto 枚举常量，不允许手写字符串 |
| 第三参数 | 默认人可读 message（英文、简洁） | 客户端可按需翻译覆盖 |

**为什么强制用 proto 枚举常量**：

- 编译期保护：漏写 / 拼错枚举值会编译失败
- 前端读 `.pb.go` / OpenAPI 能拿到完整错误码列表
- 未来用 `protoc-gen-go-errors` 做高级生成时无需改 biz

**Kratos 构造器对照表**：

| 构造器 | HTTP code | 典型场景 |
|---|---|---|
| `errors.BadRequest` | 400 | 请求参数非法 |
| `errors.Unauthorized` | 401 | 未登录 / token 失效 |
| `errors.Forbidden` | 403 | 登录了但无权限 |
| `errors.NotFound` | 404 | 资源不存在 |
| `errors.Conflict` | 409 | 唯一约束冲突 / 状态冲突 |
| `errors.InternalServer` | 500 | 不应发生的内部错误 |

### 6.3 data：翻译底层 error 到 biz 变量

```go
// internal/data/esxi_host.go
func (r *esxiHostRepo) GetByID(ctx context.Context, id string) (*biz.EsxiHost, error) {
    var po EsxiHostPO
    if err := r.data.DB.Where("uuid = ?", id).First(&po).Error; err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, biz.ErrEsxiHostNotFound   // ← 引用 biz 包里的变量
        }
        return nil, err   // 其他底层错误原样透传（encoder 兜底为 500）
    }
    return toDO(&po), nil
}
```

### 6.4 biz 与 service 层不再翻译

- biz 层拿到 `err` 已经是 Kratos `*errors.Error`，直接 `return nil, err`
- service 层同样直接透传
- HTTP / gRPC 层由 Kratos 默认 ErrorEncoder 编码成响应，业务无感

### 6.5 哪些底层错误必须翻译

| 底层错误 | 翻译目标 | 触发位置 |
|---|---|---|
| `gorm.ErrRecordNotFound` | `ErrXxxNotFound` | `GetByID` / `Delete`（Update 靠 RowsAffected 判） |
| `gorm.ErrDuplicatedKey` | `ErrXxxNameConflict` 等 | `Create` / `Update` |
| 其他 DB error | 原样透传（encoder 兜底为 500） | 任意 |

### 6.6 新增一个错误码的流程

1. 在 `api/<domain>/v1/error_reason.proto` 末尾追加枚举值
2. 跑 `make api` 重生成 `error_reason.pb.go`
3. 在对应 biz 文件里新增 `Err*` 变量（用枚举常量 + Kratos 构造器）
4. 如果是底层 error（gorm 等）触发，在 data 层对应位置加翻译分支
5. biz / service 层正常透传，无需改动

## 七、唯一约束：依赖 DB

**不**在应用层预先 SELECT 查重，由 DB 的 `uniqueIndex` 强制，data 层捕获冲突并翻译：

```go
// PO 定义
type EsxiHostPO struct {
    ID        int64     `gorm:"primarykey"`
    UUID      string    `gorm:"uniqueIndex;size:36;not null"`
    Name      string    `gorm:"uniqueIndex;size:64;not null"`   // ← DB 强制唯一
    Host      string    `gorm:"size:255;not null"`
    Username  string    `gorm:"size:64;not null"`
    Password  string    `gorm:"size:255;not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Create 实现
func (r *esxiHostRepo) Create(ctx context.Context, h *biz.EsxiHost) error {
    po := toPO(h)
    if err := r.data.DB.Create(&po).Error; err != nil {
        if errors.Is(err, gorm.ErrDuplicatedKey) {
            return biz.ErrEsxiHostNameConflict
        }
        return err
    }
    return nil
}
```

**注意事项**：

- `gorm.ErrDuplicatedKey` 需要 GORM v1.24+ 与方言驱动支持（SQLite / MySQL 最新驱动都可）
- 不支持自动映射的方言，要在 `data` 包里加一个 `isUniqueConflict(err) bool` 方言适配函数
- 表必须有对应的 `uniqueIndex`（PO tag 里写清楚），否则 DB 不会报冲突

## 八、Service 层：薄层搬运

Service 层不写业务逻辑，只做字段搬运和转换：

```go
// internal/service/esxi_host.go
type EsxiHostService struct {
    v1.UnimplementedEsxiHostServiceServer
    uc *biz.EsxiHostUsecase
}

func NewEsxiHostService(uc *biz.EsxiHostUsecase) *EsxiHostService {
    return &EsxiHostService{uc: uc}
}

func (s *EsxiHostService) CreateEsxiHost(ctx context.Context, in *v1.CreateEsxiHostRequest) (*v1.EsxiHost, error) {
    do := &biz.EsxiHost{
        Name:     in.Name,
        Host:     in.Host,
        Username: in.Username,
        Password: in.Password,
    }
    created, err := s.uc.Create(ctx, do)
    if err != nil {
        return nil, err       // 错误已是 Kratos *Error，直接透传
    }
    return toProto(created), nil
}
```

**禁令**：

- service 层**不**直接访问 `r.data.DB`
- service 层**不**判断底层错误（如 `errors.Is(err, gorm.ErrRecordNotFound)`），那是 data 层的事
- service 层**不**写业务决策（如"同名不能重复"，那是 biz 的事）

## 九、文件组织：每资源一组

假设新增资源 `Xxx`（snake_case 为 `xxx`）：

```
api/<domain>/v1/xxx.proto           ← proto 定义（另一份规范管）
api/<domain>/v1/xxx.pb.go           ← 生成
api/<domain>/v1/xxx_http.pb.go      ← 生成
api/<domain>/v1/xxx_grpc.pb.go      ← 生成

internal/biz/xxx.go                 ← DO + Repo interface + Usecase + Errors
internal/data/xxx.go                ← PO + repo 实现
internal/service/xxx.go             ← Service（实现 pb server）

internal/biz/biz.go                 ← ProviderSet += NewXxxUsecase
internal/data/data.go               ← ProviderSet += NewXxxRepo
internal/service/service.go         ← ProviderSet += NewXxxService
internal/server/http.go             ← 注入 *service.XxxService 并注册 HTTP
internal/server/grpc.go             ← 注入 *service.XxxService 并注册 gRPC
```

**单一资源的所有代码集中在三个文件里**：`biz/xxx.go` / `data/xxx.go` / `service/xxx.go`。

## 十、Wire 依赖注入

每一层有一个 `ProviderSet`（`biz.go` / `data.go` / `service.go` 里定义），新增资源时追加 constructor：

```go
// internal/biz/biz.go
var ProviderSet = wire.NewSet(NewGreeterUsecase, NewEsxiHostUsecase)

// internal/data/data.go
var ProviderSet = wire.NewSet(NewData, NewGreeterRepo, NewEsxiHostRepo)

// internal/service/service.go
var ProviderSet = wire.NewSet(NewGreeterService, NewEsxiHostService)
```

**重要**：

- `NewEsxiHostRepo` 返回类型必须是 `biz.EsxiHostRepo`（接口类型），不是 `*esxiHostRepo`（具体类型），wire 才会正确连线
- 新增资源后要重新跑 `wire ./cmd/bifrost` 重生成 `wire_gen.go`

## 十一、AutoMigrate

`internal/data/data.go` 的 `NewData` 里统一 AutoMigrate 所有 PO：

```go
func NewData(c *conf.Data) (*Data, func(), error) {
    db, err := openDB(c.Database)
    if err != nil {
        return nil, nil, fmt.Errorf("open db: %w", err)
    }
    if err := db.AutoMigrate(
        &EsxiHostPO{},
        // &OtherResourcePO{},    // 新增资源时追加一行
    ); err != nil {
        return nil, nil, fmt.Errorf("auto migrate: %w", err)
    }
    ...
}
```

## 十二、新增一个资源的 Checklist

假设资源名 `XxxYyy`（如 `EsxiDatastore`）：

1. 按 proto 规范定义 `api/<domain>/v1/xxx_yyy.proto`
2. 在 `api/<domain>/v1/error_reason.proto` 追加该资源需要的错误码枚举值（若 domain 下还没这个文件则新建）
3. 跑 `make api` 生成 `.pb.go` / `_http.pb.go` / `_grpc.pb.go` / `error_reason.pb.go`
4. `internal/biz/xxx_yyy.go`：
   - 定义 DO `XxxYyy`
   - 定义 `Err*` 变量（用 `v1.ErrorReason_XXX.String()` 作 reason）
   - 定义 `XxxYyyRepo` 接口（5 个 CRUD 方法）
   - 实现 `XxxYyyUsecase`（`Create` 里生成 UUID；所有方法调 repo 并透传错误）
   - 导出 `NewXxxYyyUsecase`
5. `internal/biz/biz.go`：`ProviderSet += NewXxxYyyUsecase`
6. `internal/data/xxx_yyy.go`：
   - 定义 PO `XxxYyyPO`（含 `uniqueIndex`、**必须带 `UpdatedAt` 字段**）
   - 实现 `toDO` / `toPO`
   - 实现 `xxxYyyRepo`（5 个方法；翻译 `ErrRecordNotFound` / `ErrDuplicatedKey`；Update 用 `Updates(map)` + `RowsAffected`）
   - 导出 `NewXxxYyyRepo` 返回**接口类型**
7. `internal/data/data.go`：
   - `ProviderSet += NewXxxYyyRepo`
   - `AutoMigrate` 列表加 `&XxxYyyPO{}`
8. `internal/service/xxx_yyy.go`：
   - 实现 `XxxYyyService`（5 个 rpc；纯 DTO ↔ DO 转换）
   - 定义 `toProto` / `toBiz` 转换函数
   - 导出 `NewXxxYyyService`
9. `internal/service/service.go`：`ProviderSet += NewXxxYyyService`
10. `internal/server/http.go`：注入 `*service.XxxYyyService`，调用 `v1.RegisterXxxYyyServiceHTTPServer(srv, s)`
11. `internal/server/grpc.go`：同上，改为 `RegisterXxxYyyServiceServer`
12. 跑 `wire ./cmd/bifrost` 重生成 `wire_gen.go`
13. `go build ./...` 确认编译通过

---

**维护者注**：本规范随项目演进持续更新。任何偏离都应在对应代码文件的注释里说明原因。
