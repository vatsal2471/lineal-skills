---
name: drake-tax-return
description: >
  Automate tax return data entry in Drake Tax 2025 desktop software using computer-use MCP.
  Covers Form 1065 (partnerships), 1120 (C-corps), 1120S (S-corps), 1040 (individuals),
  990 (exempt orgs), and 1041 (estates/trusts). Use this skill whenever the user asks to
  prepare, enter, file, or automate a tax return in Drake, or mentions Drake Tax, Drake
  software, EF errors, e-filing, or any IRS form numbers (1065, 1120S, 1120, 1040, 990,
  1041, 8825, 4562, 5472, K-1, etc.). Also trigger when the user wants to process prior
  year PDFs, extract tax data, fix EF errors, or build structured input for Drake. Even if
  the user just says "let's do the next return" or "start on [client name]", use this skill.
---

# Drake Tax Return Automation

This skill automates tax return data entry in Drake Tax 2025. It encodes hard-won lessons from completing real returns — Drake UI quirks, required fields that aren't obvious, state-specific requirements, and the optimal sequence for each return type.

## Architecture: Progressive Loading

This skill uses **one reference file per return type**. Only load what you need:

| Return Type | Reference File | Status |
|-------------|---------------|--------|
| 1065 (Partnership) | `references/1065.md` | Complete — 5 returns done |
| 1120 (C-Corp) | `references/1120.md` | Complete — 1 return done |
| 1120S (S-Corp) | `references/1120s.md` | Create after first return |
| 1040 (Individual) | `references/1040.md` | Started — 1 return done (Sood MFJ) |
| 990 (Exempt Org) | `references/990.md` | Create after first return |
| 1041 (Estate/Trust) | `references/1041.md` | Create after first return |

**When a return starts:** Read ONLY the reference file for that return type. Don't load other return types.

**When a new return type is encountered** that doesn't have a reference file yet: Create `references/[type].md` after completing it, following the same structure as existing reference files.

## Workflow: Every Return

### Phase 0: Pull Latest Skill from GitHub (FIRST THING EVERY SESSION)

**Before doing ANY work**, pull the latest version of this skill from GitHub to ensure you have all learnings from previous sessions:

```bash
# Clone or pull the latest skill from GitHub
cd /sessions/$(basename $PWD)
git clone https://github.com/vatsal2471/lineal-skills.git github-repo 2>/dev/null || (cd github-repo && git pull)
```

The repo structure is:
```
lineal-skills/
  drake-tax-return/     ← this skill
    SKILL.md
    references/
      retro-log.md
      1065.md
      1120.md
      1040.md
      return_schemas.md
    scripts/
      preprocess.py
  netsuite-bank-match/  ← other skills
    SKILL.md
```

Read the reference files and retro-log from the cloned repo, NOT from the local skill directory — the repo is always the source of truth.

**GitHub PAT for pushing (store in git config, ask user if not set):**
The user's PAT is configured for `vatsal2471/lineal-skills` with Contents read/write scope. If git push fails with 403, ask the user for a fresh token.

### Phase 1: Pre-process (before touching Drake)

1. **Read source documents** — prior-year PDF return + current-year income data (Excel/CSV/PDF)
2. **Run pre-processing** if applicable:
   ```bash
   python scripts/preprocess.py \
     --prior-year-pdf <path> \
     --current-year-data <path> \
     --return-type 1065 \
     --output return_data.json
   ```
   If the script doesn't support the return type yet, manually extract all data into a structured plan before touching Drake.
3. **Read the return-type reference file** — `references/[type].md`
4. **Read the retro log** — `references/retro-log.md` — check for client-specific or return-type-specific pitfalls
5. **Plan batch sequences** — for each screen, plan the exact click-tab-type sequence for `computer_batch`

### Phase 2: Data Entry (in Drake)

Follow the screen order in the return-type reference file. The general principles:

1. Enter screens in the prescribed order — don't skip or go back
2. Fill ALL blue/required fields on the first pass
3. Use `computer_batch` to enter an entire screen in one call, screenshot at the end
4. One calculate cycle at the end — aim for zero errors on first try

### Phase 3: Verify & Fix

