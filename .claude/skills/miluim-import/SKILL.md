---
name: miluim-import
description: Fetches IDF reserve duty (מילואים) records from the Miluim portal (miluim.idf.il) to extract Form 3010 reserve duty pay and tax withheld for a given tax year. Run during the collect-info flow to auto-populate the reserve duty income section.
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_run_code mcp__playwright__browser_wait_for mcp__playwright__browser_type mcp__playwright__browser_close Bash(ls *) Bash(mkdir *) Read Write
---

You are an automation assistant that fetches IDF reserve duty records from the Miluim portal and extracts Form 3010 pay data for Israeli tax filing.

Your goal: log in to `miluim.idf.il`, retrieve the user's reserve duty income and tax withheld for the relevant tax year via Form 3010, optionally download a reserve service certificate, and update the tax refund data file.

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

Read the `./data/` directory contents. Look for a directory named with a 9-digit ID (excluding `draft`). If found, read `<id>/info.md` and check whether `reserve_duty` income is already populated (not `PENDING` and not empty).

- **If already populated:** Show the existing data to the user and ask: "Reserve duty data already exists — would you like to keep it or refresh it from the Miluim portal?"
  - If keep → skip to STEP 6 and output the existing data.
  - If refresh → continue to STEP 2.
- **If not populated or file not found:** Continue to STEP 2.

Also extract `TAX_YEAR` from the data file (or ask if no file exists). Store it as `TAX_YEAR`.

---

## STEP 2 — LOG IN TO MILUIM PORTAL

Navigate to:
```
https://www.miluim.idf.il/auth
```

The login page has two options — always use **"כניסה עם קוד חד פעמי"** (SMS one-time code):

1. Enter the user's 9-digit Israeli ID in the "מספר ת"ז" field.
2. Click **"כניסה עם קוד חד פעמי"**.
3. On the next screen, set the **phone prefix** using the dropdown (e.g. `052`) — this is a separate combobox from the number field.
4. Enter **only the 7-digit suffix** in the "טלפון נייד" field (e.g. for 052-3000346, enter `3000346`). Do NOT enter the full 10-digit number — the field will show "מספר התווים גדול מידי" (too many characters) if you do.
5. Click **"קבלת קוד חד פעמי"** to send the SMS.
6. Ask the user for the SMS code they received on their phone.
7. Enter the code in the "סיסמה" (password) field and click **"כניסה לאתר"**.

After login, the personal area is at `https://www.miluim.idf.il/personalzone`.

---

## STEP 3 — VERIFY LOGIN SUCCESS

After the user confirms login:

1. Take a screenshot to see the current state.
2. Use the snapshot tool to get the page structure.
3. Check whether the page shows a personal dashboard (not a login screen). Look for indicators like:
   - The user's name appearing on the page
   - A navigation menu with sections like "האישורים שלי", "בירורי שכר", "טופס 3010"
   - A welcome message

If still on a login/authentication page, tell the user and wait for confirmation again.

---

## STEP 4 — GENERATE FORM 3010 (Reserve Duty Pay)

Navigate to:
```
https://www.miluim.idf.il/personalzone/milforms/form-3010
```

The form has two date fields in `DD.MM.YY` format:
- "מתאריך" (from date): `01.01.<YY>` (e.g. `01.01.22` for 2022)
- "עד לשנה" (to date): `31.12.<YY>` (e.g. `31.12.22` for 2022)

Fill both fields for `TAX_YEAR` and wait for results.

**If the page shows "מצטערים, לא נמצאו תוצאות":**
The user had no reserve duty in that year. Record:
```yaml
reserve_duty:
  income: 0
  tax_withheld: 0
  year: TAX_YEAR
```

**If results exist:**
Click **"הפקת טופס"** to generate the PDF. Extract:
- Total reserve duty income (הכנסה ממילואים)
- Tax withheld (מס שנוכה במקור)

---

## STEP 5 — RESERVE SERVICE CERTIFICATE (Automatic)

Automatically download the formal reserve service certificate — do not ask the user.

Navigate to:
```
https://www.miluim.idf.il/miluim-forms/האישורים-שלי/
```

Click **"טופס אישור שירות מילואים מזכה"**, generate the PDF, and save it to `./data/<id>/miluim_service_certificate.pdf`.

Tell the user: "(אישור שירות מילואים מזכה saved to ./data/<id>/miluim_service_certificate.pdf)"

If the certificate page is not found or the download fails, skip silently and continue to STEP 6 without blocking.

---

## STEP 6 — UPDATE TAX DATA FILE

Find the user's data file at `./data/<id>/info.md`. Update or add the `reserve_duty` section under `INCOME:`.

**If `INCOME: PENDING` exists as a single line**, replace it with a proper section block preserving any other income already collected.

The `reserve_duty` block should look like this:

```yaml
reserve_duty:
  income: NNNN     # total reserve duty pay in ILS
  tax_withheld: NN # tax withheld at source in ILS
  year: YYYY
```

If a certificate PDF was saved, add:
```yaml
  document: ./data/<id>/miluim_service_certificate.pdf
```

Write the updated file immediately after confirming the values with the user.

---

## STEP 7 — OUTPUT RESULT BLOCK

After saving, output a structured block for use by `collect-info` or other skills:

```
=== MILUIM_IMPORT START ===
tax_year: YYYY
reserve_duty_income: NNNN
reserve_duty_tax_withheld: NN
=== MILUIM_IMPORT END ===
```

If a certificate was saved, include:
```
document: ./data/<id>/miluim_service_certificate.pdf
```

Tell the user:
> "✓ Reserve duty data saved. Income: [N] ILS, Tax withheld: [N] ILS for [year]."

---

## ERROR HANDLING

| Situation | Action |
|---|---|
| Portal unavailable or redirected elsewhere | Report the URL you landed on and ask user to try again later |
| Authentication fails / OTP not received | Ask user to verify their phone number and try again |
| Form 3010 page not found | Try navigating via the side menu: click "הצג תפריט צד" → "טופס 3010" |
| PDF generation fails | Take a screenshot of the results table and extract values visually |
| Portal requires re-login mid-session | Ask the user to re-authenticate in the browser and confirm when done |

---

## IMPORTANT NOTES

- **Never store or log the user's portal credentials.** The user authenticates directly in the Playwright browser window — this skill only automates post-login navigation.
- `miluim.idf.il` only contains **reserve duty** service data. Mandatory service (שירות חובה) discharge dates are NOT available here — use the `idf-service` skill for those.
- Reserve duty income from Form 3010 is taxable income that must be reported on the tax return.
- Reserve service tax credits (if any) are separate from the mandatory service credits — consult a tax professional for eligibility.
