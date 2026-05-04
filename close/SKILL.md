---
name: close
version: 1.2.0
description: |
  Session close. Regenerates State Doc, commits git changes, and writes a session debrief.
  Use at the end of every Claude Code session.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

# Session Close

Run this at the end of every session. Saves all state so the next session picks up cleanly.

## Arguments

- `--dry-run` — print each step's intended action without executing file writes or git commits. State Doc regen, status checks, and secret scans still run (read-only from the user's POV); debrief write, Build Guide edit, and git commits are simulated and their would-be commands printed. Under `--dry-run`, a dirty repo prints what WOULD block but the run continues through the rest of the simulation without writing.
- `--claude-ai-summary` — before debrief write, prompt the user for a summary of Claude.ai-side work from this session. The debrief combines Code-side and Claude.ai work; its filename gets a `-mixed` suffix (e.g. `Session-Debrief-2026-04-20-b-mixed.md`). Use when a single session spanned both surfaces.

Flags stack: `/close --dry-run --claude-ai-summary` is valid.

## Repos in scope

All git-aware steps (status, secret scan, unpushed check, commits) run on **all three** of these repos:
- `/Users/parmstar/Documents/Python Scripts` (primary scripts)
- `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP` (vault)
- `/Users/parmstar/Documents/amp-basecamp-execution` (execution repo)

If any repo is dirty, `/close` hard-blocks (see step 2).

## Steps

