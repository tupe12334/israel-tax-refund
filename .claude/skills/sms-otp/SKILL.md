---
name: sms-otp
description: Reads recent SMS/iMessage texts from the macOS Messages app and extracts 2FA/OTP codes. Requires Full Disk Access for Terminal. Called by other skills (login, idf-service, hapoalim-import, miluim-import) to auto-fill OTP fields without user copy-paste.
allowed-tools: Bash Read Write
---

You are an SMS OTP reader. Your job is to silently read recent SMS/iMessage messages from the macOS Messages app, find the most recent 2FA / OTP code, and return it — either to the user or to the calling skill.

Detect the user's language and respond in it throughout.

---

## GROUND RULES

- **Never write message contents to disk.** Return codes in the conversation only.
- **Never store OTP codes beyond this session.**
- This skill only reads messages the user received on this Mac via the Messages app (iMessage sync or SMS relay from iPhone).
- macOS requires **Full Disk Access** for Terminal to read `~/Library/Messages/chat.db`. Guide the user if access is denied.
- Clean up all `/tmp` files before exiting.

---

## STEP 0 — PARSE INVOCATION CONTEXT

Check if the skill was invoked with arguments. Extract (if provided):
- `SENDER_HINT` — partial phone number or sender name to narrow results (e.g., "+972", "מס הכנסה", "Misim")
- `LOOKBACK_MINUTES` — how far back to search (default: `30`)
- `SERVICE_HINT` — free-text hint for which service sent the OTP (e.g., "misim", "idf", "hapoalim")

If invoked with no arguments, use defaults:
```
SENDER_HINT     = None         (search all senders)
LOOKBACK_MINUTES = 30
SERVICE_HINT    = None
```

---

## STEP 1 — CHECK DATABASE ACCESS

Run:
```bash
sqlite3 "$HOME/Library/Messages/chat.db" "SELECT count(*) FROM message;" 2>&1
```

- If it prints a number → access is granted. Continue.
- If it prints `unable to open`, `permission denied`, or `operation not permitted` → **Full Disk Access is required**.

  Tell the user (in their language):

  > **macOS Full Disk Access required.**
  > Terminal needs Full Disk Access to read your Messages database.
  >
  > 1. Open **System Settings** → **Privacy & Security** → **Full Disk Access**
  > 2. Find **Terminal** (or **iTerm2 / Warp** — whichever you use) in the list and enable it.
  > 3. If Terminal isn't listed, click **+** and add it from `/Applications/Utilities/Terminal.app`
  > 4. Re-run this skill after enabling it.
  >
  > *(Hebrew: מכשולי הרשאה: אנא פתח הגדרות מערכת ← פרטיות ואבטחה ← גישה מלאה לדיסק, ואפשר לטרמינל.)*

  Stop.

---

## STEP 2 — COPY DATABASE TO /tmp

The Messages app keeps `chat.db` locked. Copy the DB files to a temp location:

```bash
cp "$HOME/Library/Messages/chat.db" /tmp/_messages_otp.db 2>/dev/null
cp "$HOME/Library/Messages/chat.db-wal" /tmp/_messages_otp.db-wal 2>/dev/null || true
cp "$HOME/Library/Messages/chat.db-shm" /tmp/_messages_otp.db-shm 2>/dev/null || true
echo "copied"
```

If the copy fails (non-zero exit), tell the user that Messages.app may be preventing access. Ask them to quit Messages.app temporarily and retry.

---

## STEP 3 — WRITE THE EXTRACTOR SCRIPT

Write `/tmp/_sms_otp_extract.py` with the Write tool:

