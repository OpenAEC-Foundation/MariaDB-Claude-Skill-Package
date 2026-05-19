# ROADMAP : MariaDB Skill Package

## Current Status

| Phase | Description | Status | Progress |
|-------|------------|--------|----------|
| Phase 1 | Raw Masterplan | DONE | 100% |
| Phase 2 | Deep Research (Vooronderzoek) | DONE | 100% |
| Phase 3 | Masterplan Refinement | DONE | 100% |
| Phase 4 | Topic-Specific Research | DONE (merged into Phase 5 per D-011) | 100% |
| Phase 5 | Skill Creation | DONE | 100% |
| Phase 6 | Validation | DONE | 100% |
| Phase 7 | Publication | IN PROGRESS | 90% |

**Overall Progress** : 98% (Phases 1-6 complete, compliance audit 100% ; Phase 7 publication : README + banner + manifests done, GitHub repo create + push + release pending)

Final skill count : 31 (core 6, syntax 10, impl 7, errors 5, agents 3). All 5 structural validators green across all skills.

## Next Steps

1. **Phase 6 : Validation** (awaiting user-go) : run full automated validation suite + compliance audit (P-010) + functional sample-test per category. Reconcile L-010 (utf8mb4_uca1400 version 11.5 vs 11.6 in defaults-and-sql-modes skill).
2. **Phase 6.5 : Discovery manifests** : generate package.json agents.skills[], agents/openai.yaml, INDEX.md ; Keywords polish pass ; em-dash sweep.
3. **Phase 7 : Publication** (awaiting user-go) : finalize README, render social preview banner, CHANGELOG to 1.0.0, GitHub repo create under OpenAEC-Foundation, push, release tag v1.0.0, topics.

## Skill Summary

| Category | Estimated | Created | Validated (structural) |
|----------|-----------|---------|------------------------|
| core/ | 6 | 6 | 6 |
| syntax/ | 10 | 10 | 10 |
| impl/ | 7 | 7 | 7 |
| errors/ | 5 | 5 | 5 |
| agents/ | 3 | 3 | 3 |
| **Total** | **31** | **31** | **31** |

Structural validation = frontmatter + language + line-count + structure + em-dash, all green. Phase 6 adds compliance audit + functional sample-test.

## Changelog

### Phase 5 : Skill Creation complete (2026-05-20)

- 31 skills built across 11 batches (B-01 to B-11), 3 skills per batch except B-11 (1 skill)
- Each skill : SKILL.md (<500 lines) + references/methods.md + references/examples.md + references/anti-patterns.md
- All code WebFetch-verified against mariadb.com/kb and mariadb.com/docs/server
- 10 lessons logged (L-001 to L-010) from KB-divergence findings during creation
- All 5 structural validators green

### Phase 1-4 (2026-05-19)

- Phase 1 : workspace bootstrap, placeholder-replace, raw masterplan
- Phase 2 : 8669-word vooronderzoek from 3 parallel research-agents
- Phase 3 : refined masterplan, 10 refinement decisions, 11-batch plan, user-checkpoint approved
- Phase 4 : merged into Phase 5 per D-011 (dense vooronderzoek + per-skill supplementary WebFetch)
