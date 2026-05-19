# Install : MariaDB Skill Package

## Claude Code (local)

```bash
git clone https://github.com/OpenAEC-Foundation/MariaDB-Claude-Skill-Package.git
cp -r MariaDB-Claude-Skill-Package/skills/source/ ~/.claude/skills/mariadb/
```

Restart Claude Code if running. Skills auto-discover from `~/.claude/skills/`.

## As git submodule (recommended for project-specific use)

```bash
cd your-project
git submodule add https://github.com/OpenAEC-Foundation/MariaDB-Claude-Skill-Package.git .claude/skills/mariadb
git commit -m "feat: add mariadb skills as submodule"
```

## Via npm-agentskills

```bash
npx skills add @openaec/mariadb-claude-skill-package
```

## Claude.ai (web)

1. Open a Claude.ai project
2. Upload individual `SKILL.md` files from `skills/source/**/` as project knowledge
3. Skills activate based on description triggers

## Verification

After install, ask Claude :

> Welcome a MariaDB schema design question (e.g. "design a multi-tenant schema for an ERPNext-style app") and verify Claude proposes the correct table-naming convention, InnoDB engine choice, and indexing strategy.

If skill activates : install successful. If not : check `~/.claude/skills/mariadb/` exists and contains category-folders.

## Requirements

- MariaDB 10.6-LTS,10.11-LTS,11.x,12.x
- Claude Code (latest)
- MariaDB CLI (`mariadb` or `mysql` client) reachable from $PATH for running examples locally.
