# Penetration Test Report: EPSILON (Revised Version)

**Date:** 22/09/2025
**Platform:** Hack The Box
**Machine:** EPSILON (Linux - Ubuntu)
**Difficulty:** Medium
**Status:** Active (Testing in Progress)
**Author:** César López Oliva | [GitHub](https://github.com/tuusuario) | [LinkedIn](https://www.linkedin.com/in/tuperfil/)

---

## Table of Contents

1. Executive Summary
2. Scope and Rules of Engagement
3. Methodology and Tools
4. Technical Findings (with CVSS justification, evidence, and remediation)

   * 4.1 Service Enumeration Results
   * 4.2 Information Exposure via `.git`
   * 4.3 AWS Credentials in Git History
   * 4.4 AWS Infrastructure Discovery
   * 4.5 Cookie Auth (Cookie Creation)
   * 4.6 SSTI (Server Side Template Injection)
   * 4.7 Privilege Escalation (Internal Investigation)
5. Attack Chain (Diagram and Status)
6. Practical Remediations — Ready-to-Apply Snippets
7. Post-Remediation Verification (Commands and Checks)
8. Next Steps (Prioritized)
9. Executive Checklist
10. Annexes and Secure Delivery

---

## 1. Executive Summary

During the penetration test on the **EPSILON** machine, a **public `.git` directory exposure** on the web server was identified, allowing access to the commit history and configuration data. References to AWS credentials were found in the commit history, enabling enumeration of Lambda functions on `cloud.epsilon.htb`.

**Impact:** Critical. The availability of credentials with permissions on cloud resources allows exfiltration, execution, and manipulation of functions and data in the AWS infrastructure.

**Status:**

* `.git` exposure: **\[CONFIRMED]**
* Lambda enumeration using extracted credentials: **\[CONFIRMED]**
* Injection/execution tests on Lambdas: **\[IN PROGRESS]**
* SSTI on the web service (port 5000): **\[CONFIRMED]** (non-destructive tests confirmed template execution)
* Privilege escalation on internal machine: **\[UNDER INVESTIGATION]** (vulnerable files found, exploitation not completed)

**Immediate Action Recommended:** 1) Revoke/rotate any compromised AWS credentials; 2) block public access to `.git`; 3) review CloudTrail for activity performed with exposed credentials.

---

## 2. Scope and Rules of Engagement

* **Scope:** EPSILON machine within Hack The Box (IP: 10.10.11.134). All tests were conducted within the authorized environment.
* **Test Dates:** 22/09/2025 (documentation current as of this date).
* **Boundaries:** No destructive actions were performed on external services. Sensitive evidence is stored in encrypted annexes.

---

## 3. Methodology and Tools

* **Phases:** Recon → Scanning → Web Enumeration → VCS Analysis → Cloud Analysis → Application Tests (SSTI / auth) → PrivEsc attempts.
* **Primary Tools:** nmap, dirb, ffuf, git-dumper, git, awscli, burpsuite, custom scripts.

---

## 4. Technical Findings

> Each finding includes a summary, CVSS with brief justification, evidence (commands/outputs without secrets), remediation, and verification steps.

### 4.1 Service Enumeration Results

**Description:** Accessible services: SSH (22), HTTP (80 - Apache), HTTP (5000 - Flask/Werkzeug).
**CVSS:** 5.0 (Medium)
**CVSS Justification:** AV\:N (network), AC\:L (low complexity for enumeration), PR\:N (no privileges required), UI\:N; medium impact due to exposed services and patchable versions.

**Evidence:**

```bash
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4
80/tcp   open  http    Apache httpd 2.4.41
5000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.10)
```

**Remediation:**

* Segment services according to operational requirements.
* Harden SSH: disable root login, enforce key authentication, configure firewall and Fail2Ban.
* Update Apache/Werkzeug/Python to patched versions.

**Verification:** `nmap -sV -p22,80,5000 10.10.11.134`

---

### 4.2 Information Exposure via `.git`

**Description:** Web server exposes `.git` directory, allowing download of repository and commit history.
**CVSS:** 7.5 (High)
**CVSS Justification:** AV\:N, AC\:L, PR\:N, UI\:N. Source code exposure increases likelihood of discovering credentials and sensitive configurations.

**Evidence:**

```bash
# Detect .git directory
nmap -sC -sV -p80 10.10.11.134
# Enumerate with dirb
dirb http://10.10.11.134/ /usr/share/dirb/wordlists/common.txt
# Found: http://10.10.11.134/.git/HEAD (200)
# Structure enumerated with ffuf
ffuf -w wordlist.txt -u http://10.10.11.134/.git/FUZZ
# Discovered: config, index, logs, objects, refs, HEAD, info
```

