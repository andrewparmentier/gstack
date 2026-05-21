---
name: open
version: 1.1.0
description: |
  Session startup. Reads State Doc, TODOS.md, and prints a compact status summary.
  Use at the beginning of every Claude Code session.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Session Open

Run this at the start of every session. Reads current system state and prints a compact summary.

## Steps

1. Read the State Doc at `06-Areas/Claude/State Doc - Current.md` (relative to BASECAMP vault root at `/Users/parmstar/Documents/OBSIDIAN MASTER/BASECAMP/`).

2. Read TODOS.md at `06-Areas/Claude/TODOS.md`.

2.5. Read `06-Areas/Claude/Operator Warnings.md` if it exists. Include the contents of its `## Active` section in the WARNINGS block of the printed summary (step 3), each line prefixed with `⚠️ Operator Warning:`. If the file is missing, or `## Active` holds only the `(none …)` placeholder, add nothing from it.

3. Print a compact status summary in this exact format:

```
=== SESSION OPEN — {today's date} ===

STACK STATUS:
(from State Doc System Status table — list each script: name, running/not running/scheduled)

OPEN TODOS:
P1:
  - (list all open P1 items)
P2:
  - (up to 5 most recent open P2 items)
P3:
  - (count only, e.g. "3 items")

WARNINGS:
(any drift, errors, or down services from State Doc)

===================================
```

4. Do NOT regenerate any files. Do NOT run state_doc_gen.py. Just read and summarize what exists.

5. If State Doc or TODOS.md is missing, say so and suggest running `python3 ~/Documents/Python\ Scripts/state_doc_gen.py`.
