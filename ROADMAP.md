# ROADMAP : MariaDB Skill Package

## Current Status

| Phase | Description | Status | Progress |
|-------|------------|--------|----------|
| Phase 1 | Raw Masterplan | DONE | 100% |
| Phase 2 | Deep Research (Vooronderzoek) | DONE | 100% |
| Phase 3 | Masterplan Refinement | AWAITING USER-CHECKPOINT | 95% |
| Phase 4 | Topic-Specific Research | PENDING (post user-go) | 0% |
| Phase 5 | Skill Creation | PENDING (tmux-orchestration) | 0% |
| Phase 6 | Validation | PENDING (user-go) | 0% |
| Phase 7 | Publication | PENDING (user-go) | 0% |

**Overall Progress** : 35% (Phases 1-3 complete pending user-checkpoint ; Phase 4+5+6+7 await go)

Final skill count : 30 (was 28 raw ; +2 from Phase 2 findings : check-constraints, defaults-and-sql-modes ; +1 split = stored-routines into procedures-functions + triggers-events-views)

## Next Steps

1. **Phase 2 : Deep Research** : dispatch 3 opus agents in parallel against the 3 topic-clusters defined in `docs/masterplan/mariadb-masterplan.md` § Next : Phase 2 Deep Research. Merge outputs into `docs/research/vooronderzoek-mariadb.md` (>= 2000 words).
2. **Phase 3 : Masterplan Refinement** : compose Refinement Decisions table (min 1 MERGE / DROP / SPLIT), Execution Plan batches, per-skill agent prompts. STOP for user-checkpoint.
3. **Phase 4 + 5** : tmux-orchestration with 3 skill-builder workers (only after user-go on Phase 3).
4. **Phase 6 + 7** : validation + publication (only after user-go on Phase 5 end).

## Skill Summary

| Category | Estimated | Created | Validated |
|----------|-----------|---------|-----------|
| core/ | 6 | 0 | 0 |
| syntax/ | 10 | 0 | 0 |
| impl/ | 7 | 0 | 0 |
| errors/ | 5 | 0 | 0 |
| agents/ | 3 | 0 | 0 |
| **Total** | **30** | **0** | **0** |

Estimates from raw masterplan. Phase 3 refinement may merge / drop / split.

## Changelog

### Phase 1 : Raw Masterplan + Workspace Bootstrap (2026-05-19)

- Repository structure created (skills/source/, docs/masterplan/, docs/research/, agents/, .github/workflows/)
- Core files initialized (CLAUDE.md, REQUIREMENTS.md, DECISIONS.md, SOURCES.md, WAY_OF_WORK.md, LESSONS.md, CHANGELOG.md, README.md, INDEX.md, HANDOFF.md, OPEN-QUESTIONS.md, USAGE.md, INSTALL.md, START-PROMPT.md)
- All template placeholders replaced with MariaDB-specific values across 10 files
- Raw masterplan written : 28 planned skills across 5 categories with scope-bullet per skill
- SOURCES.md populated with verified MariaDB official sources (KB, GitHub, release-notes, mariadb.org, Galera, JIRA, MaxScale)
- Social preview banner customized with MariaDB brand colors and SQL code-sample
- Ready for Phase 2 deep research
