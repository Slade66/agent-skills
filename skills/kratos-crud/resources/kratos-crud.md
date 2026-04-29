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
h
## 五、Update：全字段覆盖 + RowsAffected 判 NotFound

> ⚠️ **本节 data 层 Update 实现（单次 UPDATE + 跨事务回读）已被 §十三.6 的事务化方案取代。**
> §5.1 `UpdatedAt` 铁律、§5.2 使用 `Updates(map)` 的要求仍然有效，完整 data 层代码以 §十三.6 为准。

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

> ⚠️ **本节 biz 层的错误组装方式（`errors.NotFound(v1.ErrorReason_XXX.String(), ...)`）已被 §十三.8 的 "biz 纯 sentinel + service toAPIError" 方案取代。**
> 原因：biz 依赖 `api/v1` + `kratos/errors` 违反依赖倒置，biz 不可跨协议复用。
> proto 枚举定义（§6.1）仍有效，但要加 `(errors.code)` 注解并用 `protoc-gen-go-errors` 生成 helper（见 §十三.8）。
> **新代码一律按 §十三.8 写**。下文 §6.2 ~ §6.6 仅作历史演进参考。

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

## 十三、补充经验（方案演进）

> 以下条目来自 bifrost M1–M6 的真实踩坑。每条覆盖三件套：**为什么用 / 什么不该写 / 好坏对照**。
> 新资源落地时，**优先按本节模板写**；本节与 §五 §六 冲突时以本节为准。

### 13.1 GORM `TranslateError: true` 必开启

```go
// internal/data/data.go
return gorm.Open(dialector, &gorm.Config{
    TranslateError: true,
    // ...
})
```

**为什么**：方言原生错误（`*sqlite3.Error` / `*mysql.MySQLError` / `*pq.Error`）彼此不兼容，`errors.Is` 无法统一判定。`TranslateError` 开关让 GORM 把它们翻译成 `gorm.ErrDuplicatedKey` / `gorm.ErrForeignKeyViolated` 等**方言无关 sentinel**——data 层后面所有 `errors.Is(err, gorm.ErrDuplicatedKey)` 分支才有效。不开启时，唯一冲突分支永远 false，冲突被当成 500 抛出去。

**什么不该写**：

```go
// ❌ 默认值（TranslateError 默认 false）
gorm.Open(dialector, &gorm.Config{})

// ❌ 手动拆解方言错误（不跨方言、切 MySQL 就废）
if sqErr, ok := err.(sqlite3.Error); ok && sqErr.ExtendedCode == sqlite3.ErrConstraintUnique {
    return biz.ErrXxxNameConflict
}

// ❌ 字符串匹配错误消息（方言相关、易随驱动升级失效）
if strings.Contains(err.Error(), "UNIQUE constraint failed") {
    return biz.ErrXxxNameConflict
}
```

**好写法 vs 坏写法**：

```go
// ✅ 好
gorm.Open(dialector, &gorm.Config{TranslateError: true})
// data 层
if errors.Is(err, gorm.ErrDuplicatedKey) {
    return biz.ErrXxxNameConflict
}

// ❌ 坏（不开开关 + 字符串判定）
gorm.Open(dialector, &gorm.Config{})
if strings.Contains(err.Error(), "UNIQUE") { ... }  // 脆弱且方言耦合
```

### 13.2 DO 时间字段用 `time.Time`，禁用 string / Timestamp

```go
// internal/biz/xxx.go
type Xxx struct {
    // ...
    CreatedAt time.Time   // ✅
    UpdatedAt time.Time   // ✅
}
```

**为什么**：DO 是领域对象，要让 biz 层用 Go 最自然的方式处理时间（比较、排序、间隔计算）。`time.Time` 支持纳秒、支持 Monotonic Clock、序列化/反序列化无须 parse 层。字符串（`"2026-04-22T09:38:00Z"`）要在每一层 parse/format，任意一层忘记就精度丢失且错误被吞；`*timestamppb.Timestamp` 是 protobuf 类型，不该污染 DO。

**什么不该写**：

```go
// ❌ DO 时间字段用 string
type EsxiHost struct { CreatedAt string }
// 在 data / biz / service 散落 time.Parse(time.RFC3339, ...) —— 任一层写错就炸

// ❌ DO 直接用 protobuf 类型
type EsxiHost struct { CreatedAt *timestamppb.Timestamp }

// ❌ 在 biz 或 data 层调用 timestamppb.New / AsTime
```

**好写法 vs 坏写法**：

