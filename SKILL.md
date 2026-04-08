---
name: netsuite-bank-match
description: >
  Step-by-step automation for matching imported bank deposits in NetSuite's Match Bank Data page
  (account 1002 Bank, Delco Holdings). Handles eBay and Walmart monthly clearing entries:
  creates a Journal Entry (debit 1002 Bank / credit clearing account), then matches it to the
  bank feed. Use this skill whenever the user asks to match bank deposits, process eBay deposits,
  process Walmart deposits, reconcile bank data, or do any NetSuite bank matching for Portable
  Blowout / Delco Holdings. Contains all JS automation scripts and known pitfalls — always load
  this before starting any bank matching work.
---

# NetSuite Bank Match — eBay & Walmart (Delco Holdings)

## Account Reference

| Field | Value |
|---|---|
| NetSuite Account | 4633687.app.netsuite.com |
| Match Bank Data URL | `https://4633687.app.netsuite.com/erp/confirmbanktransactions/confirmbanktransactions.nl?acctid=212` |
| Bank Account | 1002 Bank : Delco - City National Bank 7982 |
| acctid | 212 |
| Department | Delco Holdings — ID **1** |
| Location | 01 Warehouse [Delco] — ID **43** |

### Clearing Account IDs

| Vendor | GL Account | NetSuite ID |
|---|---|---|
| eBay | 1118 eBay Clearing | **831** |
| Walmart | 1117 Walmart Clearing | *(confirm on first use)* |

---

## Full Workflow — One Month

### Step 1: Navigate & Set Filters

1. Open the Match Bank Data tab (use tabId from `tabs_context_mcp` or navigate fresh).
2. If the session is expired, credentials are pre-filled — just click **Log In**.
3. If a "Bank data import failed" warning pops up, dismiss it via JS:
   ```javascript
   var btns = document.querySelectorAll('button');
   for (var b of btns) {
     if (b.textContent.trim() === '×' || b.getAttribute('aria-label') === 'Close') {
       b.click(); break;
     }
   }
   ```
4. Use the `find` tool to get refs for the left-side **From** and **To** date inputs:
   - Query: `"From date input Imported Bank Data left side"`
   - Query: `"To date input Imported Bank Data left side"`
5. `triple_click` the From field → type `MM/1/YYYY` → press `Tab`.
6. `triple_click` the To field → type `MM/31/YYYY` (or last day of month) → press `Tab`.
7. Wait 2 seconds for the filter to apply.

### Step 2: Search for Vendor Transactions

1. Use `find` with `"Search text input Imported Bank Data"` to get the ref.
2. `triple_click` it → type `ebay` (or `walmart`) → press `Return`.
3. Wait 2 seconds. Confirm **Total: N** shows on the left side.

### Step 3: Select All Left-Side Transactions

The header checkbox is a custom NetSuite element — physical coordinate clicks are required; JS `.click()` does NOT toggle `aria-checked`.

Get exact coordinates of all checkboxes:
```javascript
var checkboxes = document.querySelectorAll('[role="checkbox"]');
var result = [];
Array.from(checkboxes).forEach(function(c, i) {
    var rect = c.getBoundingClientRect();
    result.push(i + ': x=' + Math.round(rect.x) + ' y=' + Math.round(rect.y) + ' checked=' + c.getAttribute('aria-checked'));
});
result.join('\n');
```

The left-side **header** checkbox is the one at `x≈28` that appears *above* the individual row checkboxes (also at `x≈28`). Click it by coordinate: `left_click` at `(28, <header_y>)`.

Confirm all selected:
```javascript
var checkboxes = document.querySelectorAll('[role="checkbox"]');
var leftSide = Array.from(checkboxes).slice(0, 6).map(c => c.getAttribute('aria-checked'));
var diff = document.body.innerText.match(/Difference\s+([\d,.()-]+)/)?.[1];
'diff=' + diff + ' left=' + JSON.stringify(leftSide);
```

If `aria-checked` values are still `false` after clicking, try the individual row checkboxes at `(28, <row_y>)`. The left-side rows are all at `x≈28`.

**Important — indeterminate state**: If the header is in `"mixed"` state (some rows pre-selected), clicking it may *deselect* everything. Check the resulting diff. If diff drops to 0.00 and header becomes `false`, click the header again to re-select all.

### Step 4: Create the Journal Entry

Click "Add Journal Entry" via JS (do NOT use coordinate clicks):
```javascript
var btns = document.querySelectorAll('button');
for (var j = 0; j < btns.length; j++) {
    if (btns[j].textContent.trim() === 'Add Journal Entry') { btns[j].click(); break; }
}
```

A new tab opens for the JE form. Wait 3–4 seconds for it to fully load. The URL will contain `?subsidiary=1&account=212&amount=AMOUNT&date=DATE`.

### Step 5: Fill and Save the Journal Entry

Run this combined script — it fills both lines, clicks "Add", then saves. Replace `AMOUNT` with the net total (sum of all selected transactions):

