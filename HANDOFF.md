# Handoff : MariaDB-Claude-Skill-Package

> Last updated : 2026-05-19
> Generated from Skill-Package-Workflow-Template BOOTSTRAP-RUNBOOK

## Status

- **Phase** : Phase 1 (Raw Masterplan) (e.g. Phase 1 raw masterplan / Phase 5 batch N/M / v1.0.0 PUBLISHED)
- **Skills** : 0 / 22-28
- **GitHub remote** : https://github.com/OpenAEC-Foundation/MariaDB-Claude-Skill-Package
- **Last commit** : 1b2adcd feat: bootstrap MariaDB skill package workspace
- **Compliance score** : n.v.t. (pre-phase-6)%

## What is done

- Workspace bootstrap (CLAUDE.md, ROADMAP, REQUIREMENTS, DECISIONS, SOURCES, WAY_OF_WORK, LESSONS, CHANGELOG, README, INDEX, HANDOFF, package.json, agents/openai.yaml, .github/workflows)
- Placeholder-replace : all template-variables filled with MariaDB-specific values
- Raw masterplan skeleton at `docs/masterplan/mariadb-masterplan.md`
- Vooronderzoek stub at `docs/research/vooronderzoek-mariadb.md`

## What is open

- Phase 1 finalization : raw masterplan content per category (TBD this session)
- Phase 2 : dispatch deep-research agents against mariadb.com/kb
- Phase 3 : refine masterplan + user-checkpoint
- Phase 4+5 : tmux-orchestration with 3 skill-builder workers
- GitHub remote creation : deferred until Phase 7 (no push pre-checkpoint)

## Next-session entry point

Open this workspace in VS Code and run :

```
Lees START-PROMPT.md en hervat vanaf Phase 1 (Raw Masterplan).
```

Of expliciet :

```
Lees BOOTSTRAP-RUNBOOK.md van Skill-Package-Workflow-Template en hervat phase 2.
```

## Active batch (if in Phase 5)

_No active batch yet : Phase 5 not started._

| Worker | Skill | Status | tmo task ID |
|--------|-------|--------|-------------|
| worker-1 | : | : | : |
| worker-2 | : | : | : |
| worker-3 | : | : | : |

## Decisions blocking next step

- None pre-Phase-3. After Phase 3 (masterplan refinement) user-checkpoint required before Phase 4+5.

## Special notes

- MariaDB is the default DB for Frappe/ERPNext : companion-skill section required in INDEX + README
- MySQL to MariaDB migration is in-scope as a dedicated skill (not just a footnote)

---

**Anti-pattern caveat** : HANDOFF.md MOET synchroon blijven met ROADMAP.md. Bij elke phase-completion : update beide in dezelfde commit. Cross-Tech L-016 toonde dat drift tussen HANDOFF en ROADMAP leidt tot foute aannames.
