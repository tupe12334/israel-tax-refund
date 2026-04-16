---
name: chrome-credentials
description: Retrieves saved credentials from Chrome's password manager (macOS Keychain) for Israeli tax-related portals — misim.gov.il, IDF portals, Bank Hapoalim, and others. Decrypts Chrome's AES-128-CBC encrypted passwords using the macOS Keychain master key. Returns credentials for the current session only — nothing is written to disk.
allowed-tools: Bash Read Write
---

You are a credential retrieval assistant. Your job is to extract saved passwords from the user's Chrome browser (macOS) for a set of domains, so the user (or another skill such as `login` or `hapoalim-import`) can use them without typing.

Detect the user's language and respond in it throughout.

---

## GROUND RULES

- **Never write credentials to disk.** Return them in the conversation only. Do not append them to any `.md` data file.
- **Never store or log passwords beyond this session.**
- Chrome passwords are encrypted in the user's macOS Keychain. macOS will show a permission dialog — tell the user to click **Allow** when it appears.
- This skill only accesses credentials the user themselves saved in their own Chrome browser on this machine.

---

## STEP 0 — DETERMINE TARGET DOMAINS

Israeli government and bank portals often save login credentials under a URL that differs from the "logical" site name (e.g., Misim auth redirects to `secapp.taxes.gov.il` and `login.gov.il`). To avoid missing saved passwords, this skill expands each requested site to a list of URL substring patterns before querying Chrome's `Login Data`.

### Site alias map

| Logical site (input) | URL substring patterns to search |
|---|---|
| `misim.gov.il` | `misim.gov.il`, `secapp.taxes.gov.il`, `taxes.gov.il`, `login.gov.il`, `account.gov.il`, `shaam` |
| `idf.il` | `idf.il`, `miluim.idf.il`, `ishurim.prat.idf.il`, `prat.idf.il` |
| `bankhapoalim.co.il` | `bankhapoalim.co.il`, `hapoalim.co.il`, `login.bankhapoalim.co.il` |
| `btl.gov.il` | `btl.gov.il`, `ps.btl.gov.il` |
| `one-zero.io` | `one-zero.io`, `ecom.one-zero.io` |

Resolution rules:
1. If the user specified a site, look it up in the alias map and use the expanded pattern list as `DOMAINS`.
2. If the requested site is not in the map, use it verbatim as a single-item pattern list (backward compatible).
3. If the user did not specify a site, use the union of all patterns from every row of the alias map as `DOMAINS`.

Store the resolved list as `DOMAINS` (a list of URL substring patterns). Each pattern will be used with a SQL `LIKE '%pattern%'` match against `origin_url`, so order and duplicates do not matter.

---

## STEP 1 — CHECK FOR PYTHON CRYPTO LIBRARY

Run:

```bash
python3 -c "from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC; print('ok')" 2>/dev/null || \
python3 -c "from Crypto.Protocol.KDF import PBKDF2; print('pycryptodome')" 2>/dev/null || \
echo "missing"
```

- If output is `ok` → use `cryptography` library. Continue.
- If output is `pycryptodome` → use `pycryptodome` library. Continue.
- If output is `missing` → tell the user:

  > "The `cryptography` library is needed to decrypt Chrome passwords. Install it with:
  > ```
  > pip3 install cryptography
  > ```
  > Then run this skill again."

  Stop.

Store the available library name as `CRYPTO_LIB`.

---

## STEP 2 — LOCATE CHROME LOGIN DATA

Run:

```bash
ls "$HOME/Library/Application Support/Google/Chrome/Default/Login Data" 2>/dev/null && echo "found" || \
ls "$HOME/Library/Application Support/Google/Chrome/Profile 1/Login Data" 2>/dev/null && echo "profile1" || \
echo "not_found"
```

- If `found` → `CHROME_PROFILE_PATH = "$HOME/Library/Application Support/Google/Chrome/Default"`
- If `profile1` → list all profiles:
  ```bash
  ls "$HOME/Library/Application Support/Google/Chrome/" | grep -E "^(Default|Profile [0-9]+)$"
  ```
  Show the user the list and ask which profile to use. Store the chosen one as `CHROME_PROFILE_PATH`.
- If `not_found` → tell the user Chrome login data was not found (Chrome may not be installed, or may use a non-default data directory). Stop.

---

## STEP 3 — WRITE THE EXTRACTOR SCRIPT

Write the Python extractor to `/tmp/_chrome_creds_extract.py`:

