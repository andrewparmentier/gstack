---
name: close
version: 1.5.1
description: |
  Session close. Regenerates State Doc, checks IBKR↔portfolio.md drift, reconciles TODOS.md + Open Questions.md + State Doc Warnings, commits git changes, and writes a session debrief.
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

- `--dry-run` — print each step's intended action without executing file writes or git commits. State Doc regen, status checks, and secret scans still run (read-only from the user's POV); debrief write, Build Guide edit, tracking-doc reconciliation, and git commits are simulated and their would-be commands printed. Under `--dry-run`, a dirty repo prints what WOULD block but the run continues through the rest of the simulation without writing.
- `--claude-ai-summary` — before debrief write, prompt the user for a summary of Claude.ai-side work from this session. The debrief combines Code-side and Claude.ai work; its filename gets a `-mixed` suffix (e.g. `Session-Debrief-2026-04-20-b-mixed.md`). Use when a single session spanned both surfaces.
- `--skip-reconcile` — skip Step 4.5 (tracking doc reconciliation). Use only when nothing changed this session (e.g. closing a session that did pure investigation with no TODOs to add/promote/ship). Default behavior always prompts, even on no-commit sessions.

Flags stack: `/close --dry-run --claude-ai-summary --skip-reconcile` is valid.

## Repos in scope

All git-aware steps (status, secret scan, unpushed check, commits) run on **all three** of these repos:
- `/Users/parmstar/Documents/Python Scripts` (primary scripts)
- `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP` (vault)
- `/Users/parmstar/Documents/amp-basecamp-execution` (execution repo)

If any repo is dirty, `/close` hard-blocks (see step 2).

## External dependencies

Step 1a (portfolio drift check) calls `portfolio_sync.py --check` in the `amp-basecamp-execution` repo, which connects to IBKR Gateway on `127.0.0.1:4001` via `ib_async` (read-only). Gateway is normally up via the `basecamp.ibgateway` LaunchAgent. If Gateway is unavailable, step 1a degrades to a WARN — it does **not** block /close.

## Steps

