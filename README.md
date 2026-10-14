<div align="center">

# AI Agent Skills for Platform Engineering

**Domain knowledge that coding agents don't ship with.**

[![Agent Skills](https://img.shields.io/badge/Agent-Skills-6E56CF?style=flat-square)](https://github.com/LiorHaim/skills)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-D97757?style=flat-square&logo=anthropic&logoColor=white)](https://docs.claude.com/en/docs/claude-code/skills)
[![Cursor](https://img.shields.io/badge/Cursor-compatible-000000?style=flat-square&logo=cursor&logoColor=white)](https://docs.cursor.com/context/skills)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue?style=flat-square)](LICENSE)

</div>

---

General purpose coding agents know TypeScript and Kubernetes. They do not know that
`@backstage/*` packages must be peer dependencies, that scaffolder templates break on
`v1beta2`, or how a plugin gets packaged for an OCI registry. They guess, and the guesses
look plausible enough to waste an afternoon.

These skills close that gap. Each one is distilled from shipping the thing in production:
the patterns that work, the constraints that bite, and worked examples the agent can copy
rather than invent.

## Available skills

| Skill | Covers | Size |
|---|---|---|
| **[rhdh-plugin-development](./rhdh-plugin-development/)** | Full stack plugin development for Red Hat Developer Hub and Backstage on the New Backend System: backend plugins, frontend plugins, backend modules, extension points, dynamic plugin packaging, Knex and Postgres patterns, permissions, monorepo scaffolding, Scalprum and Helm configuration | ~43 KB |
| **[backstage-template-writer](./backstage-template-writer/)** | Authoring and debugging software templates: scaffolder parameters and steps, built in and custom actions, Nunjucks templating, UI field extensions, output links, and dynamic plugin actions | ~35 KB |

## Install

### Claude Code

```bash
git clone https://github.com/LiorHaim/skills.git
cp -r skills/rhdh-plugin-development ~/.claude/skills/
```

Available in every session. Claude invokes a skill automatically when the task matches its
description, or you can ask for it by name.

To scope a skill to one project instead, copy it into `.claude/skills/` in that repository
and commit it, so your whole team gets it.

### Cursor

```bash
cp -r skills/rhdh-plugin-development ~/.cursor/skills/      # personal
cp -r skills/rhdh-plugin-development <project>/.cursor/skills/   # project scoped
```

### Any other agent

Each skill is plain Markdown. Point your agent at the `SKILL.md` file as context and it
will follow the patterns and reference the supporting files as needed.

## How a skill is built

Skills use progressive disclosure. The agent reads `SKILL.md` first, which stays small
enough to keep in context, then pulls in reference files only when the task calls for them.

```
rhdh-plugin-development/
├── SKILL.md                      # entry point: frontmatter, target platform, core rules
├── monorepo-scaffold.md          # workspace layout and build configuration
├── backend-patterns.md           # createBackendPlugin, modules, extension points, database
├── frontend-patterns.md          # createPlugin, routing, API refs, components
├── dynamic-plugin-packaging.md   # rhdh-cli, OCI publishing, Helm wiring
└── README.md
```

The frontmatter is what makes a skill discoverable:

```yaml
---
name: rhdh-plugin-development
description: >-
  Build full-stack plugins for Red Hat Developer Hub and Backstage...
  Use when creating, scaffolding, or developing any Backstage or
  RHDH plugin, module, or extension point.
---
```

The `description` is the only part the agent sees until the skill fires, so it carries both
what the skill does and when to reach for it.

## Writing your own

The format is simple and worth stealing. A few things that matter more than they look:

* **Write the description for retrieval, not for humans.** It is a matching problem. Name
  the concrete triggers and the artifacts involved.
* **Keep `SKILL.md` short.** It loads on every match. Push detail into reference files.
* **State constraints as rules, not prose.** "Always use `v1beta3`, never `v1beta2`" beats a
  paragraph explaining the history.
* **Include real examples.** Agents copy far more reliably than they reason from
  description.

See [CONTRIBUTING.md](CONTRIBUTING.md) if you want to add a skill here.

## Related

* [Claude Code agent skills](https://docs.claude.com/en/docs/claude-code/skills)
* [Cursor skills](https://docs.cursor.com/context/skills)
* [Backstage software templates](https://backstage.io/docs/features/software-templates/)

## License

[Apache 2.0](LICENSE)
