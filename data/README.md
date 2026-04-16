# data/

This directory stores filer data files generated during the tax refund flow.

```
data/
├── .gitkeep              # Keeps this folder tracked in git
├── README.md             # This file — canonical schema reference (read this before any filer file)
└── <id>/                 # One directory per filer, named after the 9-digit Israeli ID number
    ├── info.md           # Collected tax data — all years for this filer
    ├── session.md        # Login session state (written by the login skill)
    ├── submission.md     # Form 135 submission state (written by the form-fill skill)
    └── *.pdf             # Supporting documents (Form 106, Form 867, IDF certificates, etc.)
```

**This directory is gitignored** — its contents are private and should never be committed.

---

## Schema — `<id>/info.md`

Each filer file holds data for **all tax years** for that person. `PERSONAL` is shared across years; all year-specific data lives under `YEARS: <year>:`.

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

YEARS:
  <year>:
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

    BANK:
      bank_number: <Israeli bank code>
      bank_name: <name>
      branch_number: <NNN>
      account_number: <account>
      account_holder: <name>

    SUBMISSION:                            # written by form-fill after filing; omit until then
      status: <submitted|failed>
      confirmation_number: <ref>
      timestamp: <ISO8601>

  <other_year>:
    ...
```
````

### Rules for skills

1. **Read `./data/README.md` first** to get the current schema before reading or writing any filer file.
2. `PERSONAL` is shared across all years — `id`, `name`, `dob`, `phone`, `email`, `marital_status`.
3. All income, credits, deductions, bank, and submission data is scoped per year under `YEARS.<year>`.
4. `SUBMISSION` is only written by `form-fill` after successful filing. `status: submitted` means that year is done.
5. When saving, read the existing `info.md` first to preserve other years, then merge and rewrite the complete file.
6. Supporting documents live in `./<id>/`. Reference them by filename only in `document:` fields.
