---
name: extract-card
description: 当
---

# 从文档中提取出知识卡片

这个 skill 用于帮助你从一篇长文档中提取出选定的一段文字，按照规范把它做成一张知识卡片，并保存到指定的目录下。

### 卡片的格式规范

- date：绝不允许伪造或推算时间。你必须在后台执行系统命令 `powershell -Command "Get-Date -Format 'yyyy-MM-ddTHH:mm:ssK'"` 获取本机真实的 ISO-8601 时间（带时区），并将其填入 YAML 的 `date` 字段中。

- [知识卡片的模板](examples/知识点卡片的模板.md)

### 卡片的保存位置

在根目录的 cards/ 文件夹下。
