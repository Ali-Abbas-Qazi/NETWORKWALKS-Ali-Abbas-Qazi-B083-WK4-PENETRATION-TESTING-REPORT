<h1 align="center">Mediroza General Hospital — Black-Box Penetration Test</h1>

**Target:** https://medirozahospital.com
**Type:** Full black-box penetration test, 5 days
**Author:** Ali Abbas Qazi

> Conducted in a controlled, authorized training environment with written permission from the client. None of the techniques below should be run against a system you don't have explicit authorization to test.

📄 **This README is a visual walkthrough.** For the full write-up — detailed methodology, CWE references, risk ratings, and remediation steps — see the complete report: [`Mediroza_Pentest_Report_W4.docx`](./Mediroza_Pentest_Report_W4.docx).

## Overview

The patient portal's login form gave away which usernames existed through its error messages, and a plain `admin'--` SQL injection payload bypassed authentication outright — no valid password needed. That led to three password-protected patient PDFs, which were cracked by extracting their hashes and running them through a dictionary attack. A follow-up Nikto scan then turned up an unauthenticated database backup sitting in an indexed, unprotected directory, exposing staff salaries, national ID numbers, and shareholder equity data.

## Tools

- **Nikto v2.6.0** — web server misconfiguration scan
- **Manual SQL injection** — login form testing
- **OnlineHashCrack.com** — PDF hash extraction (pdf2john-based)
- **Networkwalks Password Cracker** — dictionary attack, built-in wordlist + JTR `password.txt`
- **Kali Linux 2026.2** (VirtualBox)

---

## 1. Username Enumeration + SQL Injection Bypass

The login form behaved differently depending on whether a username existed, which alone is enough to let someone build a list of valid accounts:

![Username not found](1.%20Username%20Not%20found.png)
*1. Username Not found.png*

![Username found, wrong password](2.%20Username%20Found%2C%20Wrong%20Password.png)
*2. Username Found, Wrong Password.png*

The real problem was underneath: the username field went straight into the SQL query unsanitized. Typing `admin'--` closed the string early and commented out the password check entirely.

![SQL injection payload](3.%20SQLInjection%20on%20login%20page.png)
*3. SQLInjection on login page.png*

That logged me in as admin with zero valid credentials and surfaced three encrypted patient lab reports:

![Reports found](4.%20Reports%20Found.png)
*4. Reports Found.png*

## 2. Cracking the Patient Reports

Each PDF's `$pdf$` hash was extracted and saved to `hash1.txt`, `hash2.txt`, `hash3.txt`:

![Hashes extracted](5.%20Find%20hashes%20for%20all%20PDF%20files.png)
*5. Find hashes for all PDF files.png*

Report 1 and Report 2 fell to the cracker's built-in 100-word list almost immediately:

![hash1 cracked](6a.%20hash1%20cracked.png)
*6a. hash1 cracked.png*

![Patient report 1 decrypted](6a.%20Patient%20Report%201.png)
*6a. Patient Report 1.png*

![hash2 cracked](6b.%20hash2%20cracked.png)
*6b. hash2 cracked.png*

![Patient report 2 decrypted](6b.%20Patient%20Report%202.png)
*6b. Patient Report 2.png*

Report 3's password wasn't in the built-in list, so I uploaded JTR's default `password.txt` instead — that cracked it:

![hash3 cracked](6c.%20hash3%20cracked.png)
*6c. hash3 cracked.png*

![Patient report 3 decrypted](6c.%20Patient%20Report%203.png)
*6c. Patient Report 3.png*

All three passwords — a number sequence, a dictionary word, and a short keyboard string — show the encryption wasn't doing much once the hash was out.

## 3. Unauthenticated Database Backup

Running `nikto -h https://medirozahospital.com/` turned up directory indexing on three paths, with `robots.txt` disallowing (and so advertising) all of them:

![Directories found](7.%20Found%20directories.png)
*7. Found directories.png*

`/old/` was the one that mattered — it served a raw `.sql` backup to anyone who asked, no login required:

![Unprotected backup found](8.%20Unprotected%20Backup%20data%20found.png)
*8. Unprotected Backup data found.png*

![Raw backup contents](8.%20Raw%20Unprotected%20BackUp%20Data.png)
*8. Raw Unprotected BackUp Data.png*

The backup (`mediroza_db_backup_2019.sql`, full dump in [`9. Raw Unprotected BackUp Data.txt`](9.%20Raw%20Unprotected%20BackUp%20Data.txt)) held two complete plaintext tables: 30 staff records and 10 shareholder records. The breakdown, risk ratings, and fix recommendations for this and every other finding are in the full report linked at the top.

---

## Repo Contents

- `*.png` — screenshots for each step above
- `hash1.txt` / `hash2.txt` / `hash3.txt` / `hashes.txt` — extracted PDF hashes
- `9. Raw Unprotected BackUp Data.txt` — full SQL backup dump recovered from `/old/`
- `Mediroza_Pentest_Report_W4.docx` — full report with findings, risk ratings, and remediation
