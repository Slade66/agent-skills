---
created: 2026-03-17T17:08:00+08:00
---

# `sql.Open`：根据驱动名和数据源信息，创建一个可复用的数据库连接池对象（`*sql.DB`）

## 函数签名

- `func Open(driverName, dataSourceName string) (*DB, error)`

- **所属包：** `database/sql`

### 参数

- `driverName string`：数据库驱动名称（例如 `mysql`、`sqlite3`）。

- `dataSourceName string`：数据源连接信息（DSN），用于描述如何连接数据库。

### 返回值

- `db *sql.DB`：数据库连接池对象，用于后续执行查询、事务等操作。

- `err error`：创建过程中的错误信息，若成功则为 `nil`。

## 代码示例

```go
package main

import (
    "context"
    "database/sql"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

func initDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    if err := db.PingContext(ctx); err != nil {
        db.Close()
        return nil, err
    }

    return db, nil
}
```

## 注意事项

1. **`sql.Open` 不会立刻建立连接：** `sql.Open` 只是准备好数据库连接池对象，并不代表已经成功连接数据库；建议使用 `Ping` 或 `PingContext` 验证连通性。

2. **`*sql.DB` 应该全局复用：** `*sql.DB` 本质上是连接池，通常在应用启动时创建一次，并注入到 repo / dao 中贯穿整个程序生命周期复用，不要在每次请求中重复调用 `sql.Open`。
