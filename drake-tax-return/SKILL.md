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

**Before doing ANY work**, pull the latest version of this skill from GitHub to ensure you have all learnings from previous sessions.

**POST-COMPACTION RULE (added 2026-04-11 after Hammoud; regressed again Thompson 2026-04-11 — user frustration documented):** If this session is resuming from a context-window compaction mid-return (you see a "Summary" block and are told to "continue from where you left off"), you MUST still re-run the PAT-load block below before any `git push`. **Do not trust shell state inherited from pre-compaction bash calls** — each `Bash` tool call runs in a fresh shell, so any `GITHUB_PAT` env var set earlier is gone. Symptom of skipping this: `git push` fails with `could not read Username for 'https://github.com'`, and you start asking the user for a PAT that has been sitting in the persistent mcpb-cache the whole time. This failure mode has now burned the user on **Patel, Hammoud, AND Thompson**. Writing more warning paragraphs here does not fix it — the warnings are already exhaustive. The fix is behavioral: before you write ANY `git push` command, the one-liner below must already have been run in this session.

```bash
# The ONLY push command. Memorize this and do not type a different one:
PERSIST_PAT="$HOME/mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache/.github-pat"
GITHUB_PAT=$(tr -d '\r\n ' < "$PERSIST_PAT") && \
  cd "$HOME/github-repo" && \
  git push "https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git" main
```

Expected successful output: `<oldsha>..<newsha>  main -> main`. A `fatal: unable to write credential store: Operation not permitted` line ABOVE the push result is non-fatal; the push still succeeded. Only ask the user for a PAT if `$PERSIST_PAT` is missing/empty (verify with `ls -la "$PERSIST_PAT"`) — never before checking.

---

**Pull the latest version of this skill from GitHub to ensure you have all learnings from previous sessions:**