```go
// ✅ 好：biz 用 time.Time；data 读完一路 .UTC()；service 出口才转 Timestamp
// biz
type EsxiHost struct { CreatedAt time.Time }
// data
h.CreatedAt = po.CreatedAt.UTC()
// service
if !do.CreatedAt.IsZero() {
    p.CreatedAt = timestamppb.New(do.CreatedAt)
}

// ❌ 坏：DO 里用 string，一路 parse
type EsxiHost struct { CreatedAt string }
t, err := time.Parse(time.RFC3339Nano, po.CreatedAt.Format(time.RFC3339Nano))  // 白跑一遍
```

### 13.3 Server 中间件链：`recovery → logging → validate.ProtoValidate`

```go
// internal/server/http.go
http.Middleware(
    recovery.Recovery(),
    logging.Server(logger),
    validate.ProtoValidate(),
)
// gRPC 同理
```

**为什么**：三个中间件顺序和选择都有讲究：

- **`recovery` 必须最外层**：任何内层 panic 都要兜底成 500 响应而非把进程拉爆；
- **`logging` 紧跟其后**：这样即使请求被 validate 拒绝，仍能记录访问日志（可追踪"谁在刷非法请求"）；
- **`validate` 放最后**：前置基础设施就绪后再校验业务参数，一旦失败直接短路返回，handler 永远不用二次校验。

缺一不可，顺序不能调换。

**什么不该写**：

```go
// ❌ 漏 recovery
http.Middleware(logging.Server(logger), validate.ProtoValidate())

// ❌ validate 在 logging 之前 —— 非法请求不落日志
http.Middleware(recovery.Recovery(), validate.ProtoValidate(), logging.Server(logger))

// ❌ 完全不加 validate，handler 里手写校验
func CreateXxx(ctx, in) {
    if in.Name == "" { return errors.BadRequest(...) }  // 每个 handler 都要写
}
```

**好写法 vs 坏写法**：

```go
// ✅ 好
http.Middleware(recovery.Recovery(), logging.Server(logger), validate.ProtoValidate())

// ❌ 坏
http.Middleware(recovery.Recovery())  // 缺 logging、缺 validate
```

### 13.4 validate 中间件用 **contrib 版**，而非 Kratos v2 内置

```go
// ✅ 正确 import
import "github.com/go-kratos/kratos/contrib/middleware/validate/v2"
validate.ProtoValidate()
```

**为什么**：Kratos v2.8+ 把 `github.com/go-kratos/kratos/v2/middleware/validate.Validator()` 标 deprecated，因为其底层 `envoyproxy/protoc-gen-validate`（PGV）已进入 maintenance。contrib 版 `ProtoValidate()` 底层换成了 `buf.build/go/protovalidate`，**默认开启 legacy 模式**——完全兼容现有 PGV 生成的 `Validate()` 方法，proto 注解 `(validate.rules)` 零改动，升级只改 import 和函数名两处。

**什么不该写**：

```go
// ❌ 继续用 deprecated 的 v2 内置版本
import "github.com/go-kratos/kratos/v2/middleware/validate"
validate.Validator()  // go vet 报 deprecated

// ❌ 想用 protovalidate 新注解 (buf.validate.field) 却仍装 PGV 插件
// legacy 模式不识别新注解，当前 Makefile 里 --validate_out 仍是 PGV
```

**好写法 vs 坏写法**：

```go
// ✅ 好：contrib 包 + 老 PGV 注解
import "github.com/go-kratos/kratos/contrib/middleware/validate/v2"
// proto 仍写 [(validate.rules).string = {min_len: 1}]
validate.ProtoValidate()

// ❌ 坏：deprecated 内置版
import "github.com/go-kratos/kratos/v2/middleware/validate"
validate.Validator()
```

### 13.5 UUID 字段严格校验：`string.uuid = true`

```proto
// api/<domain>/v1/xxx.proto
message GetXxxRequest {
    string id = 1 [(validate.rules).string.uuid = true];
}
message UpdateXxxRequest {
    string id = 1 [(validate.rules).string.uuid = true];
}
message DeleteXxxRequest {
    string id = 1 [(validate.rules).string.uuid = true];
}
```

**为什么**：对外 ID 是 UUID（见 §四），校验时要校的是"符合 RFC 4122 的 UUID 格式"而不只是"长度 36"。PGV 内置 `uuid` 规则会校验 `8-4-4-4-12` hex + 连字符；`{len: 36}` 则会放过一堆合法长度但非法格式的字符串（例如 `"aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"` 或 `"xxxxxxxx-yyyy-zzzz-..."`），查询直接 DB miss，前端拿到 404 很困惑。

**什么不该写**：

