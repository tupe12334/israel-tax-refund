# data/

This directory stores filer data files generated during the tax refund flow.

```text
data/
├── .gitkeep              # Keeps this folder tracked in git
├── README.md             # This file — canonical schema reference (read this before any filer file)
└── <id>/                 # One directory per filer, named after the 9-digit Israeli ID number
    ├── info.md           # Personal data shared across all years
    ├── bank.md           # Bank account for refund deposit — shared across all years
    ├── <year>.md         # Year-specific tax data (e.g. 2024.md, 2023.md)
    ├── session.md        # Login session state (written by the login skill)
    ├── submission.md     # Form 135 submission state (written by the form-fill skill)
    └── *.pdf             # Supporting documents (Form 106, Form 867, IDF certificates, etc.)
```

**This directory is gitignored** — its contents are private and should never be committed.

---

## Schema — `<id>/info.md`

Holds personal data shared across all years.

````markdown
# Tax Refund Data — <name>

```tax-data
PERSONAL:
  id: <9-digit Israeli ID>
  name: <full name in Hebrew, as on ID card>
  dob: <DD/MM/YYYY>
  phone: <05X-XXXXXXX>
  email: <email address>
  marital_status: <single|married|divorced|widowed>
```
````

---

## Schema — `<id>/bank.md`

Holds the refund deposit account. Shared across all years — the Tax Authority deposits every year's refund into the same account.

````markdown
# Bank Account — <name>

```tax-data
BANK:
  bank_number: <Israeli bank code>
  bank_name: <name>
  branch_number: <NNN>
  account_number: <account>
  account_holder: <name>
```
````

---

## Schema — `<id>/<year>.md`

One file per tax year. Filename is the 4-digit year (e.g. `2024.md`).

````markdown
# Tax Data <year> — <name>

```tax-data
EMPLOYERS:
  - employer_index: <N>
    employer_name: <name>
    employer_id: <9-digit>
    field_158_taxable_income: <NIS>
    field_042_tax_withheld: <NIS>
    field_045_employee_pension: <NIS>
    field_218_study_fund: <NIS>        # optional
    months_worked: <1–12>

NII_BENEFITS:
  unemployment:
    income: <NIS>
    tax_withheld: <NIS>
  maternity:
    income: <NIS>
    tax_withheld: <NIS>
  reserve_duty:
    income: <NIS>
    tax_withheld: <NIS>
    note: <free text>
  work_injury:
    income: <NIS>
    tax_withheld: <NIS>

INVESTMENT_INCOME:                     # or the string: "none found"
  - institution: <name>
    income: <NIS>
    tax_withheld: <NIS>

TAX_CREDITS:
  military:
    service_start: <MM/YYYY>
    discharge_date: <MM/YYYY>
    service_months: <N>
    service_type: <mandatory|reserve|national>
    role: <description>
    extra_credit_points: <N>
    document: <filename inside ./<id>/ subdirectory>
  children:                            # or the string: "none"
    - birth_year: <YYYY>
      benefit_recipient: <primary|spouse>
  oleh: <MM/YYYY aliyah date>          # or the string: "none"
  academic:                            # or the string: "none"
    degree_type: <BA|BSc|teaching|other>
    graduation_year: <YYYY>
  development_area: <settlement name>  # or the string: "none"
  disability:                          # or the string: "none"
    who: <self|child|dependent>
    percentage: <N>
  single_parent: <true|false>

DEDUCTIONS:
  donations:                           # omit section if none
    - institution: <name>
      amount: <NIS>
  pension_direct:                      # omit section if none
    - institution: <name>
      annual_amount: <NIS>

SUBMISSION:                            # written by form-fill after filing; omit until then
  status: <submitted|failed>
  confirmation_number: <ref>
  timestamp: <ISO8601>
```
````

### Rules for skills

1. **Read `./data/README.md` first** to get the current schema before reading or writing any filer file.
2. `PERSONAL` data lives in `info.md` and is shared across all years.
3. `BANK` data lives in `bank.md` and is shared across all years. The same refund account is used for every year.
4. Year-specific data (income, credits, deductions, submission) lives in `<year>.md`.
5. `SUBMISSION` is only written by `form-fill` after successful filing. `status: submitted` means that year is done.
6. When saving a year file, read the existing `<year>.md` first (if it exists) to avoid overwriting data, then rewrite the complete file.
7. Supporting documents live in `./<id>/`. Reference them by filename only in `document:` fields.
