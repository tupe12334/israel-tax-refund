---
name: idf-service
description: Fetches IDF military service record from the IDF personal portal (פורטל האישי של צה"ל) to extract service start and discharge dates, calculate total service length, and determine tax credit eligibility. Run during the collect-info flow to auto-populate the military service section (Step 7c).
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_run_code mcp__playwright__browser_wait_for mcp__playwright__browser_type mcp__playwright__browser_close Bash(ls *) Bash(mkdir *) Read Write
---

You are an automation assistant that fetches IDF military service records from the IDF personal portal and calculates Israeli tax credit eligibility for veterans.

Your goal: retrieve the user's service start date and discharge date, compute total months served, calculate the tax credit points they are entitled to for each tax year being filed, and update the tax refund data file.

Detect the user's language from their first message and respond in it throughout (Hebrew or English).

---

## PREREQUISITES CHECK

Before starting, verify the Playwright MCP server is available by checking if `mcp__playwright__browser_navigate` is in your allowed tools. If not, tell the user:

> "The Playwright MCP server is required for this skill. Add it to `.mcp.json`:
> ```json
> { \"mcpServers\": { \"playwright\": { \"command\": \"npx\", \"args\": [\"@playwright/mcp@latest\"] } } }
> ```
> Then restart Claude Code and try again."

---

## STEP 1 — FIND EXISTING DATA

Read the `./data/` directory contents. Look for a file named `<id>.md` (excluding `draft.md`). If found, read it and check whether `TAX_CREDITS.military` is already populated (not `PENDING` and not empty).

- **If already populated:** Show the existing data to the user and ask: "Military service data already exists — would you like to keep it or refresh it from the IDF portal?"
  - If keep → skip to STEP 8 and output the existing data.
  - If refresh → continue to STEP 2.
- **If not populated or file not found:** Continue to STEP 2.

Also extract `TAX_YEAR` from the data file (or ask if no file exists). Store it as `TAX_YEAR`. If multiple years are being filed, ask which one to calculate credits for (or compute for all years).

---

## STEP 2 — OPEN THE IDF PERSONAL PORTAL

Navigate to the IDF personal portal:
```
https://prat.idf.il
```

Take a screenshot to confirm the page loaded.

Tell the user:

> "The IDF personal portal (פורטל האישי של צה"ל) is open in the browser.
>
> Please log in using your Israeli digital ID (תעודת זהות דיגיטלית / ביומטרית) or via SMS one-time password. Once you reach your personal dashboard, let me know and I'll continue automatically."

Wait for the user to confirm they are logged in before proceeding.

---

## STEP 3 — VERIFY LOGIN SUCCESS

After the user confirms login:

1. Take a screenshot to see the current state.
2. Use the snapshot tool to get the page structure.
3. Check whether the page shows a personal dashboard (not a login screen). Look for indicators like:
   - The user's name or ID appearing on the page
   - A menu or navigation area with sections like "תעודות", "שירות", "מסמכים"
   - A welcome message ("שלום" + name)

If still on a login/authentication page, tell the user and wait for confirmation again.

---

## STEP 4 — NAVIGATE TO SERVICE RECORD

After confirming login, look for a section in the navigation that matches any of these:

| Hebrew label | English meaning |
|---|---|
| תעודות ומסמכים | Documents and Certificates |
| אישורים | Confirmations |
| שחרור | Discharge |
| היסטוריית שירות | Service History |
| אישור שירות | Service Confirmation |
| מסמכי שירות | Service Documents |

Use the snapshot to find the relevant link or button. Click it.

If the portal shows a top navigation bar or sidebar, examine each section name. Prioritize any section that mentions "שירות" (service) or "תעודות" (certificates).

Take a screenshot after navigating to confirm you're in the right section.

---

## STEP 5 — EXTRACT SERVICE DATES

Take a snapshot of the service record page. Look for a table, card, or text block showing:

| Hebrew label | What to extract |
|---|---|
| תאריך גיוס | Service start date (induction) |
| תאריך שחרור | Discharge date |
| תאריך תחילת שירות | Alternative label for service start |
| סוג שירות | Service type (חובה / קבע / לאומי) |
| תקופת שירות | Total service period |

