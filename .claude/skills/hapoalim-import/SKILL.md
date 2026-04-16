---
name: hapoalim-import
description: Uses Playwright to log in to Bank Hapoalim (בנק הפועלים) online banking and extract Form 867 (אישור ניכוי מס שנתי) data for a given tax year. Run this during the collect-info flow to auto-populate investment income fields. Requires the Playwright MCP server to be configured.
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_run_code mcp__playwright__browser_select_option mcp__playwright__browser_type mcp__playwright__browser_fill_form mcp__playwright__browser_wait_for Bash(mkdir *) Write
---

You are an automation assistant that extracts Israeli tax data from Bank Hapoalim's (בנק הפועלים) online banking portal using Playwright.

Your goal: navigate to the Form 867 (אישור ניכוי מס שנתי) page, retrieve the tax certificate for a given year, and return the structured data for use in the tax refund interview.

Detect the user's language and respond in it throughout.

---

## PREREQUISITES CHECK

Before starting, verify the Playwright MCP server is available by checking if `mcp__playwright__browser_navigate` is in your allowed tools. If not, tell the user:

> "The Playwright MCP server is required for this skill. Add it to `.mcp.json`:
> ```json
> { \"mcpServers\": { \"playwright\": { \"command\": \"npx\", \"args\": [\"@playwright/mcp@latest\"] } } }
> ```
> Then restart Claude Code and try again."

---

## STEP 1 — GET TAX YEAR

If the tax year was not provided as input, ask:
> "Which tax year would you like to import from Bank Hapoalim? (e.g., 2022)"

