---
name: idf-service
description: Fetches IDF mandatory service record from the IDF certificates portal (ishurim.prat.idf.il) to extract service start and discharge dates, calculate total service length, and determine tax credit eligibility. Run during the collect-info flow to auto-populate the military service section (Step 7c). For reserve duty (מילואים) income, use the miluim-import skill instead.
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_run_code mcp__playwright__browser_wait_for mcp__playwright__browser_type mcp__playwright__browser_fill_form mcp__playwright__browser_close Bash(ls *) Bash(mkdir *) Read Write
---

You are an automation assistant that fetches IDF mandatory service records from the IDF certificates portal and calculates Israeli tax credit eligibility for veterans.

Your goal: retrieve the user's service start date and discharge date, compute total months served, calculate the tax credit points they are entitled to for each tax year being filed, and update the tax refund data file.

> **Note:** This skill handles **mandatory service (שירות חובה / לאומי)** only. For reserve duty (מילואים) income (Form 3010), use the `miluim-import` skill.

Detect the user's language from their first message and respond in it throughout (Hebrew or English).

---

## PREREQUISITES CHECK

Before starting, verify the Playwright MCP server is available by checking if `mcp__playwright__browser_navigate` is in your allowed tools. If not, tell the user:

> "The Playwright MCP server is required for this skill. Add it to `.mcp.json`:
> ```json
> { \"mcpServers\": { \"playwright\": { \"command\": \"npx\", \"args\": [\"@playwright/mcp@latest\"] } } }
> ```
> Then restart Claude Code and try again."

Also check if there is a lingering Playwright Chrome process that may block the browser. If `mcp__playwright__browser_navigate` fails with "Browser is already in use", run:
```bash
pkill -f "mcp-chrome" && sleep 2
```
Then retry navigation.

---

## STEP 1 — FIND EXISTING DATA

Read the `./data/` directory contents. Look for a file named `<id>.md` (excluding `draft.md`). If found, read it and check whether `TAX_CREDITS.military` is already populated (not `PENDING` and not empty).

- **If already populated:** Show the existing data to the user and ask: "Military service data already exists — would you like to keep it or refresh it from the IDF portal?"
  - If keep → skip to STEP 9 and output the existing data.
  - If refresh → continue to STEP 2.
- **If not populated or file not found:** Continue to STEP 2.

Also extract `TAX_YEAR` from the data file (or ask if no file exists). Store it as `TAX_YEAR`. If multiple years are being filed, ask which one to calculate credits for (or compute for all years).

---

## STEP 2 — OPEN THE IDF CERTIFICATES PORTAL

> **IMPORTANT:** Do NOT navigate to `prat.idf.il` (redirects to `home.idf.il` — for active soldiers only, veterans get "אין גישה").
> Do NOT navigate to `home.idf.il` or `my.idf.il`.
> The correct portal for all discharged veterans is: **`https://ishurim.prat.idf.il`**

Navigate to:
```
https://ishurim.prat.idf.il
```

Take a screenshot. The page should show "ברוכים הבאים לאתר אישורים" with two login buttons:
- **הזדהות לאומית** — national digital ID (biometric)
- **הזדהות דרך משתמש MyIdf** — MyIDF username + password

**Always use the MyIDF login option.** Click the **"הזדהות דרך משתמש MyIdf"** button automatically — do not ask the user to choose.

After clicking, take a screenshot to confirm the MyIDF login page loaded (it should show a username/email field and a phone/password field).

### Auto-fill credentials

Read the user's data file at `./data/<id>.md` to get:
- `PERSONAL.id` → used to form the MyIDF email: `<id>@idf.il`

Run the `chrome-credentials` skill to retrieve the saved password for `my.idf.il` (or `idf.il`).

Once the MyIDF login form is visible:
1. Fill the **email / username field** with `<id>@idf.il` (e.g., `123456789@idf.il`).
2. Fill the **password field** with the password retrieved from `chrome-credentials`.
3. Take a screenshot to verify the fields are populated.
4. Do **not** submit yet — tell the user to complete any remaining steps (OTP, captcha) themselves.

