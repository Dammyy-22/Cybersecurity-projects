# Network Security Fundamentals: Traffic Analysis & Hardening

## Project Overview
This project demonstrates the practical differences between secure (HTTPS) and insecure (HTTP) network traffic, alongside basic endpoint hardening techniques using Windows Firewall. The goal was to visualize data leakage risks and implement access controls.

## Lab Environment
* **Hypervisor:** VMware Workstation
* **OS:** Windows 10
* **Analysis Tool:** Wireshark
* **Security Tool:** Windows Defender Firewall with Advanced Security

## Objectives
1.  **Traffic Analysis:** Capture and compare HTTP vs. HTTPS packets to verify encryption efficacy.
2.  **Endpoint Hardening:** Configure Inbound Rules to block unauthorized ports (Port 8080).
3.  **Protocol Identification:** Analyze DNS queries and ICMP (Ping) traffic.

## Methodology & Execution

### 1. Firewall Configuration (Hardening)
* **Action:** Configured Windows Defender Firewall with Advanced Security.
* **Rule Created:** Inbound Block Rule for **TCP Port 8080**.
* **Result:** Validated that the system explicitly denies connection attempts on non-standard web ports, reducing the attack surface. <br/>


<img width="859" height="419" alt="block inbound rule" src="https://github.com/user-attachments/assets/e6c81510-a81f-47d3-b072-e4fc27150211" /> <br/>



### 2. Network Traffic Capture
Used Wireshark to capture traffic during the following activities:
* **HTTP Request:** Browsing to `http://info.cern.ch` (The world's first website).
* **HTTPS Request:** Browsing to `www.google.com` (TLS 1.3).
* **Connectivity Check:** ICMP Ping to `8.8.8.8`. <br/> <br/> <br/>


<img width="857" height="418" alt="network capture" src="https://github.com/user-attachments/assets/79fc0de2-ab03-47ae-8838-2299dc1d9fd7" /> <br/>




<img width="859" height="419" alt="Google connection" src="https://github.com/user-attachments/assets/64fdb796-8188-4bb4-abc1-4f02f1802692" /> <br/>



## Key Findings

### The HTTP Risk (Cleartext)
* **Observation:** Filtering for `http` revealed the full conversation.
* **Evidence:** In the "Line-based text data" field, the HTML code and content of the CERN website were fully visible.
* **Security Implication:** Any attacker on the same network (Man-in-the-Middle) can intercept and read this data (passwords, session tokens) without decryption tools.

### The HTTPS Protection (Encryption)
* **Observation:** Filtering for `tls` showed the connection to Google.
* **Evidence:** The "Application Data" was obfuscated. No readable text or HTML was visible.
* **Security Implication:** Even if captured, the data is useless to an attacker without the private key.

## Concept Definitions
During this lab, I researched and analyzed the following threats:
* **Worms:** Self-replicating malware that spreads laterally across networks automatically.
* **Trojans:** Malicious code disguised as legitimate software to establish backdoors.
* **Phishing:** Social engineering attacks designed to steal credentials.

## Conclusion
This lab reinforced that **encryption is non-negotiable**. While firewalls provide a necessary perimeter defense, they cannot protect data in transit. Only protocols like TLS 1.3 ensure data confidentiality.
