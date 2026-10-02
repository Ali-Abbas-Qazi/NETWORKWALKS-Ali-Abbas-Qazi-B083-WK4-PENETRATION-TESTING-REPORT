# Mediroza General Hospital — Black-Box Penetration Test

**Target:** https://medirozahospital.com
**Type:** Full black-box penetration test, 5 days
**Author:** Ali Abbas Qazi

> Conducted in a controlled, authorized training environment with written permission from the client. None of the techniques below should be run against a system you don't have explicit authorization to test.

## Summary

The patient portal's login form gave away which usernames existed through its error messages, and a plain `admin'--` SQL injection payload bypassed authentication outright — no valid password needed. That landed on the "My lab reports" page, which held three password-protected patient PDFs. I pulled each PDF's hash with a pdf2john-style extractor, cracked two of the three instantly against a 100-word dictionary (`123456`, `password`), and cracked the third (`!@#$%^&`) after switching to John the Ripper's default `password.txt`. A follow-up Nikto scan then turned up directory indexing on `/staff/`, `/patient/`, and `/old/` — all three listed in `robots.txt`, which pointed right at them — and `/old/` was quietly serving an unauthenticated SQL backup with plaintext staff salaries, national ID numbers, and shareholder equity data.

## Tools

- **Nikto v2.6.0** — web server misconfiguration scan
- **Manual SQL injection** — login form testing
- **OnlineHashCrack.com** — PDF hash extraction (pdf2john-based)
- **Networkwalks Password Cracker** — dictionary attack, built-in wordlist + JTR `password.txt`
- **Kali Linux 2026.2** (VirtualBox)

---

## 1. Username Enumeration + SQL Injection Bypass

The login form behaved differently depending on whether a username existed, which is enough on its own to let someone build a list of valid accounts:

![Username not found](1.%20Username%20Not%20found.png)
![Username found, wrong password](2.%20Username%20Found%2C%20Wrong%20Password.png)

The real problem was underneath: the username field went straight into the SQL query unsanitized. Typing `admin'--` closed the string early and commented out the password check entirely.

![SQL injection payload](3.%20SQLInjection%20on%20login%20page.png)

That logged me in as admin with zero valid credentials and surfaced three encrypted patient lab reports:

![Reports found](4.%20Reports%20Found.png)

## 2. Cracking the Patient Reports

Each PDF's `$pdf$` hash was extracted and saved to `hash1.txt`, `hash2.txt`, `hash3.txt`:

![Hashes extracted](5.%20Find%20hashes%20for%20all%20PDF%20files.png)

Report 1 and Report 2 fell to the cracker's built-in 100-word list almost immediately:

![hash1 cracked](6a.%20hash1%20cracked.png)
![Patient report 1 decrypted](6a.%20Patient%20Report%201.png)
![hash2 cracked](6b.%20hash2%20cracked.png)
![Patient report 2 decrypted](6b.%20Patient%20Report%202.png)

Report 3's password wasn't in the built-in list, so I uploaded JTR's default `password.txt` instead — that cracked it:

![hash3 cracked](6c.%20hash3%20cracked.png)
![Patient report 3 decrypted](6c.%20Patient%20Report%203.png)

All three passwords — a number sequence, a dictionary word, and a short keyboard string — show the encryption wasn't really doing much once the hash was out.

## 3. Unauthenticated Database Backup

Running `nikto -h https://medirozahospital.com/` turned up directory indexing on three paths, with `robots.txt` disallowing (and so advertising) all of them:

![Directories found](7.%20Found%20directories.png)

`/old/` was the one that mattered — it served a raw `.sql` backup to anyone who asked, no login required:

![Unprotected backup found](8.%20Unprotected%20Backup%20data%20found.png)
![Raw backup contents](8.%20Raw%20Unprotected%20BackUp%20Data.png)

The backup (`mediroza_db_backup_2019.sql`, full dump in [`9. Raw Unprotected BackUp Data.txt`](9.%20Raw%20Unprotected%20BackUp%20Data.txt)) contained two complete plaintext tables — 30 staff records (name, role, salary, national ID, contact info) and 10 shareholder records (name, equity %, shares held). A couple of rows for context:

| Employee | Role | Monthly Salary (ZAR) |
|---|---|---|
| Dr. Johan van der Merwe | Medical Director | R 160,000 |
| Sarah Botha | Chief Financial Officer | R 152,000 |
| Linda Fourie | Receptionist | R 19,000 |

| Shareholder | Equity |
|---|---|
| Dr. Rajesh Naidoo | 18% |
| Cedar Health Holdings (Pty) Ltd | 15% |
| Reddy Family Trust | 11% |

---

## Findings

| # | Vulnerability | Risk |
|---|---|---|
| 1 | Username enumeration on login form | Medium |
| 2 | SQL injection → authentication bypass | **Critical** |
| 3 | Weak/dictionary PDF passwords | High |
| 4 | Directory indexing + exposed DB backup | **Critical** |

## Fix List

- Return one generic login error ("invalid username or password") instead of two
- Parameterize every database query — stop building SQL with string concatenation
- Enforce strong, random passphrases on any patient-document encryption; move to AES-256
- Turn off directory indexing server-wide, and stop listing sensitive paths in `robots.txt`
- Move backups off the public web root and encrypt them at rest

## Repo Contents

- `*.png` — screenshots for each step above
- `hash1.txt` / `hash2.txt` / `hash3.txt` / `hashes.txt` — extracted PDF hashes
- `9. Raw Unprotected BackUp Data.txt` — full SQL backup dump recovered from `/old/`