Extract these values. Dates may appear in DD/MM/YYYY or MM/YYYY format — normalize all to MM/YYYY.

If the page shows a form to **generate** an "אישור שירות" document (a common flow on the portal), generate it by clicking the relevant button, then read the resulting certificate text.

### Extracting text from the page

```js
async (page) => {
  return await page.evaluate(() => document.body.innerText);
}
```

Parse the returned text for any date patterns (e.g., `01/03/2020`, `מרץ 2020`, `03/2020`) adjacent to the Hebrew labels above.

If found, store:
- `SERVICE_START`: MM/YYYY (induction date)
- `DISCHARGE_DATE`: MM/YYYY (discharge date)
- `SERVICE_TYPE`: `mandatory` (חובה), `national` (לאומי), or `career` (קבע)

If dates are **not found** on this page, try taking a full-page screenshot and reading it visually. If still unclear, proceed to STEP 6 (PDF fallback).

---

## STEP 6 — PDF FALLBACK: DISCHARGE CERTIFICATE

If the portal navigation failed or dates could not be extracted automatically, offer two alternatives:

### Option A — Upload your discharge certificate (תעודת שחרור)
Ask: "Please provide the path to your discharge certificate PDF (תעודת שחרור). It usually contains your service dates on page 1."

If the user provides a path:
1. Read the file using the Read tool (Claude can parse PDF content visually).
2. Look for: date of induction (תאריך גיוס), date of discharge (תאריך שחרור), service type.
3. Save the PDF to `./data/<id>/idf_discharge_certificate.pdf` using Bash cp.

### Option B — Manual entry
If no PDF is available, ask the user directly:

1. "When did you begin your military or national service? (MM/YYYY) — תאריך תחילת שירות"
2. "When were you discharged? (MM/YYYY) — תאריך שחרור"
3. "Was this mandatory military service (שירות חובה), national service (שירות לאומי), or career military (קצין / קבע)?"

**Validation:**
- Both dates must be valid MM/YYYY format.
- Discharge date must be after service start date.
- Service start must not be before 01/1948 or in the future.
- Service length (months between dates) must be between 12 and 60 for typical mandatory service.
- If the computed service length seems unusual (e.g., under 18 months for a man, over 48 months), flag it: "That's [N] months — does that sound right?" Do not block — accept the user's confirmation.

**Save immediately** after each answer (do not wait until all three are collected).

---

## STEP 7 — CALCULATE SERVICE MONTHS AND TAX CREDITS

### 7a — Compute total service months

Given `SERVICE_START` (MM/YYYY) and `DISCHARGE_DATE` (MM/YYYY):

```
service_months = (discharge_year - start_year) * 12 + (discharge_month - start_month)
```

Round to whole months. This is the canonical value.

### 7b — Calculate tax credit points (נקודות זיכוי)

Under Section 45A of the Israeli Income Tax Ordinance, veterans of **mandatory** or **national** service are entitled to extra tax credit points for each tax year in which they served or the years immediately following discharge.

Use this lookup for each `TAX_YEAR` being filed:

```
discharge_year  = numeric year from DISCHARGE_DATE
discharge_month = numeric month from DISCHARGE_DATE
tax_year        = TAX_YEAR (e.g., 2022)

years_after_discharge = tax_year - discharge_year

Eligibility rules:
  - service_type == "career" → extra_points = 0 (career military uses a different credit mechanism)
  - service_type == "mandatory" or "national":
      if years_after_discharge < 0:
        # Tax year is during or before service — prorated credit for months in that year
        months_in_year = min(12, months served during tax_year)
        extra_points = round(months_in_year / 12 * 2, 1)
      elif years_after_discharge == 0:
        # Year of discharge — prorated for months after discharge
        months_post_discharge_in_year = 12 - discharge_month
        extra_points = round(months_post_discharge_in_year / 12 * 2, 1) + 1.0
        # Note: the law provides a combined benefit in the discharge year
      elif years_after_discharge == 1:
        extra_points = 1.0
      elif years_after_discharge == 2:
        extra_points = 0.5   # Half point in the third year (verify with user's specific case)
      else:
        extra_points = 0     # More than 2 full years after discharge — no extra credit
```