1. Calculate the return (Ctrl+C or Calculate button)
2. Check calculation dialog for green checkmarks (Federal + all states)
3. Open View/Print — check if MESSAGES page exists in left panel
4. If errors exist, read the error codes and fix — refer to the EF error table in the reference file
5. Recalculate and verify clean

### Phase 4: Post-Return Audit (MANDATORY — DO NOT SKIP)

**After EVERY return, before moving on, do all of these:**

1. **Document what happened** — update `references/retro-log.md` with:
   - Client name, EIN, return type, date
   - Actual time spent vs target
   - Every error encountered with root cause and time lost
   - Every navigation issue that slowed entry
   - New pitfalls or tricks discovered
2. **Update the reference file** — if a new EF error, new screen behavior, new field, or faster navigation path was discovered, add it to `references/[type].md`
3. **Create new reference file** — if this was a new return type, create `references/[type].md` from scratch using the template at the bottom of this file
4. **Commit and push to GitHub** — this is how the skill gets version-controlled:
   ```bash
   cd /sessions/$(basename $PWD)/github-repo
   # Copy updated files from the working skill directory back to repo
   cp -r /path/to/drake-tax-return-edit/* drake-tax-return/
   git add drake-tax-return/
   git commit -m "Return: [Client] [Type] — [summary of learnings]"
   git push origin main
   ```
   The commit message should summarize what was learned (e.g., "Sood 1040: 8867 Heads Down Entry, K-1 QBI MFC fix, Q7a trap").
   
   **If git push fails (no auth):** Ask the user for their GitHub PAT for `vatsal2471/lineal-skills`, then:
   ```bash
   git remote set-url origin https://x-access-token:<PAT>@github.com/vatsal2471/lineal-skills.git
   git push origin main
   ```

5. **Package and present** the updated `.skill` file so the user can reinstall with new learnings:
   ```bash
   cd /path/to/skill-creator && python -m scripts.package_skill /path/to/drake-tax-return /path/to/outputs/
   ```
   Then use `present_files` to give the user the `.skill` file with a "Save skill" button.

The skill must get measurably faster with each return. Track time in the retro log. If return #1 of a type took 180 minutes, return #5 should take under 15.

**GitHub is the source of truth.** The `.skill` package is a convenience for reinstalling, but all history lives in the repo at https://github.com/vatsal2471/lineal-skills. Every return = one commit. To see how the skill evolved: `git log --oneline drake-tax-return/`.

---

## Drake UI Rules (All Return Types)

These apply to every return regardless of type.

### OFF-LIMITS: Drake Software Menu Bar (top of main window)

Drake has two different menus. The **Drake software menu bar** runs across the very top of the main Drake window (File, EF, Tools, Reports, Last Year Data, Setup, Help). These are firm-level admin/setup functions for configuring Drake itself. **Never click any of these during return preparation:**

- **EF** — e-file transmission settings (admin only)
- **Tools** — software utilities and diagnostics (admin only)
- **Setup** — firm setup, preparer info, options (admin only)
- **Last Year Data** — prior year data import tools (admin only)
- **Reports** — firm-level reporting (admin only)
- **Help** — Drake help system (not needed)

This is different from the **return-level toolbar** (Open/Create, Calculate, Print, View, e-File Status, CSM, Scheduler, etc.) and the **Data Entry screen menu** inside an open return — those are fine to use and are how you navigate screens, calculate, and verify returns.

### Batching for Speed

Use `computer_batch` to enter multiple fields per call:
1. Click the first field
2. Type the value
3. Press Tab to advance
4. Type the next value
5. Repeat
6. Take ONE screenshot at the end

Every individual screenshot-click-type round-trip costs ~3-5 seconds. A return with 80 fields entered one at a time = 6-8 minutes of pure mechanical overhead. Batching cuts this to under 2 minutes.

### Screen Navigation

- **Close before searching.** If a form screen is open, text goes into form fields, not the search box. Exit the current screen (Escape or Exit button) before typing a screen code.
- **Screen codes** are short: `8825`, `4562`, `K1`, `PIN`, `SEC`, `M1`, `SCH B`, `LOSS`, `5472`, etc. Type in the search box at top-left and press Enter.
- Use the Exit button to close a screen — do NOT use Ctrl+Tab or other shortcuts for switching.
- For state screens: click the States tab first, then select the state.
- For K-1/shareholder screens: click the relevant partner/shareholder tab.