```bash
# Load GitHub PAT. Checked in this priority order:
#   1. PERSISTENT mcpb-cache path (survives across sessions — primary store)
#   2. PERSISTENT skills-root fallback (user-managed, survives plugin reinstall)
#   3. Session-local credential helper /tmp/.git-session-credentials (current session only)
#   4. Nothing — prompt user to paste PAT, then WRITE it to the persistent store
#
# WHY THESE PATHS WORK (and earlier attempts didn't):
# The sandbox has many virtiofs mounts but almost all of them are under
# local_<session-UUID>/ and get wiped every session. Two paths are NOT:
#
#   (a) mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache
#       → rw, keyed by stable org/account UUIDs — PERSISTENT + SANDBOX-WRITABLE.
#         This is the primary store. Zero user involvement after first setup.
#
#   (b) mnt/.claude/skills  (and its root, e.g. mnt/.claude/skills/.github-pat)
#       → ro from sandbox, but backed by a persistent Windows folder at
#         C:\Users\<user>\AppData\Local\Packages\Claude_*\LocalCache\Roaming\Claude\
#         local-agent-mode-sessions\skills-plugin\<org-uuid>\<account-uuid>\skills
#         The user can drop .github-pat there from File Explorer; readable every
#         future session. Fallback for users who want full control or whose
#         engineering plugin gets uninstalled/reinstalled.
#
# Do NOT reintroduce mnt/.claude/session-env/ or any path containing local_<UUID>
# and call it persistent — those are all wiped between sessions.

# CRITICAL: Derive the session directory at runtime — NEVER hardcode a session name,
# and NEVER pick from `ls /sessions/*/`. The /sessions mount contains directories for
# EVERY user's sessions on this machine (often 10+), and `ls /sessions/*/ | head -1`
# picks the alphabetically-first one — which is almost never the current session's
# and returns permission-denied on every PAT read. This burned the workflow on
# Patel 2025 (2026-04-11): `affectionate-zen-mccarthy` sorted before my real
# session `beautiful-fervent-tesla`, Phase 0 never found the PAT, skill push
# silently dead-ended.
#
# The correct source is $HOME. Every sandbox session has $HOME set to its own
# session directory (e.g. /sessions/beautiful-fervent-tesla), writable by the
# current uid, and guaranteed to be THIS session and not someone else's.
SESSION_ROOT="$HOME"
# Sanity check: if $HOME isn't a /sessions/* path for some reason, fall back to
# finding the one owned by our current uid (not just the alphabetically first).
if [[ "$SESSION_ROOT" != /sessions/* ]]; then
  SESSION_ROOT=$(find /sessions -mindepth 1 -maxdepth 1 -type d -uid "$(id -u)" 2>/dev/null | head -1)
fi
PERSIST_DIR="$SESSION_ROOT/mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache"
PERSIST_PAT="$PERSIST_DIR/.github-pat"
SKILLS_ROOT_PAT="$SESSION_ROOT/mnt/.claude/skills/.github-pat"

# NOTE on /tmp/.git-session-credentials: this file is often present but owned by
# nobody:nogroup with mode 600, which means the CURRENT session's user can neither
# read nor write it (stale from a previous namespace mapping). Always test with -r
# (readable), NOT -f (exists), or the fall-through logic silently dead-ends.
if [ -r "$PERSIST_PAT" ] && [ -s "$PERSIST_PAT" ]; then
  GITHUB_PAT=$(cat "$PERSIST_PAT")
  echo "Using PAT from persistent mcpb-cache ($PERSIST_PAT)"
elif [ -r "$SKILLS_ROOT_PAT" ] && [ -s "$SKILLS_ROOT_PAT" ]; then
  GITHUB_PAT=$(tr -d '\r\n ' < "$SKILLS_ROOT_PAT")
  echo "Using PAT from persistent skills-root fallback ($SKILLS_ROOT_PAT)"
  # Mirror into mcpb-cache for faster access next session
  mkdir -p "$PERSIST_DIR" 2>/dev/null
  printf '%s' "$GITHUB_PAT" > "$PERSIST_PAT" && chmod 600 "$PERSIST_PAT" 2>/dev/null
elif [ -r /tmp/.git-session-credentials ]; then
  GITHUB_PAT=$(sed -n 's|.*x-access-token:\([^@]*\)@.*|\1|p' /tmp/.git-session-credentials | head -1)
  echo "Using PAT from session credential helper (session-local)"
else
  echo "NO PAT AVAILABLE — stop and ask the user:"
  echo "  'I need a fine-grained GitHub PAT for vatsal2471/lineal-skills (Contents: Read and write).'"
  echo "  'I'll write it to the persistent mcpb-cache so you only have to paste it once.'"
  # AFTER user pastes PAT in chat, write it to the persistent store:
  #   printf '%s' "$GITHUB_PAT" > "$PERSIST_PAT" && chmod 600 "$PERSIST_PAT"
  # Note: files in mcpb-cache can be OVERWRITTEN but NOT deleted from the sandbox.
  # To rotate the PAT, just overwrite with a new value.
fi

REMOTE_URL="https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git"

# Clone or pull the latest skill from GitHub.
# Use $SESSION_ROOT (derived from $HOME above) — NOT `ls /sessions | head -1`, which
# returns some OTHER user's session directory on a shared machine.
SESSION_DIR="$SESSION_ROOT"
cd "$SESSION_DIR"
if [ -d github-repo ]; then
  cd github-repo && git remote set-url origin "$REMOTE_URL" && git pull && git remote set-url origin "https://github.com/vatsal2471/lineal-skills.git"
else
  git clone "$REMOTE_URL" github-repo && cd github-repo && git remote set-url origin "https://github.com/vatsal2471/lineal-skills.git"
fi

# Configure git identity for the rest of the session.
cd "$SESSION_DIR/github-repo"
git config user.email "vatsal@lineal.cpa"
git config user.name "Vatsal"

# Only set up credential.helper if /tmp/.git-session-credentials is actually
# writable by us. If it's owned by nobody:nogroup (stale from a prior namespace),
# git will emit "unable to write credential store" on every push — a non-fatal
# warning but a confusing one. Instead, unset the helper and rely on the PAT
# being embedded in the remote URL for pushes.
if [ -w /tmp/.git-session-credentials ] || ( : > /tmp/.git-session-credentials 2>/dev/null ); then
  git config credential.helper "store --file=/tmp/.git-session-credentials"
  printf 'https://x-access-token:%s@github.com\n' "$GITHUB_PAT" > /tmp/.git-session-credentials
  chmod 600 /tmp/.git-session-credentials
else
  echo "Note: /tmp/.git-session-credentials not writable (stale nobody:nogroup file)."
  echo "Unsetting credential.helper and using PAT-in-URL for pushes instead."
  git config --unset credential.helper 2>/dev/null || true
  git remote set-url origin "https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git"
  # IMPORTANT: before committing the final "done" push, re-check the remote URL
  # doesn't leak the PAT into the log. The preferred workflow is to leave origin
  # as the clean https URL and pass the PAT-URL to `git push` directly:
  #   git push "https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git" main
fi
```

