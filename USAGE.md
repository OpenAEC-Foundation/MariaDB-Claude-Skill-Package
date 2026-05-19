# Usage : MariaDB Skill Package

## How skills activate

Each skill has a `description` field starting with "Use when...". Claude matches the user's prompt against all skill descriptions and loads the most relevant ones. You do not invoke skills manually : describe what you want in natural language.

## Typical workflows

### Workflow 1 : Schema design for a high-throughput table

Ask Claude :

> I need to design a 50M-row events table for time-series analytics. Pick the right MariaDB storage engine, partitioning strategy, and indexes.

Claude will load : `mariadb-core-storage-engines` + `mariadb-syntax-indexing` and produce a schema with InnoDB ROW_FORMAT=DYNAMIC, range partitioning on `created_at`, and composite indexes ordered by selectivity.

### Workflow 2 : Diagnose a slow query

Ask Claude :

> This query takes 8 seconds on a 2M-row table : `SELECT * FROM orders WHERE customer_id = ? AND status IN (1,2,3) ORDER BY created_at DESC LIMIT 100`. Tell me why.

Claude loads : `mariadb-impl-query-optimization` + `mariadb-errors-slow-queries` and produces an EXPLAIN-driven analysis with composite-index recommendation respecting `(customer_id, status, created_at)` left-to-right rule.

## Discovering which skill applies

Browse [INDEX.md](INDEX.md) for the full catalog with descriptions and dependency graph.

Or grep frontmatter Keywords :

```bash
grep -r "Keywords:" skills/source/ | grep -i "galera"
```

## When multiple skills apply

Claude loads multiple skills if descriptions match. Skills are designed to compose : `core` provides architecture, `syntax` adds API patterns, `impl` chains them into workflows.

## Anti-patterns triggered by error messages

Many `errors-*` skills activate on specific error-message strings. Paste the error verbatim and Claude loads the right diagnostic skill.

## Cross-tech usage

For multi-tech stacks : install the relevant single-tech packages plus [Cross-Tech-AEC-Claude-Skill-Package](https://github.com/OpenAEC-Foundation/Cross-Tech-AEC-Claude-Skill-Package) for boundary integration patterns.
