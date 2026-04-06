# xcf-plugins

A Claude Code plugin marketplace from the [eXperimental Computing Facility Foundation](https://github.com/0-xcf).

## Install

```
/plugin marketplace add github:0-xcf/plugins
```

## Plugins

| Plugin | Type | Description |
|--------|------|-------------|
| `rebase-resolver` | agent | Expert git rebase agent — analyzes history on both branches before resolving conflicts |
| `using-git-branchless` | skill | Reference skill for using [git-branchless](https://github.com/arxanas/git-branchless) in a stacked-diffs workflow |
| `abizer-code-review` | skill | abizer's opinionated Python code review skill |
| `maintain-changelog` | skill | Reference skill for maintaining a CHANGELOG.md across sessions |

### Install a plugin

```
/plugin marketplace add 0-xcf/plugins 
/plugin install rebase-resolver@xcf-plugins
```

## License

MIT
