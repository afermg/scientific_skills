# Scientific Skills

A collection of Claude Code skills for reproducible scientific data analysis and notebook composition.

## Skills

| Skill | Description |
|---|---|
| [aliby-pipeline](aliby-pipeline/) | Run the [aliby](https://github.com/afermg/aliby) image analysis pipeline with [nahual](https://github.com/afermg/nahual) cellpose/trackastra GPU servers for segmentation, tracking, and feature extraction |
| [compose-notebook](compose-notebook/) | Compose marimo notebooks from the aliby catalog to build microscopy image analysis pipelines (segmentation, feature extraction, deep learning embeddings) |
| [org-duckdb-analysis](org-duckdb-analysis/) | Create Org Mode literate notebooks that use DuckDB SQL queries via org-babel to explore and analyze tabular data (CSV, Parquet) inside Emacs |

## Installation

Add this repository as a skill source in your Claude Code configuration:

```bash
claude mcp add-skill-source scientific_skills /path/to/scientific_skills
```

Or copy individual skill directories into `~/.claude/skills/`.

## Adding a new skill

Each skill lives in its own directory with a `SKILL.md` file containing YAML frontmatter (`name`, `description`) and markdown instructions. See existing skills for the expected format.
