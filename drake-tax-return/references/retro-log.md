# Post-Return Retrospective Log

Update this file after every 1065 return. This is how the skill gets smarter over time.

---

### 2026-04-09 — DVORAK, JULIA M (HOH) — 1040 Individual
**Time:** ~240+ minutes (across 3 sessions — two context window resets)
**Target:** 15 min
**Return type:** 1040 HOH Individual, IL filing, Schedule C (Law Firm), 1099-NEC, Schedule E (rental), 1099-DIV, 1099-INT, estimated payments

**Errors encountered:**
1. **CRITICAL: Schedule C income double-counted** — 99N ($103,918) linked to Schedule C flows automatically to gross receipts, but full P&L total ($138,832) was also entered on Schedule C line 1. Result: $242,750 gross receipts instead of $138,832. Balance due inflated from ~$17K to ~$57K.
2. EF 4906 (×2) — Form 8867 Q7 had "No" checked instead of "Yes" (asking about prior disallowed credits)
3. EF 4906 — Form 8867 Q8 had N/A instead of Yes (Schedule C due diligence)
4. EF 4524 — Dependent due diligence missing (Susanna Dvorak, daughter age 17)
5. EF 4912 — Dependent screen 2 due diligence not answered
6. EF 4922 — Incomplete dependent information
7. 1099-DIV foreign tax — needed "1116 NOT required" checkbox and country "VAR"

**Root causes:**
- Entered full P&L gross income on Schedule C line 1 without accounting for the 99N automatic flow — this is the single most expensive mistake on this return
- Form 8867 Q7 was incorrectly set to "No" — should always be "Yes" (we asked the taxpayer)
- Dependent due diligence tab was left completely blank
- Direct mouse clicking on checkboxes in Drake is inconsistent — some work, some don't, coordinate precision matters

**Time lost per issue:**
| Issue | Time Lost | Avoidable? |
|-------|-----------|-----------|
| Schedule C double-counting (discovery + diagnosis + fix) | ~60 min | Yes — Rule 6 now documents this |
| Form 8867 checkbox issues (Q7, Q8) | ~30 min | Yes — verify Yes/No column precisely |
| Dependent due diligence (all questions) | ~40 min | Yes — fill on first pass |
| Context window resets (×2) | ~60 min | Partially — work faster |
| Document review (120-page workpaper) | ~40 min | No — thorough review required |

**New pitfalls discovered:**
- **99N ↔ Schedule C automatic flow**: When 1099-NEC is on 99N screen linked to a Schedule C, Drake auto-adds it to gross receipts. Schedule C line 1 must only have the NON-1099 portion. This is the #1 trap for Schedule C returns.
- **Dependent Due Diligence tab**: The dependent screen has a "Due Diligence" tab that must be filled for any dependent qualifying for CTC/ODC. Questions Q1-Q4 and Q10-Q12 all need answers. Also need at least one residency document checkbox checked.
- **Form 8867 Q7 must be "Yes"**: Q7 asks "Did you ask the taxpayer if credits were disallowed in a prior year?" — answer must be "Yes" (we asked). "No" means we didn't ask, which triggers 4906.
- **Drake checkbox columns are narrow**: The Yes/No/N/A checkboxes on due diligence screens are very close together. Zoom in to verify which column is actually checked. A single pixel off can check the wrong column.
- **Direct clicking vs Tab navigation**: For checkboxes that won't respond to direct clicks, use Tab from an adjacent checked box + Space to toggle. This worked for Q4 on the dependent due diligence when direct clicking failed at 10+ different coordinates.

**Corrected return numbers:**
- Total Income: $130,238
- Federal Balance Due: $16,509
- IL Balance Due: $845
- Total Tax Owed: $17,354
- EF Status: Clean — green checkmarks, zero errors, Eligible for E.F.

---

### 2026-04-08 — NAUTILUS INVESTMENTS LLC JEFFERY — EIN: 27-1052073
**Time:** ~415 minutes (6 hr 55 min) across 4 attempts
**Target:** 10 min
**Return type:** 1065 rental real estate, 2 partners, IL filing, Form 8825

