# Drake Tax Return JSON Schemas

This document defines the structured JSON format used to pass data from pre-processing into Drake data entry. Each return type has its own schema, and state-specific sections are added as needed.

## Common Fields (All Return Types)

```json
{
  "return_type": "1065 | 1120S | 1120 | 1040",
  "tax_year": 2025,
  "entity": {
    "name": "string — entity/taxpayer name exactly as it appears on the return",
    "ein": "string — XX-XXXXXXX format (SSN for 1040)",
    "address": {
      "street": "string",
      "city": "string",
      "state": "string — 2-letter code",
      "zip": "string — 5-digit"
    },
    "business_activity": "string — e.g. REAL ESTATE / RENTAL",
    "naics_code": "string — 6-digit code, e.g. 531110",
    "date_started": "string — MM/DD/YYYY",
    "accounting_method": "Cash | Accrual | Other",
    "resident_state": "string — 2-letter state code for state return filing"
  }
}
```

## Form 1065 — Partnership

```json
{
  "return_type": "1065",
  "rental_properties": [
    {
      "type": "integer — 1=Single Family, 2=Multi-family, 3=Vacation/Short-term, 4=Commercial, 5=Land, 6=Royalties, 7=Self-Rental, 8=Other",
      "address": {
        "street": "string",
        "city": "string",
        "state": "string",
        "zip": "string"
      },
      "gross_rents": "number",
      "expenses": {
        "advertising": "number | null",
        "auto_travel": "number | null",
        "cleaning_maintenance": "number | null",
        "commissions": "number | null",
        "insurance": "number | null",
        "legal_professional": "number | null",
        "management_fees": "number | null",
        "interest_mortgage": "number | null",
        "interest_other": "number | null",
        "repairs": "number | null",
        "supplies": "number | null",
        "taxes": "number | null",
        "utilities": "number | null",
        "wages": "number | null",
        "depreciation": "number | null — usually auto-calculated from Form 4562",
        "other_expenses": [
          {
            "description": "string — expense category name",
            "amount": "number"
          }
        ]
      },
      "qbi_trade_or_business": "Y | N"
    }
  ],

  "depreciation_assets": [
    {
      "description": "string — asset name (e.g. Building, Unit Remodel)",
      "date_in_service": "string — MM/DD/YYYY",
      "cost_basis": "number",
      "method": "string — ARR (MACRS Residential 27.5yr SL MM), ADB (MACRS Alt Depr), etc.",
      "life": "number — useful life in years (27.5 for residential rental, 39 for commercial, 5/7 for equipment)",
      "convention": "MM | HY | MQ — Mid-Month, Half-Year, Mid-Quarter",
      "prior_depreciation": "number — accumulated depreciation through prior year",
      "for_form": "string — MUST be '8825' for rental assets. This is the most critical field.",
      "amt_method": "string | null — AMT depreciation method (e.g. SL)",
      "amt_life": "number | null — AMT useful life (39 for residential buildings)",
      "amt_prior_depreciation": "number | null"
    }
  ],

  "partners": [
    {
      "name": "string — full name as on SSN card",
      "ssn": "string — XXX-XX-XXXX format",
      "address": {
        "street": "string",
        "city": "string",
        "state": "string",
        "zip": "string"
      },
      "ownership_pct": "number — 0-100",
      "profit_pct": "number — 0-100",
      "loss_pct": "number — 0-100",
      "capital_pct": "number — 0-100",
      "beginning_capital": "number — from prior year ending capital (can be negative)",
      "partner_type": "general | limited | llc_member_manager | llc_member",
      "is_50_pct_or_more": "boolean — triggers Schedule B-1 requirement",
      "country": "string — 2-letter code, typically US"
    }
  ],

  "schedule_b": {
    "all_no": "boolean — if true, answer all questions No",
    "specific_answers": "object | null — override specific line numbers if needed"
  },

  "sec": {
    "prior_year_income": "number — from prior year Comparison Schedule (often negative for rental)"
  },

  "schedule_m1": {
    "basis": "tax | book — Basis for Reporting Capital Accounts"
  },

  "k_credits": {
    "k2_k3_not_required": "boolean — check item 16a if true (domestic, no foreign activity)"
  },

  "pin": {
    "ero_pin": "string — always '75757' for Lineal CPA",
    "member_pin": "string — any random 5-digit number",
    "ero_entered": true,
    "signing_officer_name": "string",
    "signing_officer_title": "string — PARTNER, MEMBER, PRESIDENT, etc.",
    "signing_officer_ssn": "string — XXX-XX-XXXX, required for IL",
    "phone": "string"
  },

  "state_il": {
    "books_city": "string",
    "books_state": "string",
    "books_zip": "string",
    "principal_place_city": "string",
    "principal_place_state": "string",
    "principal_place_zip": "string",
    "il_authorization": true,
    "create_ilk1": true
  }
}
```

## Form 1120S — S-Corporation

*Schema to be defined after first 1120S return. Will share common fields (entity, depreciation, pin, sec, state) with 1065 but replace partners with shareholders and add officer compensation fields.*

## Form 1120 — C-Corporation

*Schema to be defined after first 1120 return.*

## Form 1040 — Individual

*Schema to be defined after first 1040 return.*

## Notes on Data Sources

### Prior Year PDF
The prior year return PDF is the primary source for:
- Entity information (name, EIN, address) — page 1
- Partner/shareholder details and capital accounts — Schedule K-1s
- Depreciation schedules — Form 4562
- Prior year income for SEC screen — Comparison Schedule (page 9 of Drake output)
- Schedule B answers, M-1 basis choice, state-specific info

### Current Year Excel/CSV
The current year financial data provides:
- Gross income/rents
- All expense categories
- Any new assets placed in service
- Updated partner information (if changed)

### Manual Overrides
The JSON supports manual overrides for anything the automated extraction can't handle. When building JSON manually, follow the field types exactly — Drake is sensitive to formatting (dates must be MM/DD/YYYY, EIN must include the dash, etc.).
