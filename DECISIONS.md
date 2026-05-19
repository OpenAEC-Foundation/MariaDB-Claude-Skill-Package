# Architectural Decisions

Numbered decisions (D-XXX) with rationale. Immutable once recorded — new decisions can supersede but never delete old ones.

---

## D-001: English-Only Skills

- **Date**: 2026-05-19
- **Decision**: All skill content MUST be in English only.
- **Rationale**: Skills are instructions FOR Claude, not for end users. Claude reads English and responds in the user's language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext (28 skills) and Blender-Bonsai (73 skills).
- **Consequence**: No translations needed. All descriptions, code comments, and documentation in English.

---

## D-002: MIT License

- **Date**: 2026-05-19
- **Decision**: Project uses MIT License.
- **Rationale**: Most permissive license, maximizes adoption. Consistent with OpenAEC Foundation standards.
- **Consequence**: No commercial restrictions. Community-friendly.

---

## D-003: SKILL.md Under 500 Lines

- **Date**: 2026-05-19
- **Decision**: SKILL.md files MUST be under 500 lines.
- **Rationale**: Keeps main skill focused on decision trees and quick reference. Heavy content belongs in references/ directory. Proven optimal in ERPNext (180-427 lines per skill).
- **Consequence**: Complex topics split between SKILL.md (quick reference + patterns) and references/ (complete API, examples, anti-patterns).

---

## D-004: 7-Phase Research-First Methodology

- **Date**: 2026-05-19
- **Decision**: Follow the 7-phase research-first development methodology.
- **Rationale**: Proven in ERPNext (28 skills), Blender-Bonsai (73 skills), and Tauri 2 (27 skills). Research prevents hallucination. Deterministic skills require deep understanding.
- **Consequence**: No skill creation without prior deep research. Phases are sequential with defined exit criteria.

---

## D-005: ROADMAP.md as Single Source of Truth

- **Date**: 2026-05-19
- **Decision**: ROADMAP.md is the ONLY place for project status.
- **Rationale**: Multiple status locations cause drift and "which is current?" confusion. Single source of truth enables reliable session recovery.
- **Consequence**: Never duplicate status in CLAUDE.md or other files. All status references point to ROADMAP.md.

---

## D-006: WebFetch for Research Verification

- **Date**: 2026-05-19
- **Decision**: Use WebFetch to verify all code examples against latest official documentation.
- **Rationale**: Technology APIs evolve. Training data may be stale. WebFetch ensures latest official docs are consulted, not outdated cached knowledge.
- **Consequence**: All code examples must be verified against current official documentation before inclusion in skills.

---

## D-007: GitHub Publication Under OpenAEC Foundation

- **Date**: 2026-05-19
- **Decision**: Publish all skill packages under the OpenAEC Foundation GitHub organization.
- **Rationale**: Centralized, consistent branding. Community ownership. Discoverability.
- **Consequence**: All repos follow OpenAEC naming conventions and include social preview banners with OpenAEC branding.

---

## D-008 : 30-skill scope (Phase 3 refinement)

- **Date** : 2026-05-19
- **Decision** : Final skill inventory is 30 skills (core 6, syntax 10, impl 7, errors 5, agents 3). Raw masterplan estimated 28 ; Phase 2 research findings drove two additions and one split.
- **Rationale** : Vooronderzoek L-002 (no GROUPS frame) reduced one skill scope ; vooronderzoek L-005 (JSON-as-LONGTEXT) cross-cuts but does not add a skill ; D-01 split (procedures-functions vs triggers-events-views) and D-02 add (check-constraints) and D-03 add (defaults-and-sql-modes) drove the +2 net.
- **Consequence** : Phase 5 runs 11 tmux-orchestration batches (10 batches of 3 + 1 batch of 1). Per-skill agent-prompts to be expanded post user-go.
- **Reversal cost** : adding or dropping a skill mid-Phase-5 forces re-batching ; do not change mid-execution.

---

## D-009 : Phase 4 topic-research skip-criterion

- **Date** : 2026-05-19
- **Decision** : Per skill in a batch, if the vooronderzoek section covering its scope has >= 300 words and >= 5 distinct citations, SKIP Phase 4 topic-research for that skill and reference the vooronderzoek section directly in the worker prompt. Otherwise dispatch a Phase 4 topic-research agent.
- **Rationale** : Docker package L-001 (skip-pattern) ; vooronderzoek depth varies per topic ; redundant research wastes opus-budget.
- **Consequence** : Per-batch DECISIONS.md entry must record which skills skipped Phase 4 and why (with vooronderzoek section reference).

---

## D-011 : Phase 4+5 execution via in-process Agent dispatch (deviation from P-012)

- **Date** : 2026-05-19
- **Decision** : Phase 4 + 5 (topic-research + skill-creation) run via in-process Agent tool calls in batches of 3, NOT via tmux-orchestration workers. tmux-orchestration state-files (state/messages.jsonl, role file) still written for record-keeping.
- **Rationale** : Single-session autonomous run per user-go on Phase 3. 30 skills × ~30-50k tokens per worker via independent claude-instances = high coordination overhead and risk of L-007 context-overflow. In-process Agent dispatch (opus) reuses orchestrator token-budget more efficiently and provides synchronous quality-gate.
- **Standing-order context** : CLAUDE.md P-012 mandates tmux-orchestration for packages >15 skills. Deviation accepted ONLY for this single-session run. Future re-runs in fresh sessions SHOULD use tmux-orchestration per standing-order.
- **Consequence** : Per-batch dispatch is 3 parallel Agent calls in same message. Quality-gate runs synchronously after batch return. Worker context-bundle is replaced by inlined agent-prompt with same content.
- **Skip-criterion (D-009) interaction** : Vooronderzoek covers most skills with >= 300 words + 5 citations. Per-skill agent prompt instructs the agent to do supplementary WebFetch for thin areas (e.g. C-01 architecture). Phase 4 separate dispatch is eliminated ; topic-research merges with skill-creation.

---

## D-010 : JSON LONGTEXT-alias must appear in every JSON-related skill's Quick Reference

- **Date** : 2026-05-19
- **Decision** : Every skill whose scope touches JSON (mariadb-syntax-json, mariadb-impl-schema-design, mariadb-impl-migration-mysql-to-mariadb, mariadb-errors-encoding-and-collation) MUST repeat in its Quick Reference : "MariaDB JSON is a LONGTEXT alias, not native binary. Use CHECK (JSON_VALID(col)) for structure ; use functional indexes on virtual columns for index access."
- **Rationale** : This is the single most consequential MySQL divergence (L-005) ; one-place documentation creates "I missed this" failures on read.
- **Consequence** : Polish-pass agent in Phase 6.5 explicitly checks for the LONGTEXT-alias sentence in all four skills.
