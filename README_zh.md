# Knowledge Skill

中文文档 | [English](README.md)

PDF 本地技术文档知识库 AI 问答工具。通过两级索引定位到具体章节，AI 直接读最相关页面并回答问题，避免全文阅读，节省 token 消耗。

## 前置条件

**需要 Claude Code**：在 `knowledge/` 目录下启动 Claude Code 才能使用 `/knowledge` 命令。

使用前需要安装 PDF 工具（任选其一）：

| 工具 | 安装方式 | 说明 |
|------|----------|------|
| pdfplumber | `pip install pdfplumber` | 推荐，TOC 提取效果最好 |
| pypdf | `pip install pypdf` | 备选 |
| Poppler | [下载](https://github.com/oschwartz10612/poppler-windows/releases) | 提供 pdftotext/pdftoppm |

首次使用 `/knowledge index` 时会自动检测可用工具。如未安装任何工具，会提示并停止。

## 快速开始（新项目）

1. 将 `knowledge/` 目录放置到任意位置
2. `cd knowledge/` 并在此目录启动 Claude Code
3. 运行 `/knowledge init`，按提示生成配置文件（设置 `doc_dirs` 指向文档目录，默认 `../`）
4. 运行 `/knowledge index`，建立索引
5. 用 `/knowledge <问题>` 查询

## 命令一览

| 命令 | 用途 |
|------|------|
| `/knowledge init` | 初始化项目配置（新项目首次使用） |
| `/knowledge index` | 建立或更新全部索引 |
| `/knowledge index <文件名>` | 索引或重新索引指定文档（部分匹配，覆盖已有） |
| `/knowledge status` | 查看索引健康状态 |
| `/knowledge <问题>` | 查询文档内容 |

## 配置说明

项目配置文件位于当前工作目录下的 `knowledge-config.yaml`，主要字段：

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `name` | 项目名称 | — |
| `description` | 项目描述 | — |
| `doc_dirs` | 文档扫描目录列表（相对于工作目录） | `["../"]` |
| `languages` | 主题标签语言 | `[en, zh]` |
| `doc_id_pattern` | 文档 ID 提取正则（需包含捕获组，用于子索引命名和去重分组） | — |
| `dedup.enabled` | 是否启用重复文档检测 | `false` |
| `dedup.revision_pattern` | 版本标记提取正则（仅匹配文件名，不含路径） | — |
| `dedup.revision_order` | 版本排序方式（`alphabetical` 或 `none`） | `"none"` |
| `index_root` | 索引文件根目录 | `"index/"` |
| `tmp_dir` | 临时文件目录（Unicode 文件名处理时使用） | `"tmp/"` |

详细配置说明见 [KNOWLEDGE-CONFIG-SCHEMA.md](docs/KNOWLEDGE-CONFIG-SCHEMA.md)。

## 输出示例

### `/knowledge init`

```
Project name [my-project]: 8295_qcom
Project description: SA8295 AudioReach PDF technical documents
Document directories (doc_dirs) ["../"]:

Config written to knowledge-config.yaml
Run /knowledge index to start indexing — it will auto-detect document ID patterns and suggest dedup settings.
```

### `/knowledge index`

```
Tool detected: pdfplumber
Scanning PDFs in ["../"] ... 43 files found
Comparing with main index... 42 new/modified, 1 unchanged

Indexing 80-VN500-7_AudioReach LA Architecture.pdf...
  TOC: 8 chapters extracted
  Sub index written: index/80-VN500-7.yaml
  Main index updated

... (处理其余文件)

Duplicate detection:
  80-PM164-53 → superseded by 80-PM164-53_REV_AA
  80-PM164-54 → superseded by 80-PM164-54_REV_AB

Done: 43 indexed, 4 superseded, 0 errors
```

**注意：** 大量 PDF 全量索引需要较长时间，消耗较多 token。如果中途断开，已完成的文档索引会保留，下次执行会跳过（增量索引）。

### `/knowledge index <文件名>`

```
Matching "VN500-7"... found: 80-VN500-7_AudioReach LA Architecture.pdf
Indexing...
  TOC: 8 chapters extracted
  Sub index written: index/80-VN500-7.yaml
  Main index updated
Done
```

如果部分匹配到多个文件，会列出让你选择。

### `/knowledge status`

```
Knowledge Base Status (8295_qcom)
=====================
Indexed: 1/43 PDFs

Indexed documents:
  80-VN500-6  index_status: complete

Unindexed PDFs (42):
  80-VN500-3_AudioReach Signal Processing.pdf
  ...

Superseded: 0
Errors: 0
Orphan sub-index files: 0

Summary: 1/43 indexed, 0 incomplete/error, 0 superseded, 0 orphan files
```

### `/knowledge <问题>`

用自然语言提问，自动定位到相关文档的章节并基于原文回答。

## 常见问题

**Q: 如何在新项目中使用？**

将 `knowledge/` 目录放置到任意位置，`cd knowledge/`，启动 Claude Code，然后运行 `/knowledge init` 并设置 `doc_dirs` 指向文档目录。

**Q: 提示 "Config not found" 怎么办？**

运行 `/knowledge init` 生成配置文件。

**Q: 提示 "No PDF tool found" 怎么办？**

安装任一工具即可：
```
pip install pdfplumber
```

**Q: 索引过程中断了怎么办？**

已完成的索引会保留。直接重新执行 `/knowledge index`，已索引且未修改的文件会自动跳过。

**Q: 查询结果不准或没找到？**

- 尝试用更具体的技术术语
- 如果索引不完整，先执行 `/knowledge index` 补全
- 执行 `/knowledge status` 检查索引健康状态

## 更多文档

- [设计文档](docs/design.md) — 架构设计和决策说明
- [配置 Schema](docs/KNOWLEDGE-CONFIG-SCHEMA.md) — 配置字段详细定义
