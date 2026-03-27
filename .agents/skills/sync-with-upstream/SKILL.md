---
name: sync-with-upstream
description: Use when syncing this fork with its upstream repository (kus/cs2-modded-server) - covers fetching, rebasing, conflict resolution, and updating plugin versions in README
---

# Sync With Upstream

## Overview

This skill guides the process of syncing a fork of `kus/cs2-modded-server` with its upstream. The goal is a clean linear history with the user's commits on top of upstream's latest.

**Core principle:** User's modifications are always top priority. Upstream changes are pulled in underneath. When in doubt, ask.

## Prerequisites

- Remote `upstream` must exist pointing to the base repository.
  Verify: `git remote -v | grep upstream`
  If missing: `git remote add upstream https://github.com/kus/cs2-modded-server.git`
- Working tree must be clean before starting. Stash or commit any uncommitted changes first.

## Environment Variables

Git operations during rebase MUST use these env vars to prevent interactive prompts that would hang the agent:

```bash
GIT_EDITOR=: EDITOR=: VISUAL='' GIT_SEQUENCE_EDITOR=: GIT_MERGE_AUTOEDIT=no GIT_PAGER=cat PAGER=cat
```

Prefix every `git rebase`, `git rebase --continue`, and `git commit` with these.

## The Workflow

### Phase 1: Assess Divergence

1. **Fetch upstream**
   ```bash
   git fetch upstream
   ```

2. **Analyze what's new on each side**
   ```bash
   # Upstream commits we don't have
   git log --oneline master..upstream/master

   # Our commits that upstream doesn't have
   git log --oneline upstream/master..master
   ```

3. **Find the merge base** (where histories diverged)
   ```bash
   git merge-base master upstream/master
   ```

4. **Review upstream changes in detail** (especially README.md and .gitignore)
   ```bash
   git diff $(git merge-base master upstream/master)..upstream/master -- README.md
   git diff $(git merge-base master upstream/master)..upstream/master -- .gitignore
   ```

5. **Report findings to the user** before proceeding:
   - How many upstream commits
   - How many user commits
   - Which files conflict
   - Summary of upstream changes

### Phase 2: Rebase

1. **Start the rebase**
   ```bash
   GIT_EDITOR=: EDITOR=: VISUAL='' GIT_SEQUENCE_EDITOR=: GIT_MERGE_AUTOEDIT=no GIT_PAGER=cat PAGER=cat \
     git rebase upstream/master
   ```

2. **If conflicts occur**, resolve them following the Conflict Resolution Rules below, then:
   ```bash
   git add <resolved-files>
   GIT_EDITOR=: EDITOR=: VISUAL='' GIT_SEQUENCE_EDITOR=: GIT_MERGE_AUTOEDIT=no GIT_PAGER=cat PAGER=cat \
     git rebase --continue
   ```

3. **Repeat** until the rebase completes.

4. **Verify** the rebase result:
   ```bash
   git log --oneline -20
   ```
   User's commits should appear on top of upstream's commits.

### Phase 3: Update Plugin Versions in README

After rebasing, the README may have outdated plugin versions because the user's restructured table was kept during conflict resolution.

1. **Get the upstream README plugin table** at the point of rebase:
   ```bash
   git show upstream/master:README.md
   ```

2. **Locate the plugin table** in the local README.md under the `## Mods installed` heading. The table format is:
   ```
   Mod | Version | Why
   --- | --- | ---
   [PluginName](url) | `version` | Description
   ```

3. **Compare each plugin row** between upstream and local:
   - If upstream has a newer version -> update local version
   - If upstream changed a URL (e.g., repo rename) -> update local URL
   - If a plugin exists only in local (user-added) -> **preserve it, do not touch**
   - If a plugin exists only in upstream (newly added) -> add it to the local table

4. **Known user-added plugins to preserve** (not in upstream):
   - `PlayerColorSmokes` - custom plugin, keep as-is

5. **Commit version updates** as a separate commit:
   ```bash
   GIT_EDITOR=: EDITOR=: VISUAL='' GIT_SEQUENCE_EDITOR=: GIT_MERGE_AUTOEDIT=no GIT_PAGER=cat PAGER=cat \
     git commit -am "update plugin versions from upstream"
   ```

### Phase 4: Finalize

1. **Show the user the final state**:
   ```bash
   git log --oneline -15
   git status
   ```

2. **Remind the user to force push** (never do this automatically):
   ```
   When you're ready:
   git push --force-with-lease origin master
   ```

## Conflict Resolution Rules

### Priority: User > Upstream

The user's modifications take priority in all conflicts unless the user explicitly says otherwise.

### .gitignore Conflicts

- **Keep both sides**: upstream additions AND user additions.
- Merge them together - they are almost always additive and non-conflicting.

### README.md Conflicts

- **Keep the user's layout and structure**. The user may have reorganized sections, reordered the plugin table, added custom sections, etc.
- **Delete upstream's conflict markers** and keep the user's version of the structure.
- Then update plugin versions separately in Phase 3.

### Config/Script Conflicts

- Keep user's modifications.
- If upstream added something entirely new that doesn't conflict, include it.
- **Ask the user** if:
  - Both sides modified the same setting to different values
  - Upstream removed something the user was relying on
  - The change seems structural (not just a version bump)

### When in Doubt

**Always ask.** Show the user both versions and let them decide.

## Repo-Specific Knowledge

### README Structure

- Plugin version table lives under `## Mods installed` heading
- Table columns: `Mod | Version | Why`
- Versions are wrapped in backticks: `` `1.0.0` ``
- Mod names are markdown links: `[PluginName](github-url)`
- The user may have a different plugin ordering than upstream — respect the user's order

### Remotes

- `origin` -> user's fork (dawidkulpa/cs2-modded-server)
- `upstream` -> base repo (kus/cs2-modded-server)
- Branch: `master` (not `main`)

### Common Gotchas

- Upstream may rename GitHub repos for plugins (e.g., `cs2-inventory-simulator` -> `cs2-css-inventory-simulator`). Watch for URL changes too, not just version bumps.
- The plugin table appears **twice** in the README if the user has kept the upstream's "For hosts" section. Check both occurrences.
- Some version strings are non-standard: `0.3.1x`, `1-21`, `26.02.4`. Don't normalize them — keep whatever format upstream uses.

## Red Flags

- **Never force push automatically** — only the user does this
- **Never drop user commits** during rebase
- **Never silently discard user's custom plugins** from the README table
- **Never skip Phase 3** — plugin versions must be updated after README conflict resolution
- **Never rebase if working tree is dirty** — stash or commit first
