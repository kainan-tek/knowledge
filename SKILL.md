---
name: knowledge
description: Query and manage a project PDF knowledge base with indexed access. Triggers on /knowledge commands or when users ask about project documents.
---

# Knowledge Skill

Manage a project's PDF technical documents with indexed access — index acts as locator, telling which document and chapter to read, then read actual PDF content. Avoid reading entire documents wastefully. Project-specific behavior is defined in `knowledge-config.yaml` (located in the working directory where Claude Code is launched).

## Configuration

**Before any subcommand (except `init`):** Read `knowledge-config.yaml` (relative to working directory). If the file does not exist, tell the user to run `/knowledge init` first. Resolve all `{config.*}` references from the config fields. Key conventions: `{config.doc_id_pattern}` must contain a capture group. `{INDEX_ROOT}` = `{config.index_root}` (default: `index/`). `{TMP_DIR}` = `{config.tmp_dir}` (default: `tmp/`). See [KNOWLEDGE-CONFIG-SCHEMA.md](docs/KNOWLEDGE-CONFIG-SCHEMA.md) for field definitions.

## Prerequisites

**Tool (install any ONE, auto-detected in this priority order):**
1. `pdfplumber` - `pip install pdfplumber` (best TOC extraction)
2. `pypdf` - `pip install pypdf` (text + metadata)
3. `pdftotext/pdftoppm` - Poppler (Windows: download from https://github.com/oschwartz10612/poppler-windows/releases)

**No tool available:** If none of the above are installed, stop and tell user to install one. Cannot proceed with indexing or reading until a tool is installed.

**Unicode filenames (Windows):** Copy to temp with ASCII name, cleanup after. If script output fails with encoding error, write to temp file and use Read tool.

## Architecture

```
knowledge/                  # Launch Claude Code here (requires .claude/commands/)
├── SKILL.md                  # This file — universal skill definition
├── CLAUDE.md                 # Claude Code entry guide
├── README.md                 # Getting started guide (English)
├── README_zh.md              # Getting started guide (中文)
├── knowledge-config.yaml     # Project configuration (created by init)
├── docs/
│   ├── design.md             # Detailed design document
│   └── KNOWLEDGE-CONFIG-SCHEMA.md  # Config schema reference
├── .claude/
│   └── commands/
│       └── knowledge.md      # Slash command entry point
├── index/
│   ├── knowledge-main.yaml   # Main index (doc-level summaries)
│   ├── 80-VN500-6.yaml       # Sub-indexes (chapter details, one per document)
│   └── ...
└── tmp/                      # Temp files for processing (Unicode filenames)
```

`{INDEX_ROOT}` resolves from `{config.index_root}` (default: `index/`). All paths in config are relative to the working directory where Claude Code is launched.

## Subcommands

### `/knowledge init`

Initialize or manage `knowledge-config.yaml` in the working directory.

**Steps:**
1. Check if `knowledge-config.yaml` already exists. If yes, tell user: "Config already exists. Edit it directly or delete it and re-run `/knowledge init`." Then stop.
2. Check PDF tool prerequisites (pdfplumber, pypdf, pdftotext). If none is installed, tell user which tools are available and how to install them, then stop.
3. Ask user for project name, description, and document directories (`doc_dirs`). Default: `["../"]`.
4. Write `knowledge-config.yaml` with provided values and sensible defaults (`doc_id_pattern: null`, `dedup.enabled: false`, `languages: [en, zh]`, `index_root: "index/"`, `tmp_dir: "tmp/"`).
5. Show the generated config to user. Suggest: "Run `/knowledge index` to start indexing — it will auto-detect document ID patterns and suggest dedup settings. You can also edit `knowledge-config.yaml` manually — see `docs/KNOWLEDGE-CONFIG-SCHEMA.md`."

### `/knowledge index [<filename>]`

**Without filename:** Scan → Dedup → Compare → Extract → Write

**With filename:** Extract → Write (partial match, overwrite existing)

**Steps (full scan):**
1. Ensure `{TMP_DIR}` exists. Scan `{config.doc_dirs}` recursively (Glob `**/*.pdf`), record mtime. Exclude `{INDEX_ROOT}` and `{TMP_DIR}`. Auto-detect if `doc_id_pattern` is null: infer pattern, update config.
2. Dedup (if `dedup.enabled`): group by document ID, extract revision via `revision_pattern` (basename only; group 1 else group 2 else null), order by `revision_order`, keep newest only. Set `revision` and `supersedes` fields. Report skipped files.
3. Compare with main index: unchanged skip, new/modified extract.
4. **TOC:** ① PDF metadata → ② Contents page scan → ③ Heading pattern (fallback).
5. Write sub-index `{document_id}.yaml`: chapters with page ranges, summary, key_concepts. Document ID: capture group from `doc_id_pattern`, else sanitized slug.
6. Write main index entry. `topics`: add Chinese labels if `{config.languages}` has `zh`. `all_key_concepts`: multi-word always; single-word only if unique to this doc. Set `index_status: complete`. On error: `index_status: error`.

**Steps (single file):** Ensure `{TMP_DIR}` exists, then steps 4–6 above. Overwrite existing entries.

### `/knowledge reindex <filename>`

Force re-index: match filename (partial ok, multi-match asks user) → re-extract → update mtime → update `tool_used` in main index → write sub index then main index

### `/knowledge status`

1. List PDFs in `doc_dirs`, read main index, compare counts
2. Show indexed docs: `id`, `revision`, `index_status`
3. List: unindexed PDFs, entries with missing/error status, superseded files
4. Validate sub-index refs, auto-repair inconsistent states
5. Summary: X/Y indexed, Z errors, W superseded

### `/knowledge <question>`

**Flow:** Read main index → Match docs → Read relevant sub indexes → Match chapters → Read PDF pages → Answer

**Matching strategy:**
1. Extract key terms from question (technical terms, acronyms, identifiers). Case-insensitive substring match.
2. Stage 1 (main index): `topics` +2, `chapter_titles` +1, `summary` +1. Select top 5-10 docs.
3. Stage 2 (main index, on Stage 1 results): `all_key_concepts` +3. Select top 1-3 docs.
4. Stage 3 (sub-index, per doc): `key_functions/structures` +4, `key_concepts` +2, `summary` +1. Select matching chapters.
5. Read PDF pages by `start_page/end_page`. Split into 20-page batches if large.

**No match:** Show partial matches, suggest keywords, offer wider page ranges.

**Completeness:** Skip docs with `index_status: error`, warn user and suggest `/knowledge reindex`.


## Index Formats

### Main Index (knowledge-main.yaml)

```yaml
version: 2
generated: "2026-04-24"
tool_used: "pdfplumber"

documents:
  - id: "80-PM164-51"                          # document_id (from doc_id_pattern)
    file: "80-PM164-51_SA8295 HQX Audio Bringup Guide.pdf"
    title: "SA8295 HQX Audio Bringup Guide"
    mtime: "2023-02-24T10:10:18"
    summary: "Complete audio bringup guide..."
    topics: [SA8295, audio bringup, SA8295音频调试]
    all_key_concepts: [audio_service, agmplay, agmcap]
    chapter_titles: ["1. Introduction", "2. Software architecture"]
    revision: null                              # null when filename doesn't match revision_pattern
    supersedes: []                              # list of superseded revision markers
    index_status: complete
    sub_index: "80-PM164-51.yaml"

  - id: "80-VN500-6"                            # document_id (same across all revisions)
    file: "80-VN500-6_REV_AB_AudioReach SPF CAPI API Reference.pdf"  # newest file
    title: "AudioReach SPF CAPI API Reference"
    mtime: "2023-02-24T10:10:18"
    summary: "Complete API reference for CAPI module development..."
    topics: [CAPI, API reference, CAPI接口]
    all_key_concepts: [process callback, EOF/EOS, IMCL]
    chapter_titles: ["1. Introduction", "4. Functional Description"]
    revision: "AB"                              # extracted from filename
    supersedes: ["AA"]                          # revision markers of superseded files
    index_status: complete
    sub_index: "80-VN500-6.yaml"
```

**Required:** `id`, `file`, `mtime`, `summary`, `topics`, `chapter_titles`, `all_key_concepts`, `index_status`, `sub_index`

**Optional:** `title`, `revision`, `supersedes`

### Sub Index (80-VN500-6.yaml)

```yaml
version: 2
document_id: "80-VN500-6"
file: "80-VN500-6_REV_AB_AudioReach SPF CAPI API Reference.pdf"
generated: "2026-04-24"
tool_used: "pdfplumber"

chapters:
  - id: ch4
    title: "4. Functional Description"
    start_page: 27
    end_page: 45
    page_count: 19
    summary: "Core process function mechanics, buffer handling models (buffered vs non-buffered), EOF/EOS handling, threshold-based processing, event handling during process"
    key_concepts: [process callback, EOF/EOS, threshold, buffering model, data flow, input/output buffers, stream data flags]
    key_functions: [capi_vtbl_t::process]
    key_structures: [capi_stream_data_t, capi_stream_data_v2_t, capi_buf_t]
```

**Required:** `id`, `start_page`, `end_page`, `summary`, `key_concepts`

**Optional:** `title`, `page_count`, `key_functions`, `key_structures`

## Scope

Per-project (configured via `knowledge-config.yaml`), PDFs only (first phase). Reusable across projects by running `/knowledge init` and `/knowledge index`.
