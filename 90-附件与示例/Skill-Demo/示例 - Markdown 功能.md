---
title: Obsidian Markdown 功能示例
tags:
  - demo
  - markdown
aliases:
  - Markdown 演示
cssclasses:
  - demo-note
updated: 2026-10-06
---

# Obsidian Markdown 功能示例

## 双向链接

可以链接到已有笔记：[[函数|Python 函数笔记]]，也可以链接到某个标题：[[数据结构#列表|查看列表章节]]。

## Callout

> [!note] 说明
> Obsidian Callout 用于突出提示、警告、示例和总结。

> [!warning] 注意
> 这是警告样式。

> [!question]- 点击展开问题
> Markdown 支持哪些 Obsidian 扩展？

## 文本格式

这是 ==高亮文本==，这是 **粗体**，这是 *斜体*。

## 嵌入

嵌入一张已有图片：

![[Pasted image 20260812230009.png|300]]

## 任务列表

- [x] 创建演示笔记
- [ ] 试用 Obsidian CLI 搜索
- [ ] 创建自己的 Canvas

## Mermaid 图

```mermaid
graph LR
    A[网页] --> B[Defuddle]
    B --> C[Knap 模板]
    C --> D[Obsidian 笔记]
```

## 数学公式

行内公式：$E = mc^2$

块级公式：

$$
\int_0^1 x^2 \, dx = rac{1}{3}
$$

## 脚注

这里有一个脚注[^demo]。

[^demo]: 脚注内容会显示在页面底部。
