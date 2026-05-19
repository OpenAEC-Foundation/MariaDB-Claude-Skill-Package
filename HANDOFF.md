# Handoff : MariaDB-Claude-Skill-Package

> Last updated : 2026-05-20
> Generated from Skill-Package-Workflow-Template BOOTSTRAP-RUNBOOK

## Status

- **Phase** : ALL 7 PHASES COMPLETE. v1.0.0 published 2026-05-20.
- **Skills** : 31 / 31 built, all 5 structural validators green
- **GitHub remote** : https://github.com/OpenAEC-Foundation/MariaDB-Claude-Skill-Package (public)
- **Release** : v1.0.0 live
- **Compliance score** : 100% (4/4 checks) ; functional sample-test 5/5 PASS
- **Open manual step** : upload docs/social-preview.png via repo Settings to Social preview (no API)

## What is done

- Phase 1 : workspace bootstrap, placeholder-replace, raw masterplan
- Phase 2 : 8669-word vooronderzoek (3 parallel research-agents, ~82 KB citations)
- Phase 3 : refined masterplan, 10 refinement decisions, 11-batch plan, user-checkpoint approved
- Phase 4+5 : 31 skills built across 11 batches via in-process opus Agent dispatch (D-011 deviation from tmux-orchestration for single-session efficiency)
- All 31 skills : SKILL.md (<500 lines) + 3 reference files, WebFetch-verified
- 10 lessons logged (L-001 to L-010)

## What is open

- **Phase 6 : Validation** (awaiting user-go) : compliance audit P-010, functional sample-test per category, reconcile L-010 (utf8mb4_uca1400 11.5 vs 11.6 discrepancy between defaults-and-sql-modes and encoding-and-collation skills)
- **Phase 6.5 : Discovery manifests** : package.json agents.skills[], agents/openai.yaml, INDEX.md regenerate, Keywords polish, em-dash sweep
- **Phase 7 : Publication** (awaiting user-go) : README finalize, social preview banner render, CHANGELOG 1.0.0, GitHub repo create under OpenAEC-Foundation, push, v1.0.0 release, topics
- **Masterplan arithmetic** : masterplan header says "30 skills" but inventory totals 31 (6+10+7+5+3). Correct to 31 in Phase 6.

## Next-session entry point

```
Lees ROADMAP.md en HANDOFF.md. Phase 5 is af (31 skills). Voer Phase 6 (validation + audit) uit, daarna Phase 6.5 (manifests) en Phase 7 (publication). Wacht op user-go waar BOOTSTRAP-RUNBOOK dat voorschrijft.
```

## Active batch (Phase 5 complete)

All 11 batches done. No active workers.

| Batch | Skills | Status |
|-------|--------|--------|
| B-01 to B-11 | 31 skills total | all committed, all validators green |

## Decisions blocking next step

- None. Phase 6 can start on user-go. Phase 7 (GitHub push) requires explicit user-go per original prompt.

## Special notes

- MariaDB is the default DB for Frappe/ERPNext : companion-skill section present in INDEX + README
- MySQL to MariaDB migration is a dedicated skill (mariadb-impl-migration-mysql-to-mariadb) + a validator agent skill (mariadb-agents-migration-validator)
- D-011 : Phase 4+5 ran via in-process Agent dispatch, not tmux-orchestration workers (single-session efficiency). tmux-orchestration state-files (state/) written for record only.
- 10 lessons capture KB-divergence findings : no GROUPS frame, GTID incompatible, JSON-LONGTEXT, IGNORED-vs-INVISIBLE, UPDATE-RETURNING 13.0+, query-cache not removed, slow-log rename, uca1400 11.5

---

**Anti-pattern caveat** : HANDOFF.md MOET synchroon blijven met ROADMAP.md. Bij elke phase-completion : update beide in dezelfde commit. Cross-Tech L-016 toonde dat drift tussen HANDOFF en ROADMAP leidt tot foute aannames.
