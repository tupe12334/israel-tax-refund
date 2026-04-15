---
name: leumi-import
description: Uses Playwright to log in to Bank Leumi (בנק לאומי) online banking and extract Form 867 (אישור ניכוי מס שנתי) data for a given tax year. Run this during the collect-info flow to auto-populate investment income fields. Requires the Playwright MCP server to be configured.
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_run_code mcp__playwright__browser_select_option mcp__playwright__browser_type mcp__playwright__browser_fill_form mcp__playwright__browser_wait_for Bash(mkdir *) Write
---

You are an automation assistant that extracts Israeli tax data from Bank Leumi's (בנק לאומי) online banking portal using Playwright.

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
> "Which tax year would you like to import from Bank Leumi? (e.g., 2022)"

Valid range: 2019–2024. Store as `TAX_YEAR`.

---

## STEP 2 — OPEN BANK LEUMI LOGIN

1. Navigate to the login page:
   ```
   https://hb2.bankleumi.co.il/login
   ```

2. Take a screenshot to confirm the page loaded.

3. Tell the user:
   > "The Bank Leumi login page is open. Please log in with your username (שם משתמש) and password (סיסמה) in the browser window, then let me know when you're in."

   Wait for the user to confirm they are logged in before proceeding. Bank Leumi may require OTP (one-time password) on first login from a new browser — guide the user through it if needed.

---

## STEP 3 — VERIFY LOGIN SUCCESS

After the user confirms login:

1. Take a screenshot.
2. Check the page URL — if it contains `/uniquelogin` or `/accountsmenu` or the main dashboard, login was successful.
3. If still on the login page, ask the user to log in again and wait.

---

## STEP 4 — NAVIGATE TO FORM 867 PAGE

Bank Leumi's Form 867 ordering page is located under the "פעולות בניירות ערך" / "אישורים ודוחות" menu. The known direct URL is:

```
https://hb2.bankleumi.co.il/ebanking/securities/annual-tax-certificate
```

If that URL redirects to the dashboard, navigate via the menu:
1. Open the side menu / "תפריט ראשי".
2. Click "אישורים ודוחות" (Certificates & Reports).
3. Click "אישור ניכוי מס שנתי" / "טופס 867".

Take a screenshot to confirm you're on the Form 867 page. If redirected to the dashboard, the session may have expired — ask the user to log in again.

---

## STEP 5 — SELECT TAX YEAR

The page has a year selector. Click it and choose the target year:

```js
async (page) => {
  // Try combobox first
  const combo = await page.$('select[name*="year"], [role="combobox"][aria-label*="שנת"]');
  if (combo) {
    await combo.click();
  }
  const option = await page.$(`[role="option"]:has-text("${TAX_YEAR}"), option:has-text("${TAX_YEAR}")`);
  if (option) { await option.click(); return 'selected'; }
  // Fallback: list available years
  const options = await page.$$eval('[role="option"], option', els => els.map(e => e.textContent?.trim()));
  return { notFound: true, availableYears: options };
}
```

If the year is not available, tell the user which years are available and ask them to choose.

Click **המשך** (Continue) or **הפק אישור** (Generate Certificate):
```js
async (page) => {
  const btn = await page.$('button:has-text("המשך"), button:has-text("הפק"), button:has-text("אישור")');
  if (btn) { await btn.click(); return 'submitted'; }
  return 'button not found';
}
```

---

## STEP 6 — READ THE RESULT & DOWNLOAD THE PDF

Take a screenshot and read the result page.

### 6a — Extract text content

```js
async (page) => {
  return await page.evaluate(() => document.body.innerText);
}
```

### Case A — No tax events (zero income)

If the text contains language like "לא חלו אירועי מס" or "לא נוכה מס" → **RESULT: no investment income for this year.**

### Case B — Tax was withheld (actual Form 867 data)

Look for:
- **הכנסה חייבת** / **סה"כ הכנסה חייבת** (taxable income)
- **מס שנוכה במקור** / **ניכוי מס במקור** (tax withheld at source)

Parse for these values. If unclear, take a full-page screenshot and read it visually.

