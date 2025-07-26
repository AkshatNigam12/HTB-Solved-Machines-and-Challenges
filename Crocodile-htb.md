# Hack The Box – FTP & Web Directory Exploitation

*Solved by: Akshat Nigam  |  Machine Type: Linux  |  Category: FTP Misconfiguration, Web Enumeration, Brute-force  |  Difficulty: Easy  |  Attack Surface: FTP (port 21), HTTP (port 80)*

## 🧠 Summary

This machine focused on FTP misconfiguration, web directory fuzzing, and credential brute-forcing. Below is the step-by-step walkthrough:

---

## 🔍 Initial Enumeration

* Ran an `nmap` scan to discover open services.
* Found two open ports:
  ```bash
  nmap -sC -sV <target ip> -v
  ```

  * **21/tcp** – FTP
  * **80/tcp** – HTTP

---

## 📂 FTP Anonymous Access

* `nmap` revealed **anonymous FTP login** was enabled.
* Logged in as `anonymous` using command-line FTP.

```bash
  ftp anonymous@<target ip> 
  ```
  
* Listed files using `ls -la`.
* Used `get` command to download two key files:

  
  * `usernames.txt`
  * `passwords.txt`

---

## 🌐 Web Server Analysis

* Navigated to the web service hosted on port 80.
* No login page visible initially.

---

## 🔐 FTP Brute Force Attempt

* Tried brute-forcing FTP login with `hydra`:

  ```bash
  hydra -L usernames.txt -P passwords.txt ftp://<IP>
  ```
* No valid credentials were found.

---

## 🚪 Directory Fuzzing (Gobuster)

* Used `Gobuster` to enumerate hidden web directories:

  ```bash
  gobuster dir -u http://<IP> -w /path/to/wordlist -x php,html
  ```
* Found `login.php` page.

---

## 🧑‍💻 Web Login Brute Force

* Based on the file contents and enumeration, suspected **admin** user.
* Manually attempted password guesses and gained access to the dashboard.

---

## 🏁 Capture the Flag

* Logged into the admin panel.
* Retrieved the flag from the dashboard.

---

## ✅ Key Takeaways

* **Anonymous FTP** access can leak sensitive files.
* **Directory fuzzing** helps uncover hidden paths.
* Reuse of **discovered credentials** can lead to lateral movement.
* Combining multiple attack vectors leads to compromise.

---