**The "unable to write credential store" warning on push is NON-FATAL.** If you see it, the push still went through — look at the `To https://github.com/...` line below it and check for `<oldsha>..<newsha>  main -> main`. This warning happens whenever `credential.helper=store` is configured but the target file is not writable by us. The Phase 0 block above tries to avoid setting that helper in the first place, but if you inherit a repo from a previous session that already has it configured, you will see the warning. Do not panic and do not ask the user for a new PAT — verify the push result by running `git log origin/main -1` after fetching.

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

**GitHub PAT — persistence architecture (this actually works):**

The Phase 0 block above checks four sources in order: (1) persistent mcpb-cache `.github-pat`, (2) persistent skills-root `.github-pat` fallback, (3) session-local `/tmp/.git-session-credentials`, (4) prompt the user.

**Option 1 (primary)** — `mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache/.github-pat`. This path is mounted from a Windows directory keyed by **stable** org/account UUIDs (no `local_<session-UUID>` in the path), so it persists across sessions. The mount is `rw` from inside the sandbox, so you can write the PAT here directly after the user pastes it — no Windows-side action needed. **Files can be overwritten but not deleted** from inside the sandbox; to rotate the PAT, just overwrite with a new value.

**Option 2 (fallback)** — `mnt/.claude/skills/.github-pat`. The skills folder is mounted `ro` into the sandbox but backed by a persistent Windows directory: `C:\Users\<user>\AppData\Local\Packages\Claude_*\LocalCache\Roaming\Claude\local-agent-mode-sessions\skills-plugin\<org-uuid>\<account-uuid>\skills`. Instruct the user to drop a file named `.github-pat` at the root of that folder (same level as `drake-tax-return/`, `pptx/`, etc., NOT inside any skill subfolder) containing just the PAT. Survives engineering-plugin uninstall/reinstall; user-controlled. Phase 0 mirrors it into mcpb-cache on first read for speed.

**Historical note:** earlier versions of this skill claimed `mnt/.claude/session-env/.github-pat` was persistent. That was wrong — `mnt/.claude` is mounted from a `local_<session-UUID>` path and gets wiped every session. Do NOT reintroduce any storage path that contains `local_<UUID>` or treat it as persistent. The test is: `mount | grep <path>` — if the source path contains `local_<something>`, it's session-local; if it only contains the stable org/account UUIDs, it's persistent.

### Phase 1: Pre-process (before touching Drake)

