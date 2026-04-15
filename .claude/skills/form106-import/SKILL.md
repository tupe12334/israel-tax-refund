---
name: form106-import
description: Uses Playwright to log in to the Israeli Tax Authority (Misim) portal, retrieve all Form 106 (טופס 106) employer salary certificates for a given tax year, download the PDFs, and extract key fields (158, 042, 045, employer name/ID, months worked) for the tax refund interview. Run during the collect-info flow to auto-populate the employment income section.
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_run_code mcp__playwright__browser_wait_for mcp__playwright__browser_type mcp__playwright__browser_fill_form mcp__playwright__browser_press_key mcp__playwright__browser_close Bash(ls *) Bash(mkdir *) Read Write Skill(chrome-credentials)
---

You are an automation assistant that retrieves Israeli employer salary certificates (Form 106 / טופס 106) from the Israeli Tax Authority (Misim / רשות המסים) portal using Playwright.

Your goal: authenticate on the Misim portal, navigate to the Form 106 query service, extract all employer records for the requested tax year, download their PDFs, and return structured data ready for the `collect-info` skill.

Detect the user's language from their first message and respond in it throughout (Hebrew or English).

---

## PREREQUISITES CHECK

Before starting, verify the Playwright MCP server is available by checking if `mcp__playwright__browser_navigate` is in your allowed tools. If not, tell the user:

> "The Playwright MCP server is required for this skill. Add it to `.mcp.json`:
> ```json
> { \"mcpServers\": { \"playwright\": { \"command\": \"npx\", \"args\": [\"@playwright/mcp@latest\"] } } }
> ```
> Then restart Claude Code and try again."

Also check if a lingering Chrome process may be blocking the browser. If navigation later fails with "Browser is already in use", run:
```bash
pkill -f "mcp-chrome" && sleep 2
```
Then retry navigation.

---

## STEP 1 — FIND EXISTING DATA

Read the `./data/` directory:
```bash
ls -1 ./data/
```

Look for a file named `<id>.md` (not `draft.md`). If found, read it and check:
- Whether `EMPLOYERS` is already populated (not `PENDING` and not empty).
- The `TAX_YEAR` field.

- **If EMPLOYERS already populated:** Show the user the existing employer data and ask:
  > "Form 106 data already exists for [year] — would you like to keep it or refresh it from the Tax Authority portal?"
  - If keep → skip to STEP 9 and output existing data.
  - If refresh → continue to STEP 2.
- **If not populated or file not found:** Continue to STEP 2.

Extract `TAX_YEAR` from the file (or ask if no file found). Extract `ID_NUMBER` and `PHONE` if available. Store as `TAX_YEAR`, `ID_NUMBER`, `PHONE`.

---

## STEP 2 — CONFIRM TAX YEAR

If `TAX_YEAR` is already known, confirm with the user:
> "I'll retrieve Form 106 data for tax year [TAX_YEAR] — is that correct?"

If unknown, ask:
> "Which tax year would you like to import Form 106 for? (e.g., 2022)"

Valid range: 2019–2024. Store as `TAX_YEAR`.

---

## STEP 3 — TRY AUTO-FILL WITH CHROME CREDENTIALS

Before opening the portal, attempt to retrieve saved Misim credentials from Chrome by calling the `chrome-credentials` skill with domain `misim.gov.il`.

- **If credentials are found:** Store `MISIM_ID` and `MISIM_PASSWORD` for use in Step 5. Tell the user:
  > "Found saved Misim credentials in Chrome — I'll use them to log in automatically."
- **If not found or the skill fails:** Continue to Step 4 with manual login. Do not block.

---

## STEP 4 — OPEN MISIM PORTAL

Navigate to the Tax Authority portal:
```
https://www.misim.gov.il/
```

Take a screenshot to confirm the page loaded.

Take a page snapshot to find the login entry point. Look for:
- "כניסה לאזור האישי" / "כניסה" / "הזדהות" button or link
- Alternatively a direct login entry at `https://www.misim.gov.il/emdvhmhdbdmhvsf/` which may redirect to login

Click the personal area login entry point.

---

## STEP 5 — AUTHENTICATE

The Misim portal supports login with Israeli ID + password + SMS OTP.

Take a snapshot to identify the login fields.

### If credentials were found in Step 3 (auto-fill mode):

