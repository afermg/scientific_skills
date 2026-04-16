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

This skill handles everything: patching `ob-sql` to support the DuckDB engine,
structuring the org file, writing the queries, executing them from within Emacs,
and saving the final notebook.

## Prerequisites

- **Emacs** with a running server (`M-x server-start`)
- **DuckDB CLI** installed and on `$PATH` (`duckdb --version` to verify)
- **org-babel** (ships with Emacs — `ob-sql` specifically)
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

### 2. Add DuckDB engine support to ob-sql

`ob-sql` does not natively support DuckDB. Inject support via Emacs Lisp
advice. This must run before any SQL blocks are executed.

```elisp
(require 'ob-sql)

(defun org-babel-execute:sql--duckdb-support (orig-fn body params)
  "Add DuckDB engine support to ob-sql."
  (let ((engine (cdr (assq :engine params))))
    (if (string= engine "duckdb")
        (let* ((result-params (cdr (assq :result-params params)))
               (cmdline (cdr (assq :cmdline params)))
               (database (cdr (assq :database params)))
               (colnames-p (not (equal "no" (cdr (assq :colnames params)))))
               (in-file (org-babel-temp-file "sql-in-"))
               (out-file (or (cdr (assq :out-file params))
                             (org-babel-temp-file "sql-out-")))
               (db-arg (if (and database (not (string-empty-p database)))
                           (shell-quote-argument database)
                         ""))
               (command (format "duckdb %s -list -separator '\t' %s < %s > %s"
                                db-arg
                                (or cmdline "")
                                (org-babel-process-file-name in-file)
                                (org-babel-process-file-name out-file))))
          (with-temp-file in-file
            (insert (org-babel-expand-body:sql body params)))
          (org-babel-eval command "")
          (org-babel-result-cond result-params
            (with-temp-buffer
              (insert-file-contents-literally out-file)
              (buffer-string))
            (with-temp-buffer
              (when colnames-p
                (with-temp-buffer
                  (insert-file-contents out-file)
                  (goto-char (point-min))
                  (forward-line 1)
                  (insert "-\n")
                  (write-file out-file)))
              (org-table-import out-file '(16))
              (org-babel-reassemble-table
               (mapcar (lambda (x)
                         (if (string= (car x) "-")
                             'hline
                           x))
                       (org-table-to-lisp))
               (org-babel-pick-name (cdr (assq :colname-names params))
                                    (cdr (assq :colnames params)))
               (org-babel-pick-name (cdr (assq :rowname-names params))
                                    (cdr (assq :rownames params)))))))
      (funcall orig-fn body params))))

(advice-add 'org-babel-execute:sql :around #'org-babel-execute:sql--duckdb-support)

(org-babel-do-load-languages
 'org-babel-load-languages
 (cons '(sql . t) org-babel-load-languages))
```

Evaluate this via `eval-elisp.sh` before opening the org file.

### 3. Structure the org file

Use this skeleton:

```org
#+TITLE: <Descriptive Title>
#+PROPERTY: header-args:sql :engine duckdb :database /tmp/<name>.duckdb :colnames yes :results output example
```

The file-level `#+PROPERTY` line sets defaults for every SQL block:
- `:engine duckdb` — routes through the advice above
- `:database /tmp/<name>.duckdb` — persistent file so tables created in one
  block are available in subsequent blocks (acts as a shared session)
- `:colnames yes` — includes column headers in output
- `:results output example` — wraps output in `#+begin_example` blocks for
  consistent formatting

To ensure short outputs also use `#+begin_example` (instead of the `: ` prefix),
set this buffer-local variable before executing:

```elisp
(setq-local org-babel-min-lines-for-block-output 0)
```

### 4. Write the notebook sections

Each section follows this pattern:

```org
* <Question about the data>

<Brief motivation or context for the question — one or two sentences.>

#+begin_src sql
<SQL query that answers the question>
#+end_src
```

**The first section** should always load data into tables:

```org
* Setup - Load data into DuckDB tables

#+begin_src sql
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
  (org-set-regexps-and-options)  ;; parse file-level properties
  (setq-local org-babel-min-lines-for-block-output 0)
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

- **DuckDB CLI flags**: use `-list -separator '\t'` for tab-separated output.
  There is no `-tabs` flag despite what some docs suggest.
- **Multiple result sets**: if a block has multiple SELECT statements, their
  outputs concatenate and confuse the table parser. Use one SELECT per block.
- **Database file**: clean up `/tmp/<name>.duckdb` before a fresh run if you
  want to start from scratch (`rm -f /tmp/<name>.duckdb*`).
- **ob-sql sessions**: `ob-sql` does not support `:session`. The shared database
  file achieves the same effect — tables persist across blocks.
- **File-level properties**: after modifying `#+PROPERTY`, call
  `(org-set-regexps-and-options)` to re-parse without reverting the buffer.
