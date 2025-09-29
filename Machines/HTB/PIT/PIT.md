# Penetration Test Report: PIT (Revised)

**Date:** 26/09/2025  
**Platform:** Hack The Box  
**Machine:** PIT (Linux)  
**Difficulty:** Medium  
**Status:** Retired  
**Author:** César López Oliva | [GitHub](https://github.com/Cesar-pentest/Penetration-Testing-Notes) | [LinkedIn](https://www.linkedin.com/in/c%C3%A9sar-lopez-oliva-2145b0216)

---

## Executive Summary

During the penetration test of the **PIT** machine we identified and exploited multiple weaknesses that resulted in full system compromise. The initial foothold was gained via SNMP enumeration (default `public` community) which disclosed a hidden web path hosting a vulnerable SeedDMS instance. SeedDMS’s upload functionality allowed a web shell upload and limited code execution. Using information harvested from the webshell and the application, we obtained credentials to access the Cockpit web console and used its built-in terminal to obtain a user shell. Post-enumeration revealed writable scripts and SNMP management capabilities that were abused to achieve root privileges despite SELinux being enabled.

**Overall risk:** High — the combination of insecure SNMP configuration, vulnerable web application, and poorly protected management interfaces allowed a complete compromise.

**Immediate actions recommended:**  
1. Change or disable SNMP community strings (do not use `public`).  
2. Patch SeedDMS (or remove/segment the service) and fix file-upload handling.  
3. Remove or restrict access to Cockpit and other management interfaces.  
4. Rotate any credentials discovered and review logs (Auditd/SSH).

---

## 1. Scope and Rules of Engagement

* **Scope:** PIT machine within Hack The Box (IP: **10.10.10.241**). All actions confined to the HTB environment.  
* **Test Date:** 26/09/2025 (document current as of this date).  
* **Rules / Limits:** Non-destructive testing by default; any potentially destructive steps were not performed unless needed to validate a remediation. Sensitive artifacts (dumps, screenshots containing secrets) are exported in encrypted annexes.

---

## 2. Methodology and Tools

**Phases performed:** Recon → Port/service scanning → SNMP enumeration → Web enumeration → File upload exploitation → Post-exploitation (user) → Privilege escalation → Clean documentation.

**Main tools used:** `nmap`, `snmpwalk` / `snmpenum`, `dirb`/`gobuster`, browser, `Burp Suite`, `curl`, `git` (for local analysis), `Cockpit` UI, `linpeas`/`linux-smart-enum` (post-exploitation), custom scripts.

---

## 3. Technical Findings (detailed)

> Each finding includes: description, CVSS score + short justification, evidence (commands/outputs without secrets), remediation and verification steps.

### Finding 1 — SNMP: Default/Weak Community Disclosure (Initial Foothold)
**Description:** SNMP (UDP/161) responded to `snmpwalk` with the default `public` community, exposing system data and configuration, including filesystem paths and hostnames that led to a hidden SeedDMS instance and user information.

**CVSS:** 6.5 (Medium)  
**Justification:** AV:N (network), AC:L (low complexity to query), PR:N (no privileges), UI:N. Confidentiality is affected because internal information and paths were exposed, increasing likelihood of further exploit.

**Evidence (non-sensitive):**
```bash
# Example enumeration (conceptual)
snmpwalk -v1 -c public 10.10.10.241
# Output included OIDs revealing mount points, hostnames and references to dms-pit.htb or web paths
```

**Impact:** Disclosure of hidden web paths and user names that enabled targeted web and application exploitation.

**Remediation:**
* Disable SNMP v1/v2 or restrict to management network; prefer SNMPv3 with authentication/encryption.
* Change community strings to strong, unique values; restrict SNMP access via firewall (ACL).
* Monitor SNMP queries and audit logs for unusual activity.

**Verification:**
```bash
# After remediation, this should return no data or require SNMPv3 credentials:
snmpwalk -v1 -c public 10.10.10.241 || echo "No public SNMP access"
```

---

### Finding 2 — Hidden Web Application (SeedDMS) Discovered by SNMP
**Description:** SNMP disclosed a path to a DMS web application (SeedDMS) running on the host under a separate vhost (e.g., `dms-pit.htb`). SeedDMS version hosted was vulnerable to file upload / remote code execution in older versions.

**CVSS:** 8.0 (High)  
**Justification:** AV:N; AC:M (requires targeted upload and exploitation); PR:N; UI:N. Exploitable web application allowed file uploads and subsequent code execution.

**Evidence:**
* Verified vhost via TLS certificate and host header: `dms-pit.htb` discovered.  
* Visiting SeedDMS login / file upload area confirmed typical SeedDMS workflow.

**Remediation:**
* Patch SeedDMS to the latest secured version.
* Restrict uploads: validate file types server-side, store uploaded files outside the web root, and ensure uploaded files are not executable.
* Apply web application firewall (WAF) rules and limit access to the DMS by network segmentation.

**Verification:**
* Re-scan host and verify updated SeedDMS version and that upload endpoint blocks unsafe uploads.

---

### Finding 3 — File Upload → Web Shell (Authenticated/Unauthenticated Abuse)
**Description:** The SeedDMS upload functionality was abused to upload a web shell (or other executable code). The shell allowed browsing filesystem content and limited command execution within the webserver context.

**CVSS:** 9.1 (Critical)  
**Justification:** AV:N; AC:M; PR:N; UI:N. A successful web shell leads to code execution on the host process and may lead to credential disclosure and lateral movement.

**Evidence (non-destructive):**
* File uploaded and accessed via web path (screenshot and file path documented in encrypted annex).  
* Commands executed via webshell returned filesystem and config information (no sensitive secrets shown in-line here).

**Remediation:**
* Harden upload handling (reject executable extensions, check MIME types, store outside webroot).
* Use least-privilege for web server process; run with reduced permissions and disable execution in upload folders.
* Enable SELinux/AppArmor policy checks to minimize impact of code executed in web server context.

**Verification:**
* Attempt safe upload of a known-bad type and confirm server rejects or blocks it.
* Confirm uploaded files cannot be executed by web server.

---

### Finding 4 — Credential Discovery and Cockpit Access (Authenticated Lateral Move)
**Description:** Using information discovered via the webshell (e.g., reused or retrievable credentials), login to the Cockpit management interface on port **9090** was achieved for a user account (e.g., `michelle`). Cockpit provided a built-in terminal/console giving an interactive shell.

**CVSS:** 8.6 (High)  
**Justification:** AV:N; AC:L for local web interface login if credentials are available; PR:L (web session needed). Management consoles exposed to the network increase attack impact.

**Evidence:**
```text
# Observed service:
9090/tcp open  https   Cockpit Web Console (SSL cert references dms-pit.htb)
# Successful login to Cockpit via discovered credentials (screenshots in annex)
```

**Remediation:**
* Restrict access to Cockpit to trusted networks and enforce MFA where possible.
* Review and rotate credentials; enforce unique strong passwords and avoid credential reuse across services.
* Log and alert on Cockpit logins from unusual IPs.

**Verification:**
* Attempt to access Cockpit from an external host (should be blocked) or require authentication and MFA.

---

### Finding 5 — Privilege Escalation via Writable Script / SNMP Management
**Description:** Post-user access enumeration revealed writable scripts (e.g., a backup script) and SNMP server capabilities that allow execution of system commands or that can be used to schedule execution as a privileged user. By placing a crafted script or using SNMP to trigger execution, root privileges were achieved despite SELinux being enabled.

**CVSS:** 9.5 (Critical)  
**Justification:** Local vector (AV:L) but trivial to exploit once user access is obtained; impact is full system compromise (C:H / I:H / A:H).

**Evidence (summarized):**
* File `/usr/bin/backup.sh` (or similar) found with write permissions accessible to the compromised user.  
* Cockpit console and observed SNMP management abilities allowed placement or triggering of command execution.  
* Final `root` shell was obtained (root flag retrieved). Screenshots and command outputs stored in encrypted annex (secrets redacted).

**Remediation:**
* Fix file permissions — remove writable bits from system scripts and ensure only trusted accounts can modify them.
* Audit and restrict any management tooling that can execute scripts (SNMP write, cron, systemd timers) — require strict authentication and restrict to management networks.
* Implement file integrity monitoring (AIDE/OSSEC) and alert on unexpected changes to critical scripts.
* Harden SELinux policies to prevent escalation paths involving webserver or management interfaces.

**Verification:**
* Re-run local enumeration to confirm scripts are not writable by unprivileged users:  
```bash
find /usr/bin -perm -002 -type f -exec ls -l {} \;
``` 
* Confirm SNMP write operations are disabled or protected.

---

## 4. Attack Chain (Concise)

Recon / Port Scan → SNMP enumeration (public) → discovery of `dms-pit.htb` → SeedDMS upload vulnerability exploited → web shell → discovery of credentials → Cockpit login → interactive shell (user) → find writable script & SNMP management → trigger execution → root.

---

## 5. Tools Used

* Recon/Port scanning: `nmap`  
* SNMP: `snmpwalk`, `snmpenum`  
* Web enumeration: `gobuster`, browser, `Burp Suite`  
* Post-exploitation: Cockpit console, `linpeas`, `bash`  
* Utility: custom scripts and standard Unix toolset

---

## 6. Recommendations & Remediation Summary (Prioritized)

**Immediate (0–4 hours):**
1. Change or disable SNMP community strings; remove public community access; restrict SNMP to management networks (or upgrade to SNMPv3 with authentication and encryption).  
2. Patch/disable SeedDMS or temporarily restrict access to the service; remove its public exposure while patching.  
3. Rotate any credentials discovered; notify owners and reset passwords.  
4. Block external access to Cockpit; restrict to admin networks and require MFA.

**Short term (24–72 hours):**
1. Harden file upload handling for SeedDMS: validate content, store outside webroot, deny execution rights on upload directories.  
2. Remove write permissions from system scripts; check for improperly permissioned files (SUID, world-writable).  
3. Audit cron jobs, systemd timers, and management interfaces for unsafe script execution.

**Medium term (1–2 weeks):**
1. Implement network segmentation: management interfaces (SNMP, Cockpit) only reachable from trusted networks.  
2. Deploy file integrity monitoring and centralized logging/alerting for unusual file changes or administrative logins.  
3. Conduct internal security review for credentials reuse and implement a password policy + secrets management.

**Long term:**
* Regularly scan for web upload vulnerabilities and test management interfaces for safe configuration; perform periodic penetration tests and developer training on secure deployments.

---

## 7. Post-Remediation Verification (examples)

* Confirm SNMP queries using `public` community fail:
```bash
snmpwalk -v2c -c public 10.10.10.241 || echo "SNMP public disabled"
```
* Confirm SeedDMS patch/version:
```bash
# Check DMS access/version info from authorized admin interface
```
* Verify uploaded files cannot be executed:
```bash
# Try to access an uploaded placeholder file from the web and expect 403 / non-executable response
```
* Ensure Cockpit is not reachable from untrusted network segments:
```bash
curl -k https://10.10.10.241:9090 || echo "Cockpit not accessible from this host"
```

---

## 8. Annexes and Secure Delivery

**Note:** All artifacts containing sensitive data (web shell dumps, screenshots showing credentials, command outputs with passwords) have been exported and encrypted (GPG symmetric). Annex files are grouped under `anexes/`:

* `anexes/pit_webshell_dump.enc`  
* `anexes/cockpit_login_screenshots.enc`  
* `anexes/command_logs.enc`  

**Key delivery:** share the decryption passphrase via an out-of-band, secure channel (e.g., Signal, in-person, or an encrypted ticketing system). Do not include passphrases in the same email as the report.

---

## 9. Conclusion

The PIT machine was compromised due to a chain of weaknesses: an insecure SNMP configuration that provided discovery data and a vulnerable SeedDMS instance that allowed webshell upload and local code execution. Management interfaces (Cockpit) and writable system scripts allowed the escalation from a limited web context to full system control. The combined impact shows how insecure management protocols and misconfigured web applications with poor upload handling constitute a severe risk.

Addressing the prioritized remediations above (SNMP hardening, patching SeedDMS, limiting management access, fixing permissions) will remove the primary attack paths exploited.

---

## 10. Appendix — Quick checklist for remedial actions

- [ ] Disable SNMP v1/v2 or set a secure SNMPv3 configuration.  
- [ ] Remove public SNMP community strings and apply ACLs.  
- [ ] Patch SeedDMS and fix upload handling (store outside webroot; deny execution).  
- [ ] Rotate all relevant credentials; enforce unique passwords and MFA.  
- [ ] Restrict Cockpit to management LAN and enable MFA.  
- [ ] Remove write permissions on system scripts; audit SUID executables.  
- [ ] Implement file integrity monitoring and alerting.  
- [ ] Retest after mitigation.
