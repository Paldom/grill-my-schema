# Grill My Schema

[![CI](https://github.com/Paldom/grill-my-schema/actions/workflows/ci.yml/badge.svg)](https://github.com/Paldom/grill-my-schema/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![skills.sh](https://skills.sh/b/Paldom/grill-my-schema)](https://skills.sh/Paldom/grill-my-schema)

Asks challenging design-review questions for application-serving database schemas built on an analytics gold layer - covering access patterns, modeling, denormalization, sync, and evolution. Platform independent.

Agent Skills for [Claude Code](https://code.claude.com/docs/en/skills) (and any
[Agent Skills](https://agentskills.io)-compatible tool). Each skill is a folder under
[`skills/`](skills/) with a single-purpose `SKILL.md`, trigger evals, and optional
scripts/references — validated on every write, commit, and PR.

## Quick start

Install with the [skills CLI](https://skills.sh) — auto-detects 70+ agents
(Claude Code, Codex, Cursor, Copilot, pi, …):

```bash
npx skills add Paldom/grill-my-schema                  # all detected agents
npx skills add Paldom/grill-my-schema -a codex -a pi   # or target specific agents
```

Or with the [GitHub CLI](https://cli.github.com/manual/gh_skill_install) (≥ 2.90),
including version-pinned installs from releases:

```bash
gh skill install Paldom/grill-my-schema
gh skill install Paldom/grill-my-schema <skill> --pin <tag>
```

Or as a Claude Code plugin:

```
/plugin marketplace add Paldom/grill-my-schema
/plugin install grill-my-schema@grill-my-schema
```

Or copy a single skill into a project:

```bash
git clone https://github.com/Paldom/grill-my-schema.git
cp -r grill-my-schema/skills/<skill-name> your-project/.claude/skills/
```

Then just describe the task — the skill activates on its description — or invoke it
explicitly with `/<skill-name>`.

## Skills

| Skill | Description |
| --- | --- |
| [grill-my-schema](skills/grill-my-schema/) | Grills the design of an app-serving database schema on an analytics gold layer with prioritized challenging questions — access patterns, freshness, writes, tenancy, evolution — before DDL exists. |
| [serving-schema-review](skills/serving-schema-review/) | Reviews a concrete app-serving schema (DDL, ERD, table defs, or data contract) replicated from an analytics gold layer — severity-ranked findings with fixes. |

The two compose: grill the design until the big questions are answered, then
review the concrete schema before go-live — see
[docs/setup-prompt.md](docs/setup-prompt.md) for a paste-ready session prompt
that runs the full loop.

## Repository structure

```
skills/                  # distributed skills, one folder per skill (SKILL.md + evals/ + scripts/)
docs/                    # skill-authoring guide, eval methodology, deployment guide
scripts/                 # deterministic validator used by hooks and CI
skills.sh.json           # skills.sh repo-page customization (groupings)
.claude/                 # agentic dev setup: hooks + bundled add-skill / publish-repo skills
.claude-plugin/          # plugin + marketplace manifests (makes this repo installable)
.local/                  # gitignored working area: sources, research, PROMPT.md (see below)
```

## Working on this repo with an agent

This repo is agent-native: canonical agent instructions live in
[AGENTS.md](AGENTS.md) (CLAUDE.md imports it), hooks validate every `SKILL.md` on
write, `make check` runs the full validator, and CI enforces the same gate on every
PR. The bundled `add-skill` skill walks the eval-first authoring workflow described
in [docs/skill-authoring.md](docs/skill-authoring.md). Maintainers drive sessions
with their own (gitignored, personal) `.local/PROMPT.md` goal prompt.

## Contributing

Contributions welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the skill-proposal
process, the authoring workflow, and the PR checklist. Please note the
[Code of Conduct](CODE_OF_CONDUCT.md).

## Support

Questions, ideas, or something not working? Start with [SUPPORT.md](SUPPORT.md) —
bugs and skill proposals have [issue templates](../../issues/new/choose), and
security concerns go through [SECURITY.md](SECURITY.md) (never a public issue).

## License

[MIT](LICENSE) © 2026 Paldom
