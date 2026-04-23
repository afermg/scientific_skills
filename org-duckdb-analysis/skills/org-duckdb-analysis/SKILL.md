---
name: org-duckdb-analysis
description: Create an Org Mode literate notebook that uses DuckDB SQL queries via org-babel to explore and analyze CSV/Parquet data files. Trigger whenever the user wants to analyze tabular data (CSV, Parquet, TSV) using SQL inside Emacs, asks for exploratory data analysis in org-mode, wants a reproducible data notebook, or mentions DuckDB with org-babel. Also trigger when the user says "analyze these CSVs", "get statistics on this data", "explore this dataset", or asks for a literate SQL analysis — even if they don't explicitly mention DuckDB or org-mode.
---

# Org-mode Reproducible Analysis with DuckDB

## What this skill is for

When a user has tabular data files (CSV, Parquet, TSV) and wants to explore them
interactively inside Emacs, the right approach is an **Org Mode literate
notebook** with DuckDB SQL blocks executed via org-babel. Each heading poses a
question about the data, followed by a SQL query that answers it. The results
appear inline as formatted blocks, making the document both the analysis and
its report.

This skill uses [ob-duckdb](https://github.com/gggion/ob-duckdb), which
provides native DuckDB support for org-babel with persistent sessions, async
execution, and multiple output formats. The session runs as a live DuckDB
process in an Emacs buffer, so tables created in one block are immediately
available in subsequent blocks.

## Prerequisites

- **Emacs 30.1+** with a running server (`M-x server-start`)
- **DuckDB CLI** installed and on `$PATH` (`duckdb --version` to verify)
- **ob-duckdb** (installed automatically — see below)
- The emacs-pair skill scripts at `~/.claude/skills/emacs-pair/scripts/`

## Step-by-step process

### 1. Discover the data

List the files in the target directory. Read the first few lines of each CSV or
Parquet file to understand column names, types, and any quirks (e.g., columns in
different order across files, date formats, NULL representations).

Pay attention to:
- Column name differences between files that describe the same concept
- Column ordering inconsistencies (e.g., `lat, long` vs `long, lat`)
- Timestamp formats and timezone info
- Categorical columns (inspect distinct values later via queries)

### 2. Install ob-duckdb in Emacs

Before creating the org file, ensure `ob-duckdb` is available. Evaluate this
via `eval-elisp.sh`:

```elisp
(unless (featurep 'ob-duckdb)
  (if (fboundp 'straight-use-package)
      (straight-use-package '(ob-duckdb :host github :repo "gggion/ob-duckdb"))
    (progn
      (require 'package)
      (unless (assoc "melpa" package-archives)
        (add-to-list 'package-archives '("melpa" . "https://melpa.org/packages/") t))
      (package-refresh-contents)
      (package-install 'ob-duckdb)))
  (require 'ob-duckdb)
  (org-babel-do-load-languages
   'org-babel-load-languages
   (append org-babel-load-languages '((duckdb . t)))))
```

### 3. Structure the org file

The org file should start with a title, an auto-install block for
reproducibility, and a property line:

```org
#+TITLE: <Descriptive Title>
#+begin_src emacs-lisp :results silent :exports none
;;; Install ob-duckdb if not available
(unless (featurep 'ob-duckdb)
  (if (fboundp 'straight-use-package)
      (straight-use-package '(ob-duckdb :host github :repo "gggion/ob-duckdb"))
    (progn
      (require 'package)
      (unless (assoc "melpa" package-archives)
        (add-to-list 'package-archives '("melpa" . "https://melpa.org/packages/") t))
      (package-refresh-contents)
      (package-install 'ob-duckdb)))
  (require 'ob-duckdb)
  (org-babel-do-load-languages
   'org-babel-load-languages
   (append org-babel-load-languages '((duckdb . t)))))
#+end_src
#+PROPERTY: header-args:duckdb :session <name> :db /tmp/<name>.duckdb :format markdown :results output example
```

The auto-install block ensures anyone opening the notebook can execute it
without manual setup — they just run the first block.

The file-level `#+PROPERTY` line sets defaults for every `duckdb` block:
- `:session <name>` — runs a persistent DuckDB process in an Emacs buffer
  (`*DuckDB:<name>*`), so tables created in one block are available in all
  subsequent blocks
- `:db /tmp/<name>.duckdb` — persistent database file backing the session
- `:format markdown` — markdown table output (can also use `box`, `csv`, `json`)
- `:results output example` — wraps output in `#+begin_example` blocks

### 4. Write the notebook sections

Each section follows this pattern:

```org
* <Question about the data>

<Brief motivation or context for the question — one or two sentences.>

#+begin_src duckdb
<SQL query that answers the question>
#+end_src
```

**The first section** should always load data into tables:

```org
* Setup - Load data into DuckDB tables

#+begin_src duckdb
CREATE OR REPLACE TABLE my_table AS
SELECT * FROM read_csv_auto('/absolute/path/to/data.csv');

SELECT 'my_table' AS "table", count(*) AS records FROM my_table;
#+end_src
```

Use `CREATE OR REPLACE TABLE` so the notebook is idempotent — safe to re-run.
Use `read_csv_auto()` for CSVs or `read_parquet()` for Parquet files.

**Subsequent sections** should each pose a question and answer it with a query.
Good analytical questions to consider (adapt to the dataset):

- Record counts and basic shape of the data
- Distribution of categorical variables (with percentages)
- Trends over time (by year, month, hour)
- Top-N rankings (most frequent, highest, largest)
- Cross-tabulations between two dimensions
- Comparisons and ratios between related tables
- Seasonal or temporal patterns

Keep each block to a **single SELECT statement** so the output is clean. DDL
statements (CREATE, DROP) produce no tabular output and can precede the SELECT
in the same block.

### 5. Execute and save

From Elisp (via `eval-elisp.sh`), open the file, execute all blocks, and save:

```elisp
(with-current-buffer (find-file-noselect "/path/to/notebook.org")
  (org-set-regexps-and-options)
  (let ((org-confirm-babel-evaluate nil))
    (org-babel-execute-buffer))
  (save-buffer))
```

To re-execute after changes, first remove stale results:

```elisp
(with-current-buffer "notebook.org"
  (org-babel-remove-result-one-or-many t)
  (let ((org-confirm-babel-evaluate nil))
    (org-babel-execute-buffer))
  (save-buffer))
```

After execution, a `*DuckDB:<session-name>*` buffer will be available in Emacs
for interactive ad-hoc queries.

## ob-duckdb header arguments reference

| Argument | Purpose |
|----------|---------|
| `:session NAME` | Persistent DuckDB process shared across blocks |
| `:db FILE` | Database file (created if missing) |
| `:format MODE` | Output format: `box`, `csv`, `json`, `markdown`, `table` |
| `:async yes` | Non-blocking execution (requires `:session`) |
| `:max-rows N` | Limit output to N rows |
| `:output buffer` | Send results to a dedicated buffer instead of inline |
| `:results table` | Convert markdown to org table (requires `:format markdown`) |

## DuckDB SQL tips for exploratory analysis

- `read_csv_auto()` auto-detects delimiters, headers, and types
- `read_parquet()` for Parquet files — supports glob patterns like `'data/*.parquet'`
- Window functions work: `count(*) / sum(count(*)) OVER ()` for percentages
- `monthname()`, `dayname()`, `year()`, `month()`, `hour()` for temporal analysis
- `UNION ALL` to combine summary rows from multiple tables
- `COALESCE(val, 0)` for safe joins with missing matches
- `round(expr, N)` to control decimal places in output
- String functions: `upper()`, `lower()`, `contains()`, `regexp_matches()`
- `LIMIT N` for top-N queries — always pair with `ORDER BY`

## Gotchas

- **Multiple result sets**: if a block has multiple SELECT statements, their
  outputs concatenate. Use one SELECT per block for clean output.
- **Database file**: clean up `/tmp/<name>.duckdb` before a fresh run if you
  want to start from scratch (`rm -f /tmp/<name>.duckdb*`).
- **File-level properties**: after modifying `#+PROPERTY`, call
  `(org-set-regexps-and-options)` to re-parse without reverting the buffer.
- **Session buffer**: the `*DuckDB:<name>*` buffer stays open after execution.
  The user can switch to it for interactive queries outside the notebook.