```proto
// ❌ 只校长度
string id = 1 [(validate.rules).string = {len: 36}];

// ❌ 手写 UUID 正则，大小写 / 格式细节易错
string id = 1 [(validate.rules).string = {pattern: "^[0-9a-f]{8}-[0-9a-f]{4}-..."}];

// ❌ 不校验（依赖 DB 查询 miss 兜底）
string id = 1;
```

**好写法 vs 坏写法**：

```proto
// ✅ 好
string id = 1 [(validate.rules).string.uuid = true];

// ❌ 坏：只校长度
string id = 1 [(validate.rules).string = {len: 36}];
```

### 13.6 Update 路径事务化（方案 A）——取代 §五

```go
// biz 接口：签名返回最新 DO
type XxxRepo interface {
    // ...
    Update(ctx context.Context, h *Xxx) (*Xxx, error)
}

// biz usecase 透传
func (uc *XxxUsecase) Update(ctx context.Context, h *Xxx) (*Xxx, error) {
    return uc.repo.Update(ctx, h)
}

// data 实现：同事务内 UPDATE + First
func (r *xxxRepo) Update(ctx context.Context, h *biz.Xxx) (*biz.Xxx, error) {
    var po XxxPO
    err := r.data.DB.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        result := tx.Model(&XxxPO{}).
            Where("uuid = ?", h.ID).
            Updates(map[string]any{
                "name":     h.Name,
                // ... 所有业务字段
            })
        if result.Error != nil {
            if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
                return biz.ErrXxxNameConflict
            }
            return result.Error
        }
        if result.RowsAffected == 0 {
            return biz.ErrXxxNotFound
        }
        // 同事务内回读：拿到刚写入的行（含自动更新的 updated_at）
        return tx.Where("uuid = ?", h.ID).First(&po).Error
    })
    if err != nil {
        return nil, err
    }
    return toDO(&po), nil
}

// service 直接用返回值
updated, err := s.uc.Update(ctx, do)
if err != nil {
    return nil, toAPIError(err)
}
return toProto(updated), nil
```

**为什么**：朴素写法是 "data.Update 只返 error，service 拿到 OK 后再发一次 `GetByID` 回读"——但这两步是**跨事务**的：中间窗口别人可能删行或再改，客户端拿到的不是"我刚提交的状态"。事务化：

- 回读在**同事务内**，读到的必然是本次写入；
- 其他连接的并发修改被隔离；
- repo 返回 `(*DO, error)`，service 省一次 DB RTT 也省一层代码。

**什么不该写**：

```go
// ❌ service 层再发一次 GetByID（跨事务）
s.uc.Update(ctx, do)
updated, _ := s.uc.GetByID(ctx, do.ID)  // 中间窗口被别人改/删就读错

// ❌ 不回读，service 直接返回 客户端传进来的 do
s.uc.Update(ctx, do)
return toProto(do)  // 丢失 updated_at 等 DB 维护字段

// ❌ 用 Save(&po) 代替 Updates(map)
tx.Save(&po)  // Save 跳过零值（空串、0、false），丢失"空串也覆盖"语义

// ❌ Updates(struct) 同样丢零值
tx.Model(&po).Updates(XxxPO{Name: h.Name, Host: h.Host})  // 空串被跳过
```

**好写法 vs 坏写法**：

```go
// ✅ 好：事务内 Updates(map) + First 回读
db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    r := tx.Model(&PO{}).Where(...).Updates(map[string]any{...})
    if r.RowsAffected == 0 { return biz.ErrNotFound }
    return tx.Where(...).First(&po).Error
})

// ❌ 坏：跨事务回读
_ = db.Model(&PO{}).Where(...).Updates(...)
_ = db.Where(...).First(&po)
```

### 13.7 HTTP 响应 envelope：`{code, message, data}`

```go
// internal/server/http_encoder.go
type envelope struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data"`
}

func responseEncoder(w http.ResponseWriter, _ *http.Request, v interface{}) error {
    data := v
    if data == nil {
        data = struct{}{}   // 避免 "data": null
    }
    body, err := json.Marshal(envelope{Code: 0, Message: "ok", Data: data})
    if err != nil {
        return err
    }
    w.Header().Set("Content-Type", "application/json")
    _, err = w.Write(body)
    return err
}

