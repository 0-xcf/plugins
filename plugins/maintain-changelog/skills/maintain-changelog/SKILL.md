---
name: maintain-changelog
description: |
  Maintain a CHANGELOG.md in the repo documenting changes made across sessions. Activates
  at the END of every session that modifies files. Use when: (1) you have completed work
  that changed files in the repo, (2) the user asks to update the changelog, (3) starting
  a session and needing to understand what was done previously. Records what changed, why,
  and which files were touched — creating a human-readable audit trail across sessions.
author: Claude Code
version: 1.0.0
date: 2026-02-09
---

# Maintain Changelog

## When to Activate

**End of every session that modified files.** Before wrapping up or when the user signals
they're done, append an entry to `CHANGELOG.md` in the repo root.

Also activate when:
- The user explicitly asks to update the changelog
- Starting a new session — read the changelog first to understand recent history

## Changelog Location

`CHANGELOG.md` in the repository root. Create it if it doesn't exist.

## Entry Format

```markdown
## YYYY-MM-DD — <brief title>

### What changed
- Bullet points describing each meaningful change
- Group related changes together
- Include file paths for significant additions/modifications

### Why
- One or two sentences explaining the motivation or context

### Files touched
- `path/to/new-file.yaml` (new)
- `path/to/modified-file.yaml` (modified)
- `path/to/deleted-file.yaml` (deleted)
```

## Rules

1. **Be concise.** Each entry should be scannable in 10 seconds. No paragraphs.
2. **Focus on "what" and "why", not "how".** Don't describe implementation details —
   the diff tells that story. The changelog tells the *human* story.
3. **Group by logical change, not by file.** "Added shared Postgres instance" is one
   entry even if it touched 3 files.
4. **Newest entries at the top.** Reverse chronological order.
5. **Don't log trivial changes.** Typo fixes, formatting, or changelog-only updates
   don't need entries.
6. **Include context that won't be obvious later.** Item IDs, IP addresses, design
   decisions — things that help future sessions understand the state of the world.
7. **Don't duplicate commit messages.** The changelog is higher-level than git log.
   One changelog entry may span multiple commits.

## Example

```markdown
# Changelog

## 2026-02-09 — Add shared PostgreSQL instance

### What changed
- Created CNPG Cluster `home-1` in `postgres` namespace (single instance, 10Gi on zfs-nfs)
- Added managed roles for `kutt` and `linkding` with ESO-managed passwords (auto-rotation via CNPG)
- Created 4 Vaultwarden login items for postgres credentials (superuser, app, kutt, linkding)
- Added ArgoCD Application for postgres

### Why
Shared Postgres instance for apps migrating off SQLite. Per-app user isolation with
automated password rotation through the ESO → CNPG pipeline.

### Files touched
- `apps/postgres/postgres.yaml` (new)
- `apps/external-secrets/externalsecrets.yaml` (modified — 4 new ExternalSecrets)
- `apps/argocd/apps.yaml` (modified — postgres Application added)

### Next steps
- Create `kutt_db` and `linkding_db` databases manually after cluster is healthy
- Rotate change-me passwords in Vaultwarden
- Migrate Kutt from SQLite to Postgres
```

## Starting a New Session

When beginning work, read `CHANGELOG.md` to understand:
- What was done recently
- What "next steps" were left by the previous session
- The current state of the project

This provides continuity across sessions without relying on memory files alone.

## Notes

- The changelog complements (not replaces) git history. Git shows *what code changed*;
  the changelog shows *what decisions were made and why*.
- If the repo doesn't have a CHANGELOG.md yet, create one with a header:
  ```markdown
  # Changelog
  ```
- Don't commit the changelog separately — include it in the same commit as the changes
  it documents.
