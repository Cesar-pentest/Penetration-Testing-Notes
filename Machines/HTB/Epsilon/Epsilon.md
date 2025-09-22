# Penetration Test Report: EPSILON

**Date:** 22/09/2025  
**Platform:** Hack The Box  
**Machine Type:** Linux (Ubuntu)  
**Difficulty:** Medium  
**Status:** Active (Testing in Progress)  
**Author:** César López Oliva | [GitHub](https://github.com/tuusuario) | [LinkedIn](https://www.linkedin.com/in/tuperfil/)

---

## Executive Summary

A penetration test was conducted on the target machine EPSILON. The assessment has revealed critical information exposure vulnerabilities that could lead to full system compromise. The most significant finding was **an exposed `.git` repository containing sensitive AWS configuration data**. Further exploitation is underway to determine the full impact of these exposures.

## Technical Findings

### 1. Information Exposure via Exposed Git Repository
**Vulnerability:** Source Code Disclosure via Public .git Directory  
**CVSS Score:** 7.5 (High)  
**Attack Vector:** Network  
**Description:** 
The target's web server on port 80 exposes the `.git` directory, allowing unauthorized access to the complete version control history and source code.

**Evidence:**
*   **Enumeration Command:**
    ```bash
    nmap -sC -sV -p80 10.10.11.134
    # Output revealed: 10.10.11.134:80/.git/
    ```
*   **Directory Brute-forcing Results:**
    ```bash
    dirb http://10.10.11.134/ /usr/share/dirb/wordlists/common.txt
    # Found: http://10.10.11.134/.git/HEAD (CODE:200|SIZE:23)
    ```
*   **Git Structure Enumeration:**
    ```bash
    ffuf -w wordlist.txt -u http://10.10.11.134/.git/FUZZ
    # Discovered: config, index, logs, objects, refs, HEAD, info
    ```

**Remediation:**
*   Immediately restrict access to the `.git` directory via web server configuration (e.g., Apache `.htaccess` rules).
*   Implement proper `.gitignore` files and ensure deployment procedures exclude version control directories.
*   Conduct regular security scans to detect accidental source code exposure.

### 2. AWS Credentials Exposure via Git History
**Vulnerability:** Hardcoded AWS Access Keys in Version Control  
**CVSS Score:** 9.5 (Critical)  
**Attack Vector:** Network  
**Description:** 
Analysis of the git commit history revealed hardcoded AWS credentials in previous commits, providing full access to the AWS Lambda infrastructure.

**Evidence:**
*   **Dowloading .git from URL:**
    ```bash
    git-dumper http://10.10.11.134/.git/ /home/user/Maquinas/Epsilon/content/website/
    ```

*   **Git Commit Analysis:**
    ```bash
    git log --oneline
    # Commit: c622771 - Fixed Typo
    # Previous commits contained AWS credentials
    ```
*   **Exposed Credentials:**
    ```
    AWS Access Key: ****REDACTED****
    AWS Secret Key: ****REDACTED****
    ```
*   **AWS Infrastructure Access:**
    ```bash
    aws lambda list-functions --endpoint-url http://cloud.epsilon.htb
    # Successfully enumerated Lambda functions using exposed credentials
    ```

**Remediation:**
*   **IMMEDIATE:** Revoke the exposed AWS credentials in the AWS IAM console.
*   Implement AWS credential scanning in CI/CD pipelines to prevent future exposure.
*   Use AWS IAM Roles instead of hardcoded credentials for Lambda functions.
*   Implement git pre-commit hooks to detect sensitive information.


### 3. AWS Infrastructure Discovery
**Vulnerability:** Sensitive Cloud Configuration Exposure  
**CVSS Score:** 8.0 (High)  
**Attack Vector:** Network  
**Description:** 
Analysis of the exposed git repository revealed Python application code interacting with AWS Lambda functions and cloud infrastructure, including internal domain names and potential API endpoints.

**Evidence:**
*   **Extracted Information:**
    *   Domain name: `cloud.epsilon.htb`
    *   AWS Lambda function identifiers
    *   Python application code handling cloud operations
*   **Analysis:** The exposed source code provides blueprint of cloud architecture, potentially revealing attack vectors against AWS services.

**Remediation:**
*   Rotate all AWS access keys and credentials exposed in the git history.
*   Implement AWS IAM roles with least privilege principles instead of hardcoded credentials.
*   Use AWS Secrets Manager or environment variables for sensitive configuration.

### 4. Service Enumeration Results
**Vulnerability:** Multiple Services Exposed  
**CVSS Score:** 5.0 (Medium)  
**Attack Vector:** Network  
**Description:** 
Network scanning revealed three accessible services providing multiple potential attack surfaces.

**Evidence:**
*   **Nmap Scan Results:**
    ```bash
    PORT     STATE SERVICE VERSION
    22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4
    80/tcp   open  http    Apache httpd 2.4.41
    5000/tcp open  http    Werkzeug httpd 2.0.2 (Python 3.8.10)
    ```
*   **Service Details:**
    *   SSH (port 22): Potential brute-force or key-based attack vector
    *   HTTP (port 80): Apache server with exposed git repository
    *   HTTP (port 5000): Python Flask application ("Costume Shop")

**Remediation:**
*   Implement network segmentation to restrict unnecessary service exposure.
*   Harden SSH configuration (disable root login, use key-based authentication).
*   Apply security patches to all exposed services.

## Attack Chain Diagram (Current Progress)
Reconnaissance
↓
Port Scanning (Nmap) → Found ports 22, 80, 5000
↓
Service Enumeration → Discovered exposed .git repository
↓
Git Analysis → Extracted source code and AWS configuration
↓
Cloud Infrastructure Mapping → Identified cloud.epsilon.htb domain
↓
[IN PROGRESS] AWS Lambda Injection Testing


## Tools Used
*   **Reconnaissance:** `nmap`, `ping`
*   **Web Enumeration:** `dirb`, `ffuf`
*   **Version Control Analysis:** `git-dumper`, `git log`
*   **Cloud Analysis:** Manual code review, AWS CLI

## Next Steps
Based on current findings, the following attack vectors require further investigation:
1.  AWS Lambda function injection via extracted API endpoints
2.  Subdomain enumeration and virtual host routing
3.  Python application analysis (port 5000)
4.  SSH service security assessment

---
**Disclaimer:** This report documents ongoing testing of an active Hack The Box machine. All testing is conducted within the platform's authorized scope for educational purposes only.