**Try to auto-read the OTP from Messages.app** — run the `sms-otp` skill immediately after the login form appears:
```
Skill(sms-otp) --service idf --minutes 10
```
- If `sms-otp` returns a code:
  - Extract the `code:` value.
  - Type it into the OTP input field using `mcp__playwright__browser_type`.
  - Take a screenshot to confirm entry.
  - Tell the user: "I found your OTP in Messages.app and entered it automatically."
  - Wait for the user to click submit / confirm the OTP step.
- If `sms-otp` returns no result:
  - Tell the user:

> "פורטל האישורים של צה\"ל פתוח — לחצתי על **הזדהות דרך משתמש MyIdf** ומילאתי אוטומטית את כתובת המייל והסיסמה.
>
> אנא השלם/י את ההתחברות (קוד אימות / OTP) ולאחר שמגיעים לדשבורד, תודיע/י לי ואמשיך אוטומטית."
>
> (English: "The IDF Certificates Portal is open — I've clicked **MyIDF login** and pre-filled your email and password. Please complete login (OTP / captcha). Let me know when you reach the dashboard.")

Wait for user to confirm login.

---

## STEP 3 — VERIFY LOGIN SUCCESS

After the user confirms login:

1. Take a screenshot.
2. Take a snapshot to inspect the page.
3. Confirm you're on the dashboard — look for:
   - Page title "בית" and URL containing `ishurim.prat.idf.il/ords/r/hr/ishurim/`
   - The user's name in the top navigation (e.g., "אופק גבאי")
   - A search box labeled "חיפוש אישור"
   - A section "אישורים פופולרים" with certificate cards

If still on the login page, tell the user and wait for another confirmation.

---

## STEP 4 — REQUEST SERVICE CERTIFICATE

On the dashboard you should see "אישורים פופולרים" listing several certificates. Look for:

**"אישור על מהלך שירות צבאי (830)"**

This is certificate #830 (internal cert_id 6743). It contains induction date (תאריך גיוס), discharge date (תאריך שחרור), service type, and units.

Click the **"להגשה"** link next to this certificate. A modal dialog will appear titled "הגשת בקשה לקבלת אישור".

If the certificate is not visible in the popular list, use the search box to search for "830" or "מהלך שירות".

### In the request modal:
1. The certificate "אישור על מהלך שירות צבאי (830)" should already be selected.
2. Fill in the email field **"כתובת המייל לשליחת אישורים"** with the user's email address.
   - Use the email from the data file if available, otherwise ask the user.
3. Click **"שליחת בקשות (1)"**.

A success screen will appear: "הגשתך בוצעה בהצלחה" confirming the request was submitted. The certificate will be sent to email **within 2 hours** and will also appear under "אישורים שלי".

---

## STEP 5 — CONFIRM REQUEST STATUS AND IMMEDIATELY CHECK FOR CERTIFICATE

After submission, click "למעבר למעקב סטטוס הזמנה" or navigate to **"אישורים שלי"** in the top navigation.

Click the **"סטטוס בקשות"** tab and confirm the request shows "1 אישורים בתהליך".

Tell the user:
> "✓ הבקשה הוגשה בהצלחה. אני בודק עכשיו אם האישור כבר זמין…"

Then **immediately proceed to STEP 6** — do not wait for the user to do anything.

---

## STEP 6 — AUTOMATICALLY CHECK FOR CERTIFICATE AND EXTRACT DATES

Run the following checks in order. Do NOT ask the user to check anything — do it yourself.

### Check A — Gmail (run immediately)
Use the Gmail MCP tool (`mcp__claude_ai_Gmail__gmail_search_messages`) to search for the certificate email right now:
- Search query: `from:no-reply@idf.il אישור על מהלך שירות`
- Also try: `subject:אישור שירות צבאי`

If found, read the email with `gmail_read_message` and extract the PDF attachment if present. The certificate PDF contains:
- **תאריך גיוס** (induction date) — service start
- **תאריך שחרור** (discharge date)
- **סוג שירות** (service type)