1. **Read source documents — EVERY PAGE OF EVERY DOCUMENT (repeat offender, flagged 2026-04-11).** Skimming is a Phase 1 failure and the #1 way to ship a wrong return.

   **Mandatory document intake procedure:**
   - **(a) Enumerate.** List every uploaded file with its full path. Use `ls /sessions/*/mnt/uploads/` or the paths the user provided. Print a numbered list — this is the "documents provided" ground truth.
   - **(b) Get page counts BEFORE reading.** For each PDF: `python3 -c "from pypdf import PdfReader; print(len(PdfReader('<file>').pages))"` or `pdfinfo <file>`. For each .xlsx: list sheets with row counts. Print `filename → N pages` for each.
   - **(c) Read every page.** PDFs over 10 pages need `pages:` ranges with the `Read` tool (20 pages/call cap), so read in chunks and keep a running counter. After each read, log `<filename>: pages 1-N read, N/N complete`. Do not move to (d) until every file's counter equals its total.
   - **(d) Read every sheet of every spreadsheet.** An 8-sheet workpaper is 8 documents, not 1. Do not assume Sheet1 is all that matters.
   - **(e) Do NOT rely on the Cowork context-window preview.** When a user uploads a PDF, Cowork sometimes renders the first few pages inline as images. That preview is NOT a substitute for `Read`-ing the file — it's truncated and always misses pages past the preview budget. If a document is on disk, open it with the appropriate tool regardless of what the inline render shows.
   - **(f) Cross-check.** The plan output must contain a "Documents read" section listing every file + "pages X of X" for each. If counts don't match step (a), Phase 1 isn't finished — go back.
   - **Exception:** if the user explicitly excludes a document (e.g., "skip the Pershing 1099, that's a different trust"), document the exclusion in the plan so Phase 4 audit can match it.

   **Why this keeps burning us:** tax workpapers bury critical data on arbitrary pages — a K-1 on page 23 of a 40-page PDF, a 1099-R stuck between brokerage statement pages, a Schedule E rental disclosed on page 7 of a "cover letter." Missing it at Phase 1 means either a wrong return or a mid-Phase-2 re-plan. Phase 1 is where reading is cheapest and missing is smallest — read everything there, once, completely. See 1040.md Rule 10 for the full policy.

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

**Drake field color legend — READ THIS BEFORE YOU TOUCH THE KEYBOARD (REPEAT OFFENDER, flagged again 2026-04-11):**

| Color | Meaning |
|-------|---------|
| **BLUE** (background) | **Required for e-file.** Must be filled on first pass. Missing blue = EF error. |
| **YELLOW** (background) | **The field currently has focus (cursor indicator).** Has ZERO correlation with required status. |
| Gray / masked dots | Privacy masking on SSN/EIN fields, OR a blank required field. Verify with HDE before assuming populated. |
| Red outline | Validation error after calc. |

**The trap that keeps catching me:** when you land on a blue required field, it **turns yellow** because focus lands on it, so the screenshot shows yellow and I pattern-match "yellow = important/required." That's wrong — the yellow is the cursor highlight, and it moves with wherever I clicked last. If I want to know whether a field is actually required, I must: (a) check the atlas in `references/[type].md`, or (b) calculate and read the EF errors, or (c) look at the field's **unfocused** color (which requires clicking away from it first). **Never** conclude a field is required from a screenshot where it's yellow — that's the focus color.

Rule-of-thumb: if writing notes that say "field X is yellow," STOP. Yellow is not a persistent property of the field. Write the label instead — "BLUE required" or just "required."

---

Follow the screen order in the return-type reference file. The general principles:

1. Enter screens in the prescribed order — don't skip or go back
2. **`Ctrl+N` is the FIRST action on every screen, no exceptions** (Rule 1 in 1040.md)
3. Fill ALL **blue** required fields on the first pass (NOT yellow — yellow means focused, see color legend above)
4. Use `computer_batch` to enter an entire screen in one call, screenshot at the end
5. One calculate cycle at the end — aim for zero errors on first try
6. First time you encounter a screen type: transcribe the HDE field-number overlay into the atlas in `references/[type].md` BEFORE data entry (Rule 27). Every subsequent encounter: skip the screenshot, go straight to `computer_batch` from the atlas + preprocessed values.

### Phase 3: Verify, Fix, and Ship

1. Calculate the return (Ctrl+C or Calculate button)
2. Check calculation dialog for green checkmarks (Federal + all states)
3. Open View/Print — check if MESSAGES page exists in left panel
4. If errors exist, read the error codes and fix — refer to the EF error table in the reference file
5. Recalculate and verify clean
6. **IMMEDIATELY update skill and push to GitHub** — see Phase 4. Do this NOW, not "later."

### Phase 4: Post-Return Audit (MANDATORY — DO NOT SKIP)

**After EVERY return, before moving on, do all of these:**

