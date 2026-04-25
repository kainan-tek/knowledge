# Knowledge Skill

[中文文档](README_zh.md) | English

An AI-powered Q&A tool for local PDF technical documents. Uses a two-stage index to locate relevant chapters, then AI reads only the matching pages to answer questions — avoiding full-document reads and saving token consumption.

## Prerequisites

**Requires Claude Code**: Launch Claude Code in the `knowledge/` directory to use `/knowledge` commands.

Install any ONE PDF tool (auto-detected in priority order):

| Tool | Install | Notes |
|------|---------|-------|
| pdfplumber | `pip install pdfplumber` | Recommended — best TOC extraction |
| pypdf | `pip install pypdf` | Alternative |
| Poppler | [Download](https://github.com/oschwartz10612/poppler-windows/releases) | Provides pdftotext/pdftoppm |

The tool is auto-detected on first `/knowledge index`. If none is installed, you'll be prompted to install one.

## Quick Start (New Project)

1. Copy or place the `knowledge/` directory anywhere
2. `cd knowledge/` and launch Claude Code there
3. Run `/knowledge init` and follow the prompts (set `doc_dirs` to point to your documents, default `../`)
4. Run `/knowledge index` to build the index
5. Query with `/knowledge <question>`

## Commands

| Command | Description |
|---------|-------------|
| `/knowledge init` | Initialize project config (first time setup) |
| `/knowledge index` | Build or update full index |
| `/knowledge index <filename>` | Index a single document (supports partial match) |
| `/knowledge reindex <filename>` | Force re-index a document |
| `/knowledge status` | Show index health status |
| `/knowledge <question>` | Query document content |

## Configuration

Project config is at `knowledge-config.yaml` (in the working directory). Key fields:

| Field | Description | Default |
|-------|-------------|---------|
| `name` | Project name | — |
| `description` | Project description | — |
| `doc_dirs` | Directories to scan for documents (relative to working directory) | `["../"]` |
| `languages` | Topic label languages | `[en, zh]` |
| `doc_id_pattern` | Document ID regex (must include a capture group; used for sub-index naming and dedup grouping) | — |
| `dedup.enabled` | Enable duplicate/revision detection | `false` |
| `dedup.revision_pattern` | Revision marker extraction regex (matches filename basename only) | — |
| `dedup.revision_order` | Revision ordering method (`alphabetical` or `none`) | `"none"` |
| `index_root` | Index files root directory | `"index/"` |
| `tmp_dir` | Temp directory (used for Unicode filename processing) | `"tmp/"` |

See [KNOWLEDGE-CONFIG-SCHEMA.md](docs/KNOWLEDGE-CONFIG-SCHEMA.md) for full field definitions.

## Example Output

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

... (processing remaining files)

Duplicate detection:
  80-PM164-53 → superseded by 80-PM164-53_REV_AA
  80-PM164-54 → superseded by 80-PM164-54_REV_AB

Done: 43 indexed, 4 superseded, 0 errors
```

**Note:** Full indexing of many PDFs takes time and consumes tokens. If interrupted, completed indexes are preserved — re-running skips already-indexed unchanged files.

### `/knowledge index <filename>`

```
Matching "VN500-7"... found: 80-VN500-7_AudioReach LA Architecture.pdf
Indexing...
  TOC: 8 chapters extracted
  Sub index written: index/80-VN500-7.yaml
  Main index updated
Done
```

If partial match finds multiple files, you'll be prompted to choose.

### `/knowledge reindex <filename>`

```
Matching "VN500-7"... found: 80-VN500-7_AudioReach LA Architecture.pdf
Re-indexing...
  TOC: 8 chapters extracted
  Sub index written: index/80-VN500-7.yaml
  Main index updated
Done
```

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

### `/knowledge <question>`

Ask in natural language. The skill locates relevant document chapters and answers based on original content.

## FAQ

**Q: How to use in a new project?**

Copy the `knowledge/` directory anywhere, `cd knowledge/`, launch Claude Code, then run `/knowledge init` and set `doc_dirs` to point to your documents.

**Q: "Config not found" error?**

Run `/knowledge init` to generate the config file.

**Q: "No PDF tool found" error?**

Install any one tool:
```
pip install pdfplumber
```

**Q: Indexing was interrupted?**

Completed indexes are preserved. Just re-run `/knowledge index` — already-indexed unchanged files are skipped.

**Q: Query results are inaccurate or not found?**

- Try more specific technical terms
- If index is incomplete, run `/knowledge index` first
- Run `/knowledge status` to check index health

## More Docs

- [Design Document](docs/design.md) — Architecture and design decisions
- [Config Schema](docs/KNOWLEDGE-CONFIG-SCHEMA.md) — Full config field definitions
