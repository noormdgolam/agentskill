# Agents

255 subagent definitions mirrored from [msitarzewski/agency-agents](https://github.com/msitarzewski/agency-agents)
(MIT license — see `LICENSE-agency-agents.txt`), synced here on 2026-08-04 for
portability across machines/projects. Also installed globally at
`~/.claude/agents/` (flat, no subfolders — that's what Claude Code expects).
Browse the full upstream set (including ones not pulled in here) at
[aiskill.market/agents](https://aiskill.market/agents).

## Layout

Organized by category, matching upstream:

```
agents/
  academic/            6 agents
  design/              10 agents
  engineering/          58 agents
  finance/              5 agents
  game-development/     6 agents
  gis/                  13 agents
  healthcare/           3 agents
  integrations/         1 agent
  marketing/            36 agents
  paid-media/           7 agents
  product/              5 agents
  project-management/   7 agents
  sales/                9 agents
  security/             12 agents
  spatial-computing/    6 agents
  specialized/          57 agents
  strategy/             3 agents
  support/              6 agents
  testing/              9 agents
```

## Reinstall / update globally

```bash
git clone https://github.com/msitarzewski/agency-agents.git /tmp/agency-agents
cp /tmp/agency-agents/*/*.md ~/.claude/agents/
rm -rf /tmp/agency-agents
```

Note: the upstream repo also contains `examples/*.md` (workflow write-ups) and
a few stray docs (`README.md`, `EXECUTIVE-BRIEF.md`, `QUICKSTART.md`,
`nexus-strategy.md`) mixed into category folders — none of those have agent
frontmatter, so they were excluded from both this mirror and the global
install.