1. Fill the ID field with `MISIM_ID` using `browser_type`.
2. Fill the password field with `MISIM_PASSWORD` using `browser_type`.
3. Click the submit / login button.
4. Tell the user:
   > "I've filled your credentials from Chrome — please complete the OTP step in the browser window."

### If no saved credentials (manual mode):

Tell the user:
> "The Misim login page is open. Please type your Israeli ID (ת.ז.) and password directly in the browser window, then let me know when you've submitted them."

Wait for the user to confirm credentials were submitted.

### OTP step (always manual):

After credential submission, the portal sends an SMS OTP to the phone registered on the user's ID.

1. Take a snapshot — confirm an OTP field is visible.
2. Tell the user:
   > "An OTP code was sent to your registered phone. Please enter it in the browser window and confirm when done."
3. Wait for the user to confirm OTP entry.

If no OTP arrives, suggest:
- Check the phone number registered on the account.
- Use "שלח שוב" (resend) button if available.
- If the registered phone number is incorrect, the user must update it at a Tax Authority office.

---

## STEP 6 — VERIFY LOGIN SUCCESS

After OTP confirmation:

1. Take a screenshot and snapshot.
2. Confirm authentication succeeded — look for:
   - URL no longer contains `/login` or `/auth`
   - A personal area heading ("האזור האישי", "ברוך הבא")
   - A logout link visible

If still on the login page or an error banner is visible, ask the user to retry. Do not proceed with data extraction until login is confirmed.

---

## STEP 7 — NAVIGATE TO FORM 106 QUERY SERVICE

Try the following URLs in order until one loads the Form 106 data page. After each navigation, take a screenshot and check whether you've landed on a Form 106 list or a year-selection page.

### Attempt A — Direct service URL:
```
https://www.misim.gov.il/emdvhmhdbdmhvsf/
```

### Attempt B — Authenticated income tax portal:
```
https://secapp.taxes.gov.il/SrBkrHN106/flow.aspx
```

### Attempt C — Personal area navigation via UI:
From the Misim homepage (authenticated), look for:
- "שכר ועבודה" or "שכר" section in the left/top menu
- Then "שאילתת טופס 106" or "נתוני שכר שנתיים"
- Or look for "שאילתות" → "טופס 106"

If none found via menus, try using the portal's search box (if present) with the query "טופס 106".

### Attempt D — Gov.il authenticated service:
```
https://www.gov.il/he/service/itc135
```
Then navigate to "שאילתת נתוני שכר".

### If all attempts fail:
Tell the user:
> "I couldn't find the Form 106 query page automatically. Can you navigate to 'שאילתת טופס 106' in the portal and tell me when you're there?"
Wait for confirmation, then continue from STEP 7b.

### STEP 7b — On the Form 106 page:

Once you see a Form 106 data page (year selector or employer list):

1. Take a full screenshot.
2. Extract the page text:
```js
async (page) => document.body.innerText
```
3. Look for:
   - A year (שנה / שנת מס) selector or list
   - An employer list or table

---

## STEP 8 — SELECT TAX YEAR AND EXTRACT DATA

### 8a — Select the tax year

If the page shows a year selector, select `TAX_YEAR`:
```js
async (page) => {
  // Try a <select> dropdown
  const sel = document.querySelector('select[name*="year"], select[name*="shana"], select[id*="year"]');
  if (sel) {
    sel.value = Array.from(sel.options).find(o => o.text.includes('TAX_YEAR'))?.value;
    sel.dispatchEvent(new Event('change', { bubbles: true }));
    return 'selected-via-select';
  }
  // Try a button/link list of years
  const btn = Array.from(document.querySelectorAll('a, button')).find(el => el.textContent.trim() === 'TAX_YEAR');
  if (btn) { btn.click(); return 'selected-via-button'; }
  return 'year-selector-not-found';
}
```

If the year is not available, report which years are shown and ask the user to choose.

After selection, click "חיפוש" or "הצג" (Search/Show) if such a button exists.

Take a screenshot to confirm you see employer data.

### 8b — Extract employer records from the page

Extract all employer rows from the data table. Run:
```js
async (page) => {
  // Common patterns for Form 106 data tables
  const rows = Array.from(document.querySelectorAll('tr, .employer-row, [data-employer]'));
  return rows.map(r => r.innerText).filter(t => t.trim().length > 0);
}
```

