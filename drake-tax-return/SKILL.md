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

> **⚠️ NON-NEGOTIABLE — READ THIS FIRST:**
>
> **After EVERY return, before telling the user "done":**
> 1. Update `references/retro-log.md` with errors, time lost, new pitfalls
> 2. Update `references/[type].md` with new learnings (field maps, EF errors, coordinates)
> 3. `git add`, `git commit`, `git push` to `vatsal2471/lineal-skills`
>
> The return is NOT DONE until the skill is updated and pushed. Do not wait for the user to remind you. This has been a recurring failure — the user has had to ask every single time. Fix this by treating the GitHub push as the final step of calculate-verify, not as a separate "Phase 4" afterthought.

This skill automates tax return data entry in Drake Tax 2025. It encodes hard-won lessons from completing real returns — Drake UI quirks, required fields that aren't obvious, state-specific requirements, and the optimal sequence for each return type.

## Architecture: Progressive Loading

This skill uses **one reference file per return type**. Only load what you need:

| Return Type | Reference File | Status |
|-------------|---------------|--------|
| 1065 (Partnership) | `references/1065.md` | In progress — 2 returns documented (Nautilus Jeffery, Nautilus Maple). SSD/Cornell/7220 visible in CSM but never had retro entries captured. |
| 1120 (C-Corp) | `references/1120.md` | In progress — 1 return done (Atamin) |
| 1120S (S-Corp) | `references/1120s.md` | Create after first return |
| 1040 (Individual) | `references/1040.md` | In progress — 3 returns done (Sood MFJ, Dvorak HOH, Link MFJ) |
| 990 (Exempt Org) | `references/990.md` | Create after first return |
| 1041 (Estate/Trust) | `references/1041.md` | Create after first return |

**When a return starts:** Read ONLY the reference file for that return type. Don't load other return types.

**When a new return type is encountered** that doesn't have a reference file yet: Create `references/[type].md` after completing it, following the same structure as existing reference files.

## Workflow: Every Return

### Phase 0: Pull Latest Skill from GitHub (FIRST THING EVERY SESSION)

**Before doing ANY work**, pull the latest version of this skill from GitHub to ensure you have all learnings from previous sessions:

