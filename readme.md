---
type: Note
category: "[[vault-guides]]"
---
# README

这是一个基于 Tolaria 的个人知识库，所有内容以本地 Markdown 文件形式存储，并通过 wikilink 组织成知识图谱。

从 [[knowledge-base-index|知识库总索引]] 开始浏览全部内容。

## 结构说明

- 根目录中的 `.md` 文件：普通笔记或类型定义
- `views/`：保存的视图配置
- `attachments/`：附件资源文件
- `category` 属性：笔记所属的主题分类

## 约定

### 笔记

每篇笔记都是一个 Markdown 文件，推荐使用 kebab-case 文件名，例如：

- `my-note.md`
- `project-plan.md`

笔记通常包含：

- YAML frontmatter
- 正文内容
- 第一个 H1 作为显示标题

示例：

```markdown
---
type: Note
status: Active
---

# 我的笔记

这里是正文内容。
```
