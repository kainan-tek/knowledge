# CLAUDE.md — Knowledge Skill Platform Adapter

## Knowledge Skill

A generic skill for querying and managing PDF technical document knowledge bases. Project-specific behavior is defined in `knowledge-config.yaml`. Must launch Claude Code from the `knowledge/` directory (requires `.claude/commands/` for slash commands). `doc_dirs` in config specifies where to find documents — the directory itself can be placed anywhere.

## Configuration Loading

On skill activation (except `/knowledge init`):
1. Read `knowledge-config.yaml` (relative to working directory)
2. If not found, tell user: "Config not found. Run `/knowledge init` first."
3. Resolve paths from config:
   - `{INDEX_ROOT}` = `{config.index_root}` (default: `index/`)
   - Sub-index paths: `{INDEX_ROOT}` + `sub_index` value
   - `{TMP_DIR}` = `{config.tmp_dir}` (default: `tmp/`)
   - Doc scan dirs: `{config.doc_dirs}` (relative to working directory)

## Structure

- [SKILL.md](SKILL.md) - Main skill entry point with subcommand definitions, index formats, and matching logic
- [KNOWLEDGE-CONFIG-SCHEMA.md](docs/KNOWLEDGE-CONFIG-SCHEMA.md) - Config schema reference
- [docs/design.md](docs/design.md) - Detailed design document
- [README.md](README.md) - Getting started guide
- [index/](index/) - Index data: `knowledge-main.yaml` (main) + `*.yaml` (per-document sub-indexes)

## Platform Adapter for Claude Code

- **Tool mapping:**
  - Read file → `Read` tool
  - List files → `Glob` tool
  - Search content → `Grep` tool
  - Execute command → `Bash` tool
  - Read PDF pages → `Read` tool (with `pages` parameter)
- **Skill entry point:** `.claude/commands/knowledge.md` (loaded by Claude Code slash command system)

## Using This Skill

When the user asks about project documents, PDFs, or runs `/knowledge` commands, load [SKILL.md](SKILL.md) and follow its instructions. All paths are resolved from `knowledge-config.yaml`.