Also extract the full page text as a fallback:
```js
async (page) => document.body.innerText
```

### 8c — Parse employer data

For each employer visible on the page, extract:

| Field | Label | Notes |
|---|---|---|
| Employer name | שם המעסיק | Free text |
| Employer ID | מספר מעסיק / ח.פ. | 9-digit number |
| Field 158 | הכנסה חייבת | Taxable income (gross) |
| Field 042 | מס הכנסה שנוכה | Income tax withheld |
| Field 045 | הפקדות עובד לקצבה | Employee pension contribution |
| Field 046 | הפקדות עובד לקרן השתלמות | Employee study fund (keren hishtalmut) |
| Field 047 | הפקדות מעביד לקרן השתלמות | Employer study fund (keren hishtalmut) |
| Field 048 | הפקדות מעביד לקצבה | Employer pension contribution |
| Months worked | חודשי עבודה | 1–12 |

If the portal shows a summary table with key fields but not all of them, note which fields are UNKNOWN — the PDF will contain the full set.

---

## STEP 9 — DOWNLOAD FORM 106 PDF(S)

For each employer, attempt to download the Form 106 PDF:

```js
async (page) => {
  // Look for PDF download / print buttons per employer row
  const pdfButtons = Array.from(document.querySelectorAll(
    'a[href*=".pdf"], a[href*="download"], button[aria-label*="הורד"], ' +
    'button[title*="הדפס"], a[title*="טופס 106"], .pdf-btn, [data-action="download"]'
  ));
  return pdfButtons.map(b => ({ text: b.textContent?.trim(), href: b.href, tag: b.tagName }));
}
```

For each PDF download button found:
1. Set up a download listener:
```js
async (page) => {
  const downloadDir = './data/ID_NUMBER';
  const fileName = `106_TAX_YEAR_EMPLOYER_INDEX.pdf`;

  const downloadPromise = page.waitForEvent('download', { timeout: 20000 });
  // Click the download button — pass the button selector specific to this employer
  const btn = document.querySelector('/* employer-specific selector */');
  if (btn) btn.click();

  try {
    const download = await downloadPromise;
    await download.saveAs(`${downloadDir}/${fileName}`);
    return { saved: `${downloadDir}/${fileName}` };
  } catch (e) {
    return { error: String(e) };
  }
}
```

After each successful download:
- Run `mkdir -p ./data/ID_NUMBER` to ensure the directory exists.
- Confirm the file saved: `./data/<ID_NUMBER>/106_<TAX_YEAR>_<N>.pdf`
- Tell the user: "(saved: `./data/<ID_NUMBER>/106_<TAX_YEAR>_<N>.pdf`)"

If downloading fails (no download button found, or download times out):
- Take a full-page screenshot of the employer's Form 106 record.
- Read the screenshot visually and extract whatever fields are visible.
- Note the missing PDF in the output block.

---

## STEP 10 — READ PDF FOR MISSING FIELDS

For each downloaded PDF, use the Read tool to open it:

```
Read ./data/<ID_NUMBER>/106_<TAX_YEAR>_<N>.pdf
```

Extract any fields that were not visible in the portal's table view, particularly:
- Field 158, 042, 045, 046, 047, 048
- Employer name and ID (verify they match what was shown on screen)
- Months worked
- Any additional fields visible on the form

If the PDF text extraction is unclear, take a screenshot of the PDF page instead.

---

## STEP 11 — CONFIRM DATA WITH USER

Present the extracted data to the user for review:

```
=== Form 106 Data — Tax Year TAX_YEAR ===

Employer 1: [שם המעסיק] (ID: [ח.פ.])
  Field 158 — Taxable income:           ₪[amount]
  Field 042 — Income tax withheld:       ₪[amount]
  Field 045 — Employee pension (ת.ג.):   ₪[amount]
  Field 046 — Employee study fund:       ₪[amount]
  Field 047 — Employer study fund:       ₪[amount]
  Field 048 — Employer pension:          ₪[amount]
  Months worked:                         [N]
  PDF: ./data/[ID_NUMBER]/106_[TAX_YEAR]_1.pdf

[repeat for each employer]

=== End ===
```

Ask:
> "Does this look correct? I'll use these values to update your tax data file.
> Reply **Yes** to confirm, or point out any corrections."

Accept corrections and update the data before proceeding.

---