1. **Time + shortcut audit (MANDATORY — do this FIRST, before writing the retro-log)** — the skill only gets faster if every return ends with an honest click-by-click review. Do not skip this even if the return went well. Do not wait to be asked.
   - Scroll back through the session's tool calls for this return. Count: total elapsed time, # of screenshots, # of `left_click` calls, # of `computer_batch` calls, # of times HDE was used vs. mouse/tab data entry.
   - For EVERY `left_click` on a toolbar button, tab header, menu item, or Exit button, ask: **"could this have been a keyboard shortcut or a screen-code search?"** Every click that should have been a keystroke is a time-leak finding.
   - For every data-entry screen, ask: **"did I enter HDE (Ctrl+N) immediately, stay in HDE the whole screen, and exit with Esc/Ctrl+N?"** Any screen where HDE was dropped mid-entry is a discipline finding.
   - For every screen, count tool calls. The target is **3 tool calls per screen** (screenshot → computer_batch with all HDE actions → verification screenshot). Any screen that exceeded 3 calls is a screenshot-discipline finding.
   - Record findings as a bulleted "Shortcut audit" subsection inside this return's retro-log entry. Format: `[click description] → [shortcut that should have been used] → [est. seconds lost]`. Sum the lost seconds at the bottom.
   - If a shortcut was discovered or confirmed during the return, promote it from the "TO VERIFY" column to the "VERIFIED" column of the Drake keyboard shortcut table in `references/[type].md`.
   - If the same time-leak shows up in 2+ consecutive returns, it becomes a rule in `references/[type].md` (not just a retro-log note) so the next return is forced to fix it.
2. **Document what happened** — update `references/retro-log.md` with:
   - Client name, EIN, return type, date
   - Actual time spent vs target, plus the audit findings from step 1
   - **Documents read** — explicit list of every uploaded file with `pages X of X` or `sheets Y of Y` confirmation. If this line is missing or shows incomplete counts, that's a Rule 10 violation — call it out as a discipline finding in the retro and note what was missed. No return is "done" until this line is present and complete.
   - **HDE atlas capture audit** — which screens did this return encounter that were in 📋 TODO state in the atlas? For each: was the field-number overlay captured and transcribed to the atlas? If no, that's a Rule 27 violation and goes in the retro as a discipline finding.
   - Every error encountered with root cause and time lost
   - Every navigation issue that slowed entry
   - New pitfalls or tricks discovered