### 6b — Download the PDF (ALWAYS)

```js
async (page) => {
  const downloadDir = './data/{ID_NUMBER}';
  const fileName = `867_leumi_{TAX_YEAR}.pdf`;

  const downloadPromise = page.waitForEvent('download', { timeout: 15000 });

  const pdfBtn = await page.$('a[href*=".pdf"], button[aria-label*="הורד"], button:has-text("הורדה"), .pdf-download');
  if (pdfBtn) {
    await pdfBtn.click();
  } else {
    await page.evaluate(() => {
      const icons = document.querySelectorAll('.certificate-actions button, .doc-toolbar a, [class*="download"]');
      if (icons[0]) icons[0].click();
    });
  }

  try {
    const download = await downloadPromise;
    await download.saveAs(`${downloadDir}/${fileName}`);
    return { saved: `${downloadDir}/${fileName}` };
  } catch (e) {
    return { error: 'Download timed out', message: String(e) };
  }
}
```

If the download approach fails, take a full-page screenshot of the certificate for manual data extraction.

**After downloading:**
1. Create the directory if needed: `mkdir -p ./data/{ID_NUMBER}`
2. Confirm the file exists at `./data/{ID_NUMBER}/867_leumi_{TAX_YEAR}.pdf`
3. Tell the user: "(saved: `./data/{ID_NUMBER}/867_leumi_{TAX_YEAR}.pdf`)"

---

## STEP 7 — HANDLE MULTIPLE ACCOUNTS

If the user has multiple Leumi accounts, the page may show an account selector. If present:

1. Show the user the list of accounts found.
2. Ask: "Which account should I generate the Form 867 for? (or say 'all' to go through each one)"
3. Repeat Steps 5–6 for each requested account.
4. Combine results — if multiple accounts have income, list each separately under `INVESTMENT_INCOME`.

---

## STEP 8 — OUTPUT RESULT

Return structured output for use in `collect-info`:

### If no investment income:
```
=== LEUMI_IMPORT START ===
TAX_YEAR: {TAX_YEAR}
BANK: בנק לאומי (10)

INVESTMENT_INCOME: NONE
# Form 867 certified: no taxable events in {TAX_YEAR}
DOCUMENT: ./data/{ID_NUMBER}/867_leumi_{TAX_YEAR}.pdf
=== LEUMI_IMPORT END ===
```

### If investment income exists:
```
=== LEUMI_IMPORT START ===
TAX_YEAR: {TAX_YEAR}
BANK: בנק לאומי (10)

INVESTMENT_INCOME:
  - institution: בנק לאומי
    income: {הכנסה חייבת}
    tax_withheld: {ניכוי מס במקור}
DOCUMENT: ./data/{ID_NUMBER}/867_leumi_{TAX_YEAR}.pdf
=== LEUMI_IMPORT END ===
```

Show the user a summary and ask them to confirm the extracted numbers before finishing.

---

## ERROR HANDLING

| Situation | Action |
|---|---|
| Redirected to homepage after navigating to Form 867 URL | Session expired — ask user to log in again |
| Year not available in dropdown | Show available years, ask user to choose or skip |
| Page text unclear / numbers not found | Take a full-page screenshot and read the certificate visually |
| OTP / two-factor prompt appears | Ask user to complete it in the browser and confirm when done |
| Certificate takes time to generate | Use `browser_wait_for` to wait for the content to render |
| Page shows an error message | Screenshot and report the error message text to the user |

---

## IMPORTANT NOTES

- **Never store or log the user's bank credentials.** This skill only automates navigation — the user types their credentials directly in the browser.
- The Playwright browser is **isolated** from the user's personal browser. The user must log in inside the Playwright browser window.
- The Form 867 covers: savings account interest, investment income (ניירות ערך), foreign currency deposits. It does NOT cover employer salary (that's Form 106).
- Bank number for Bank Leumi is **10** (used in the bank account section of the tax form).
- Leumi's UI layout may change; if the selectors above don't match, fall back to `browser_snapshot` and let the user point out the right element.
