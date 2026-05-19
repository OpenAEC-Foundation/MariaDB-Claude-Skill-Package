# INDEX : MariaDB Skill Package Catalog

## Overview

0 skills across 5 categories for MariaDB 10.6-LTS,10.11-LTS,11.x,12.x.

## Summary

| Category | Skills | Focus |
|----------|:------:|-------|
| **core** | 0 | Architecture, cross-cutting concerns |
| **syntax** | 0 | API syntax, signatures, code patterns |
| **impl** | 0 | Development workflows, end-to-end implementations |
| **errors** | 0 | Error handling, debugging, anti-patterns |
| **agents** | 0 | Validation, code generation, orchestration |
| **Total** | **0** | |

> Skill rows are auto-populated by `generate-index.js` after Phase 5. Pre-Phase-5 the catalog shows the planned per-category structure only.

## Core Skills (0 / planned post-research)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| _TBD : populated from frontmatter after Phase 5_ | _generated_ | None |

## Syntax Skills (0 / planned post-research)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| _TBD_ | _generated_ | core-* |

## Implementation Skills (0 / planned post-research)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| _TBD_ | _generated_ | syntax-* |

## Error Skills (0 / planned post-research)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| _TBD_ | _generated_ | impl-* |

## Agent Skills (0 / planned post-research)

| Skill | Description | Dependencies |
|-------|-------------|--------------|
| _TBD_ | _generated_ | ALL |

## Dependency Graph

```
                    ┌─────────────────────┐
                    │   core-{topic}      │
                    └──────┬──────────────┘
                           │
                  ┌────────▼────────┐
                  │  syntax-{topic} │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │   impl-{topic}  │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  errors-{topic} │
                  └────────┬────────┘
                           │
                  ┌────────▼────────┐
                  │  agents-{topic} │
                  └─────────────────┘
```

## Discovery

- npm-agentskills manifest : [package.json](package.json) `agents.skills[]`
- OpenAI Codex : [agents/openai.yaml](agents/openai.yaml)
- GitHub topic : `agentskills`

This INDEX.md is generated from skills frontmatter. To regenerate after adding skills :

```bash
node /path/to/Skill-Package-Workflow-Template/scripts/generate-index.js .
```
