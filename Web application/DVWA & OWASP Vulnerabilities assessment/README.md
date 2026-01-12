# Web Application Vulnerability Assessment: DVWA & OWASP ZAP

## Project Overview

This project documents a vulnerability assessment conducted on a deliberately vulnerable web application (**DVWA** hosted on **Metasploitable 2**). The goal was to identify, analyze, and manually exploit common OWASP Top 10 vulnerabilities using both automated scanners and manual techniques.

## Lab Environment

* **Attacker Machine:** Kali Linux (Rolling 2024)
* **Target Machine:** Metasploitable 2 (Hosting DVWA)
* **Tools Used:** * **OWASP ZAP:** For automated vulnerability scanning and discovery.
  * **Firefox Browser:** For manual exploitation and verification.
  * **DVWA:** The target application (Security Level: Low).

## Methodology & Findings

### 1. Automated Analysis (OWASP ZAP)

I initiated the assessment with an automated scan using **OWASP ZAP** against the target IP.

* **Process:** Configured ZAP to spider and scan `http://[Target_IP]/dvwa/`.
* **Result:** ZAP identified multiple high-priority alerts, including SQL Injection, Cross-Site Scripting (Reflected), and Directory Traversal.
* **Verification:** I used these alerts as a roadmap for manual exploitation.

<img width="958" height="455" alt="alerts" src="https://github.com/user-attachments/assets/c4ea66f2-57d8-4214-8e02-eca1abd78456" />


---

### 2. Manual Exploitation

#### A. Command Injection (Remote Code Execution)

This was the most critical finding. The application failed to sanitize user input in a "Ping" utility, allowing for the execution of arbitrary system commands.

* **Payload Used:** `8.8.8.8; cat /etc/passwd`
* **Observation:** The application executed the ping, followed immediately by the `cat` command.
* **Impact:** I was able to retrieve the sensitive `/etc/passwd` file, revealing the list of all system users (e.g., `root`, `msfadmin`, `service` accounts). This confirms **Remote Code Execution (RCE)**.

<img width="958" height="461" alt="Command ls execution" src="https://github.com/user-attachments/assets/fe42c224-f952-41b0-8795-2b07728e718f" />


<img width="960" height="463" alt="Command passwd execution" src="https://github.com/user-attachments/assets/096cd563-7f09-43d1-b3e0-b7f9297525fb" />



#### B. Reflected Cross-Site Scripting (XSS)

I identified an input field that reflected user data back to the browser without validation.

* **Payload Used:** `<script>alert('Hacked')</script>`
* **Observation:** Upon submission, the browser executed the JavaScript immediately, displaying a popup alert.
* **Impact:** An attacker could use this to steal session cookies, redirect users to phishing sites, or perform actions on behalf of the victim.

<img width="959" height="460" alt="XSS Script" src="https://github.com/user-attachments/assets/2383437c-5e68-4b1d-b3fc-772e1024a5f9" />

<img width="960" height="431" alt="XSS Popup" src="https://github.com/user-attachments/assets/2f305dc6-9a9e-443c-bc50-9abfd32e9da2" />


#### C. SQL Injection (SQLi)

The "User ID" lookup field was found to be vulnerable to SQL injection, allowing direct interaction with the backend database.

* **Test 1 (Enumeration):** Inputting ID `1`, `2`, etc., enumerated valid user accounts (`admin`, `GordonB`).
* **Test 2 (Bypass):** Inputting `' OR '1'='1' #` forced the database to return all records by making the query logically true.
* **Impact:** Full database disclosure and potential authentication bypass.\

<img width="958" height="460" alt="SQL injection" src="https://github.com/user-attachments/assets/200c9381-c178-4ebd-b53f-44183d738a07" />

## Remediation Recommendations

To secure this application, the following code changes are required:

1. **Input Sanitization:** Implement strict whitelisting for all user inputs.
2. **Parameterized Queries:** Use Prepared Statements for all database queries to prevent SQLi.
3. **Output Encoding:** Encode special characters (like `<` and `>`) before rendering them in the browser to prevent XSS.
4. **Disable Shell Execution:** Avoid using functions like `exec()` or `system()` in PHP. If necessary, ensure inputs are strictly validated.

## Conclusion

This assessment demonstrated how easily a web application can be compromised if basic security practices are ignored. By combining automated scanning (ZAP) with manual verification, I successfully escalated access from a web visitor to an attacker with system-level visibility.
