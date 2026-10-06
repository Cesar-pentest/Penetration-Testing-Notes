# Hack The Box — Touch

> **Platform:** Hack The Box
> **OS:** Windows
> **Difficulty:** Easy
> **Machine:** Touch
>
> **Note:** This write-up documents the methodology and attack chain used to compromise the machine. Flags, passwords and the final weaponized payload are intentionally omitted while the machine remains active.

---
## 0. Machine Information

"In the Layover machine, you uncovered a name and a booking confirmation code. Maybe they'll come in handy here? Jenny Crawford / KS7X2M"

## 1. Reconnaissance

I started with a standard TCP scan against the target.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn IP
```

The scan revealed the following interesting ports:

| Port       | Service | Description                 |
| ---------- | ------- | --------------------------- |
| `135/tcp`  | MSRPC   | Microsoft RPC               |
| `3389/tcp` | RDP     | Microsoft Terminal Services |
| `5985/tcp` | HTTPAPI | Windows Remote Management   |
| `8443/tcp` | HTTPS   | Nexion DeviceHub            |

The most interesting service was clearly **8443**, since it exposed a web application and we do not have any credentials to access RDP or WinRM.

---

# 2. Nexion DeviceHub

Navigating to:

```text
http://IP:8443/
```

presented a login page.

The application identified itself as:

<img width="1917" height="957" alt="image" src="https://github.com/user-attachments/assets/504a9a79-1f43-4254-ad94-690f8f7086c3" />

The login form only requested a password.

There was also an interesting option:

```text
Scan Staff Badge
```

which sent a request to:

```text
POST /login/badge
```
But lead to nothing at all.

While looking at the **source code** of the application , the html contained a very important clue:

> The default password is the device serial number included in the DeviceHub packaging.

This suggested that the application might expose device information somewhere that could potentially be accessed without authentication.

---

# 3. Web Enumeration

I started enumerating the web application.

For example:

```bash
ffuf -c -u https://IP:8443/FUZZ \
     -w /usr/share/wordlists/dirb/common.txt \
     -fw 1
```

One interesting endpoint was:

```text
/api
```

which initially returned:

```text
403 Forbidden
```

The application also exposed:

```text
/login
```

While i was investigating i decided to see if i could list endpoint of the API so i decided to execute ffuf:

```bash
ffuf -c -u https://IP:8443/api/FUZZ \
     -w /usr/share/wordlists/Discovery/.... \
     -fw 1
