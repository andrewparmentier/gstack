# gstack update flow (post-fork)

Operator-authored doc, not part of upstream gstack. Documents how this clone's
remotes and branches are organized after the 2026-05-04 fork, and the safe
update flow that preserves local commits.

## Layout

- `origin` → `andrewparmentier/gstack` (fork, writable)
- `upstream` → `garrytan/gstack` (original, read-only for us)

## Branch model

- `main` — mirrors `upstream/main`. DO NOT commit operator changes here directly.
- `amp/local-customizations` — operator's local commits (e.g. /close allowlist
  fix). All AMP-specific work lives here.
- The active /close skill at `~/.claude/skills/close/SKILL.md` is an APFS
  clone of `~/.claude/skills/gstack/close/SKILL.md` and tracks whichever
  branch is checked out in this worktree.

## Why feature branch instead of committing to main

At fork time (2026-05-04), the local clone was at upstream commit `1b317aa`
while `garrytan/main` had advanced ~200 commits to v1.26.x. Forking inherited
the full upstream tree on `origin/main`, so local commits (`38646d8`,
`b2fcf78`) couldn't fast-forward push to `origin/main`. Using a feature
branch preserves both histories cleanly:

- `origin/main` continues to mirror `upstream/main` for clean update flow
- `origin/amp/local-customizations` carries all operator work, including the
  2026-05-04 /close daemon-allowlist fix

## To pull updates from garrytan/gstack/main (no operator commits affected)

```bash
cd ~/.claude/skills/gstack
git fetch upstream
git checkout main
git merge upstream/main           # or: git rebase upstream/main (linear)
git push origin main              # publishes to fork's main
```

This keeps `origin/main` in lockstep with `upstream/main`. Operator commits
on `amp/local-customizations` are unaffected.

## To pull upstream updates INTO operator's customizations branch

```bash
cd ~/.claude/skills/gstack
git fetch upstream
git checkout amp/local-customizations
git merge upstream/main           # likely conflicts on close/SKILL.md; resolve manually
git push origin amp/local-customizations
```

Conflicts are expected on any file the operator has touched (notably
`close/SKILL.md` after the 2026-05-04 allowlist fix). Resolve by hand,
preserving the AMP customizations.

## Active /close branch in this worktree

The harness reads `~/.claude/skills/close/SKILL.md` (APFS clone of
`~/.claude/skills/gstack/close/SKILL.md`). Whichever branch is checked out
in this worktree is the active /close.

- Checkout `amp/local-customizations` → AMP-customized /close (with
  daemon-allowlist + secret scan + hard-block + dry-run + claude-ai-summary)
- Checkout `main` → upstream-only /close (without AMP customizations)

Default working state: stay on `amp/local-customizations`. Switch to `main`
only when intentionally testing upstream behavior.

## DO NOT

- Run `git fetch upstream && git reset --hard upstream/main`. That wipes
  operator commits. The original gstack docs may recommend this pattern
  pre-fork; ignore it post-fork.
- Commit AMP-specific changes to `main`. Use `amp/local-customizations`.
- Force-push to `origin/main`. Breaks the fork-mirrors-upstream contract
  and complicates future updates.

---

**Last forked:** 2026-05-04
**Local commits at fork time, on `amp/local-customizations`:**

- `38646d8` add /close and /open skills: session management for AMP Basecamp
- `b2fcf78` /close: allowlist daemon-produced and /close-regenerated paths
  (closes 2026-05-03 P3 TODO + 2026-05-04 recurrence)

**Upstream tip at fork time:** `db9447c3` (gstack v1.26.3.0)
