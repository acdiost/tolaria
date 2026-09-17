---
type: Note
status: Active
category: "[[vault-guides]]"
---
# Tolaria 入门指南

Tolaria 是一个基于 Markdown 文件的个人知识管理应用，所有内容以纯文本形式存储在本地文件夹中，形成一个可互联的知识图谱。

## 核心概念

### 笔记（Note）

每个 `.md` 文件就是一篇笔记。文件名使用 kebab-case 格式，例如 `my-first-note.md`。笔记由两部分组成：

- **Frontmatter**：文件顶部 `---` 之间的 YAML 元数据，用于存储类型、状态、关系等属性
- **正文**：标准 Markdown 内容，第一个 H1 标题作为笔记的显示标题

### 类型（Type）

每篇笔记都有一个 `type:` 属性，用于分类管理。类型本身也是笔记，存放在 vault 根目录，`type` 字段值为 `Type`。

常见内置类型包括 `Note`、`Type` 等，你也可以自定义类型。

### 关系（Relationship）

在 frontmatter 中使用 [[wikilink]] 语法建立笔记之间的关联：

```yaml
related_to: "[[tolaria]]"
belongs_to: "[[my-project]]"
```

正文中也可以使用 [[笔记标题]] 跳转到其他笔记。

## 快速上手

### 1. 创建第一篇笔记

新建 `my-note.md`，写入：

```markdown
---
type: Note
status: Active
---

# 我的第一篇笔记

在这里写内容……
```

### 2. 定义自定义类型

在 vault 根目录新建类型文件，例如 `project.md`：

```yaml
---
type: Type
_icon: rocket
_color: "#3b82f6"
---

# Project
```

### 3. 创建保存的视图

在 `views/` 文件夹中新建 `active-notes.yml`：

```yaml
name: 活跃笔记
sort: modified:desc
filters:
  all:
    - field: type
      op: equals
      value: Note
    - field: status
      op: equals
      value: Active
```

## 常用 Frontmatter 属性

| 属性           | 说明                       |
| ------------ | ------------------------ |
| `type`       | 笔记类型，必填                  |
| `status`     | 状态，如 `Active`、`Archived` |
| `related_to` | 关联笔记（wikilink）           |
| `url`        | 外部链接                     |

## 文件结构概览

```text
vault/
├── note.md          # Note 类型定义
├── type.md          # Type 类型定义
├── my-first-note.md # 普通笔记
├── views/
│   └── my-view.yml  # 保存的视图
└── attachments/     # 附件资源（不作为笔记处理）
```

## 小贴士

- Frontmatter 以 `_` 开头的属性（如 `_icon`、`_color`）由 Tolaria 管理，一般不手动修改
- 笔记正文的第一个 H1 优先作为显示标题，无需在 frontmatter 中重复写 `title:`
- 视图文件名使用 kebab-case，文件名即视图的稳定 ID