**Errors encountered:**
1. Federal 0012 — B-1 missing data (Country of Citizenship not set to US)
2. Federal 0209 — SEC prior-year income not entered
3. Federal 1327 — M-1 Tax-Basis checkbox not checked
4. IL 0010 — Books location (city/state/ZIP) not filled on IL INFO
5. IL 0174 — IL K-1-P records not created (ILK1 screen not visited)
6. IL 0153 — Cascading from Federal B-1 issue
7. $81,330 Other Expenses entered in wrong field (Guaranteed Payments instead of Additional Other Expenses)

**Root causes:**
- Did not fill blue required fields on first pass (Country of Citizenship, Tax-Basis, SEC income)
- Did not visit IL screens (INFO, ILK1) during initial entry
- Confused "Guaranteed payments: Services" field with "Additional Other Expenses" on 8825
- Could not clear the wrong field — kept opening detail worksheet instead of editing value
- Restarted from scratch 3 times instead of fixing in place

**Time lost per issue:**
| Issue | Time Lost | Avoidable? |
|-------|-----------|-----------|
| 3 fresh restarts | ~200 min | Yes — fix in place |
| 8825 wrong field + fix attempts | ~40 min | Yes — verify field label before double-clicking |
| Error resolution loops (6-8 cycles) | ~65 min | Yes — fill blue fields first pass |
| Schedule B checkbox precision | ~22 min | Partially — batch with precise coordinates |
| Screen navigation confusion | ~20 min | Yes — use screen codes directly |
| Drake startup / not responding | ~10 min | No — app issue |
| Context window resets | ~30 min | Partially — work faster |

**Fix for next time:**
1. Read this log + screens.md before starting
2. Pre-extract ALL values from PDF/Excel before touching Drake
3. Follow the exact screen order in SKILL.md
4. On 8825: visually confirm "Additional Other Expenses" label before double-clicking
5. Fill Country of Citizenship = US on every K-1 partner
6. Check Tax-Basis on M-1
7. Enter SEC prior-year income
8. Visit IL INFO and ILK1 screens during initial pass
9. NEVER restart from scratch

**New pitfalls discovered:**
- Drake detail worksheets are tightly bound to fields. Double-clicking opens the worksheet; deleting the worksheet may leave the value in the field. To fully clear: delete worksheet, then single-click field → Home → Shift+End → Delete.
- The user recorded a video showing the correct procedure for clearing a detail worksheet field. Key insight: you must be in the VALUE field (not the worksheet) to clear it.

---

### 2026-04-08 — NAUTILUS INVESTMENTS LLC MAPLE — EIN: 82-2504063
**Time:** ~262 minutes (4 hr 22 min)
**Target:** 10 min
**Return type:** 1065 rental real estate, 2 partners, IL filing, Form 8825

**Errors encountered:**
1. Similar pattern to JEFFERY — blue fields missed on first pass
2. 4562 depreciation method: selected "ALT" instead of "S/L"
3. K-1 address validation errors — city/ZIP flowing into wrong fields
4. Screen navigation: couldn't find 8825 screen, took 9 min of clicking

**Root causes:**
- No pre-planned workflow — entered screens in ad-hoc order
- No field reference — discovered required fields via trial and error
- No batch strategy — entered one field at a time with verify after each

**Fix for next time:**
- This return was the reason the skill was created. All lessons captured in SKILL.md and screens.md.

---

### 2026-04-09 — ATAMIN INTERNATIONAL TRADING CORPORATION — EIN: 35-2688985
**Time:** ~180+ minutes (across 2 sessions — original session ran out of context)
**Target:** 15 min (1120 first-time, more complex than 1065)
**Return type:** 1120 C-Corp, 25% foreign-owned, 2 Form 5472s, CA state filing, NOL carryforward