```python
#!/usr/bin/env python3
"""
SMS OTP extractor — reads recent messages from a copy of ~/Library/Messages/chat.db
and finds 2FA / OTP codes.

Usage:
  python3 /tmp/_sms_otp_extract.py [--minutes N] [--sender HINT] [--service HINT]

Outputs JSON: list of {date, sender, text, otp_candidates} sorted newest-first.
"""
import sqlite3, re, sys, json, argparse
from datetime import datetime, timezone, timedelta

# Apple Core Data epoch offset: seconds from Unix epoch to 2001-01-01
APPLE_EPOCH_OFFSET = 978307200

# OTP patterns — ordered from most specific to least
OTP_PATTERNS = [
    # "קוד האימות שלך הוא: 123456" / "Your code is 123456"
    re.compile(r'(?:קוד[^:]*:|code[^:]*:|otp[^:]*:|verification[^:]*:|אימות[^:]*:)\s*(\d{4,8})', re.IGNORECASE),
    # "123456 הוא קוד האימות שלך" (code before Hebrew context)
    re.compile(r'\b(\d{4,8})\b(?=\s*(?:הוא|is|:)\s*(?:קוד|code|otp))', re.IGNORECASE),
    # Standalone 6-digit code (most common OTP length) — high confidence
    re.compile(r'\b(\d{6})\b'),
    # 4-digit code fallback (some banks/services)
    re.compile(r'\b(\d{4})\b'),
    # 8-digit code fallback
    re.compile(r'\b(\d{8})\b'),
]

# Keywords that indicate this message is OTP-related
OTP_KEYWORDS = re.compile(
    r'otp|קוד|code|אימות|verification|סיסמה חד.פעמית|one.time|חד.פעמי|authenticate|2fa|two.factor|login|התחברות|כניסה|misim|hapoalim|idf|מס.הכנסה|ביטוח.לאומי|בנק|bank',
    re.IGNORECASE
)

def apple_ts_to_datetime(ts):
    """Convert Apple Core Data timestamp (seconds or nanoseconds) to UTC datetime."""
    if ts > 1_000_000_000_000:  # nanoseconds
        ts = ts / 1_000_000_000
    return datetime.fromtimestamp(ts + APPLE_EPOCH_OFFSET, tz=timezone.utc)

def extract_otp(text):
    """Return list of OTP candidate strings found in text, highest-confidence first."""
    candidates = []
    for pattern in OTP_PATTERNS:
        for m in pattern.finditer(text):
            code = m.group(1)
            if code not in candidates:
                candidates.append(code)
    return candidates

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--minutes', type=int, default=30,
                        help='Look back N minutes (default: 30)')
    parser.add_argument('--sender', default=None,
                        help='Partial sender phone/name filter')
    parser.add_argument('--service', default=None,
                        help='Service hint (e.g. misim, hapoalim, idf)')
    args = parser.parse_args()

    cutoff = datetime.now(tz=timezone.utc) - timedelta(minutes=args.minutes)
    db_path = '/tmp/_messages_otp.db'

    try:
        conn = sqlite3.connect(db_path)
        conn.row_factory = sqlite3.Row
        cur = conn.cursor()

        cur.execute("""
            SELECT
                m.ROWID,
                m.date          AS raw_date,
                m.text,
                m.is_from_me,
                h.id            AS sender_id,
                COALESCE(r.display_name, h.id, 'Unknown') AS sender_name
            FROM message m
            LEFT JOIN handle h       ON m.handle_id = h.ROWID
            LEFT JOIN (
                SELECT handle_id, display_name
                FROM chat_handle_join chj
                JOIN chat c ON c.ROWID = chj.chat_id
                LIMIT 0   -- placeholder; real display names come from AddressBook, not chat.db
            ) r ON r.handle_id = h.ROWID
            WHERE m.is_from_me = 0
              AND m.text IS NOT NULL
              AND m.text != ''
            ORDER BY m.date DESC
            LIMIT 200
        """)
        rows = cur.fetchall()
        conn.close()
    except Exception as e:
        print(json.dumps({'error': str(e)}))
        sys.exit(1)

    results = []
    for row in rows:
        raw_date = row['raw_date']
        if not raw_date:
            continue
        try:
            msg_dt = apple_ts_to_datetime(raw_date)
        except Exception:
            continue

        if msg_dt < cutoff:
            continue  # older than lookback window

        text = row['text'] or ''
        sender = row['sender_id'] or 'unknown'
        sender_name = row['sender_name'] or sender

        # Apply sender filter
        if args.sender and args.sender.lower() not in sender.lower() and args.sender.lower() not in sender_name.lower():
            continue

        # Apply service keyword filter (if provided) against message text
        if args.service and not re.search(re.escape(args.service), text, re.IGNORECASE):
            # Still include if text has any OTP keyword
            if not OTP_KEYWORDS.search(text):
                continue

        otp_candidates = extract_otp(text)

        # Only include messages that either have OTP candidates or contain OTP keywords
        if not otp_candidates and not OTP_KEYWORDS.search(text):
            continue

        results.append({
            'date': msg_dt.astimezone().strftime('%Y-%m-%d %H:%M:%S %Z'),
            'date_utc': msg_dt.isoformat(),
            'sender': sender,
            'sender_name': sender_name,
            'text': text,
            'otp_candidates': otp_candidates,
        })

    print(json.dumps(results, ensure_ascii=False, indent=2))

if __name__ == '__main__':
    main()
```

Use the Write tool to save the exact content above to `/tmp/_sms_otp_extract.py`.

---

## STEP 4 — RUN THE EXTRACTOR

Build the command from the parsed input (Step 0):

```bash
python3 /tmp/_sms_otp_extract.py \
  --minutes <LOOKBACK_MINUTES> \
  [--sender "<SENDER_HINT>" if provided] \
  [--service "<SERVICE_HINT>" if provided]
```

Examples:
```bash
# Default — last 30 minutes, all senders
python3 /tmp/_sms_otp_extract.py --minutes 30

# Last 10 minutes from a sender matching "+972"
python3 /tmp/_sms_otp_extract.py --minutes 10 --sender "+972"

# Last 60 minutes, any Misim-related SMS
python3 /tmp/_sms_otp_extract.py --minutes 60 --service misim
```

Run the command. The output is a JSON array.

---

## STEP 5 — PARSE AND RETURN RESULTS

### If `{"error": ...}` is returned:

