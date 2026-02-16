# Wazuh SIEM: Single-Node Docker Deployment & Troubleshooting Guide

This repository documents a **real-world Wazuh SIEM deployment**. It deliberately skips polished, marketing-grade walkthroughs and instead records what actually breaks, why it breaks, and the exact steps required to recover when it does.

The focus is on:

* Docker-based Wazuh Manager deployment
* Windows Agent enrollment
* File Integrity Monitoring (FIM)
* Authentication, agent, and configuration failures you are likely to hit in practice

---

## 🎯 Objectives

* Deploy a **single-node Wazuh Manager** using Docker Compose on an Ubuntu VM (`192.168.100.3`).
* Enroll a **Windows Agent** (`192.168.100.4`) to collect system and security logs.
* Configure **File Integrity Monitoring (FIM)** to detect real-time changes in sensitive directories.

---

## 🏗️ Phase 1: Docker Deployment (Manager + Indexer + Dashboard)

The deployment starts with the official Wazuh Docker repository. On paper, this step looks trivial.

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
docker compose up -d
```

Containers came up cleanly. No crashes. No obvious errors.

And yet the system was already broken.

---

## 🛑 Challenge 1: The “Uninitialized” Authentication Failure

The Wazuh Dashboard loaded, but **default admin credentials failed repeatedly**. No typo. No browser issue. Hard failure.

### Discovery

Checking the indexer logs revealed the real problem:

```bash
docker compose logs wazuh.indexer
```

Recurring error:

```
ERROR BackendRegistry Not yet initialized (you may need to run securityadmin)
```

**What this actually means:**
The OpenSearch security backend failed during its *first initialization*. Once that happens, the indexer never recovers on its own.

---

### ❌ What Does *Not* Work

* Installing tools inside the container
* Running `securityadmin.sh` manually
* Editing files inside the container

Dockerized Wazuh containers are **immutable and minimal by design**. If the first boot fails, the data volumes are poisoned.

---

### ✅ The Only Reliable Fix: Nuclear Reset

```bash
docker compose down -v   # -v is mandatory
docker volume prune
docker compose up -d
```

This wipes the corrupted OpenSearch security indices and forces a clean bootstrap.

**Lesson learned:**
If the indexer security backend corrupts on first boot, **resetting volumes is faster and safer than attempting repairs**.

---

## 💻 Phase 2: Windows Agent Enrollment

With the Manager fully functional, the next step was enrolling a Windows endpoint using the MSI installer.

The agent was pointed to the Ubuntu host (`192.168.100.3`) during installation.

---

## 🛑 Challenge 2: The “Ghost Agent” Problem

Immediately after installation, the dashboard showed an **already-registered agent** in a disconnected state.

This was not the current machine.
It was a remnant from a **previous failed registration**.

Worse: the dashboard UI refused to delete it cleanly.

---

### Discovery

Agents are stored server-side. Failed enrollments leave artifacts behind, especially when reusing hostnames or IPs.

The only way out was direct CLI access.

---

### ✅ Solution: Manual Agent Cleanup

Identify the manager container:

```bash
docker ps | grep manager
```

List registered agents:

```bash
docker exec -it wazuh.manager /var/ossec/bin/manage_agents -l
```

Remove the ghost agent by ID:

```bash
docker exec -it wazuh.manager /var/ossec/bin/manage_agents -r 001
```

Once removed, the Windows agent enrolled normally.

---

## 🩹 Phase 3: The XML Corruption Nightmare (FIM Setup)

The goal was simple: monitor file changes in `C:\test`.

The result was catastrophic.

After editing `ossec.conf`, the agent stopped working entirely.

---

### The Error

The Windows Agent UI showed:

```
Unable to set OSSEC Server IP address
Internal error on the XML write
```

No line number. No syntax hint. Just failure.

---

### Discovery

Two separate issues caused this:

1. **Malformed XML**

   * A stray `</syscheck>` tag closed the block early
   * Remaining tags were left floating

2. **Wrong File Encoding**

   * The file was saved as UTF-16
   * Wazuh expects UTF-8

Wazuh’s XML parser is unforgiving. One misplaced tag or wrong encoding and the agent will not start.

---

## ✅ Final Solution: PowerShell Automation (No Manual Editing)

Manual Notepad edits are unreliable. Permissions, encoding, and syntax all fail silently.

The fix was to **force-write a clean XML configuration using PowerShell**.

```powershell
$config = @"
<ossec_config>
  <client>
    <server>
      <address>192.168.100.3</address>
      <port>1514</port>
    </server>
    <config-profile>windows, windows10</config-profile>
  </client>
  <syscheck>
    <disabled>no</disabled>
    <frequency>43200</frequency>
    <directories realtime="yes" check_all="yes">C:\test</directories>
    <directories realtime="yes" check_all="yes">C:\Users\Dammy\Documents\test</directories>
  </syscheck>
</ossec_config>
"@

Set-Content -Path "C:\Program Files (x86)\ossec-agent\ossec.conf" -Value $config -Encoding UTF8
Restart-Service -Name "Wazuh"
```

This guarantees:

* Valid XML structure
* Correct UTF-8 encoding
* Proper overwrite of a protected file

---

## 🔍 Phase 4: Verifying File Integrity Monitoring (FIM)

Once the agent restarted:

* Creating a file in `C:\test` triggered an alert within seconds
* Modifying the file generated an integrity change event
* Deletions were logged correctly

The Wazuh Dashboard under **Integrity Monitoring → Events** showed full context:

* File path
* Action (created, modified, deleted)
* Timestamp
* Agent name

---

## 🧠 Final Takeaways

* Dockerized Wazuh fails hard and early if initialization breaks
* Volume resets are not optional when OpenSearch security corrupts
* Ghost agents persist unless removed manually
* `ossec.conf` is fragile: syntax and encoding both matter
* Automation beats manual edits every time

This setup now runs stable, detects file changes in real time, and reflects what deploying Wazuh *actually* looks like outside sanitized documentation.
