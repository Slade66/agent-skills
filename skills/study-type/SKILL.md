---
name: study-type
description: 用于按照模板解释类型（结构体）的功能和用法。
---

### 操作步骤

1. 我会给你一个结构体，你需要按照模板来解释它。

2. 你需要在根目录下的 `cards` 文件夹中创建一个新的 markdown 文件，文件名为「包名.结构体名」（`sql.DB`），并将解释内容写入该文件中。

    - YAML 头部的 `created` 字段要在 PowerShell 里调用 `Get-Date -Format 'yyyy-MM-ddTHH:mm:ssK'`获取真实的时间。

3. 在原文档中留下像这样的内容链接：

```markdown
### `sql.DB`：表示「一个操作数据库资源的凭证 + 复用多个数据库底层连接的连接池」

[[type DB struct](sql.DB.md 的相对路径)]
```

### 模板

```markdown
# `sql.DB`：表示「一个操作数据库资源的凭证 + 复用多个数据库底层连接的连接池」

## 🧠 类型签名

- `type DB struct`

- **所属包：** `database/sql`

### 🧠 `DB` 是并发安全的

- 它可以安全地供多个 goroutine 并发使用，而不需要你自己加锁。

### 🧠 `DB` 应该全局复用

- 由于 `DB` 本质上是连接池，一个应用通常只需要创建一个实例：在应用启动时初始化，并注入到各层复用；程序退出时再调用 `Close()` 释放资源。不应该在每次请求中重复调用 `sql.Open` 创建新的 `DB`。
```