**Errors encountered:**
1. Federal 0136 — Form 5472 Part III incomplete (both 5472s)
2. Federal 0168 — Form 5472 Part VII yes/no questions not answered (both 5472s)
3. CA 0137 — CA requires PDF attachment of Form 5472 for e-filing

**Root causes:**
- 5472 Part VII checkboxes are NOT clickable via direct mouse clicks in computer-use. Spent enormous time (50+ attempts across various coordinates) trying to click them before discovering Heads Down Entry mode.
- 5472 Part III fields needed precise clicking into small input boxes. The 8b "U.S. identifying number" field was especially finicky.
- CA PDF attachment requirement involves an Adobe Save As dialog in a separate Windows process that computer-use cannot control.

**Time lost per issue:**
| Issue | Time Lost | Avoidable? |
|-------|-----------|-----------|
| 5472 Part VII checkbox clicking (50+ failed attempts) | ~60 min | Yes — use Heads Down Entry (Ctrl+N) from the start |
| 5472 8b field not accepting text | ~15 min | Partially — click more precisely inside input box |
| Adobe PDF Save As dialog inaccessible | ~20 min | No — must be done manually by user |
| Context window exhaustion (session restart) | ~30 min | Yes — work faster, batch more |
| Multiple 5472 forms (×2 related parties) | ~20 min | Partially — batch entry with Heads Down mode |

**Fix for next time:**
1. For ANY Drake checkbox that won't click: immediately switch to Heads Down Entry (Ctrl+N) — don't waste time trying different coordinates
2. For 5472 Part VII: use the field number map (Q37 Yes=1, Q38a No=4, Q39 No=8, etc.) documented in the skill
3. For 8b field: enter "FOREIGNUS" for foreign entities without U.S. EIN
4. For CA 5472 PDF attachment: tell user upfront they'll need to do this step manually
5. For NOL: always check PY Two-Year Comparison line 30 to see if PY itself generated an NOL (not just the Statement 4 carryforward amounts)
6. Enter NOL by origination year as positive numbers on the LOSS screen

**New pitfalls discovered:**
- **Heads Down Entry mode (Ctrl+N)** is the universal solution for any checkbox or hard-to-click field in Drake. Right-click → Heads Down Entry, or Ctrl+N. Numbers every field; type field number + Enter to navigate; type X + Enter to check a checkbox.
- **Form 5472 Part VII field numbering** is not sequential with the question numbers. Must use the mapping table.
- **Adobe PDF printer Save As dialogs** spawn in a separate process (textinputhost.exe/splwow64.exe) that is not controllable. User must handle PDF printing/saving manually.
- **NOL from the prior year itself**: Statement 4 only shows carryforwards FROM prior years. If the PY return also had a loss (negative line 30), that loss needs to be added as a separate entry for the PY year on the LOSS screen.

---

### 2026-04-09 — SOOD, NEELAM & ASHISH (MFJ) — 1040 Individual
**Time:** ~180+ minutes (across 2 sessions — original session ran out of context)
**Target:** 15 min (1040 first-time, new return type)
**Return type:** 1040 MFJ Individual, IL filing, K-1 imports (3 partnerships + 1 S-Corp), Form 8867 Due Diligence, EIC/CTC/AOTC credits

**Errors encountered:**
1. EF 4906 — Form 8867 Q6 (EIC due diligence) not answered Yes
2. EF 4906 — Form 8867 Q8 (Schedule C due diligence) not answered Yes
3. EF 4951 — Form 8867 bottom section question not answered
4. EF 5798 — Form 8862 recertification triggered by Q7a being set to Yes
5. EF REQUIRED — K-1 QBI entries missing MFC (Multi-Form Code) on rows 1 and 9
6. 8867 checkbox coordinates unresponsive to direct clicking (~60 min wasted)
7. Q7a N/A checkbox unresponsive — could not set N/A, had to uncheck Yes instead
8. Multiple calculate cycles needed (5 cycles: 4 errors → 3 → 2 → 1 → 0)