1. **Regenerate State Doc and CLAUDE.md:**
   ```bash
   cd "/Users/parmstar/Documents/Python Scripts" && python3 state_doc_gen.py
   ```
   This updates: State Doc, Onboarding Paste, and BASECAMP CLAUDE.md.
   Runs unchanged in `--dry-run` (regen is read-only from the user's perspective; outputs get checked into git only if step 6 commits them).

   **1a. Portfolio drift check — HARD BLOCK:**

   ```bash
   python3 /Users/parmstar/Documents/amp-basecamp-execution/portfolio_sync.py --check
   ```

   This compares the BASECAMP `portfolio.md` `## Owned` section against live IBKR positions via `ib_async` (read-only, port 4001). Exit codes:

   - **`0`** — `portfolio.md` in sync with IBKR. Continue to step 2.
   - **`1`** — drift detected. **HARD BLOCK.** Print the diff captured on stdout from `--check`, then print:
     ```
     === /close BLOCKED — portfolio.md drift from IBKR detected ===

     Run `python3 /Users/parmstar/Documents/amp-basecamp-execution/portfolio_sync.py` to apply the IBKR snapshot to portfolio.md, then re-run /close.
     ```
     **STOP `/close` immediately.** Do not run step 2. Do not write debrief. Do not update Build Guide. Do not print the summary block.

     No override. No "proceed anyway." Same hard-block semantics as step 2.
   - **`2`** — Gateway unreachable (`ib_async` could not connect to `127.0.0.1:4001`). **WARN-but-proceed.** Print:
     ```
     WARN: portfolio_sync --check could not reach IBKR Gateway. Skipping drift check; continuing /close. Verify Gateway is up if drift is suspected before next session.
     ```
     Continue to step 2. Do not block — Gateway availability is not under operator control at the moment of /close, and blocking would create a dependency loop with `basecamp.ibgateway` LaunchAgent state.

   **Why this is a hard block:** `portfolio.md` feeds three downstream consumers — `state_doc_gen.py` (BASECAMP State Doc + CLAUDE.md regenerated in step 1 above), watcher relevance gates via `portfolio_loader.py` (3 daemons: `dropzone_watcher.py`, `youtube_watcher.py`, `book_watcher.py`), and Claude memory `portfolio_state.md`. Letting a session close with stale `portfolio.md` propagates drift into all three surfaces, which silently poisons every subsequent session and every relevance-gated ingest until the next manual run. Catching drift here, before commit, is the single chokepoint that prevents that propagation.

   **Why this isn't covered by step 2:** the dirty-repo check looks at the git working tree. Drift between `portfolio.md` (committed clean) and live IBKR is invisible to git. The two checks are complementary, not redundant.

   **Exception — `--dry-run` only:** if exit code is `1`, print the BLOCKED message and diff, then continue through the rest of the dry-run simulation without writing anything. The dry-run summary must report `Portfolio-drift: yes (would block)` in the summary block.

   No secret scan needed: `portfolio_sync.py --check` output never contains secrets by construction (it emits only cost/quantity/listing data from IBKR plus operator-curated sectors/themes from `portfolio.md`, none of which is sensitive).

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
   - **Scheduler-produced daily brief:** any path under `02-Daily Notes/` in the BASECAMP repo (written by `daily_brief_generator.py` LaunchAgent every 6am, file name `Daily Brief - YYYY-MM-DD.md`)
   - **Daemon-appended preferences log:** `06-Areas/Claude/Preferences.md` in the BASECAMP repo (appended by `daily_brief_generator.py` LaunchAgent every 6am with a daily progress entry — `### YYYY-MM-DD\n- Yesterday's brief surfaced N items\n- PERSONAL.md current state: X checked, Y unchecked`)
   - **/close-regenerated state doc:** `06-Areas/Claude/State Doc - Current.md` in the BASECAMP repo (rewritten by step 1 of /close itself via `state_doc_gen.py`)
   - **/close-regenerated vault CLAUDE.md:** `CLAUDE.md` at the BASECAMP repo root only (rewritten by step 1 of /close itself via `state_doc_gen.py`)

   These exclusions apply to the BASECAMP vault repo only. Identical filenames in other repos (e.g. `CLAUDE.md` in `Python Scripts` or `amp-basecamp-execution`) are NOT excluded — those represent operator-authored guidance and must continue to block /close when dirty.

   Allowlist principle: paths added here must be both (a) deterministically written by a daemon, scheduler, or /close itself, and (b) considered noise from a session-debrief standpoint. If you find yourself wanting to allowlist `TODOS.md`, `Open Questions.md`, anything under `04-Builds/`, anything under `Session Debriefs/`, or anything under `00-Inbox/` / `03-Projects/` / `07-Archive/` / `99-Archive/` — stop. Those are operator-authored work and must continue to block.

   **TODOS.md and Open Questions.md — chokepoint + gate, complementary:** These two files are operator-authored AND now reconciled through Step 4.5 as a structured authoring chokepoint (added /close v1.5.0). They continue to block drift detection here when modified outside /close — that is correct behavior, not a leftover. Step 4.5 is where intentional edits should land; this step 2 gate catches anything that bypassed 4.5 (manual Obsidian edits, daemon writes that shouldn't touch these files, etc.). Do not add either file to the allowlist above.

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

4.5. **Reconcile tracking docs:**

   This step is the structured authoring chokepoint for TODOS.md, Open Questions.md, and the State Doc Operator Warnings section. After this step, `/open` on the next session reads guaranteed-fresh state — zero in-session reconciliation work required.

   Skipped entirely if `--skip-reconcile` was passed. Otherwise always prompts, even if no commits landed this session (catches "I forgot to write that down" cases).

   **Read current state:**
   - `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/06-Areas/Claude/TODOS.md`
   - `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/06-Areas/Claude/Open Questions.md`
   - Last `/close` timestamp: parse the YYYY-MM-DD from the most recent file under `06-Areas/Claude/Session Debriefs/Session-Debrief-*.md` (lexicographic max excluding today's just-written debrief). If none, fall back to `git log -1 --format=%cI -- 06-Areas/Claude/TODOS.md` in the BASECAMP repo.

   **Survey commits since last /close** across all three in-scope repos:
   ```bash
   for repo in "/Users/parmstar/Documents/Python Scripts" "/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP" "/Users/parmstar/Documents/amp-basecamp-execution"; do
     git -C "$repo" log --since="<last_close_timestamp>" --oneline
   done
   ```

   **Build the numbered item lists (do this ONCE, before prompting — numbering must stay stable for the whole Step 4.5 cycle).**

   Two SEPARATE numbering namespaces — TODOS and Open Questions. A number means item N *within the section it appears under*; the same number can refer to different items in different sections.

   - **TODOS namespace.** Scan TODOS.md for `- [ ]` (open) and `- [~]` (partial) lines. SKIP every `- [x]` (done — history, never shown, never numbered). Parse each line's priority from the `Pn:` token after the checkbox. Number CONTINUOUSLY across tiers in this order: all P0, then all P1, then P2 (10 most recent by `(added YYYY-MM-DD)` date, newest first). P3 is not displayed or numbered (direct-edit those in Obsidian). The first displayed item is `[1]` and the counter increments by one straight through the tier boundaries (so if there are 2 P0 and 9 P1, the first P2 shown is `[12]`).
   - **Open Questions namespace.** Scan Open Questions.md for `- [ ]` lines (skip `- [x]` resolved). Display the 10 most recent by `(raised: YYYY-MM-DD)` date, newest first (items with no raised date sort oldest). Number `[1]`…`[M]` — independent of the TODOS numbering.

   If multiple ops fire against the same list, apply them ALL against this original numbering. Never renumber mid-cycle (deleting `[7]` does not turn `[8]` into `[7]` for the rest of this same reply).

   **Print the reconciliation prompt** (single block, exact format):
   ```
   === RECONCILE TRACKING DOCS ===

   Debrief just written: <path from Step 4>

   Commits since last /close (<timestamp>):
     <repo-shortname>: <hash> <subject>
     ...

   Current TODOS:
     P0:
       [n] <text> (<added-date>)        | or "(none)" if the tier is empty
     P1:
       [n] <text> (<added-date>)
       ...
     P2 (10 most recent):
       [n] <text> (<added-date>)
       ...

   Current Open Questions:
       [n] <text> (<raised-date>)
       ...

   Operations — one per line, numeric ID refers to the bracketed number in THIS section:
     ✓ N            mark item N shipped (TODOS) / resolved (Open Questions)
     ✓ N, M, ...    multiple items in one op (comma-separated)
     - N            remove item N
     ↑ N            promote one tier (P3→P2, P2→P1, P1→P0)   [TODOS only]
     ↓ N            demote one tier (P0→P1, P1→P2, P2→P3)    [TODOS only]
     + P1: <text>   add new item (free text; include the priority prefix for TODOS)

   TODOS numbers and Open Questions numbers are SEPARATE namespaces.
   Empty section = no changes. Type "done" to finish a section. Type "abort" to bail Step 4.5 entirely.

   > TODOS:
   > Open Questions:
   > State Doc Warnings:
   ```

   **Collect the operator's reply as a conversation message.** Do NOT use AskUserQuestion — the three-section format is multiline and benefits from being reviewed/edited as one block. The reply is one message containing all three sections delimited by `> TODOS:`, `> Open Questions:`, `> State Doc Warnings:` headers; each section ends at `done` on its own line (or at the next `>` header). Empty sections (header followed immediately by `done`) are valid and mean "no changes for that section."

   **Validate each section BEFORE applying anything — fail loud, per-section atomic.**

   For the `> TODOS:` and `> Open Questions:` sections, validate every op line against that section's numbered list first:
   - `✓ N`, `- N`, `↑ N`, `↓ N` — `N` must parse as an integer in `1..len(list)`. Comma forms (`✓ 5, 6, 12`) validate every number independently.
   - `+ <text>` — always valid (free-text add; no number).
   - Any op that references an out-of-range number, an unrecognized op character, or a malformed line FAILS the entire section.

   On ANY validation failure within a section:
   - Apply NO ops from that section (atomic — a valid op in the same block does not partially land).
   - Print the specific error:
     - out-of-range number → `Item #N not found. Valid range: 1-<len>. Section "<TODOS|Open Questions>" input rejected — paste again.`
     - malformed/unrecognized line → `Unrecognized operation: "<line>". Section "<TODOS|Open Questions>" input rejected — paste again.`
   - Re-prompt that section only. Do NOT advance to the next section until one of: valid input received, OR empty input (= no changes for that section), OR the operator types `abort` (= bail Step 4.5 entirely, write NOTHING to any file, print `Reconcile: aborted by operator` and continue /close at Step 5).

   The principle: never silently no-op. Either an op applies visibly, or the section is rejected visibly and resolution is forced in-session.

   **Apply edits (only after the whole section validates). Resolve each `N` to its original file line via the frozen numbering above.**

   For **TODOS.md**:
   - `✓ N` — remove the line entirely AND record its text (stripped of the `- [ ]` / `- [~]` prefix) for the debrief's "Shipped this session" section.
   - `- N` — delete the line.
   - `↑ N` — replace the priority prefix on the matched line (P3→P2, P2→P1, P1→P0; P0 is the ceiling, no-op + warn).
   - `↓ N` — reverse direction (P0→P1, P1→P2, P2→P3; P3 is the floor).
   - `+ <priority>: <text>` — prepend to a `## Session {YYYY-MM-DD} reconciliation` section at the top of the file (create the section just under any frontmatter if today's section doesn't exist yet; append to today's section if it does). Line format: `- [ ] {priority}: {text}  *(added {YYYY-MM-DD})*`. If `<priority>` is omitted or unrecognized, default to `P2` and warn in the summary.

   For **Open Questions.md**:
   - `✓ N` — remove the line from the open list AND append a corresponding entry under a `## Resolved {YYYY-MM-DD}` section at the bottom of the file (create if absent; append if present). Resolved line format: `- [x] {original text} (resolved: {YYYY-MM-DD})`.
   - `- N` — delete the line.
   - `↑ N` / `↓ N` — no-op (OQ items carry no priority tier); print `↑/↓ N: skipped (Open Questions items have no priority tier)`.
   - `+ <text>` — prepend to a `## Reconciliation {YYYY-MM-DD}` section placed after the frontmatter + `## Protocol` block, before existing category sections (create today's section if absent; append if present). Line format: `- [ ] {text} (raised: {YYYY-MM-DD}, context: /close reconciliation)`. No priority prefix expected.

   For **State Doc Warnings** (free-text lines, no numbering):
   - Treat each non-blank operator line under `> State Doc Warnings:` as one warning bullet.
   - **Write to State Doc — `State Doc - Current.md`:** write a `## Operator Warnings` section at the very top (above the existing `## System Status` table; below any frontmatter). Body is a bulleted list of the operator's lines verbatim. If the section already exists from a prior /close cycle, replace its contents (do not concatenate). If the operator supplied zero lines, remove any existing `## Operator Warnings` section entirely. Do NOT touch the auto-generated `## System Status` table or anything below it — that is `state_doc_gen.py`'s territory. (State Doc is gitignored; this copy is informational and refreshed at every regen.)
   - **Write to Operator Warnings.md — `06-Areas/Claude/Operator Warnings.md`** (NEW; tracked in git, survives fresh clone):
     - If the operator supplied ≥1 line: move the current `## Active` block contents to the TOP of `## Archive` under a `### Rotated {YYYY-MM-DD}` header (skip the rotation if `## Active` holds only the `(none …)` placeholder), then replace `## Active` with the new lines verbatim.
     - If the operator supplied zero lines: leave Operator Warnings.md unchanged — no rotation, no clearing. The tracked file retains last-known warnings until explicitly replaced (the State Doc clear above is the "warnings resolved this session" signal; the tracked file is the durable record).
     - Bump the `updated:` frontmatter to today.

   In TODOS.md and Open Questions.md (and Operator Warnings.md if touched): bump the `updated:` frontmatter field to today's date. Preserve frontmatter shape otherwise (do not reformat keys, do not add new keys).

   **Print a per-file summary:**
   ```
   TODOS.md:             +N added, -M removed, ↑P promoted, ↓Q demoted, ✓R shipped
   Open Questions.md:    +N added, -M removed, ✓P resolved
   State Doc:            Operator Warnings updated (N lines)  |  cleared  |  unchanged
   Operator Warnings.md: Active replaced (N lines, prior rotated to Archive)  |  unchanged
   ```

   **If any `✓` ops were applied to TODOS,** append a `## Shipped this session` section to the debrief file written in Step 4, listing each shipped item verbatim (item text minus the `- [ ]` checkbox).

   **Under `--dry-run`:** still build the numbered lists, print the prompt, collect + VALIDATE the reply (including fail-loud rejection + section re-prompt), and print the per-file summary — but compute and display the would-be diffs (`+++ added: …`, `--- removed: …`, etc.) without writing. TODOS / Open Questions / State Doc / Operator Warnings.md / debrief edits are all simulated.

   **Under `--skip-reconcile`:** skip this entire step. Print one line: `Reconcile: skipped (--skip-reconcile)`.

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

6. **Commit debrief + Build Guide update + reconciled tracking docs:**
   Stage only: the debrief, the Build Guide, the regenerated State Doc + CLAUDE.md, and any of `06-Areas/Claude/TODOS.md` / `06-Areas/Claude/Open Questions.md` / `06-Areas/Claude/Operator Warnings.md` modified in Step 4.5. Never stage log files, `.claude/settings.local.json`, or anything flagged by the secret scan without override.

   Ask via AskUserQuestion: "Commit debrief + Build Guide update? (yes/no)". If yes, commit with a `/close` style message.

   **Commit message suffix:** if Step 4.5 modified TODOS.md, Open Questions.md, or Operator Warnings.md (i.e., any per-file summary counter was non-zero — including State Doc Operator Warnings updates), append ` (+ reconciled tracking docs)` to the commit subject line. Suffix omitted if Step 4.5 was skipped or produced zero edits.

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
   Drift check:  {clean / blocked / gateway-unavailable / dry-run-blocked}
   Debrief:      {written <path> / skipped / override-stub / dry-run}
   Build Guide:  {updated / unchanged / dry-run}
   Reconciled:   {TODOS:+N -M ↑P ↓Q ✓R | OQ:+N -M ✓P | Warnings:N lines | skipped (--skip-reconcile) | dry-run}
   Secrets:      {clean / overrides: N / aborted}
   Git:          {committed: <hashes> / no changes / skipped}
   Unpushed:     {e.g. "Python Scripts: 1, BASECAMP: 2, execution: 0" / none}
   Would-block:  {no / yes (N repos | portfolio drift) — dry-run only}
   Mode:         {normal / --dry-run / --claude-ai-summary / --skip-reconcile / combos}
   ==========================================
   ```
