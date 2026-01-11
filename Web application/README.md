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



> ![](C:\Users\hp\Documents\Projects\CyberSecurity\Cybersecurity-projects\Web%20application\screenshots\alerts.png)

---

### 2. Manual Exploitation

#### A. Command Injection (Remote Code Execution)

This was the most critical finding. The application failed to sanitize user input in a "Ping" utility, allowing for the execution of arbitrary system commands.

* **Payload Used:** `8.8.8.8; cat /etc/passwd`
* **Observation:** The application executed the ping, followed immediately by the `cat` command.
* **Impact:** I was able to retrieve the sensitive `/etc/passwd` file, revealing the list of all system users (e.g., `root`, `msfadmin`, `service` accounts). This confirms **Remote Code Execution (RCE)**.
  
  

> ![](C:\Users\hp\Documents\Projects\CyberSecurity\Cybersecurity-projects\Web%20application\screenshots\Command%20ls%20execution.png)



> ![](C:\Users\hp\Documents\Projects\CyberSecurity\Cybersecurity-projects\Web%20application\screenshots\Command%20passwd%20execution.png)



#### B. Reflected Cross-Site Scripting (XSS)

I identified an input field that reflected user data back to the browser without validation.

* **Payload Used:** `<script>alert('Hacked')</script>`
* **Observation:** Upon submission, the browser executed the JavaScript immediately, displaying a popup alert.
* **Impact:** An attacker could use this to steal session cookies, redirect users to phishing sites, or perform actions on behalf of the victim.

> ![](C:\Users\hp\Documents\Projects\CyberSecurity\Cybersecurity-projects\Web%20application\screenshots\XSS%20Script.png)



> ![](C:\Users\hp\Documents\Projects\CyberSecurity\Cybersecurity-projects\Web%20application\screenshots\XSS%20Popup.png)



#### C. SQL Injection (SQLi)

The "User ID" lookup field was found to be vulnerable to SQL injection, allowing direct interaction with the backend database.

* **Test 1 (Enumeration):** Inputting ID `1`, `2`, etc., enumerated valid user accounts (`admin`, `GordonB`).
* **Test 2 (Bypass):** Inputting `' OR '1'='1' #` forced the database to return all records by making the query logically true.
* **Impact:** Full database disclosure and potential authentication bypass.\



> ![](C:\Users\hp\Documents\Projects\CyberSecurity\Cybersecurity-projects\Web%20application\screenshots\SQL%20injection.png)



## Remediation Recommendations

To secure this application, the following code changes are required:

1. **Input Sanitization:** Implement strict whitelisting for all user inputs.
2. **Parameterized Queries:** Use Prepared Statements for all database queries to prevent SQLi.
3. **Output Encoding:** Encode special characters (like `<` and `>`) before rendering them in the browser to prevent XSS.
4. **Disable Shell Execution:** Avoid using functions like `exec()` or `system()` in PHP. If necessary, ensure inputs are strictly validated.

## Conclusion

This assessment demonstrated how easily a web application can be compromised if basic security practices are ignored. By combining automated scanning (ZAP) with manual verification, I successfully escalated access from a web visitor to an attacker with system-level visibility.
