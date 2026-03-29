# AI Agent Skills

A collection of reusable [Cursor Agent Skills](https://docs.cursor.com/context/skills) for software engineering workflows.

## Available Skills

| Skill | Description |
|-------|-------------|
| [rhdh-plugin-development](./rhdh-plugin-development/) | Build full-stack plugins for Red Hat Developer Hub (RHDH) and Backstage |
| [backstage-template-writer](./backstage-template-writer/) | Write and maintain Backstage / RHDH software templates (template.yaml) |

## What Are Skills?

Skills are structured markdown files that teach AI coding agents (Cursor, Claude) how to perform specific tasks with domain-specific knowledge. They provide patterns, templates, and rules that the agent wouldn't otherwise know — distilled from real-world project experience.

## Using a Skill

### In Cursor (Personal)

Copy the skill folder to your personal skills directory:

```bash
cp -r rhdh-plugin-development ~/.cursor/skills/
```

The skill will be automatically available across all your Cursor projects.

### In Cursor (Project-Scoped)

Copy into a specific project to share with collaborators:

```bash
cp -r rhdh-plugin-development /path/to/project/.cursor/skills/
```

### With Claude / Other Agents

Reference the `SKILL.md` file as context when starting a conversation. The agent will follow the patterns and rules defined in the skill and its reference files.

## Contributing

Each skill follows this structure:

```
skill-name/
├── SKILL.md              # Main entry point (required, <500 lines)
├── reference-1.md        # Detailed patterns (progressive disclosure)
├── reference-2.md        # More detail as needed
└── ...
```

See the [Cursor Skills documentation](https://docs.cursor.com/context/skills) for authoring guidelines.

## License

Apache-2.0