Valid range: `Y - 6` through `Y - 1`, where `Y` is the current calendar year (derived from today's date). Store as `TAX_YEAR`.

---

## STEP 2 — OPEN BANK HAPOALIM LOGIN

1. Navigate to the login page:
   ```
   https://login.bankhapoalim.co.il/ng-portals/auth/he/login
   ```

2. Take a screenshot to confirm the page loaded.

3. Tell the user:
   > "The Bank Hapoalim login page is open. Please log in with your username (קוד משתמש) and password (סיסמה) in the browser window, then let me know when you're in."

   Wait for the user to confirm they are logged in before proceeding.

---

## STEP 3 — VERIFY LOGIN SUCCESS

After the user confirms login:

1. Take a screenshot.
2. Check the page URL — if it contains `/homepage`, login was successful.
3. If still on the login page, ask the user to log in again and wait.

---

## STEP 4 — NAVIGATE TO FORM 867 PAGE

Navigate directly to the annual tax certificate page:
```
https://login.bankhapoalim.co.il/ng-portals/rb/he/deposits-and-savings/annual-tax-certificate-order
```

Take a screenshot to confirm you're on "הזמנת אישור מס שנתי (טופס 867)".

If redirected to homepage, the session may have expired — ask the user to log in again.

---

## STEP 5 — SELECT TAX YEAR

The page has a year combobox with id `taxYearSelect`. It defaults to the current year.

1. Click the combobox to open the dropdown:
   ```js
   async (page) => {
     await page.getByRole('combobox', { name: 'בחר שנת מס' }).click();
   }
   ```

2. Select the target year using `browser_run_code`:
   ```js
   async (page) => {
     const option = await page.$(`[role="listbox"] [role="option"]:has-text("${TAX_YEAR}"), [id="listbox-taxYearSelect"] button:has-text("${TAX_YEAR}")`);
     if (option) { await option.click(); return 'selected'; }
     // Fallback: list available years
     const options = await page.$$eval('[role="listbox"] [role="option"], [id="listbox-taxYearSelect"] button', els => els.map(e => e.textContent?.trim()));
     return { notFound: true, availableYears: options };
   }
   ```

   If the year is not available, tell the user which years are available and ask them to choose.

3. Click **המשך** (Continue):
   ```js
   async (page) => { await page.getByRole('button', { name: 'המשך' }).click(); }
   ```

---

## STEP 6 — CONFIRM REQUEST

The page moves to step 2 showing: "אנא אשר כי ברצונך לקבל אישור מס לשנת {TAX_YEAR}"

Click **אשר** (Confirm):
```js
async (page) => {
  const btn = await page.$('button:has-text("אשר")');
  if (btn) { await btn.click(); return 'confirmed'; }
  return 'button not found';
}
```

---

## STEP 7 — READ THE RESULT & DOWNLOAD THE PDF

Take a screenshot and read the result page (step 3 — סיום).

### 7a — Extract text content

Extract all text from the page:
```js
async (page) => {
  return await page.evaluate(() => document.body.innerText);
}
```

### Case A — No tax events (zero income)

The certificate text will contain:
> "הריני לאשר כי לא חלו אירועי מס בחשבון ... בשנת {TAX_YEAR}. לפיכך לא נוכה מס בגין פעילות בחשבון זה."

If this message is present → **RESULT: no investment income for this year.**

### Case B — Tax was withheld (actual Form 867 data)

The certificate will contain fields such as:
- **הכנסה חייבת** (taxable income)
- **מס שנוכה / ניכוי מס במקור** (tax withheld at source)

Parse for these values. If unclear, take a full-page screenshot and read it visually.

### 7b — Download the PDF (ALWAYS, regardless of Case A or B)

The result page shows a certificate preview with a PDF download button (📄 icon). Download it using Playwright's download event:

```js
async (page) => {
  const downloadDir = './data/{ID_NUMBER}';
  const fileName = `867_{TAX_YEAR}.pdf`;

  // Set up download listener BEFORE clicking
  const downloadPromise = page.waitForEvent('download', { timeout: 15000 });

  // Click the PDF download icon (first icon in the certificate toolbar)
  const pdfBtn = await page.$('a[href*=".pdf"], button[aria-label*="הורד"], .pdf-icon, [data-cc*="pdf"]');
  if (pdfBtn) {
    await pdfBtn.click();
  } else {
    // Fallback: click the first icon in the certificate preview toolbar
    await page.evaluate(() => {
      const icons = document.querySelectorAll('.approval-preview button, .certificate-toolbar button, .doc-toolbar a');
      if (icons[0]) icons[0].click();
    });
  }

  try {
    const download = await downloadPromise;
    await download.saveAs(`${downloadDir}/${fileName}`);
    return { saved: `${downloadDir}/${fileName}` };
  } catch (e) {
    return { error: 'Download timed out or no download triggered', message: String(e) };
  }
}
```

If the download approach fails, try intercepting the PDF URL instead:
```js
async (page) => {
  // Look for a direct PDF link in the page
  const pdfUrl = await page.evaluate(() => {
    const links = Array.from(document.querySelectorAll('a[href]'));
    const pdfLink = links.find(a => a.href.includes('.pdf') || a.href.includes('download') || a.href.includes('print'));
    return pdfLink ? pdfLink.href : null;
  });
  return pdfUrl;
}
```

If a URL is found, use `page.goto(pdfUrl)` to navigate to it and then trigger a download by using Playwright's download-event listener on the resulting page. If that also fails, take a full-page screenshot of the PDF for manual data extraction.

**After downloading:**
1. Create the directory if needed: `mkdir -p ./data/{ID_NUMBER}`
2. Confirm the file exists at `./data/{ID_NUMBER}/867_{TAX_YEAR}.pdf`
3. Tell the user: "(saved: `./data/{ID_NUMBER}/867_{TAX_YEAR}.pdf`)"

---

## STEP 8 — HANDLE MULTIPLE ACCOUNTS

If the user has multiple bank accounts, the form may show a dropdown to select an account before requesting. Check for an account selector and, if present:

1. Show the user the list of accounts found.
2. Ask: "Which account should I generate the Form 867 for? (or say 'all' to go through each one)"
3. Repeat Steps 5–7 for each requested account.
4. Combine results — if multiple accounts have income, list each separately under `INVESTMENT_INCOME`.

---

## STEP 9 — OUTPUT RESULT

Return structured output for use in `collect-info`. Wrap the payload between `=== HAPOALIM_IMPORT START ===` and `=== HAPOALIM_IMPORT END ===`. Include `TAX_YEAR`, the populated `INVESTMENT_INCOME` section following the `./data/example/<year>.md` layout (or `INVESTMENT_INCOME: NONE` when Form 867 certifies no taxable events), and `DOCUMENT: <pdf path>`.

Show the user a summary and ask them to confirm the extracted numbers before finishing.

The `DOCUMENT` field records where the PDF was saved.

---

## ERROR HANDLING

| Situation | Action |
|---|---|
| Redirected to homepage after navigating to Form 867 URL | Session expired — ask user to log in again |
| Year not available in dropdown | Show available years, ask user to choose or skip |
| Page text unclear / numbers not found | Take a full-page screenshot and read the certificate visually |
| OTP / two-factor prompt appears | Ask user to complete it in the browser and confirm when done |
| Page shows an error message | Screenshot and report the error message text to the user |

---

## IMPORTANT NOTES

- **Never store or log the user's bank credentials.** This skill only automates navigation — the user types their credentials directly in the browser.
- The Playwright browser is **isolated** from the user's personal browser. The user must log in inside the Playwright browser window.
- Bank Hapoalim may require OTP (one-time password) on first login from a new browser. Guide the user through it if needed.
- The Form 867 covers: savings account interest, investment income (ניירות ערך), foreign currency deposits. It does NOT cover employer salary (that's Form 106).
- Bank number for Bank Hapoalim is **12** (used in the bank account section of the tax form).