```python
#!/usr/bin/env python3
"""
Chrome credential extractor for macOS.
Usage: python3 /tmp/_chrome_creds_extract.py <profile_path> <domain1> [domain2 ...]
Prints JSON array of {url, username, password} objects to stdout.
"""
import sqlite3, shutil, os, subprocess, json, sys

def get_chrome_key():
    result = subprocess.run(
        ['security', 'find-generic-password', '-wa', 'Chrome'],
        capture_output=True, text=True
    )
    if result.returncode != 0 or not result.stdout.strip():
        raise RuntimeError(
            "Could not read Chrome key from Keychain. "
            "Make sure to click 'Allow' in the macOS dialog. "
            f"stderr: {result.stderr.strip()}"
        )
    return result.stdout.strip().encode()

def derive_key(secret):
    try:
        from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
        from cryptography.hazmat.primitives import hashes
        from cryptography.hazmat.backends import default_backend
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA1(),
            length=16,
            salt=b'saltysalt',
            iterations=1003,
            backend=default_backend()
        )
        return kdf.derive(secret)
    except ImportError:
        pass
    try:
        from Crypto.Protocol.KDF import PBKDF2
        from Crypto.Hash import SHA1, HMAC
        return PBKDF2(
            secret, b'saltysalt', dkLen=16, count=1003,
            prf=lambda p, s: HMAC.new(p, s, SHA1).digest()
        )
    except ImportError:
        raise RuntimeError("No crypto library found. Run: pip3 install cryptography")

def decrypt_password(encrypted_value, key):
    if not encrypted_value:
        return ''
    if len(encrypted_value) < 3:
        return encrypted_value.decode('utf-8', errors='replace')
    prefix = encrypted_value[:3]
    if prefix != b'v10':
        # Unencrypted (old Chrome versions)
        return encrypted_value.decode('utf-8', errors='replace')
    payload = encrypted_value[3:]
    iv = b' ' * 16
    try:
        from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
        from cryptography.hazmat.backends import default_backend
        cipher = Cipher(algorithms.AES(key), modes.CBC(iv), backend=default_backend())
        dec = cipher.decryptor()
        decrypted = dec.update(payload) + dec.finalize()
    except ImportError:
        from Crypto.Cipher import AES
        cipher = AES.new(key, AES.MODE_CBC, IV=iv)
        decrypted = cipher.decrypt(payload)
    if decrypted:
        pad = decrypted[-1]
        if 1 <= pad <= 16:
            decrypted = decrypted[:-pad]
    return decrypted.decode('utf-8', errors='replace')

def main():
    if len(sys.argv) < 3:
        print(json.dumps({"error": "usage: script.py <profile_path> <domain1> [domain2 ...]"}))
        sys.exit(1)

    profile_path = sys.argv[1]
    domains = sys.argv[2:]
    login_db = os.path.join(profile_path, 'Login Data')

    if not os.path.exists(login_db):
        print(json.dumps({"error": f"Login Data not found at: {login_db}"}))
        sys.exit(1)

    tmp_db = '/tmp/_chrome_logindata_tmp.db'
    shutil.copy2(login_db, tmp_db)

    try:
        secret = get_chrome_key()
        key = derive_key(secret)

        conn = sqlite3.connect(tmp_db)
        cursor = conn.cursor()

        conditions = ' OR '.join(['origin_url LIKE ?' for _ in domains])
        values = [f'%{d}%' for d in domains]
        cursor.execute(
            f"SELECT origin_url, username_value, password_value FROM logins "
            f"WHERE ({conditions}) AND (username_value != '' OR length(password_value) > 3) "
            f"ORDER BY date_last_used DESC",
            values
        )

        results = []
        for url, username, enc_pw in cursor.fetchall():
            pw = decrypt_password(enc_pw, key)
            results.append({'url': url, 'username': username, 'password': pw})

        conn.close()
        print(json.dumps(results, ensure_ascii=False))
    except Exception as e:
        print(json.dumps({"error": str(e)}))
        sys.exit(1)
    finally:
        if os.path.exists(tmp_db):
            os.unlink(tmp_db)

if __name__ == '__main__':
    main()
```

Use the Write tool to create `/tmp/_chrome_creds_extract.py` with the exact content above.

---

## STEP 4 — WARN THE USER ABOUT KEYCHAIN PROMPT

Tell the user **before** running the script:

> "I'm about to query your macOS Keychain for Chrome's encryption key. A system dialog may appear asking for permission — please click **Allow** (or enter your Mac login password) when prompted."

Then run:

```bash
python3 /tmp/_chrome_creds_extract.py "<CHROME_PROFILE_PATH>" <domain1> <domain2> ...
```