```bash
# Load GitHub PAT. Checked in this priority order:
#   1. $GITHUB_PAT env var — TRULY persistent if user set it in Claude Code settings
#   2. Session-local credential helper at /tmp/.git-session-credentials (this session only)
#   3. Nothing — prompt user to paste PAT, then save to BOTH #1's session env AND the helper
#
# IMPORTANT — READ THIS CAREFULLY:
# There is NO persistent writable filesystem location inside this sandbox. Earlier
# versions of this skill claimed to save the PAT at mnt/.claude/session-env/.github-pat
# and called it "persistent" — that was WRONG. mnt/.claude is mounted from a
# session-local UUID path (local_<UUID>) and is wiped every new session. Do not
# re-introduce that mechanism. The ONLY way to get a PAT to survive across sessions
# is for the user to set it as a Claude Code env var ($GITHUB_PAT).

if [ -n "$GITHUB_PAT" ]; then
  echo "Using PAT from \$GITHUB_PAT env var (persistent)"
elif [ -f /tmp/.git-session-credentials ]; then
  # Previously set this session, not persistent across sessions
  GITHUB_PAT=$(sed -n 's|.*x-access-token:\([^@]*\)@.*|\1|p' /tmp/.git-session-credentials | head -1)
  echo "Using PAT from session credential helper (session-local)"
else
  echo "NO PAT AVAILABLE — stop and ask the user:"
  echo "  'I need a fine-grained GitHub PAT for vatsal2471/lineal-skills (Contents: Read and write).'"
  echo "  'For true persistence across sessions, set GITHUB_PAT as a Claude Code env var.'"
  echo "  'Otherwise I'll keep asking for it every session — this is a sandbox limitation, not a skill bug.'"
  # After user pastes PAT, save to session credential helper:
  #   printf 'https://x-access-token:%s@github.com\n' "$GITHUB_PAT" > /tmp/.git-session-credentials
  #   chmod 600 /tmp/.git-session-credentials
  #   git config --global credential.helper 'store --file=/tmp/.git-session-credentials'
fi

REMOTE_URL="https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git"

# Clone or pull the latest skill from GitHub
cd /sessions/$(basename $PWD)
if [ -d github-repo ]; then
  cd github-repo && git remote set-url origin "$REMOTE_URL" && git pull && git remote set-url origin "https://github.com/vatsal2471/lineal-skills.git"
else
  git clone "$REMOTE_URL" github-repo && cd github-repo && git remote set-url origin "https://github.com/vatsal2471/lineal-skills.git"
fi

# Configure git identity and credential helper for the rest of the session
cd /sessions/$(basename $PWD)/github-repo
git config user.email "vatsal@lineal.cpa"
git config user.name "Vatsal"
git config credential.helper "store --file=/tmp/.git-session-credentials"
printf 'https://x-access-token:%s@github.com\n' "$GITHUB_PAT" > /tmp/.git-session-credentials
chmod 600 /tmp/.git-session-credentials
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

**GitHub PAT — persistence reality check:**

The Phase 0 block above tries three sources in order: (1) `$GITHUB_PAT` env var, (2) `/tmp/.git-session-credentials` file from earlier in this same session, (3) prompt the user. **Only option 1 survives across sessions** — and only if the user has added `GITHUB_PAT` to their Claude Code environment variables. Everything else in this sandbox is session-local.

If the user is being asked for the PAT every session, tell them plainly: *"This is a sandbox limitation — `mnt/.claude/` is mounted from a per-session UUID path, so there's no persistent writable location. To stop being asked, add `GITHUB_PAT=<your_pat>` as a Claude Code env var."* Do **not** claim to save the PAT to `mnt/.claude/session-env/` or any other in-sandbox path and call it persistent — that claim is false and has burned the user multiple times.

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

### Phase 3: Verify, Fix, and Ship

1. Calculate the return (Ctrl+C or Calculate button)
2. Check calculation dialog for green checkmarks (Federal + all states)
3. Open View/Print — check if MESSAGES page exists in left panel
4. If errors exist, read the error codes and fix — refer to the EF error table in the reference file
5. Recalculate and verify clean
6. **IMMEDIATELY update skill and push to GitHub** — see Phase 4. Do this NOW, not "later."

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
   
   **If git push fails with auth error:** Phase 0 should have configured a session credential helper at `/tmp/.git-session-credentials`. If the push still fails, the helper is missing or expired. Ask the user for a fresh fine-grained PAT for `vatsal2471/lineal-skills` (Contents: Read and write), then:
   ```bash
   GITHUB_PAT='<paste from user>'
   printf 'https://x-access-token:%s@github.com\n' "$GITHUB_PAT" > /tmp/.git-session-credentials
   chmod 600 /tmp/.git-session-credentials
   git config credential.helper 'store --file=/tmp/.git-session-credentials'
   git push origin main
   ```
   Do NOT try to write the PAT to `mnt/.claude/session-env/` and claim it'll persist — it won't. If the user wants cross-session persistence, tell them to set `GITHUB_PAT` as a Claude Code env var.

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

### Heads Down Entry (HDE) — PRIMARY navigation mode for data entry

**HDE is not an escape hatch — it is the default, fastest, most reliable way to enter data into any Drake screen.** Pixel-clicking was the old approach. After three consecutive 1040 returns where HDE dramatically outperformed direct clicking (including 60+ min wasted on Form 8867 in Sood, 20+ min on Form 8863 in Link), the policy is now: **turn HDE on at the top of every screen that has more than a handful of fields and stay in HDE until the screen is done.** Switch back to direct clicking only for the narrow exceptions listed below.

**HARD ENFORCEMENT RULE — read this before every screen:**

Before executing ANY `computer_batch` that touches a Drake data entry screen, you MUST write one line of text in your own turn stating which mode you're using. One of:
- `"HDE mode — pressing Ctrl+N first"` (the expected default)
- `"Direct click — this screen qualifies for exception X because Y"` (must cite a specific exception from the list below)

If you catch yourself about to pixel-click a field that lives inside a screen grid without having declared an exception, STOP. That is a skill violation. Back up, press Ctrl+N, and enter HDE. The user has flagged this same drift five times across Sood, Dvorak, Link, and Nielsen — it is the #1 cause of wasted time on this skill and the #1 reason the user has to repeat themselves. No more.

The drift happens when a screen "looks simple" (one or two fields) or when you already have a field coordinate in memory from a prior screenshot. Both are traps: "just this one click" turns into ten, and cached coordinates go stale the moment Drake's layout shifts. Always default to HDE.

**Why HDE wins over pixel-clicking:**
- **Resolution-independent.** Field 17 is field 17 regardless of window size, DPI, Drake version, or theme. Coordinate maps break the instant Drake's layout shifts.
- **Unambiguous.** No risk of hitting the adjacent Yes/No/N/A column on a due-diligence checkbox — type the field number and you land exactly on that input.
- **Batchable.** The entire HDE flow for a screen is keyboard-only (click input → type number → Return → click value → type value → Return), which means you can pack dozens of field entries into a single `computer_batch` call. Pixel-clicking requires a fresh screenshot every few actions to re-verify alignment.
- **Works uniformly for every field type.** Text, amount, date, dropdown, checkbox — the HDE entry pattern is the same. Direct clicking has different rules for each.
- **No double-click traps.** Single-click on an amount field in direct mode can accidentally become a double-click and open a detail worksheet. HDE never does this.
- **Immune to "triple-click deletes data" bugs** that have bitten returns before.

**How to turn it on and off:**
1. Press **Ctrl+N** (or right-click → Heads Down Entry). Every field on the screen gets a number badge.
2. Stay in HDE for the whole screen.
3. Press **Escape** (or Ctrl+N again) when you're done with that screen.

**The HDE entry pattern (batch-friendly):**

For each field, HDE gives you an "input box" where you type the field number and a "value box" where you type the value. The coordinates below are what Sood/Dvorak/Link all used on Drake Tax 2025 — verify with a screenshot once per screen, then batch.

**For text / amount / date fields** (anything where you're typing a value):
```
click (548, 87)    # field-number input box
type "17"          # the HDE number of the field you want
key Return
click (585, 91)    # value input box
type "2500"        # the text/amount/date you want
key Return
# repeat for next field — no screenshot needed between fields
```

**For checkbox fields — CRITICAL PATTERN, verified on Nielsen:**
```
click (548, 87)    # field-number input box
type "17"          # the HDE number of the checkbox
key Return
key space          # SPACE toggles the checkbox
key Return         # commit the toggle
```

**DO NOT use `type "X"` for checkboxes.** An earlier version of this skill told you to type `X + Return` — that is WRONG and cost real time on Sood, Dvorak, Link, and Nielsen before it was figured out. The working pattern is literally `field_num → Return → Space → Return`. The `Space` key is what actually flips the box; `Return` commits and advances.

Pack as many of these as you need into one `computer_batch`. A full Form 8867 (12+ checkboxes) takes one batch and one verification screenshot instead of dozens of click-screenshot rounds.

**Yes/No/N/A groups:** each column has its own field number. To set Q7 to "Yes", toggle the "Yes" column's field number ON with Space, then toggle the "No" and "N/A" columns for that same question OFF with another Space (space toggles, so if they're already off you'll turn them on — check with a screenshot first). For a fresh screen with nothing selected yet, just Space-toggle the one correct column.

**Text, amount, date values:** just type the value in the value box and press Return. HDE handles tabbing out automatically.

**The narrow exceptions — when direct clicking is still appropriate:**
- Opening a detail worksheet (double-click on an amount field) — HDE won't open the sub-window.
- Navigating between tabs at the top of a screen (States tab, PTR K tab, etc.) — those are outside the HDE field grid.
- Clicking Exit/Save/Calculate/View buttons on the toolbar — toolbar buttons aren't HDE-numbered.
- Selecting from an open dropdown list — fine to click the item once the dropdown is open.
- Screen navigation via the search bar at top-left.

Everything else — every text field, every amount, every checkbox, every dropdown selection inside a screen — should go through HDE by default.

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