```javascript
// Line 1: already populated with account 1002 — just set department & location
nlapiSetCurrentLineItemValue('line', 'department', '1');
nlapiSetCurrentLineItemValue('line', 'location', '43');
nlapiCommitLineItem('line');

// Line 2: credit clearing account
nlapiSelectNewLineItem('line');
nlapiSetCurrentLineItemValue('line', 'account', '831');   // eBay: 831 | Walmart: check ID
nlapiSetCurrentLineItemValue('line', 'department', '1');
nlapiSetCurrentLineItemValue('line', 'credit', 'AMOUNT'); // e.g. '61.56'
nlapiSetCurrentLineItemValue('line', 'location', '43');
nlapiCommitLineItem('line');

// Click Add then Save
setTimeout(function() {
    var btns = document.querySelectorAll('button');
    for (var i = 0; i < btns.length; i++) {
        if (btns[i].textContent.trim() === 'Add') { btns[i].click(); break; }
    }
    setTimeout(function() {
        document.getElementById('btn_multibutton_submitter').click();
    }, 600);
}, 800);
```

Wait 5 seconds for save. The URL will change to `?id=NETSUITE_ID&...`.

Get the JE transaction number:
```javascript
document.body.innerText.match(/1551355\d{4,}/)?.[0]
```
This returns something like `15513550990`.

### Step 6: Match the JE to the Bank Feed

Switch back to the Match Bank Data tab.

1. Use `find` with `"Search text input Account Transactions right side"` to get the ref (usually `ref_475` or similar).
2. `triple_click` it → type the JE number (e.g., `15513550990`).
3. `left_click` on the field again (physical focus) → press `Return`.
4. Wait 2 seconds. Confirm right-side shows **Total: 1**.

> **Why Enter must be physical**: Dispatching `KeyboardEvent` via JS does NOT trigger NetSuite's search. You must use the `key: "Return"` tool action after physically clicking the field.

5. Use `find` with `"unchecked checkbox in Account Transactions journal row"` to get the row checkbox ref.
6. Click it via ref. The diff should drop to **0.00**.

Check and click Match:
```javascript
var diff = document.body.innerText.match(/Difference\s+([\d,.()-]+)/)?.[1];
if (diff === '0.00') {
    var btns = document.querySelectorAll('button');
    for (var j = 0; j < btns.length; j++) {
        if (btns[j].textContent.trim() === 'Match') { btns[j].click(); break; }
    }
}
diff;
```

Wait 3 seconds. The left-side transactions disappear — month is done.

---

## Processing Multiple Months

Process one month at a time. For each month:
1. Update the From/To date filter (use `triple_click` + `type` + `Tab` on the date fields — JS date changes do NOT trigger the filter refresh)
2. Keep the vendor search term in place
3. Repeat Steps 3–6

Check for remaining months by advancing the date range. When the left side shows **Total: 0** with the ebay/walmart filter, that vendor is done.

---

## Handling Months with Mixed Debits and Credits

Some months include both credits and debits (e.g., December 2025 eBay had a -$27.95 debit alongside credits). The approach is the same — select all transactions, sum them algebraically, use the net total as the JE amount. The JE still debits 1002 Bank for the net amount.

Example: -27.95 + 4.60 + 26.57 + 26.87 + 31.47 = **61.56**

---

## Completed History

| Month | Vendor | JE # | Amount |
|---|---|---|---|
| Aug 2025 | eBay | 15513550986 | $116.22 |
| Sep 2025 | eBay | 15513550987 | $570.36 |
| Oct 2025 | eBay | 15513550988 | $534.24 |
| Nov 2025 | eBay | 15513550989 | $419.51 |
| Dec 2025 | eBay | 15513550990 | $61.56 |

---

## Common Issues & Fixes

| Problem | Fix |
|---|---|
| Browser disconnects mid-task | Call `tabs_context_mcp(createIfEmpty: true)`, then navigate back. Use `switch_browser` if the tab group keeps dying. |
| Header checkbox doesn't respond | Get exact y-coordinate via JS `getBoundingClientRect()` — don't guess. Click `(28, exact_y)`. |
| Checkbox appears unchecked even after click | The `aria-checked` attribute takes a moment to update. Re-query after 1 second. |
| Diff doesn't reach 0.00 | Verify you matched the correct JE number. Check that the JE debit amount equals the sum of selected bank transactions. |
| Right-side search doesn't filter | Click the field physically first (`left_click` on ref), THEN press `Return`. JS `KeyboardEvent` dispatch doesn't work. |
| "From" date field gets contaminated with JE number | Clear it: `document.querySelector('input[placeholder="From"]').value = ''; ` then dispatch a `change` event. |
| JE saves with empty lines | The combined script must call `nlapiCommitLineItem` before `nlapiSelectNewLineItem`. Also ensure the "Add" button click runs before Save (the setTimeout chain handles this). |
| Warning toast on page load | Dismiss via JS — find the `×` button and `.click()` it. Do NOT click by coordinate (it's near the nav bar and can navigate away). |
| Session expired (login page) | Credentials are pre-filled. Click the Log In button. URL redirect will return to Match Bank Data after login. |