```

---

# 4. Authentication and the Device API

After obtaining an authenticated session, the application exposed credentials.

The interesting account was:

```text
KioskUser
```

along with its corresponding password.

> Credentials are intentionally redacted from this public write-up while the machine remains active.

This was the first major pivot from the web application to the underlying Windows system.

---

# 5. RDP Access

The initial thought was to test the recovered credentials against the exposed Windows services.

RDP was particularly interesting because port `3389` was open.

The first attempts with NetExec produced an error related to the RDP security configuration:

```text
SSL_NOT_ALLOWED_BY_SERVER
```
Rather than assuming the credentials were invalid, I switched to a native RDP such as **rdesktop** client and adjusted the security negotiation.

This eventually resulted in a successful RDP connection.

<img width="1907" height="947" alt="image" src="https://github.com/user-attachments/assets/01f15485-6307-493f-a36c-8e19d6ea7643" />

---

# 6. The Windows Kiosk

The RDP session did not provide a normal Windows desktop.

Instead, the account was placed inside an **airline self-check-in kiosk environment**.

This significantly restricted the normal Windows interface.

At this point, the objective changed from traditional remote-access enumeration to **kiosk escape**.

The important observation was that the kiosk application interacted with several local components, including the scanner/printer functionality.

Manipulating the state of one of these components caused the application to generate a native Windows error.

That error exposed functionality that was not intended to be available from the kiosk interface.

<img width="1917" height="942" alt="image" src="https://github.com/user-attachments/assets/d465a46e-4da8-46d8-b83c-04217cba3ff3" />


From there it was possible to reach a more general Windows interface and ultimately obtain a command shell as the kiosk user.

The resulting context was:

```cmd
whoami
```

```text
<kiosk user>
```

This gave direct access to the Windows filesystem and allowed local privilege-escalation enumeration.

---

# 7. Local Enumeration

Once a shell was available, I began enumerating the system.

Some of the first checks were:

```cmd
whoami
whoami /all
whoami /priv
whoami /groups
systeminfo
```

I also enumerated services and processes:

```cmd
tasklist /v
sc query
```

and looked for interesting files and application configuration.

One particularly interesting discovery was that the kiosk application used a **local MySQL database**.

---

# 8. MySQL Configuration

The application configuration revealed a local database connection.

The relevant parameters were:

```text
Host:     127.0.0.1
Port:     3306
Database: htb_airways
User:     kiosk_app
```

A database password was also present in the application configuration.

> Credentials are omitted from this public write-up.

The MySQL server was therefore completely local to the machine:

```text
127.0.0.1:3306
```

I connected from the Windows shell using the MySQL client:

```bash
mysql -h 127.0.0.1 -P 3306 -u kiosk_app -p
```

After authenticating, I enumerated the available databases and privileges. and lead to nothign.

Afert checking everything about the program a mysql root user was found in the Program data file of windows

---

# 9. MySQL Privilege Enumeration with Root

One of the most important discoveries was that the database account had extremely powerful privileges.

Checking the current account:

```sql
SELECT USER(), CURRENT_USER();
```

and:

```sql
SHOW GRANTS;
```

revealed that the account had administrative privileges over the MySQL instance.

This was a significant finding because MySQL was not simply being used as a database:

> **MySQL itself became a potential privilege-escalation primitive.**

---

# 10. MySQL Running as SYSTEM

The next step was to determine which Windows account was running the MySQL server.

From Windows:

```cmd
tasklist /v | findstr /i mysqld
```

and service enumeration:

```cmd
sc query state= all | findstr /i mysql
```

followed by:

```cmd
sc qc <mysql-service>
```

showed that the MySQL service was running as:

```text
NT AUTHORITY\SYSTEM
```

This was the critical observation.

The attack path now looked like:

```text
KioskUser
    │
    ▼
MySQL administrative privileges
    │
    ▼
mysqld.exe
    │
    └── running as NT AUTHORITY\SYSTEM
```

The remaining question was whether MySQL provided a mechanism to execute code inside its own process.

---

# 11. MySQL Plugin Directory

I checked the MySQL configuration:

```sql
SHOW VARIABLES LIKE 'plugin_dir';
```

The plugin directory was:

```text
C:\MySQL\lib\plugin
```

I then checked the Windows ACLs:

```cmd
icacls "C:\MySQL\lib\plugin"
```

The permissions showed that members of:

```text
Authenticated Users
```

had sufficient permissions over the directory.

Since the current Windows user belonged to that group, the plugin directory was effectively writable from the compromised context.

This produced an interesting combination:

```text
MySQL administrative privileges
        +
Writable plugin directory
        +
mysqld running as SYSTEM
```

That is a classic setup for investigating **MySQL User Defined Functions (UDFs)**.

---

# 12. MySQL UDF Privilege Escalation

A MySQL UDF is a native function implemented through a shared library.

On Windows, MySQL can load a DLL from its plugin directory and expose functions from that library to SQL.

Conceptually:

```text
DLL
 │
 ▼
MySQL plugin directory
 │
 ▼
CREATE FUNCTION
 │
 ▼
SELECT function(...)
 │
 ▼
mysqld.exe
 │
 ▼
Windows process context
```

Since:

```text
mysqld.exe = NT AUTHORITY\SYSTEM
```

successful execution through a malicious UDF would inherit the SYSTEM context.

---

## 12.1 Identifying the MySQL Version

The MySQL version was:

```sql
SELECT VERSION();
```

Result:

```text
8.0.42
```

The architecture was:

```text
x86_64
```

Therefore the UDF had to be compiled as a compatible 64-bit Windows DLL.

---

## 12.2 Building the UDF

Instead of using Metasploit, I took a more minimal approach and created a small native DLL implementing a MySQL UDF.

The function accepts a command string and passes it to the Windows command interpreter.

The important point here is that the DLL is not interesting because of the DLL itself.

The interesting part is **where it executes**:

```text
MySQL UDF
     ↓