// internal/server/http.go
http.NewServer(
    http.Middleware(...),
    http.ResponseEncoder(responseEncoder),   // 只改成功响应
    // 错误响应继续走 Kratos 默认 ErrorEncoder
)
```

**为什么**：团队前端约定所有成功响应是 `{code:0,message:"ok",data:<payload>}` 的固定结构，便于统一拦截器（loading、错误弹窗、data 解构）。Kratos 默认 encoder 直接返 payload，不符合约定。错误路径保持走 Kratos 默认 `ErrorEncoder`（输出 `{code,reason,message,metadata}`），两类响应各自独立、不互相干扰。

**什么不该写**：

```go
// ❌ 把 ErrorEncoder 也改掉，试图把错误塞进同一个 envelope
http.ErrorEncoder(customErrorEncoder)
// 会和 pb.ErrorXxx 生成的 *kratos.Error 结构冲突，前端也无法区分成功/失败

// ❌ 不处理 nil，直接让 "data":null 进响应
body, _ := json.Marshal(envelope{Data: v})  // v 是 nil 时炸前端

// ❌ 把错误手动包进成功 envelope
if err != nil {
    json.Marshal(envelope{Code: 500, Message: err.Error(), Data: nil})
}
// 丢失 Kratos 错误体的 reason / metadata 字段
```

**好写法 vs 坏写法**：

```go
// ✅ 好
http.NewServer(
    http.ResponseEncoder(responseEncoder),   // 只管成功
    // 错误: Kratos 默认
)

// ❌ 坏
http.NewServer(
    http.ResponseEncoder(responseEncoder),
    http.ErrorEncoder(customErrorEncoder),   // 破坏错误体结构
)
```

### 13.8 错误处理分层：biz 纯 sentinel + service `toAPIError`——取代 §六

#### 结构

```go
// biz：纯 stdlib sentinel，零框架依赖
package biz
import "errors"

var (
    ErrXxxNotFound     = errors.New("xxx not found")
    ErrXxxNameConflict = errors.New("xxx name conflict")
)
// biz 包内**禁止** import api/v1、禁止 import kratos/errors

// data：翻译 GORM 错误 → biz sentinel
if errors.Is(err, gorm.ErrRecordNotFound) {
    return nil, biz.ErrXxxNotFound
}
if errors.Is(err, gorm.ErrDuplicatedKey) {
    return nil, biz.ErrXxxNameConflict
}
return nil, err   // 未知错误透传

// service：集中翻译到对外协议错误（适配层职责）
func toAPIError(err error) error {
    switch {
    case err == nil:
        return nil
    case errors.Is(err, biz.ErrXxxNotFound):
        return pb.ErrorXxxNotFound("%s", err.Error())
    case errors.Is(err, biz.ErrXxxNameConflict):
        return pb.ErrorXxxNameConflict("%s", err.Error())
    default:
        return err
    }
}

// handler 失败路径统一包装
func (s *XxxService) GetXxx(ctx, in) (*pb.Xxx, error) {
    do, err := s.uc.GetByID(ctx, in.Id)
    if err != nil {
        return nil, toAPIError(err)
    }
    return toProto(do), nil
}
```

#### proto：加 `errors.options` 注解 + 生成器

```proto
import "errors/errors.proto";

enum XxxErrorReason {
    option (errors.default_code) = 500;

    XXX_UNSPECIFIED = 0;
    XXX_NOT_FOUND   = 1 [(errors.code) = 404];
    XXX_NAME_CONFLICT = 2 [(errors.code) = 409];
}
```

#### Makefile：装 `protoc-gen-go-errors` 并在 api target 生成

```makefile
init:
    go install github.com/go-kratos/kratos/cmd/protoc-gen-go-errors/v2@latest

api:
    protoc ... --go-errors_out=paths=source_relative:./api ...
```

`make api` 生成 `xxx_errors.pb.go`，含 `pb.ErrorXxxNotFound(...)` 构造器 + `pb.IsXxxNotFound(err)` 判断器。

**为什么**（完整论证见项目根目录 `20260422T092600 - 架构决策 - Kratos 错误处理规范（...）.md`）：

- **依赖倒置**：biz 是领域内核，不依赖任何外层（api / kratos/errors 都算外层）。Clean Architecture 允许 data → biz（Infra → Domain），不允许 biz → api。
- **跨协议复用**：biz 层只用 stdlib sentinel，CLI / Cron / MQ consumer 都能无痛复用；若 biz 用 `kratos.NotFound`，CLI 一个命令行工具凭什么知道 HTTP 404？
- **抽象泄露**：`*kratos.Error` 含 HTTP Code，biz 一旦返回它就已经在"假装不知道却已经知道"HTTP。
- **可测性**：biz 单测只要 `errors.Is(err, biz.ErrXxxNotFound)`，不需要 import kratos 或起 HTTP 服务器。
- **集中翻译**：`toAPIError` 让"新增错误 → 加 switch 分支"集中一处；分散到每个 handler 必漏。

**什么不该写**：

```go
// ❌ biz import api/v1
package biz
import v1 "xxx/api/xxx/v1"
var Err = errors.NotFound(v1.ErrorReason_XXX.String(), "...")  // 违反 DIP

