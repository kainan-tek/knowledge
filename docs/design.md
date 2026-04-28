# Knowledge Skill Design

## Problem

Projects contain technical PDF documents that need efficient querying. Reading all documents each time is slow and wasteful of tokens.

## Solution

A configurable `knowledge` skill with two core subcommands (`index`, `status`) plus query and `init`. Project-specific behavior is defined in `knowledge-config.yaml`. The skill uses a **two-stage index architecture** — the main index enables document-level filtering; sub indexes provide chapter-level precision. This reduces context consumption by 75-85% compared to reading entire PDFs.

See [SKILL.md](../SKILL.md) for index formats, subcommand flows, and matching logic.
See [KNOWLEDGE-CONFIG-SCHEMA.md](KNOWLEDGE-CONFIG-SCHEMA.md) for config field definitions.

## Config-Driven Architecture

### Why config instead of hard-coding

The original implementation hard-coded Qualcomm SA8295 specific patterns (document ID regex `80-XX###-##`, version ordering AA<AB<AC, bilingual topics). Extracting these into `knowledge-config.yaml` allows:

1. **Zero-code adoption:** New projects run `/knowledge init` → `/knowledge index`, no file editing needed
2. **Domain flexibility:** Different document naming schemes (Qualcomm `80-XX###-##`, NXP `IMX-*`, TI `SPRU*`, internal `DOC-XXXX`) each define their own `doc_id_pattern`
3. **Clean separation:** Skill logic in SKILL.md is generic; project knowledge is in config + index data

### Config loading sequence

```
/knowledge <command>
  → Read CLAUDE.md (platform adapter)
  → Read SKILL.md (skill logic)
  → Read knowledge-config.yaml (project config)
  → Execute command with config-resolved paths and rules
```

If config is missing, only `init` works. All other subcommands fail with a clear message.

## Design Decisions

### Why Two-Stage Index

A single-file index (v1) combined all documents into one YAML (~3600 lines for 45 docs). Every query read the entire file even when only one document was relevant. The two-stage approach separates document-level metadata (main index, ~150 lines) from chapter details (sub indexes, ~200 lines each). A typical query reads main index + 2-3 sub indexes = ~350 lines.

### Why Config for Dedup

Different organizations use different document ID schemes. Qualcomm uses `80-XX###-##` with `_REV_AA_` revision markers. Other vendors may use entirely different patterns. The config's `doc_id_pattern` (for sub-index naming; must contain a capture group) and `dedup.revision_pattern` (for revision detection; uses if-else capture group logic) regexes let each project define their own scheme, while the ordering and marking logic stays generic.

### Why Bilingual Topics Are Configurable

Not all projects need Chinese topic labels. The `languages` config field defaults to `[en, zh]`. Projects that only need English can set `[en]`.

### Why page_count Is Redundant

`page_count = end_page - start_page + 1`. It's stored as a convenience field for batch planning without arithmetic. Writers must ensure consistency; if inconsistent, recompute from page range.

### Why index_status Uses Absence for Interrupted

When indexing is interrupted (session crash, user abort), no entry is written for the incomplete document. Using field absence to mean "interrupted" is natural — there's no moment to write `partial`. The two explicit values `complete` and `error` cover cases where we *can* write the status.

### Why Matching Strategy Is Score-Based

LLM matching without explicit priorities produces inconsistent results — sometimes matching on a vague summary, other times ignoring an exact function name hit. The scoring heuristic (`key_functions/structures` +4 > `key_concepts` +2 > `summary` +1) ensures identifier-precise queries consistently win over vague semantic matches.

## Efficiency Analysis

**Context consumption comparison:**

| Approach | Lines read per query | Token savings |
|----------|---------------------|---------------|
| No index | Read entire PDFs (100+ pages) | Baseline |
| Single-file index (v1) | ~3600 lines (45 docs combined) | ~50% vs baseline |
| **Two-stage index (v2)** | Main index ~150 + 2-3 sub indexes ~200 = ~350 lines | **75-85%** vs single-file |

**Example query "What is IMCL?":**
- Stage 1: Read 150-line main index → match doc 80-VN500-6
- Stage 2: Read 200-line sub index → match ch6 (pages 48-49)
- Stage 3: Read 2 PDF pages (not 203)

## Scope

Per-project (configured via `knowledge-config.yaml`), PDFs only (first phase). Reusable across projects by running `/knowledge init` and `/knowledge index`.