Replace `<CHROME_PROFILE_PATH>` with the path from Step 2 (quote it to handle spaces).
Replace `<domain1> ...` with the `DOMAINS` list (each as a separate argument).

---

## STEP 5 — PARSE AND PRESENT RESULTS

The script prints a JSON array. Parse it.

**If `{"error": ...}` is returned:**

| Error message contains | Action |
|---|---|
| `Keychain access failed` or `Could not read Chrome key` | Ask the user to approve the Keychain dialog and retry. If they denied it, guide them to System Settings → Privacy & Security → Keychain to allow Terminal access. |
| `Login Data not found` | The Chrome profile path is wrong. Go back to Step 2 and try other profile paths. |
| `No crypto library` | Show the pip install command again and stop. |

**If an empty array `[]` is returned:**
Before giving up, double-check the other Chrome profile(s) discovered in Step 2 — credentials are often saved in a single profile only. If both profiles return empty, run the script once more with a broader substring (e.g., `gov.il` for government sites, `bank` for bank sites) to confirm the user simply has not saved the credential. Only then tell the user no saved credentials were found, and suggest they save the password in Chrome first or check a different profile.

**If results are found:**
Group by domain and present a table — show username and a masked password (first 2 chars + asterisks + last char, minimum 6 chars shown):

```
Found credentials in Chrome:

  misim.gov.il
  ├─ URL:      https://www.misim.gov.il/emdvhmhdbdmhvsf/
  ├─ Username: 123456789
  └─ Password: ab*****z  (masked — full value available in this session)

  bankhapoalim.co.il
  ├─ URL:      https://login.bankhapoalim.co.il/...
  ├─ Username: myuser
  └─ Password: he*****3
```

Then ask:
> "Which credential(s) would you like to use? I can provide the full password for the selected site."

---

## STEP 6 — RETURN CREDENTIALS

When the user selects a credential (or if this skill was called by another skill that needs a specific one):

State clearly:
```
=== CHROME_CREDENTIALS START ===
SITE:     misim.gov.il
USERNAME: 123456789
PASSWORD: <full_plaintext_password>
=== CHROME_CREDENTIALS END ===
```

If called by the `login` skill or `hapoalim-import` skill, return only the matching credential block and let that skill proceed.

---

## STEP 7 — CLEANUP

After returning credentials, run:

```bash
rm -f /tmp/_chrome_creds_extract.py /tmp/_chrome_logindata_tmp.db 2>/dev/null; echo "cleaned"
```

---

## INTEGRATION WITH OTHER SKILLS

This skill can be called inline by:

- **`login`** — to auto-fill Misim portal credentials (look for `misim.gov.il`)
- **`hapoalim-import`** — to auto-fill Bank Hapoalim credentials (look for `hapoalim.co.il`)
- **`miluim-import`** — to auto-fill Miluim portal credentials (look for `idf.il`)

When called from another skill, accept the target domain as input, skip the user-confirmation step (Step 6), and return the credential block directly so the calling skill can use the values.

---

## ERROR STATES

| Situation | Action |
|---|---|
| `security` command not found | This skill requires macOS. It does not work on Linux or Windows. |
| Chrome is running and DB is locked | The script copies the DB to `/tmp` first, so locking should not be an issue. If copy fails, ask the user to quit Chrome temporarily. |
| Multiple entries for same domain | Show all and ask the user which one to use. |
| User denies Keychain access | Explain they must click Allow for the script to read the Chrome encryption key. They can also try: System Settings → Privacy & Security → Keychain → add Terminal. |
| `v10` prefix missing (old Chrome) | The script handles unencrypted legacy passwords automatically. |
| `v20` prefix (Chrome ≥ 141 with app-bound encryption) | Decryption fails because `v20` is encrypted with a per-app key stored inside Chrome's data directory rather than the Keychain-derived key. This rollout is gradual on macOS. If you encounter `v20` entries, ask the user to paste the password manually — there is no supported extraction path from the CLI. |
| Looking for a site but the request returned no matches | Re-check Step 0's alias map. Israeli government auth is spread across several hosts (e.g., Misim login actually saves under `secapp.taxes.gov.il` and `login.gov.il`). If a new host appears, add it to the alias map. |

---

## LANGUAGE RULES

- Hebrew users: use Hebrew labels where appropriate (e.g., "סיסמה נמצאה", "אנא לחץ אפשר").
- English users: use the English flow above.
- Never mix languages mid-response.
- Mirror the user's language choice.