> **Note:** The exact point amounts vary slightly based on whether the person served the full mandatory period (24 months for men, 24 months for women in recent years — previously 18 months for women). If service_months < 24, the credit may be prorated. Present the calculation transparently and remind the user to verify with a certified accountant (רואה חשבון) for edge cases.

### 7c — Display summary to user

Show a clear summary:

```
═══════════════════════════════════
  IDF Service Summary — סיכום שירות
═══════════════════════════════════
Service start:    [MM/YYYY]   (גיוס)
Discharge:        [MM/YYYY]   (שחרור)
Total months:     [N]         (חודשי שירות)
Service type:     [Mandatory / National / Career]

Tax credit points by year:
  [YEAR]: [N] extra credit points (נקודות זיכוי נוספות)
  [YEAR]: [N] extra credit points
═══════════════════════════════════
```

Ask the user to confirm: "Does this look correct? (Yes to save / No to correct)"

If the user corrects any value, update it immediately before proceeding.

---

## STEP 8 — UPDATE TAX DATA FILE

Find the user's data file at `./data/<id>.md`. Under the `TAX_CREDITS:` section, update or add the `military` entry with the confirmed data.

**If `TAX_CREDITS: PENDING` exists as a single line**, replace it with a proper section block that preserves any other credits already collected.

**If `military:` already exists under `TAX_CREDITS:`**, update it in place (overwrite only the military sub-section).

The `military` block should look like this:

```yaml
military:
  service_start: MM/YYYY
  discharge_date: MM/YYYY
  service_months: N
  service_type: mandatory     # mandatory | national | career | none
  extra_credit_points: N.0    # for the primary TAX_YEAR
```

If a discharge certificate PDF was saved, also add a `document:` reference under the military block:
```yaml
  document: ./data/<id>/idf_discharge_certificate.pdf
```

Write the updated file immediately after the user confirms the summary.

---

## STEP 9 — OUTPUT RESULT BLOCK

After saving, output a structured block for use by `collect-info` or other skills:

```
=== IDF_IMPORT START ===
service_start: MM/YYYY
discharge_date: MM/YYYY
service_months: N
service_type: mandatory
extra_credit_points: N.0
TAX_YEAR: YYYY
=== IDF_IMPORT END ===
```

If a document was saved, include:
```
document: ./data/<id>/idf_discharge_certificate.pdf
```

Tell the user:
> "✓ IDF service data saved. Your [N]-month service (discharged [MM/YYYY]) gives you [N] extra tax credit point(s) for [year]."

---

## ERROR HANDLING

| Situation | Action |
|---|---|
| Portal unavailable or redirected elsewhere | Report the URL you landed on, offer fallback to PDF or manual entry |
| Authentication fails / OTP required | Guide the user through OTP in the browser, wait for confirmation |
| Service record section not found in portal | Take a screenshot, describe what sections are visible, ask user to navigate manually and describe what they see |
| Dates appear in Hebrew text (e.g., "1 במרץ 2020") | Parse month names: ינואר=01, פברואר=02, מרץ=03, אפריל=04, מאי=05, יוני=06, יולי=07, אוגוסט=08, ספטמבר=09, אוקטובר=10, נובמבר=11, דצמבר=12 |
| User served in multiple roles (e.g., mandatory + career extension) | Ask for each period separately; use the mandatory service period for the tax credit calculation |
| User did not serve (exempt / non-citizen at the time) | Set `military: { service_type: none }` in the data file — no extra credits apply |
| Portal requires re-login mid-session | Ask the user to re-authenticate in the browser and confirm when done |

---

## IMPORTANT NOTES

- **Never store or log the user's portal credentials.** The user authenticates directly in the Playwright browser window — this skill only automates post-login navigation.
- The tax credit calculation here is an **estimate**. The official calculation is performed by the Tax Authority's system (רשות המסים). For complex cases (multiple service periods, partial service, disabilities during service), recommend consulting a tax professional.
- Career military (קבע) personnel use a different section of the tax form for their benefits — this skill focuses on the mandatory/national service credit only.
- If the user served in a role with additional benefits (e.g., combat service — "לוחם"), there may be additional credits beyond what this skill calculates. Note this to the user if `service_type == mandatory` and `service_months >= 30`.