**Remediation (Immediate):**

* Block public access to `.git` via server configuration (see section 6 — ready-to-apply snippets).
* Ensure deployment pipelines do not include VCS directories.

**Verification:**

```bash
curl -I http://10.10.11.134/.git/HEAD
# Expected: 403/404 after mitigation
```

---

### 4.3 AWS Credentials in Git History

**Description:** AWS credentials found in previous commits.
**CVSS:** 9.5 (Critical)
**CVSS Justification:** AV\:N, AC\:L, PR\:N, UI\:N. Exposure of credentials allows access to cloud environment with full impact (C\:H / I\:H / A\:H).

**Evidence:**

* `.git` dumped with `git-dumper` (full outputs stored in encrypted annexes).
* Enumerated Lambda functions using exposed credentials:

```bash
aws lambda list-functions --endpoint-url http://cloud.epsilon.htb
# Successful enumeration (outputs in encrypted annexes)
```

**Remediation:**

1. Revoke/rotate affected AWS credentials immediately.
2. Review CloudTrail for actions taken using these credentials.
3. Implement credential scanning in CI/CD (git-secrets, truffleHog, pre-commit hooks).
4. Use IAM roles and Secrets Manager for credentials.

**Verification:** Test old credentials:

```bash
aws sts get-caller-identity || echo "Credentials invalid"
```

---

### 4.4 AWS Infrastructure Discovery

**Description:** Code reveals AWS Lambda, internal endpoints, and domain (`cloud.epsilon.htb`) aiding infrastructure mapping.
**CVSS:** 8.0 (High)
**Justification:** While not direct credentials, exposure of architecture increases targeted attack potential; high impact if combined with compromised keys.

**Evidence:** Code paths, function names, and domain references (full artifacts in encrypted annexes).

**Remediation:** Limit sensitive info in repos; use environment variables; enforce least privilege.

---

### 4.5 Cookie Auth (Cookie Creation)

**Description:** Cookie generation logic extracted from source allowed creation of valid cookies for `http://epsilon.htb:5000`.
**CVSS:** 8.0 (High)
**Justification:** Exposed secret keys for cookie signing allow forged authentication; high impact on session confidentiality.

**Evidence:** Logic located in scripts (`./scripts` in dump); example commands in annexes.

**Remediation:** Rotate signing keys; manage secrets securely; enforce short cookie expiration.

---

### 4.6 Server Side Template Injection (SSTI)

**Description:** Endpoint `/Order` executes template expressions from input.
**CVSS:** 10.0 (Critical)
**Justification:** SSTI can lead to RCE; maximum impact (C\:H / I\:H / A\:H).

**Evidence (Non-destructive PoC):**

```
POST /Order
Body: costume={{7*7}}&q=1&addr=1
Response: Your order of "49" has been placed successfully.
```

**Remediation:** Sanitize inputs; use sandboxed template engines; avoid `render_template_string` with user input.

**Verification:** Resend PoC; confirm expression no longer evaluated.

---

### 4.7 Privilege Escalation (Internal Investigation)

**Description:** Internal exploration revealed potential escalation vectors: scripts and `backup.sh` in `/usr/bin/` with sensitive permissions.
**CVSS:** High risk (dependent on exploit vector; under investigation).

**Evidence:** Screenshots and command outputs (annexes: `screenshots/` and `filelists/`).

**Remediation:** Review permissions/ownership of binaries; audit cron jobs; enforce least privilege.

---

## 5. Attack Chain (Current Status)

Reconnaissance
↓
Port Scanning (Nmap) → 22, 80, 5000
↓
Service Enumeration → `.git` exposed
↓
Git Analysis → Retrieved code and AWS references
↓
Cloud Mapping → `cloud.epsilon.htb` identified
↓
AWS Lambda Enumeration → \[CONFIRMED]
↓
Cookie Creation → \[CONFIRMED]
↓
SSTI at /Order → \[CONFIRMED]
↓
Privilege Escalation → \[UNDER INVESTIGATION]

---

## 6. Practical Remediations — Ready-to-Apply Snippets

**Block `.git` in Apache:**

```apache
<DirectoryMatch "^/.*/\.git/">
    Require all denied
</DirectoryMatch>
<LocationMatch "^/\.git">
    Require all denied
</LocationMatch>
```

**Remove `.git` from deployment:**

```bash
rm -rf $CI_PROJECT_DIR/.git
```

**Clean history/remove secrets:**

```bash
git clone --mirror git@repo:project.git
cd project.git
git filter-repo --path-glob 'secrets.txt' --invert-paths
git push --force --all
```

**Rotate/revoke AWS keys:**

```bash
aws iam list-access-keys --user-name impacted-user --profile admin
aws iam update-access-key

```