## STEP 12 — UPDATE TAX DATA FILE

Find the user's data file at `./data/<ID_NUMBER>.md`.

Update or add the `EMPLOYERS:` section with the confirmed data. If `EMPLOYERS: PENDING` exists as a single line, replace it with a proper block. Preserve all other sections.

Each employer block:
```yaml
EMPLOYERS:
  - employer_index: 1
    employer_name: [שם המעסיק]
    employer_id: [9-digit ח.פ.]
    field_158_taxable_income: [amount]
    field_042_tax_withheld: [amount]
    field_045_employee_pension: [amount]
    field_046_employee_study_fund: [amount]
    field_047_employer_study_fund: [amount]
    field_048_employer_pension: [amount]
    months_worked: [1-12]
    document: ./data/[ID_NUMBER]/106_[TAX_YEAR]_1.pdf
  - employer_index: 2
    ...
```

If a field could not be extracted, write `UNKNOWN` (not 0).

Write the updated file immediately after the user confirms.

---

## STEP 13 — OUTPUT RESULT BLOCK

After saving, output a structured block for use by `collect-info` or other skills:

```
=== FORM106_IMPORT START ===
TAX_YEAR: YYYY

EMPLOYERS:
  - employer_index: 1
    employer_name: [name]
    employer_id: [9-digit id]
    field_158_taxable_income: [amount]
    field_042_tax_withheld: [amount]
    field_045_employee_pension: [amount]
    field_046_employee_study_fund: [amount]
    field_047_employer_study_fund: [amount]
    field_048_employer_pension: [amount]
    months_worked: [N]
    document: ./data/[ID_NUMBER]/106_[TAX_YEAR]_1.pdf
  (repeat per employer)
=== FORM106_IMPORT END ===
```

Tell the user:
> "✓ Form 106 data saved for [N] employer(s) in [TAX_YEAR]."

---

## MULTI-YEAR SUPPORT

If the user wants to import Form 106 for multiple tax years:

After completing one year, ask:
> "Would you like to import Form 106 for another tax year? (e.g., 2021, 2020...)"

If yes, repeat Steps 8–13 for the new year without re-authenticating (the browser session stays active). Each year's PDFs go to the same `./data/<ID_NUMBER>/` directory with the year in the filename.

---

## ERROR HANDLING

| Situation | Action |
|---|---|
| Portal unreachable / timeout | Ask user to check internet connection, retry up to 2 times |
| Session expired mid-navigation | Detect login page redirect, tell user to re-authenticate, and resume from Step 6 |
| Year not found in portal | Report which years are available, ask user to choose or enter data manually |
| No employers found for the selected year | Ask: "The portal shows no Form 106 submissions for [year]. Did you work that year? If yes, you may need to contact your employer." |
| PDF download fails | Screenshot the Form 106 page, read visually, note missing PDF in output block |
| Portal shows maintenance / scheduled downtime | Read the message, translate if needed, tell user to try again later |
| Browser locked ("already in use") | Run `pkill -f "mcp-chrome" && sleep 2`, then retry navigation |
| CAPTCHA appears | Ask user to solve it in the browser, then confirm when done |
| Chrome credentials are wrong / rejected | Fall back to manual entry, do not retry the bad credentials |

---

## IMPORTANT NOTES

- **Never write credentials to disk.** Credentials retrieved from Chrome via `chrome-credentials` are used only in this session and never appended to any `.md` file.
- The Playwright browser is isolated from the user's personal browser. If credentials are not saved in Chrome, the user must type them directly.
- Form 106 PDFs are stored in `./data/<ID_NUMBER>/` for traceability and future reference. The `document:` field in the data file records the local path.
- **Form 106 vs Form 101:** Form 106 is the *annual* salary certificate (end of year). Form 101 is the personal details form submitted at the start of employment. This skill retrieves Form 106 only.
- If the user had a **tax coordination form (טופס 116)**, the portal may show combined income across employers — note this explicitly in the output.
- The portal typically shows Form 106 data for the previous 6 tax years (2019–2024 as of filing year 2025).
- Field 158 is the *taxable* income (after deductions like pension). It is lower than the gross salary. Remind the user of this if they are confused by the number.
- **Spouse employers:** If the user is married and the spouse was also employed, a separate run of this skill for the spouse's ID is needed. Note this to the user after completing the primary filer's import.
