---
name: git-branchless
description: Use when working in a git repo that uses git-branchless or stacked-diffs workflow. Triggers - git sl in CLAUDE.md, branchless in git config, user mentions stacks or stacked diffs, need to rebase/move/reorder commits, submit stacked PRs.
---

# git-branchless

Stacked-diffs workflow for Git. Commits are the unit of work, not branches. Think Mercurial/Phabricator-style development.

## Mental Model

- **Stack**: a chain of draft commits off `main()`/`master`. Stacks can branch.
- **Draft commits**: your in-progress work. Visible in smartlog even without branch names.
- **Public commits**: merged into main. Immutable.
- Rebases happen **in-memory** first (fast, safe). Falls back to on-disk only if needed.
- `git undo` can reverse almost any operation.

## Command Quick Reference

| Task | Command |
|------|---------|
| View commit graph | `git sl` |
| Go to specific commit | `git checkout <hash>` or `git branchless switch <hash>` |
| Navigate down stack (relative) | `git prev [n]` (interactive if stack branches) |
| Navigate up stack (relative) | `git next [n]` (interactive if stack branches) |
| Fuzzy-search checkout | `git branchless switch -i` |
| Create commit | `git record -m "msg"` |
| Insert commit mid-stack | `git record --insert -m "msg"` |
| Amend HEAD commit | `git amend` (stage changes first) |
| Amend + keep descendants identical | `git amend --reparent` |
| Reword commit message | `git reword -m "new msg"` |
| Move/rebase commits | `git move -s <src> -d <dest>` |
| Reorder: swap A and B | `git move -x <B> -d <parent-of-A>` |
| Squash into dest | `git move -x <src> -d <dest> --fixup` |
| Insert between dest and children | `git move -s <src> -d <dest> --insert` |
| Fix orphaned commits after amend | `git restack` |
| Sync all stacks onto updated main | `git sync --pull` |
| Sync without fetching | `git sync` |
| Push existing PR branches | `git submit` |
| Create new PR branches | `git submit --create` |
| Push as draft PRs | `git submit --create --draft` |
| Hide/discard a stack | `git hide -r <commit>` |
| Recover hidden commits | `git unhide <commit>` |
| Undo last operation | `git undo` |

## Critical Distinctions

**`git sync` vs `git restack`** - agents confuse these constantly:
- `git sync` = rebase stacks onto **updated main** (runs `git fetch` + in-memory rebase, NOT `git pull` - so `branch.autosetuprebase` is irrelevant)
- `git restack` = fix **orphaned descendants** after you amend/reword a mid-stack commit
- After amending mid-stack: `git restack`
- After fetching new main: `git sync --pull`

**`git move` modes:**
- `-s/--source`: move commit **and all descendants**
- `-b/--base`: move entire branch from merge-base (default if nothing specified)
- `-x/--exact`: move **only** specified commits, leave descendants in place

## Revset Language

Revsets select commits. Used by `git sl`, `git move`, `git submit`, `git query`, etc.

### Essential Functions

| Revset | Meaning |
|--------|---------|
| `@` | HEAD |
| `stack()` | Current stack (default for many commands) |
| `draft()` | All draft (non-public) commits |
| `main()` | Tip of main branch |
| `ancestors(x)` / `::x` | All ancestors of x |
| `descendants(x)` / `x::` | All descendants of x |
| `children(x)` | Immediate children |
| `parents(x)` | Immediate parents |
| `branches(pattern)` | Commits with matching branch names |
| `message(pattern)` | Commits matching message text |
| `paths.changed(pattern)` | Commits touching matching file paths |
| `author.date(pattern)` | Commits by date |

### Operators

| Op | Meaning |
|----|---------|
| `x + y`, `x \| y` | Union |
| `x & y` | Intersection |
| `x - y` | Difference (spaces required! `foo-bar` = branch name) |
| `x % y` | Only: ancestors of x not ancestors of y |

### Date/Text Patterns

```
author.date("after:1 week ago")     # relative dates
author.date("before:2025-01-01")    # absolute dates
message("substr:fix")               # substring (default)
message("regex:feat.*auth")         # regex
paths.changed("glob:src/**/*.ts")   # glob
```

## Common Workflows

### Amend a mid-stack commit
```bash
git checkout <hash>           # go to the target commit (or git prev N for relative)
# make edits, git add
git amend                     # amend in place
git restack                   # rebase descendants onto amended commit
git checkout <stack-tip>      # return to where you were
```

### Create a new stack for independent work
```bash
git branchless switch master  # go to main
# make changes
git record -m "feat: new thing" -c my-branch  # commit + create branch
```

### Submit stacked PRs to GitHub
```bash
git submit --create --forge github   # first time: create PRs
git submit                           # subsequent: update existing PRs
```

### Sync after main branch updated
```bash
git sync --pull               # fetch + rebase all stacks onto new main
```

### Move a stack on top of another
```bash
git move -s <root-of-stack-2> -d <tip-of-stack-1>
```

## What NOT To Do

- **Don't use `git rebase -i`** for reordering/squashing. Use `git move` instead.
- **Don't create branches manually** for stacked work. Commits are the unit; branches are created at submit time.
- **Don't confuse `git restack` with `git sync`**. Restack = fix orphans. Sync = update to new main.
- **Don't use `git merge`** for incorporating upstream changes. Use `git sync --pull`.