3. **Update the reference file** — if a new EF error, new screen behavior, new field, or faster navigation path was discovered, add it to `references/[type].md`
4. **Create new reference file** — if this was a new return type, create `references/[type].md` from scratch using the template at the bottom of this file
5. **Commit and push to GitHub** — this is how the skill gets version-controlled.

   **USE THIS EXACT BLOCK. Do not rewrite it. Do not "simplify" it. Copy-paste as-is.** It is self-contained: it loads the PAT inline from the persistent store, so it works even if Phase 0 was skipped (e.g., post-compaction resume, mid-session recovery, continuing a return from a prior session). This block is the single source of truth for "push to lineal-skills" — every other push path in this skill should route through here.

   ```bash
   set -e
   cd "$HOME/github-repo"

   # --- Inline PAT load (self-contained — does NOT depend on Phase 0 env state) ---
   # Each Bash tool call runs in a fresh shell. Any $GITHUB_PAT from an earlier call
   # is GONE. Do not assume it's loaded. Always re-read from the persistent store here.
   PERSIST_PAT="$HOME/mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache/.github-pat"
   SKILLS_ROOT_PAT="$HOME/mnt/.claude/skills/.github-pat"
   if [ -r "$PERSIST_PAT" ] && [ -s "$PERSIST_PAT" ]; then
     GITHUB_PAT=$(tr -d '\r\n ' < "$PERSIST_PAT")
   elif [ -r "$SKILLS_ROOT_PAT" ] && [ -s "$SKILLS_ROOT_PAT" ]; then
     GITHUB_PAT=$(tr -d '\r\n ' < "$SKILLS_ROOT_PAT")
     # Mirror into mcpb-cache for next session
     mkdir -p "$(dirname "$PERSIST_PAT")" 2>/dev/null
     printf '%s' "$GITHUB_PAT" > "$PERSIST_PAT" && chmod 600 "$PERSIST_PAT"
   else
     echo "ERROR: no PAT in persistent store. Ask user for PAT, then write it with:"
     echo "  printf '%s' \"\$GITHUB_PAT\" > \"$PERSIST_PAT\" && chmod 600 \"$PERSIST_PAT\""
     exit 1
   fi
   # -------------------------------------------------------------------------------

   # Stage + commit (change files list as appropriate)
   git add drake-tax-return/
   git commit -m "Return: [Client] [Type] — [summary of learnings]"

   # Push using PAT-in-URL directly. This intentionally does NOT rely on
   # credential.helper or the remote URL in .git/config — both of those have
   # burned us when /tmp/.git-session-credentials is stale-owned by nobody:nogroup.
   git push "https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git" main

   # Verify the push actually landed on origin (cheap sanity check)
   git fetch origin main 2>&1
   git log origin/main -1 --oneline
   ```

   The commit message should summarize what was learned (e.g., "Sood 1040: 8867 Heads Down Entry, K-1 QBI MFC fix, Q7a trap").

   **Why the inline PAT load (added 2026-04-11 after Hammoud):** Phase 0's PAT load sets `$GITHUB_PAT` in the shell that ran Phase 0 — which is a DIFFERENT shell than the one running Phase 4, because every `Bash` tool call spawns a fresh shell. If I skip Phase 0 (which happens on post-compaction resume, when the summary tells me to "continue where you left off" and I jump straight to the remaining work), Phase 4's push fails with "could not read Username for 'https://github.com'" even though the PAT is sitting in the persistent cache the whole time. Embedding the load inline makes Phase 4 work regardless of whether Phase 0 ran. This is a repeat offender — it burned Patel AND Hammoud. The block above MUST be used verbatim; do not optimize it away on the grounds that "Phase 0 already did this."

   **Diagnostic if the inline load itself fails:**
   ```bash
   echo "HOME=$HOME"
   ls -la "$HOME/mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache/.github-pat" 2>&1
   ls -la "$HOME/mnt/.claude/skills/.github-pat" 2>&1
   ```
   If either file exists, is non-empty, and is readable by the current uid, **the PAT is already there** — the inline load must have a bug. Debug the bug. Do NOT prompt the user. The Patel 2026-04-11 failure happened because Phase 0 was looking at the wrong session's directory (`ls /sessions | head -1` instead of `$HOME`) and concluded the PAT was missing when it wasn't.

   **If git push ACTUALLY fails with auth error** (i.e. the persistent PAT was loaded and GitHub rejected it — token expired or revoked): ask the user for a fresh fine-grained PAT for `vatsal2471/lineal-skills` (Contents: Read and write), then write it to the persistent store:
   ```bash
   GITHUB_PAT='<paste from user>'
   # Use $HOME — NEVER `ls /sessions | head -1`. On a shared sandbox, /sessions
   # contains 10+ other users' session dirs and ls|head-1 picks the wrong one.
   PERSIST_PAT="$HOME/mnt/.remote-plugins/plugin_01GC5sHmfRpUwySPemYHW7n5/.mcpb-cache/.github-pat"
   mkdir -p "$(dirname "$PERSIST_PAT")" 2>/dev/null
   printf '%s' "$GITHUB_PAT" > "$PERSIST_PAT" && chmod 600 "$PERSIST_PAT"
   # Only write /tmp/.git-session-credentials if we actually own it or can create it
   if [ -w /tmp/.git-session-credentials ] || ( : > /tmp/.git-session-credentials 2>/dev/null ); then
     printf 'https://x-access-token:%s@github.com\n' "$GITHUB_PAT" > /tmp/.git-session-credentials
     chmod 600 /tmp/.git-session-credentials
     git config credential.helper 'store --file=/tmp/.git-session-credentials'
   fi
   # Push using PAT-in-URL directly (leaves origin clean in config)
   git push "https://x-access-token:${GITHUB_PAT}@github.com/vatsal2471/lineal-skills.git" main
   ```
   The mcpb-cache write means next session picks it up automatically — Phase 0 reads from `$HOME/mnt/.remote-plugins/.../.mcpb-cache/.github-pat` on startup. Do NOT suggest Claude Code env vars or `mnt/.claude/session-env/` — both are broken paths that burned the user multiple times.

6. **Package and present** the updated `.skill` file so the user can reinstall with new learnings:
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