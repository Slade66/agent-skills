---
name: study-func
description: 用于按照模板解释函数的功能和用法。
---

### 操作步骤

1. 我会给你一个函数，你需要按照这个 [函数模板](examples/sql.Open.md) 来解释这个函数的功能、用法、注意事项。

2. 你需要在根目录下的 `cards` 文件夹中创建一个新的 markdown 文件，文件名为「包名.函数名」，并将解释内容写入该文件中。

    - YAML 头部的 `created` 字段要在 PowerShell 里调用 `Get-Date -Format 'yyyy-MM-ddTHH:mm:ssK'`获取真实的时间。

3. 在原文档中留下像这样的内容链接：

```markdown
### `sql.Open`：根据驱动名和数据源信息，创建一个可复用的数据库连接池对象（`*sql.DB`）

[[func Open(driverName, dataSourceName string) (*DB, error)](sql.Open.md 的相对路径)]
```
