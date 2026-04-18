---
name: close
version: 1.1.0
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

## Steps

1. **Regenerate State Doc and CLAUDE.md:**
   ```bash
   cd "/Users/parmstar/Documents/Python Scripts" && python3 state_doc_gen.py
   ```
   This updates: State Doc, Onboarding Paste, and BASECAMP CLAUDE.md.

2. **Show git status:**
   ```bash
   cd "/Users/parmstar/Documents/Python Scripts" && git status && git diff --stat
   ```
   Show the user what's changed but NOT committed. Do NOT commit automatically — ask first.

3. **Ask the user what was accomplished this session:**
   Use AskUserQuestion:
   - "What was built or fixed this session?" (free text)
   
   If the user provides notes, proceed to step 4. If they say skip, go to step 5.

4. **Write session debrief:**
   Create a new file at:
   `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/06-Areas/Claude/Session Debriefs/Session-Debrief-{YYYY-MM-DD}.md`
   
   If a debrief already exists for today, add a suffix: `-a`, `-b`, etc.
   
   Use this structure:
   ```markdown
   ---
   type: session-debrief
   date: {YYYY-MM-DD}
   ---
   
   # Session Debrief — {YYYY-MM-DD}
   
   ## What Was Built or Fixed
   (from user's answer)
   
   ## What's Still Pending
   (extract open P1 items from TODOS.md)
   
   ## Next Session P1
   (the single most important thing for next session)
   ```

5. **Update the Build Guide Current Phase block:**

   File: `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/04-Builds/Master Financial Pipeline/AMP_Basecamp_Build_Guide.md`

   First, verify the `## Current Phase` block exists in the file. If it is missing, STOP and print:
   ```
   ERROR: Current Phase block not found in Build Guide. Aborting close.
   ```

   If present, derive four values from the session debrief just written + recent git log + TODOS.md:
   - **Phase:** current phase label (e.g. "Phase 0 (partial)", "L4 in progress", "L4.2 complete")
   - **Next step:** the single next concrete action in one sentence
   - **Last shipped:** most recent completed milestone with date (e.g. "L2 event_log + rollup — 2026-04-17")
   - **Blockers:** any open blockers in one sentence, or "None"

   Rules for derivation:
   - If a value cannot be determined from session context, keep the existing value from the block — never write "Unknown" or leave blank.
   - Use git log to confirm the last shipped item and date if not clear from the debrief.
   - Use open P1 items from TODOS.md as the source for Next step and Blockers.

   Replace ONLY the four bold-labeled lines inside the block. Use Edit with precise old/new strings scoped to those exact lines. Do NOT touch any other part of the Build Guide.

   The four lines to replace have this form:
   ```
   **Phase:** <value>
   **Next step:** <value>
   **Last shipped:** <value>
   **Blockers:** <value>
   ```

6. **Ask about git commit:**
   If there are uncommitted changes, ask:
   - "Commit these changes? If yes, I'll draft a commit message."
   
   If approved, stage relevant files (NOT log files or .claude/settings.local.json) and commit. Include the Build Guide in the same commit as the session debrief.

7. **Print session summary:**
   ```
   === SESSION CLOSED — {today's date} ===
   State Doc: regenerated
   Debrief: {written / skipped}
   Build Guide: Current Phase updated
   Git: {committed hash / no changes / skipped}
   ==========================================
   ```