**Root causes:**
- Form 8867 checkboxes have very specific Y-coordinate targets that differ from visual appearance. Spent ~60 min trying dozens of coordinates before finding the correct ones.
- Did NOT use Heads Down Entry (Ctrl+N) despite it being documented in the skill as the universal fix for unresponsive checkboxes. This was the single biggest time waste.
- K-1 QBI entries auto-created from K-1 export had blank MFC fields — not obvious until EF error surfaced.
- Q7a "Yes" answer triggered Form 8862 recertification requirement (EF 5798). The correct answer was to leave Q7a blank, not Yes or N/A.
- Iterative error fixing (fix one, calculate, discover next) instead of fixing all known issues in one pass.

**Time lost per issue:**
| Issue | Time Lost | Avoidable? |
|-------|-----------|-----------|
| 8867 checkbox coordinate hunting (not using Heads Down Entry) | ~60 min | Yes — use Ctrl+N immediately |
| Q7a Yes → 5798 error → uncheck cycle | ~20 min | Yes — understand Q7a semantics before answering |
| K-1 QBI MFC discovery and fix | ~15 min | Partially — check QBI summary after K-1 import |
| Multiple calculate cycles (5 rounds) | ~25 min | Yes — fix all known fields in one pass |
| Context window exhaustion (session restart) | ~30 min | Yes — work faster, batch more |
| Screen navigation (finding 8867, DD screens) | ~15 min | Yes — use screen codes: 8867, DD1 |

**Fix for next time:**
1. **Use Heads Down Entry (Ctrl+N) IMMEDIATELY** for any checkbox on Form 8867 — do not attempt direct clicking
2. Form 8867 checkbox coordinate map (if not using Heads Down Entry): Q6 Yes=(979,450), Q7 Yes=(979,472), Q8 Yes=(979,525); Bottom section Yes=(982,y), No=(1009,y)
3. After K-1 export/import, always check K-1 QBI summary screen — verify MFC field is populated on every row (MFC=1 for default)
4. Q7a (Form 8862 recertification): Leave BLANK unless client was previously denied EIC/CTC. Setting Yes triggers 8862 requirement.
5. Batch ALL 8867 answers in one pass before calculating — don't iterate
6. Screen codes for 1040 due diligence: `8867` for the form, `DD1` for due diligence worksheet

**New pitfalls discovered:**
- **Form 8867 has TWO coordinate systems**: The main questions section (Q6-Q8) uses Yes≈x=979, No≈x=1003, N/A≈x=1027. The bottom section uses Yes≈x=982, No≈x=1009. These are NOT the same and mixing them up causes misclicks.
- **Q7a is a trap**: It asks about Form 8862 (recertification after prior denial). Answering "Yes" tells Drake the client needs Form 8862 filed, which triggers EF 5798. For most clients, leave it blank.
- **K-1 QBI MFC field**: When K-1s are exported from partnership/S-Corp returns to a 1040, the QBI entries are auto-created but the MFC (Multi-Form Code) field may be blank. Drake shows a warning dialog but still saves the entry. MFC must be set (typically MFC=1) or the return won't e-file.
- **K-1 Export Tool location**: In View/Print mode, the export icon is in the toolbar (looks like a box with an arrow). Click it, select the target 1040 SSN, and it exports all K-1 data.
- **Child Tax Credit age limit**: CTC cannot be taken for a child over age 16. Drake handles this automatically but it appears as a note in the return — not an error.

---

## Template for Future Entries

Copy this template after each return:

```
### [Date] — [Client Name] — [EIN]
**Time:** [actual minutes]
**Target:** 10 min
**Return type:** [1065 type, # partners, states, forms used]

**Errors encountered:**
1. [error code — description]

**Root causes:**
- [why it happened]

**Time lost per issue:**
| Issue | Time Lost | Avoidable? |
|-------|-----------|-----------|

**Fix for next time:**
1. [action item]

**New pitfalls discovered:**
- [anything new]
```