| Error contains | Action |
|---|---|
| `no such table` | The DB schema is unexpected. Tell the user their macOS Messages database schema may be different (uncommon). |
| `unable to open` / `permission` | Full Disk Access was lost — repeat Step 1 guidance. |
| Any other error | Show the error and stop. |

### If empty array `[]` is returned:

Tell the user:
> "No 2FA SMS messages found in the last [N] minutes. Possible reasons:
> - The OTP hasn't arrived yet — wait a few seconds and re-run.
> - Messages.app is not set up to relay SMS from your iPhone (check Settings → Messages → Text Message Forwarding on iOS).
> - The code was sent more than [N] minutes ago — re-run with `--minutes 60`.
> - The sender is filtered out — try without the sender hint."

### If results are found:

1. **Pick the best candidate** — use the first `otp_candidates` entry from the most recent message that has candidates. This is the "primary OTP".

2. **If only one result with one candidate** — return it immediately without asking:
   ```
   === SMS_OTP START ===
   code:    <6-digit-code>
   sender:  <sender>
   message: <full message text>
   date:    <timestamp>
   === SMS_OTP END ===
   ```

3. **If multiple results or multiple candidates** — show a numbered list and ask the user to confirm:

   ```
   Found 2FA messages in the last [N] minutes:

   [1] 14:23:05 | from: +97250-XXX-XXXX
       "קוד האימות שלך הוא: 482951"
       → OTP candidate: 482951  ✓ (6-digit)

   [2] 14:19:40 | from: +97250-YYY-YYYY
       "Your one-time code: 3847"
       → OTP candidate: 3847  (4-digit)

   Which code should I use? [1/2/other]
   ```

4. **If calling skill requested a specific service** (e.g., `service=misim`) and one result clearly matches, return it without prompting.

---

## STEP 6 — OUTPUT THE CODE

After the code is confirmed (or auto-selected), output the structured block:

```
=== SMS_OTP START ===
code:    482951
sender:  +97250XXXXXXX
message: קוד האימות שלך הוא: 482951. תקף ל-5 דקות.
date:    2025-04-16 14:23:05 IDT
=== SMS_OTP END ===
```

If called by another skill (e.g., `login`, `idf-service`, `hapoalim-import`, `miluim-import`), return only this block so the calling skill can extract and use the code directly.

---

## STEP 7 — CLEANUP

```bash
rm -f /tmp/_sms_otp_extract.py /tmp/_messages_otp.db /tmp/_messages_otp.db-wal /tmp/_messages_otp.db-shm 2>/dev/null; echo "cleaned"
```

---

## INTEGRATION WITH OTHER SKILLS

This skill is called inline by:

| Calling skill | When to call | Context |
|---|---|---|
| **`login`** | After triggering OTP on Misim portal | Pass `--service misim --minutes 10` |
| **`idf-service`** | After triggering MyIDF OTP | Pass `--service idf --minutes 10` |
| **`hapoalim-import`** | After triggering Bank Hapoalim OTP | Pass `--service hapoalim --minutes 10` |
| **`miluim-import`** | After triggering Miluim portal OTP | Pass `--service idf --minutes 10` |
| **`form106-import`** | After triggering Misim OTP | Pass `--service misim --minutes 10` |

When invoked by another skill, do not ask the user to confirm the code unless ambiguous — just return the `=== SMS_OTP START ===` block directly.

### How to invoke from another skill

Add this note to the OTP step of any skill that currently asks the user to type their OTP:

```
Instead of asking the user to type their OTP, run the `sms-otp` skill with:
  Skill(sms-otp) --service <name> --minutes 10
Extract the `code:` field from the SMS_OTP block and type it automatically with
  mcp__playwright__browser_type into the OTP input field.
If sms-otp returns no result, fall back to asking the user to type the code manually.
```

---

## PREREQUISITES FOR SMS RELAY (iOS → Mac)

For SMS (non-iMessage) texts to appear on Mac, iPhone SMS relay must be enabled:

1. iPhone → **Settings** → **Messages** → **Text Message Forwarding**
2. Enable forwarding to this Mac.
3. Both devices must be signed into the same Apple ID.

iMessages (blue bubble) are always stored in `chat.db` regardless of relay settings.

---

## ERROR STATES

| Situation | Action |
|---|---|
| Full Disk Access denied | Guide user through System Settings (see Step 1) |
| Messages.app has DB locked | Ask user to quit Messages.app temporarily, then retry Step 2 |
| `chat.db` not found | Messages.app may never have been opened on this Mac, or iCloud sync moved it. Check `ls ~/Library/Messages/` |
| No SMS relay from iPhone | Explain Text Message Forwarding setup (see Prerequisites section) |
| OTP expired before extraction | Tell user codes are typically valid 5 minutes — ask them to request a new OTP and re-run |
| Multiple unrelated codes found | Show all and ask user to confirm which one matches |

---

## LANGUAGE RULES

- Hebrew users: label outputs in Hebrew (שולח, קוד, הודעה, תאריך).
- English users: use English labels.
- Mirror the user's language — never mix mid-response.