// ❌ biz import kratos/errors
package biz
import "github.com/go-kratos/kratos/v2/errors"
var Err = errors.NotFound("XXX", "...")  // biz 污染框架

// ❌ biz 用 fmt.Errorf 不带 sentinel
return fmt.Errorf("xxx not found")  // 上游无法 errors.Is 判定

// ❌ data 裸抛 GORM 错误
if err := tx.First(&po).Error; err != nil { return nil, err }
// ErrRecordNotFound 会一路透传，客户端收到 500

// ❌ service 不 toAPIError 直接透传 biz sentinel
return nil, err   // biz.ErrXxxNotFound → kratos 包 500

// ❌ 每个 handler 手写 switch（散落，新增错误必漏）
if errors.Is(err, biz.ErrXxxNotFound) {
    return nil, pb.ErrorXxxNotFound("...")
}

// ❌ proto enum 不加 (errors.code)
XXX_NOT_FOUND = 1;   // 走 default_code 500，404 变成 500

// ❌ proto enum 0 值给业务错误
XXX_NOT_FOUND = 0;   // proto3 零值必须是 UNSPECIFIED
```

**好写法 vs 坏写法**：

```go
// ✅ 好：biz 纯 stdlib sentinel
package biz
import "errors"
var ErrXxxNotFound = errors.New("xxx not found")

// ❌ 坏：biz 耦合 v1 + kratos
package biz
import (
    "github.com/go-kratos/kratos/v2/errors"
    v1 "xxx/api/xxx/v1"
)
var ErrXxxNotFound = errors.NotFound(v1.ErrorReason_XXX.String(), "xxx not found")
```

```go
// ✅ 好：service 集中翻译
if err != nil {
    return nil, toAPIError(err)
}

// ❌ 坏：每个 handler 手写分支
if errors.Is(err, biz.ErrXxxNotFound) {
    return nil, pb.ErrorXxxNotFound("...")
}
return nil, err
```

#### 新增错误的流程（取代 §6.6）

1. **proto**：在 `XxxErrorReason` 枚举末尾追加，带 `[(errors.code) = N]`；
2. `make api` 生成 `ErrorXxx / IsXxx`；
3. **biz**：新增 `var ErrXxx = errors.New("...")` sentinel；
4. **data**：若是底层错误触发，在对应位置加翻译分支 `return biz.ErrXxx`；
5. **service**：在 `toAPIError` 的 switch 追加 `case errors.Is(err, biz.ErrXxx): return pb.ErrorXxx(...)`。

### 13.9 本节与 §五 §六 的关系速查

| 原章节 | 状态 | 替代 |
|---|---|---|
| §五 Update 非事务实现 | ⛔️ 过时 | §13.6 |
| §5.1 `UpdatedAt` 字段铁律 | ✅ 仍生效 | — |
| §5.2 `Updates(map)` 而非 struct | ✅ 仍生效（纳入 §13.6） | — |
| §六 biz 用 `kratos.NotFound(v1.XXX.String(), ...)` | ⛔️ 过时 | §13.8 |
| §6.1 proto enum 定义 | ✅ 仍生效，但要加 `(errors.code)` 注解 | §13.8 |
| §6.5 "哪些底层错误必须翻译" 对照表 | ✅ 仍生效 | — |

### 13.10 Checklist 增量（对 §十二 的补丁）

§十二 的新增资源 Checklist 仍然适用，只是要把第 4、6、8 步的**具体代码模板**换到本节：

- 第 4 步 biz：`var Err* = errors.New(...)`（纯 stdlib），而非 `errors.NotFound(v1.XXX.String(), ...)`；
- 第 6 步 data：Update 方法**签名 `(*DO, error)` + 实现事务化**（§13.6）；
- 第 8 步 service：新增 `toAPIError` 函数；所有 handler 失败路径统一 `return nil, toAPIError(err)`；
- 额外：`internal/server/http.go` 必须挂 `http.ResponseEncoder(responseEncoder)`（§13.7）；
- 额外：`internal/server/http.go` 和 `grpc.go` 的中间件链按 §13.3 顺序；validate 用 contrib 包（§13.4）；
- 额外：`internal/data/data.go` 的 `NewData` 开 `TranslateError: true`（§13.1）。

---

**维护者注**：本规范随项目演进持续更新。任何偏离都应在对应代码文件的注释里说明原因。