→ If data extracted: proceed to STEP 7.

### Check B — Portal "אישורים שהתקבלו" tab (run if Check A found nothing)
Navigate back to "אישורים שלי" → tab **"אישורים שהתקבלו"**. Inspect the list — if certificate #830 appears, click its download link and extract the dates from the PDF.

→ If data extracted: proceed to STEP 7.

### Check C — Manual entry fallback (only if both A and B found nothing)
The certificate has not arrived yet (it may take up to 2 hours). Ask the user to enter their service dates now, and note that the data can be refreshed later by re-running this skill:

1. "When did you begin your military or national service? (MM/YYYY) — תאריך תחילת שירות"
2. "When were you discharged? (MM/YYYY) — תאריך שחרור"
3. "Was this mandatory military service (שירות חובה), national service (שירות לאומי), or career military (קצין / קבע)?"

**Validation:**
- Both dates must be valid MM/YYYY format.
- Discharge date must be after service start date.
- Service start must not be before 01/1948 or in the future.
- Service length (months) must be between 12 and 60 for typical mandatory service.
- If computed service length seems unusual, flag it: "That's [N] months — does that sound right?" Do not block — accept user's confirmation.

**Save immediately** after each answer.

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
| Navigating to `prat.idf.il` redirects to `home.idf.il` with "אין גישה" | This is the **active soldiers** portal — ignore it. Navigate to `ishurim.prat.idf.il` instead |
| Browser locked: "Browser is already in use" | Run `pkill -f "mcp-chrome" && sleep 2`, then retry navigation |
| Portal requires re-login mid-session | Ask the user to re-authenticate in the browser and confirm when done |
| Certificate not in popular list | Use the search box on the dashboard to search "830" or "מהלך שירות" |
| Email not received after 2 hours | Navigate to "אישורים שלי" → "סטטוס בקשות" yourself and check the status; if the request shows as failed, resubmit automatically; if still pending, fall back to manual entry (Check C in STEP 6) |
| Dates appear in Hebrew text (e.g., "1 במרץ 2020") | Parse month names: ינואר=01, פברואר=02, מרץ=03, אפריל=04, מאי=05, יוני=06, יולי=07, אוגוסט=08, ספטמבר=09, אוקטובר=10, נובמבר=11, דצמבר=12 |
| User served in multiple roles (e.g., mandatory + career extension) | Ask for each period separately; use the mandatory service period for the tax credit calculation |
| User did not serve (exempt / non-citizen at the time) | Set `military: { service_type: none }` in the data file — no extra credits apply |

---

## PORTAL REFERENCE

| URL | Purpose |
|---|---|
| `https://ishurim.prat.idf.il` | **Certificates portal** — correct entry point for all users including veterans |
| `https://www.home.idf.il` | Active soldiers only (צ360) — veterans get "אין גישה", do not use |
| `https://www.prat.idf.il` | Redirects to `home.idf.il` — do not use for veterans |
| `https://my.idf.il` | MyIDF account management — used for login credentials only |
| `https://www.miluim.idf.il` | Reserve duty portal — use `miluim-import` skill for this |

### Certificate #830 details
- Name: **אישור על מהלך שירות צבאי**
- Internal cert_id: `6743`
- Contains: induction date, discharge date, service type, units served
- Delivery: emailed as PDF within 2 hours of request
- Monthly quota: 8 requests per month

---

## IMPORTANT NOTES

- **Never store or log the user's portal credentials.** The user authenticates directly in the Playwright browser window — this skill only automates post-login navigation.
- The tax credit calculation here is an **estimate**. The official calculation is performed by the Tax Authority's system (רשות המסים). For complex cases (multiple service periods, partial service, disabilities during service), recommend consulting a tax professional.
- Career military (קבע) personnel use a different section of the tax form for their benefits — this skill focuses on the mandatory/national service credit only.
- If the user served in a role with additional benefits (e.g., combat service — "לוחם"), there may be additional credits beyond what this skill calculates. Note this to the user if `service_type == mandatory` and `service_months >= 30`.