1. **Regenerate State Doc and CLAUDE.md:**
   ```bash
   cd "/Users/parmstar/Documents/Python Scripts" && python3 state_doc_gen.py
   ```
   This updates: State Doc, Onboarding Paste, and BASECAMP CLAUDE.md.
   Runs unchanged in `--dry-run` (regen is read-only from the user's perspective; outputs get checked into git only if step 6 commits them).

2. **Uncommitted work check — HARD BLOCK:**
   For each repo in scope:
   ```bash
   git -C "$REPO" status --porcelain
   ```
   Ignore these paths when deciding "dirty" (they are noise, not real work):
   - any `*.log` file
   - `.claude/settings.local.json`
   - **Daemon-produced source notes:** any path under `05-Resources/Sources/` in the BASECAMP repo (written continuously by `dropzone_watcher.py`, `youtube_watcher.py`, `x_watcher.py`, `article_watcher.py`, `inbox_router.py`, `book_watcher.py`)
   - **Daemon-modified PERSONAL.md:** `06-Areas/Claude/PERSONAL.md` in the BASECAMP repo (auto-tagged by `morning_router.py` every 15 min — routed-URL `[routed YYYY-MM-DD]` markers)
   - **Scheduler-produced vault audit:** `06-Areas/Claude/Vault Audit - Latest.md` in the BASECAMP repo (overwritten by `vault_auditor.py` LaunchAgent)
   - **/close-regenerated state doc:** `06-Areas/Claude/State Doc - Current.md` in the BASECAMP repo (rewritten by step 1 of /close itself via `state_doc_gen.py`)
   - **/close-regenerated vault CLAUDE.md:** `CLAUDE.md` at the BASECAMP repo root only (rewritten by step 1 of /close itself via `state_doc_gen.py`)

   These exclusions apply to the BASECAMP vault repo only. Identical filenames in other repos (e.g. `CLAUDE.md` in `Python Scripts` or `amp-basecamp-execution`) are NOT excluded — those represent operator-authored guidance and must continue to block /close when dirty.

   Allowlist principle: paths added here must be both (a) deterministically written by a daemon, scheduler, or /close itself, and (b) considered noise from a session-debrief standpoint. If you find yourself wanting to allowlist `TODOS.md`, `Open Questions.md`, anything under `04-Builds/`, anything under `Session Debriefs/`, or anything under `00-Inbox/` / `03-Projects/` / `07-Archive/` / `99-Archive/` — stop. Those are operator-authored work and must continue to block.

   If **any** repo has remaining dirty state (modified, staged, or untracked) after those exclusions:
   - Show the user the full `git status` for every dirty repo.
   - Run the **secret scan (step 2a)** against the dirty files first. If step 2a stops, `/close` stops there (secret handling takes precedence).
   - Print:
     ```
     === /close BLOCKED — uncommitted work in N repo(s) ===
     ```
   - List each dirty repo with its file count, e.g.:
     ```
     - /Users/parmstar/Documents/Python Scripts: 3 file(s)
     - /Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP: 2 file(s)
     ```
   - Print: `Commit, stash, or discard changes, then re-run /close.`
   - **STOP `/close` immediately.** Do not write debrief. Do not update Build Guide. Do not print the summary block.

   No override. No "proceed anyway." Hard block means hard block.

   **Exception — `--dry-run` only:** print what WOULD block (the same BLOCKED message, repo list, and instruction) and then continue through the rest of the dry-run simulation without writing anything. The dry-run summary must report `Would-block: yes (N repos)` in the summary block.

   **2a. Secret scan (runs inside step 2, gates every commit path):**

   For each dirty file across all repos:
   - **Filename check** (case-insensitive): flag if the path contains any of: `token`, `secret`, `credential`, `oauth`, `.env`, `apikey`, `api_key`, `password`, `private_key`.
   - **Content check**: scan file contents (working copy, not staged blob) for:
     ```
     xoxb-[A-Za-z0-9-]+
     xoxp-[A-Za-z0-9-]+
     sk-[A-Za-z0-9]{20,}
     sk-ant-[A-Za-z0-9_-]{20,}
     ghp_[A-Za-z0-9]{36}
     github_pat_[A-Za-z0-9_]{22,}
     AKIA[A-Z0-9]{16}
     -----BEGIN [A-Z ]+PRIVATE KEY-----
     ```

   If any match:
   - **STOP** — do not commit, do not write debrief (except the override stub per step 3 if the user picks "proceed anyway").
   - Show the file path and matching line. **Redact the matched secret** in terminal output (e.g. `sk-****...****`, keeping at most 4 leading + 4 trailing chars). Never echo the full token.
   - Ask via AskUserQuestion: `delete file / move to .env / proceed anyway / abort`.
     - **delete file** → `git rm -f <path>` if tracked, `rm <path>` if untracked. Re-run the scan.
     - **move to .env** → ask the user to move the secret manually; wait for confirmation; re-run the scan.
     - **proceed anyway** → record the decision (filename + pattern + user's justification) for the debrief override section. The hard block in step 2 still applies — this override only pertains to the secret-scan gate, not to the uncommitted-work gate.
     - **abort** → stop `/close` immediately.

   Secret scan is always active; `--dry-run` does not disable it.

3. **Ask the user what was accomplished this session:**
   Use AskUserQuestion:
   - "What was built or fixed this session?" (free text)

   If `--claude-ai-summary` is set, also ask:
   - "Summarize the Claude.ai-side work from this session." (free text)

   If the user says skip, go to step 5 without writing a debrief **unless** any "proceed anyway" override was granted in step 2a. In that case, force-write a stub debrief containing **only** the `## Secret scan overrides` section (plus frontmatter and title). Filename rules from step 4 still apply. This guarantees every override leaves a paper trail.

4. **Write session debrief:**
   File path:
   `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/06-Areas/Claude/Session Debriefs/Session-Debrief-{YYYY-MM-DD}.md`

   Suffix rules (applied in order):
   - If a debrief already exists for today, add a letter suffix `-a`, `-b`, … (next unused letter).
   - If `--claude-ai-summary` was used, append `-mixed` after any letter suffix.

   Structure:
   ```markdown
   ---
   type: session-debrief
   date: {YYYY-MM-DD}
   surface: {claude-code | mixed}
   ---

   # Session Debrief — {YYYY-MM-DD}

   ## What Was Built or Fixed
   (from user's answer)

   ## Claude.ai-side Work
   (only if --claude-ai-summary; from the second answer)

   ## What's Still Pending
   (extract open P1 items from TODOS.md)

   ## Secret scan overrides
   (only if any "proceed anyway" was chosen in step 2a)

   ## Next Session P1
   (the single most important thing for next session)
   ```

   Under `--dry-run`, print the target path and rendered content to stdout; do not write.

5. **Update the Build Guide Current Phase block:**

   File: `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/04-Builds/Master Financial Pipeline/AMP_Basecamp_Build_Guide.md`

   First, verify the `## Current Phase` block exists. If missing, STOP and print:
   ```
   ERROR: Current Phase block not found in Build Guide. Aborting close.
   ```

   If present, derive four values from the session debrief just written + recent git log + TODOS.md:
   - **Phase:** current phase label (e.g. "Phase 0 (partial)", "L4 in progress", "L4.2 complete")
   - **Next step:** the single next concrete action in one sentence
   - **Last shipped:** most recent completed milestone with date (e.g. "L2 event_log + rollup — 2026-04-17")
   - **Blockers:** any open blockers in one sentence, or "None"

   Rules for derivation:
   - If a value cannot be determined from session context, keep the existing value — never write "Unknown" or leave blank.
   - Use git log to confirm the last shipped item and date if not clear from the debrief.
   - Use open P1 items from TODOS.md for Next step and Blockers.

   Replace ONLY the four bold-labeled lines inside the block. Use Edit with precise old/new strings.

   Under `--dry-run`, print the derived new values side-by-side with existing values; do not edit.

6. **Commit debrief + Build Guide update:**
   Stage only: the debrief, the Build Guide, and (optionally) the regenerated State Doc + CLAUDE.md. Never stage log files, `.claude/settings.local.json`, or anything flagged by the secret scan without override.

   Ask via AskUserQuestion: "Commit debrief + Build Guide update? (yes/no)". If yes, commit with a `/close` style message.
   Under `--dry-run`, print the commands and intended message; do not commit.

7. **Unpushed commits check:**
   For each repo in scope:
   ```bash
   git -C "$REPO" log origin/main..HEAD --oneline
   ```
   If any commits listed in any repo, print:
   > "N unpushed commits on <repo>/main. Run `git push` when ready."
   Never auto-push.

8. **Print session summary:**
   ```
   === SESSION CLOSED — {today's date} ===
   State Doc:    regenerated
   Debrief:      {written <path> / skipped / override-stub / dry-run}
   Build Guide:  {updated / unchanged / dry-run}
   Secrets:      {clean / overrides: N / aborted}
   Git:          {committed: <hashes> / no changes / skipped}
   Unpushed:     {e.g. "Python Scripts: 1, BASECAMP: 2, execution: 0" / none}
   Would-block:  {no / yes (N repos) — dry-run only}
   Mode:         {normal / --dry-run / --claude-ai-summary / both}
   ==========================================
   ```
