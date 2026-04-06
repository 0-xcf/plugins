# xcf-plugins

Claude Code plugin marketplace maintained by the eXperimental Computing Facility Foundation.

## Structure

```
.claude-plugin/marketplace.json   — marketplace catalog (source of truth)
plugins/<name>/
  .claude-plugin/plugin.json      — plugin manifest
  agents/                         — subagent definitions (.md)
  skills/<skill-name>/SKILL.md    — skill definitions
```

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` with at minimum `name`, `description`, `version`, `author`
2. Add skill or agent content under the plugin directory
3. Add an entry to `.claude-plugin/marketplace.json` with `name`, `source`, `description`, `version`
4. Commit and push

## Conventions

- Plugin and skill names use kebab-case
- The `name` field in plugin.json, marketplace.json, and SKILL.md frontmatter must all match
- Skill directories match their skill name: `skills/<name>/SKILL.md`
- Agent files live under `agents/<name>.md`
- All plugins are MIT licensed
