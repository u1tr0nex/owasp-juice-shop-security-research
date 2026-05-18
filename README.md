# 🔐 OWASP Juice Shop – Injection, Access Control & Client-Side Attacks

## 📌 Overview

This repository documents hands-on testing performed against **OWASP Juice Shop**, focusing specifically on:

- Injection attacks
- Broken access control & authentication issues
- Client-side vulnerabilities

All tests were executed only against the **official OWASP Juice Shop instance**, which is intentionally vulnerable and designed for security training.

> **Researcher:** Abhimanyu Rawat  
> **Role:** Technical Research & Cyber Security Operations Intern – GreySentinel  
> **Target:** `https://juice-shop.herokuapp.com/#/`  
> **Date:** May 17–18, 2026  

---

## 🗂 Repository Structure

```text
.
├── README.md
├── juice_shop_vulnerability_assessment_report.docx
└── screenshots/
    ├── 01_sql_injection/
    ├── 02_xxe/
    ├── 03_ssti/
    ├── 04_auth_access_control/
    └── 05_client_side/
```

- `juice_shop_vulnerability_assessment_report.docx`  
  → Detailed Word report with findings and screenshot placeholders.  
- `screenshots/`  
  → Burp, browser, and terminal screenshots used as evidence.

---

## 🔴 Injection Attacks Performed

### 1. SQL Injection – Login Bypass (Login as Admin)

**Goal:** Log in as `admin@juice-sh.op` without knowing the password.

- **Location:** `/#/login`
- **Endpoint:** `POST /rest/user/login`
- **Payload used (Email field):**
  ```text
  ' OR 1=1 --
  ```
- **Result:**
  - Login succeeded as `admin@juice-sh.op` (user ID 1).
  - Admin panel at `/#/administration` became accessible.
- **Impact:** Full authentication bypass and admin session without valid credentials.

**Evidence (screenshots folder):**

- `01_sql_injection/`  
  - Burp request showing the SQLi payload  
  - UI showing admin logged in

---

### 2. SQL Injection – UNION-Based Data Extraction

**Goal:** Extract all user emails and hashed passwords.

- **Location:** `/rest/products/search?q=`
- **Vector:** Vulnerable `q` (search) parameter.
- **Payload (in `q` parameter):**
  ```text
  qwert')) UNION SELECT id,email,password,'4','5','6','7','8','9' FROM Users--
  ```
- **Result:**
  - Response contained a JSON array with all user records (emails + hashes).
- **Impact:** Full credential data exposure from the Users table.

**Evidence:**

- `01_sql_injection/`  
  - URL / request with UNION payload  
  - Response body showing user emails and hashed passwords

---

### 3. XXE Injection – Reading `/etc/passwd`

**Goal:** Demonstrate XML External Entity (XXE) vulnerability by reading a local file from the server.

- **Location (UI):** `/#/complain` (Complaint / Invoice upload)
- **Backend Endpoint:** `POST /file-upload`
- **Payload file:** `xxe.xml`
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE foo [
    <!ENTITY xxe SYSTEM "file:///etc/passwd">
  ]>
  <stockCheck>
    <productId>&xxe;</productId>
  </stockCheck>
  ```
- **Result:**
  - Server responded with `410 Gone` and an error message:
    - Text included the **contents of `/etc/passwd`**, such as:
      - `root:x:0:0:root:/root:/sbin/nologin`
      - `nobody:x:65534:65534:nobody:/nonexistent:/sbin/nologin`
      - `nonroot:x:65532:65532:nonroot:/home/nonroot:/sbin/nologin`
  - This also completed the **Deprecated Interface** challenge, proving the old XML upload interface was still reachable.

- **Impact:** Local file disclosure from the server via XXE.

**Evidence:**

- `02_xxe/`  
  - `xxe.xml` content screenshot  
  - Burp request to `/file-upload` with XML payload  
  - Response showing 410 error and embedded `/etc/passwd` data

---

### 4. Server-Side Template Injection (SSTI) – Not Vulnerable in Tested Field

**Goal:** Check if the profile username field is vulnerable to SSTI.

- **Location:** `/profile` (server-rendered profile page)
- **Test payloads in username field:**
  ```text
  #{6*6}
  #{7*7}
  #{9*9}
  ```
- **Observation:**
  - The username was stored and returned exactly as `#{9*9}`.
  - JWT token (`Set-Cookie: token=...`) included `"username":"#{9*9}"`.
  - No evaluation to `36`, `49`, or `81` occurred.

