---
name: form106-import
description: Uses Playwright to log in to the Israeli Tax Authority (Misim) portal and retrieve every available Form 106 (טופס 106) employer salary certificate. Defaults to importing ALL years the portal exposes (usually 2017 forward, up to 9 years) in a single run — downloads each PDF and extracts fields 158, 042, 045, 218, employer name/ID, and months worked. Accepts an optional year filter for a single year or subset. Run during the collect-info flow to auto-populate the employment income section across multiple years in one pass.
allowed-tools: mcp__playwright__browser_navigate mcp__playwright__browser_snapshot mcp__playwright__browser_click mcp__playwright__browser_take_screenshot mcp__playwright__browser_evaluate mcp__playwright__browser_tabs mcp__playwright__browser_wait_for mcp__playwright__browser_type mcp__playwright__browser_fill_form mcp__playwright__browser_press_key mcp__playwright__browser_close Bash(ls *) Bash(mkdir *) Bash(python3 *) Bash(find *) Bash(rm *) Read Write Skill(chrome-credentials)
---

You are an automation assistant that retrieves Israeli employer salary certificates (Form 106 / טופס 106 — nispach ב') from the Israeli Tax Authority (Misim / רשות המסים) portal.

**Default behaviour: download every year the portal exposes.** The "טפסי 106" page lists each available tax year as a collapsible card (typically 2017 forward — up to 9 years). In one run this skill iterates over every year with data, downloads each employer's PDF, extracts the key fields, and merges the results into `YEARS.<year>.EMPLOYERS` in `info.md`. If the user requests a specific year (or a subset), scope to those only.

Detect the user's language from their first message and respond in it throughout (Hebrew or English).

---

## PREREQUISITES CHECK

Verify the Playwright MCP server is available. If `mcp__playwright__browser_navigate` isn't in scope, tell the user:

> "The Playwright MCP server is required for this skill. Add it to `.mcp.json`:
> ```json
> { \"mcpServers\": { \"playwright\": { \"command\": \"npx\", \"args\": [\"@playwright/mcp@latest\"] } } }
> ```
> Then restart Claude Code and try again."

If navigation later fails with "Browser is already in use":
```bash
pkill -f "mcp-chrome" && sleep 2
```
Then retry.

---

## STEP 1 — LOCATE THE FILER DIRECTORY

List `./data/`:
```bash
ls -1 ./data/
```

Find a directory named with a 9-digit Israeli ID (ignore `draft`, `README.md`, `.gitkeep`). Read `<id>/info.md` and record:
- `ID_NUMBER` — the 9-digit directory name (also the filer's ת.ז.)
- `PERSONAL.name` — for logging and portal-heading cross-check
- The list of years under `YEARS:` that already have `EMPLOYERS` populated

If no ID directory exists, ask the user for their 9-digit ID so the skill knows where to save PDFs. Create the directory later with `mkdir -p` when a PDF is first written.

---

## STEP 2 — DETERMINE SCOPE

Parse the user's request into `SCOPE`:

- `all` — default when the user says "all", "every year", or gives no year argument. The skill will import every year the portal shows that has at least one employer record.
- A single year (e.g. `2023`) — import only that year.
- A list of years (e.g. `2022, 2023, 2024`) — import only those.

Then compare `SCOPE` against the years already saved in `info.md`. If any years overlap, ask once for the whole overlap:

> "I already have Form 106 data saved for [year list]. Overwrite them with fresh data from the portal? (yes / skip)"

- `yes` (default) → include these years in the refresh.
- `skip` → remove them from `SCOPE`.

Years whose `YEARS.<year>.SUBMISSION.status` is `submitted` must never be overwritten automatically — always skip those and list them in the final output as "already filed, data preserved".

Tell the user what you're about to do:

> "I'll import Form 106 for: [year list]. Logging in to Misim now."

---

## STEP 3 — AUTO-FILL WITH CHROME CREDENTIALS

Call `chrome-credentials` inline with site `misim.gov.il`. The skill's alias map expands the search to the hosts where Misim credentials are actually saved (`secapp.taxes.gov.il`, `login.gov.il`, `account.gov.il`, `shaam`, …).

- Credentials returned → store `MISIM_ID` (9-digit ת.ז.) and `MISIM_PASSWORD` (קוד משתמש קבוע, the "permanent user code"). Tell the user:
  > "Found saved Misim credentials in Chrome — I'll fill them in automatically."
- Nothing returned → continue to Step 4 without auto-fill. Do not block.

Do not display the password in chat. Type it directly into the browser.

---

## STEP 4 — OPEN PORTAL AND AUTHENTICATE

Navigate to:
```
https://secapp.taxes.gov.il/SrSherutAtzmi/#/clientMainDashboard
```

If not yet authenticated, the portal redirects to `secapp.taxes.gov.il/taxes-login/login/general`. The login page has:
- `textbox "מספר זהות"` — Israeli ID
- `textbox "קוד משתמש קבוע"` — permanent user code
- `button "המשך"` — submit

### With saved credentials
Type `MISIM_ID` into `מספר זהות`, type `MISIM_PASSWORD` into `קוד משתמש קבוע`, then click `המשך`.

### Without saved credentials
Ask the user to fill both fields in the browser window and click `המשך`, then wait for confirmation.

### OTP step
The portal sends an SMS OTP to the phone registered on the user's ID. Take a snapshot to locate the OTP field. If the `sms-otp` skill is available, call it inline to auto-read the code from macOS Messages and fill it in; otherwise ask the user to paste the code and submit.

---

## STEP 5 — VERIFY LOGIN

Snapshot and confirm:
- URL no longer contains `/taxes-login` or `/login`
- Header shows `אזור אישי` with the user's name (e.g. `שלום גבאי אופק,`)
- A `ניתוק` (logout) link is visible

If the login page is still showing, have the user retry. Do not proceed without authentication.

---

## STEP 6 — NAVIGATE TO THE FORM 106 PAGE

Navigate directly to:
```
https://secapp.taxes.gov.il/sr-ezor-ishi/main/form106
```

(Equivalent: click the "טפסי 106" card on the authenticated dashboard.)

Wait for these page elements:
- heading `"טפסי 106"` (h1)
- subheading `"טפסי 106 של <id> - <name>"` (h2) — verify `<id>` matches `ID_NUMBER`; if not, stop and ask the user which filer directory to use.

Snapshot the page.

---

## STEP 7 — DISCOVER AVAILABLE YEARS

Year cards are `details/summary` elements. Each summary contains an `h3` with the 4-digit year and an optional `p` with text like `"נמצאו N מעסיקים"` when the year has data. Enumerate them:

```js
async () => {
  const summaries = Array.from(document.querySelectorAll('summary'));
  return summaries.map(s => ({
    year: s.querySelector('h3')?.textContent?.trim(),
    note: s.querySelector('p')?.textContent?.trim() || ''
  })).filter(x => /^\d{4}$/.test(x.year || ''));
}
```

Rules:
- `note` matches `/נמצאו (\d+) מעסיקים/` → year has that many employers; include if in `SCOPE`.
- `note` is empty or missing → year has no data; skip.
- If the page shows a `button "טפסים משנים קודמות"` and `SCOPE` includes years older than the initially visible set, click it, wait for the expansion, and re-run the discovery query.

Store as `YEARS_TO_IMPORT` — the intersection of `SCOPE` and years that actually have data. Report to the user: "Found Form 106 data for [year list] — downloading now."

---

## STEP 8 — DOWNLOAD PDFs (OUTER LOOP — PER YEAR)

Iterate `YEARS_TO_IMPORT` newest → oldest. For each `<year>`:

### 8a. Expand the year card

Take a snapshot, find the `summary` whose visible text starts with `<year>`, click it. The expansion reveals:
- A combined-summary box: `משכורות ותשלומים` (sum of 158) and `סכום מס הכנסה` (sum of 042). Record these as `portal_totals[<year>]` for cross-checking after PDF parsing.
- One section per employer with `שם המעביד`, `תיק ניכויים` (9-digit employer id), and a button `להצגת טופס 106`.
- Optional `למסמך ריכוז הכנסות` button at the top of the year.

### 8b. Enumerate employers

For the expanded card, collect `{ employer_name, employer_id, button_ref }` for every `להצגת טופס 106` button inside this year's section. Do not confuse with buttons from other years — match on DOM ancestry of the specific `details` element.

### 8c. Download each employer's PDF (INNER LOOP)

For each employer indexed `N = 1..K`:

1. Click the employer's `להצגת טופס 106` button. The portal opens a new tab (index 1) with a `blob:` URL containing the PDF.
2. `browser_tabs` → `select` index `1`.
3. Save the PDF bytes without pulling them through your context. Call `browser_evaluate` and pass a `filename` parameter so the result streams to disk:

   ```js
   async () => {
     const resp = await fetch(window.location.href);
     const buf = await resp.arrayBuffer();
     const bytes = new Uint8Array(buf);
     let binary = '';
     for (let i = 0; i < bytes.byteLength; i++) binary += String.fromCharCode(bytes[i]);
     return { size: buf.byteLength, b64: btoa(binary) };
   }
   ```

   Pass `filename: "_form106_<year>_<N>.b64.txt"` — this writes the evaluate result to a file under the Playwright working directory and keeps the base64 out of chat context. **Never omit `filename` here**; a 60 KB PDF encoded inline will blow past the tool-result token limit.

4. Decode and save the PDF to the filer directory, then remove the temp file:

   ```bash
   python3 - <<'PY'
   import re, base64, glob, sys
   src_matches = glob.glob('**/_form106_<year>_<N>.b64.txt', recursive=True)
   if not src_matches:
       sys.exit("no b64 file found")
   src = src_matches[0]
   dst = "./data/<ID_NUMBER>/106_<year>_<N>.pdf"
   text = open(src).read()
   m = re.search(r'"b64"\s*:\s*"([^"]+)"', text)
   if not m:
       sys.exit("no b64 payload in " + src)
   open(dst, "wb").write(base64.b64decode(m.group(1)))
   print(f"saved {dst}")
   import os; os.unlink(src)
   PY
   ```

   Run `mkdir -p ./data/<ID_NUMBER>` beforehand if this is the first write for the filer.

5. `browser_tabs` → `close` index `1`. Focus returns to the year-list tab automatically.

### 8d. (Optional) Consolidated income summary

If the user asked for the yearly summary too, click `למסמך ריכוז הכנסות`, follow the same blob-download flow, and save as `106_<year>_summary.pdf`. Skip by default to save time — the per-employer PDFs already hold every field needed for Form 135.

### 8e. Collapse the year card

Click the `summary` again to collapse it. This keeps the DOM small for the next iteration. If the collapse fails it's harmless — continue.

---

## STEP 9 — PARSE EACH DOWNLOADED PDF

For every `./data/<ID_NUMBER>/106_<year>_<N>.pdf`, call `Read` on the PDF path. Claude Code renders the PDF visually, so you can extract the Hebrew labels directly.

The portal exports the "nispach ב'" (Appendix B) layout. Map its labels to the `info.md` schema (see `./data/README.md`):

| Hebrew label on the PDF | Schema field (`info.md`) |
|---|---|
| `שם המעביד` | `employer_name` |
| `מס' תיק ניכויים` | `employer_id` (9 digits) |
| Count of `V` marks in `חדשי עבודה בשנת המס` (or the `סה"כ` cell) | `months_worked` |
| `סה"כ (158/172)` | `field_158_taxable_income` |
| `סה"כ ניכויי מס (042)` | `field_042_tax_withheld` |
| `הפקדות העובד לקופ"ג לקצבה (...086/045)` | `field_045_employee_pension` |
| `בסיס להשתלמות (218/219)` | `field_218_study_fund` |

Rules:
- If a row is absent from the PDF, the value is `0` — write `0`, not `UNKNOWN`. Small employers (e.g. short side gigs with no pension) commonly omit `086/045` and adjacent rows.
- Strip currency symbols and commas before saving (`73,521 ש"ח` → `73521`).
- Informational rows (`הכנסה מבוטחת (244/245)`, `תשלומי מעסיק לקרן השתלמות`, `בסיס לאובדן כושר`, `סכום הפרשות המעביד לקופות גמל לקצבה (248/249)`, credit points) are **not** stored in `info.md`. Ignore them unless explicitly needed downstream.

After parsing every PDF for a year, verify `sum(field_158) == portal_totals[<year>].משכורות ותשלומים` and `sum(field_042) == portal_totals[<year>].סכום מס הכנסה` to ±1 ₪. If they disagree, flag the year in the final summary and ask the user to eyeball the PDFs.

---

## STEP 10 — PRESENT A CONSOLIDATED SUMMARY

Print one block per imported year, then portal-confirmed totals so the user can cross-check. Order newest → oldest:

```
=== Form 106 Import — [M] year(s), [N] employer(s) total ===

YEAR 2024
  Employer 1: <name> (ID: <emp_id>)
    Field 158 (Taxable income):   ₪<n>
    Field 042 (Tax withheld):     ₪<n>
    Field 045 (Employee pension): ₪<n>
    Field 218 (Study fund basis): ₪<n>
    Months worked:                <n>
    PDF: ./data/<ID_NUMBER>/106_2024_1.pdf
  [repeat per employer]
  Year totals — income: ₪<sum 158>, tax: ₪<sum 042>   (portal: ₪<portal-summary>)

YEAR 2023
  [same structure]

=== End ===
```

Ask:

> "Does this look right? Reply **Yes** to save all years, or tell me which year/field to correct."

Accept corrections year-by-year before writing to disk.

---

## STEP 11 — WRITE TO `<year>.md`

Follow the schema in `./data/README.md` (use `./data/example/<year>.md` as the reference sample). For each confirmed year in `YEARS_TO_IMPORT`, read the existing `./data/<ID_NUMBER>/<year>.md` (if it exists) with the Read tool, replace the file's top-level `EMPLOYERS` list with the new data, preserve every other section, and rewrite the complete `<year>.md`. Personal and bank data live in `info.md` and `bank.yaml` respectively — do not touch them.

Rules:
- Never rewrite the `EMPLOYERS` block of a year with `SUBMISSION.status: submitted`. Preserve it and note in the output that the year was skipped.
- The PDF path is predictable (`./data/<ID_NUMBER>/106_<year>_<N>.pdf`); don't write a `document:` field under `EMPLOYERS` (the project schema doesn't include one there).
- If `field_218_study_fund` is zero, omit the line — it's optional per the schema.

---

## STEP 12 — OUTPUT RESULT BLOCK

Emit one consolidated machine-readable block between `=== FORM106_IMPORT START ===` and `=== FORM106_IMPORT END ===`. Include:

- `FILER_ID`, `YEARS_IMPORTED`, and (when non-empty) `YEARS_SKIPPED_ALREADY_SUBMITTED` and `YEARS_SKIPPED_USER_CHOICE`.
- A `YEARS:` map. Each year's `EMPLOYERS` list follows the shape in `./data/example/<year>.md`.

Downstream skills (including `collect-info`) parse this directly.

Tell the user:

> "✓ Form 106 data saved for [N] employer(s) across [M] year(s): [year list]."

---

## ERROR HANDLING

| Situation | Action |
|---|---|
| Portal unreachable / timeout | Ask the user to check the internet, retry up to 2 times. |
| Session expired mid-loop (URL redirects to `/taxes-login`) | Re-run Step 4, then resume Step 8 at the next unfinished year. PDFs already saved for earlier years are kept. |
| Year list is empty | The user has no Form 106 records at the portal. Ask them to contact each employer for a paper copy. |
| A year shows no employer entries | Skip quietly; don't create an empty `EMPLOYERS` block. |
| `browser_evaluate` returns a huge base64 string inline | Always pass `filename: "_form106_..."` to stream the result to disk. Re-run the evaluate with the filename if you forgot it the first time. |
| PDF download fails for one employer (blob fetch errors, tab never opens) | Take a full-page screenshot of the expanded year card and extract fields from the on-page per-employer summary if present. Note `PDF: MISSING` for that employer in the final output. |
| Portal maintenance banner | Translate if needed, stop, tell the user to retry later. |
| Browser locked ("already in use") | `pkill -f "mcp-chrome" && sleep 2`, then retry navigation. |
| CAPTCHA appears mid-flow | Ask the user to solve it in the browser window and confirm. |
| Chrome credentials rejected | Do not retry the same password. Fall back to manual entry; tell the user to update Chrome once logged in. |
| Portal heading ID differs from `ID_NUMBER` | Stop. The logged-in account is not the same filer as the data directory. Ask the user which directory to write to, or to log in with the correct ID. |

---

## IMPORTANT NOTES

- **Never write credentials to disk.** Credentials returned by `chrome-credentials` live in this session only — never append them to any `.md` file.
- **Blob downloads are the bottleneck.** Each "להצגת טופס 106" click opens a new tab with a `blob:` URL; Playwright can't stream it to the local filesystem directly. The supported path is: switch to the blob tab, call `browser_evaluate` with `filename:` to dump base64 into a file, then decode with Python. Do not try to return the base64 from `evaluate` inline — it will overflow the tool-result token budget.
- PDFs live in `./data/<ID_NUMBER>/` named `106_<year>_<N>.pdf`. The filename is self-describing; no `document:` schema field is needed.
- **Form 106 vs Form 101.** Form 106 is the end-of-year salary certificate; Form 101 is the personal-details form filled at the start of employment. This skill retrieves Form 106 only.
- **Tax coordination form (טופס 116).** If the user had one, the portal may also show a combined row — note this explicitly in the output and trust the per-employer PDFs over any summary line.
- The portal typically shows 2017 forward (about 9 years of data). Older tax years require a paper request at the user's tax office.
- Field 158 is *taxable* income (after pension deductions). It is lower than gross salary. If the user questions the number, reassure them Form 135 uses 158, not gross.
- **Spouse employers.** For married filers, the spouse's Form 106s must be retrieved in a separate run after logging in with the spouse's ID.
