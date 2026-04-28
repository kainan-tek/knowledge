# Knowledge Config Schema

## Overview

Each project using the knowledge skill must have a `knowledge-config.yaml` in the working directory (where Claude Code is launched). The skill reads this file at startup to determine project-specific behavior.

## Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | (required) | Project name, displayed in `/knowledge status` |
| `description` | string | (required) | Short description, used in skill description field |
| `doc_dirs` | list of strings | `["../"]` | Directories to scan for documents, relative to working directory or absolute paths |
| `languages` | list | `[en, zh]` | Languages for bilingual topic generation. `[en, zh]` adds Chinese topic labels |
| `doc_id_pattern` | regex | `null` | Regex to extract document ID from filename. Must contain a capture group; first non-null capture group is the ID. Used for sub-index naming and dedup grouping |
| `dedup.enabled` | bool | `false` | Whether to detect duplicate/revision documents. Requires `doc_id_pattern` to be set |
| `dedup.revision_pattern` | regex | `null` | Regex to extract revision marker from filename (basename only, not full path). If capture group 1 is non-null, use it; otherwise use capture group 2. If both null, revision is null. Pattern example: `(?:_REV_([A-Z]+)_?|_(?:REV_)?([A-Z]{2})[ _])` matches `_REV_AA_` or `_AA ` in filenames like `80-VN500-6_REV_AA_xxx.pdf` |
| `dedup.revision_order` | string | `"none"` | `"alphabetical"` for AA<AB<AC ordering, `"none"` to skip ordering. Files with null revision are treated as oldest
| `index_root` | string | `"index/"` | Directory for index files, relative to working directory |
| `tmp_dir` | string | `"tmp/"` | Temp directory for PDF processing, relative to working directory |

## Behavior when config is missing

If `knowledge-config.yaml` is not found:
- `/knowledge init`: Run normally (creates config)
- `/knowledge status`: Show "Config not found. Run /knowledge init first."
- `/knowledge index`: Show "Config not found. Run /knowledge init first."
- `/knowledge <query>`: Show "Config not found. Run /knowledge init first."

## Minimal config example

```yaml
name: "my-project"
description: "My project PDF documents"
doc_dirs: ["../"]
languages: [en, zh]
doc_id_pattern: null
dedup:
  enabled: false
index_root: "index/"
tmp_dir: "tmp/"
```

## Qualcomm SA8295 config example

```yaml
name: "8295_qcom"
description: "SA8295 AudioReach PDF technical documents"
doc_dirs: ["../"]
languages: [en, zh]
doc_id_pattern: "^(80-[A-Z]{2}\\d{3}-\\d+)"
dedup:
  enabled: true
  revision_pattern: "(?:_REV_([A-Z]+)_?|_(?:REV_)?([A-Z]{2})[ _])"
  revision_order: alphabetical
index_root: "index/"
tmp_dir: "tmp/"
```