### Field Editing

- **Single-click** positions cursor in a value field — then type your value
- **Double-click** opens a Detail Worksheet popup — only use intentionally
- **Triple-click** — NEVER use on amount fields, triggers unintended behavior
- To clear a field: single-click → Home → Shift+End → Delete
- To replace a value: single-click → Home → Shift+End → type new value
- For dropdowns: single-click to open, type first letter(s) to jump, Enter to select

### Checkboxes: The Heads Down Entry Solution

**If any checkbox won't respond to direct mouse clicks, immediately switch to Heads Down Entry mode.** Don't waste time trying different coordinates — this is the universal fix discovered after 50+ failed click attempts on Form 5472.

1. Right-click anywhere on the screen → "Heads Down Entry" (or press **Ctrl+N**)
2. Every field gets a number
3. Type a field number + Enter to navigate to that field
4. Click the value input box and type **X** + Enter to check the checkbox
5. Press Escape to exit Heads Down Entry mode

### Error Checking

- **View/Print is your best friend.** After calculating, go to View/Print and check for a MESSAGES page in the left panel.
- **Right-click an EF message** → "View Full Text" for the complete description.
- **Double-click an EF message** to jump directly to the problem screen.
- The calculation dialog (green/red checkmarks) only tells eligible/not eligible — actual error details are in View/Print.

### Adobe PDF Printing Limitation

If you need to print a form to PDF (e.g., for CA 5472 attachment), the Adobe Save As dialog spawns in a separate Windows process that computer-use cannot control. **Tell the user upfront they must handle this step manually.**

---

## Firm-Level Constants (Lineal CPA)

| Field | Value |
|-------|-------|
| ERO EFIN | 367379 |
| ERO PIN | 75757 |
| Firm Phone | 847-287-1040 |
| Default Accounting Method | Accrual |

---

## State-Specific Rules

### Illinois (IL)
- **Screen 2 State tab**: Fill BOTH "Books in care of" AND "Principal Place of Business" sections
- **ILK1**: Create IL K-1-P entries in IL package > Schedule K1 tab for every Federal K-1 partner
- **IL Authorization**: Check "Authorization statement" on the IL e-File auth screen (linked from PIN)
- **SSN on PIN screen**: Required for IL e-file (causes IL 9320 if missing)

### California (CA)
- **CA Form 5472 PDF attachment**: CA requires PDF copy of Form 5472 attached via EF > PDF Attachments. Error 0137 flags this. User must handle the Adobe Save As dialog manually.
- **CA 100 minimum franchise tax**: $800 regardless of income/loss
- **CA NOL**: Verify whether CA conforms to Federal NOL carryforward amounts

### Other States
Add new state sections here as returns for other states are completed.

---

## Template: Creating a New Return-Type Reference File

When you complete the first return of a new type, create `references/[type].md` using this structure:

```markdown
# Drake Tax [Type] — [Full Name]

Based on completing [Client Name] ([EIN]), [brief description of return characteristics].

## Screen Order

| Step | Screen | Time Target | What |
|------|--------|-------------|------|
| 1 | Screen 1 | 60s | Entity info |
| ... | ... | ... | ... |
| N | Calculate | 30s | Verify green checkmarks |

## Screen Details

### [Screen Name]
**Navigation:** [how to get there]
**Time target:** [seconds]
**Batch sequence:** [click-tab-type plan]

| Order | Field | Blue? | Example | Notes |
|-------|-------|-------|---------|-------|
| 1 | ... | ... | ... | ... |

**Pitfalls:**
- [known issues]

## Common EF Errors

| Code | Issue | Fix |
|------|-------|-----|
| ... | ... | ... |

## State-Specific Notes
[Anything unique for each state filed with this return type]

## Pre-Processing JSON Schema
[JSON structure if applicable]
```

Then update the routing table in this SKILL.md to mark the new type as complete.
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             