- **Conclusion:**  
  - This specific field **did not evaluate template expressions**; input was treated as data, not template code.  
  - SSTI was **not observed** on this input vector.

**Evidence:**

- `03_ssti/`  
  - Burp response showing `username":"#{9*9}"` in the JWT  
  - UI showing literal `#{9*9}` as username

---

## 🟠 Broken Access Control & Authentication

### 5. Brute Force – No Rate Limiting / Lockout

**Goal:** Check whether the login endpoint defends against brute-force attempts.

- **Location:** `/#/login`
- **Endpoint:** `POST /rest/user/login`
- **Test method:**
  - Used Burp Suite Intruder (Community Edition) to replay multiple login attempts against `admin@juice-sh.op` with different passwords.
  - Observed responses and server behavior.

- **Findings:**
  - No account lockout, no CAPTCHA, and no noticeable delay after multiple failed attempts.
  - Burp Intruder was throttled (Community Edition), but the application showed **no built-in rate limiting** or protections.

- **Impact:**  
  - Login endpoint is susceptible to brute-force attacks using external tools (Hydra, ffuf, etc.).

**Evidence:**

- `04_auth_access_control/`  
  - Burp Intruder/HTTP history showing repeated login attempts and responses

---

### 6. Admin Panel Access & Privilege Escalation

**Goal:** Show that SQL injection leads directly to admin-level functionality.

- **Step 1:** Login with SQL injection (see Section 1).
- **Step 2:** Navigate to:
  ```text
  https://juice-shop.herokuapp.com/#/administration
  ```
  - As admin: full admin panel loads (users, feedback, etc.).
  - As unauthenticated or normal user: access restricted.

- **API Evidence:** `GET /rest/user/whoami`
  ```json
  {
    "user": {
      "id": 1,
      "email": "admin@juice-sh.op",
      "lastLoginIp": "",
      "profileImage": "assets/public/images/uploads/defaultAdmin.png"
    }
  }
  ```

- **Impact:**
  - SQL injection → direct **vertical privilege escalation** to the built-in admin account.
  - Admin panel allows extensive control over application data.

**Evidence:**

- `04_auth_access_control/`  
  - Screenshot of `/#/administration` as admin  
  - Burp Repeater showing `GET /rest/user/whoami` with `email":"admin@juice-sh.op"`

---

## 🟡 Client-Side Vulnerabilities

### 7. DOM-Based XSS

**Goal:** Prove client-side XSS in the search functionality.

- **Location:** Homepage search bar (`/#/search`)
- **Payload (example):**
  ```html
  <iframe src="javascript:alert('XSS')">
  ```
- **Result:**
  - Browser executed a JavaScript alert when the payload was processed.
  - Confirms that untrusted input is inserted into the DOM unsafely.

**Evidence:**

- `05_client_side/`  
  - Search input with payload  
  - Alert popup screenshot

---

### 8. CORS Misconfiguration

**Goal:** Check if API responses are overly exposed to other origins.

- **Observation on multiple responses (e.g., `/rest/user/whoami`):**
  ```http
  Access-Control-Allow-Origin: *
  ```
- **Impact:**
  - Any external website can make cross-origin requests to Juice Shop APIs.
  - If a victim is logged in, a malicious page could read API responses (e.g., user email, ID).

---

## 📄 Full Report

All of the above is documented in detail in:

- **`juice_shop_vulnerability_assessment_report.docx`**  
  → Contains methodology, payloads, findings tables, and explicit screenshot placeholders (A1, A2, A4, etc.)

---

## ⚠️ Disclaimer

- Testing was performed solely against **OWASP Juice Shop**, an intentionally vulnerable application.
- No production systems were tested.
- This work is for **learning, internal review, and portfolio purposes** only.