mysqld.exe
     ↓
SYSTEM
```

The resulting DLL was placed into:

```text
C:\MySQL\lib\plugin
```

using the previously identified write permissions.

---

# 13. Registering the UDF

Once the DLL was available inside the MySQL plugin directory, the function was registered from the MySQL administrative context.

Conceptually:

```sql
CREATE FUNCTION <function_name>
RETURNS INTEGER
SONAME '<udf.dll>';
```

The registered function could then be invoked using SQL:

```sql
SELECT <function_name>(<command>);
```

For the public version of this write-up, the exact payload and command execution sequence are intentionally omitted.

---

# 14. Confirming Code Execution

The first test was deliberately harmless.

Rather than immediately attempting to retrieve the root flag, I executed:

```cmd
whoami
```

and redirected the output to a temporary file.

Reading that file returned:

```text
nt authority\system
```

This confirmed that the UDF was executing inside the MySQL process with the privileges of the MySQL service.

The privilege escalation was therefore complete.

---

# 15. Final Attack Chain

The complete attack chain was:

```text
                         TOUCH
                           │
                           ▼
                  Nmap enumeration
                           │
                           ▼
                 HTTPS :8443 DeviceHub
                           │
                           ▼
                 Device/API enumeration
                           │
                           ▼
                 KioskUser credentials
                           │
                           ▼
                       RDP :3389
                           │
                           ▼
                    Airline Kiosk
                           │
                           ▼
                      Kiosk escape
                           │
                           ▼
                    Windows shell
                           │
                           ▼
              Local application enumeration
                           │
                           ▼
                  MySQL credentials
                           │
                           ▼
                    MySQL :3306
                           │
                           ▼
                  MySQL administrative
                       privileges
                           │
                           ▼
              mysqld.exe running as SYSTEM
                           │
                           ▼
                Writable plugin directory
                           │
                           ▼
                    MySQL UDF / DLL
                           │
                           ▼
               Code execution in mysqld
                           │
                           ▼
                  NT AUTHORITY\SYSTEM
```

---

# 16. Key Findings

### 1. Sensitive information exposed through the DeviceHub API

The authenticated API exposed information that should have been treated as sensitive, including device credentials.

**Impact:**

Credentials intended for the kiosk environment could be recovered through application enumeration.

---

### 2. Kiosk breakout

The RDP account was restricted to a kiosk environment, but the kiosk's interaction with native Windows functionality provided a route to a command shell.

**Impact:**

The kiosk restriction did not provide an effective security boundary.

---

### 3. Sensitive database credentials stored in application configuration

The kiosk application contained credentials for its local MySQL instance.

**Impact:**

Access to the application context resulted in access to the backend database.

---

### 4. MySQL running as SYSTEM

The MySQL service was configured to run under:

```text
NT AUTHORITY\SYSTEM
```

Running database services with SYSTEM unnecessarily increases the impact of vulnerabilities in the database or applications interacting with it.

---

### 5. Writable MySQL plugin directory

The MySQL plugin directory was writable by a broad Windows group:

```text
Authenticated Users
```

Combined with MySQL administrative privileges, this created a dangerous code-loading primitive.

---

### 6. UDF-based privilege escalation

The combination of:

```text
MySQL administrative privileges
+
writable plugin directory
+
MySQL running as SYSTEM
```

allowed code execution inside the MySQL process and resulted in:

```text
NT AUTHORITY\SYSTEM
```

---

## Conclusion

Touch was a good exercise in **chaining application-level enumeration with Windows privilege escalation**.

The most interesting part of the machine was not obtaining the initial shell, but recognizing that the local MySQL instance could be turned into a SYSTEM-level execution primitive.

The decisive combination was:

```text
Writable MySQL plugin directory
+
MySQL administrative privileges
+
mysqld.exe running as SYSTEM
```

Once those three conditions were identified, the UDF route provided a direct path from the kiosk account to:

```text
NT AUTHORITY\SYSTEM
```

**Machine completed.**
