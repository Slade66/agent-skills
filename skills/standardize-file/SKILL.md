---
name: standardize-file
description: 用于把文件的文件名、路径和文件头部的创建时间标准化。
---

# 文件标准化

### 操作步骤

1. 在 PowerShell 中运行 `Get-Date -Format "yyyy-MM-ddTHH:mm:sszzz"` 命令获取当前时间戳，将其填入文件头部 YAML frontmatter 的 `created` 字段。

2. 先将文件名与文件中的 H1 标题保持一致，再将其移动到根目录下的 `cards/` 文件夹。

### 注意事项

1. 你只能增加 YAML frontmatter、修改文件名和文件路径，不得删减原有内容。
