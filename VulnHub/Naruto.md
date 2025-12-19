# Lab Report: HA: Naruto (VulnHub)

## 1. Objective

The goal of this lab was to perform a full penetration test on the "Naruto" virtual machine, moving from initial network reconnaissance to gaining administrative (**root**) privileges.

## 2. Reconnaissance & Enumeration

The initial phase focused on identifying the target and its services.

* **Network Discovery:** Used `nmap -sN 192.168.56.0/24` to locate the target at **192.168.56.109**.
* **Service Scanning:** Conducted an aggressive scan with `nmap -A -p- 192.168.56.109` to identify open ports and versions.
* **SMB Enumeration:** Discovered a shared folder using `smbclient -L`. Accessed the `/Naruto` share and retrieved files like `summon.md` to gather hints for the initial entry.

## 3. Initial Foothold & The "Metasploit Hurdle"

* **Vulnerability:** The target was running a Drupal CMS vulnerable to RESTful API exploitation.
* **Observation:** An attempt was made to use `msfconsole` with the `drupal_restws_unserialize` module. However, the exploit failed to create a session (VHOST/path issues).
* **Note:** As per the instructor's feedback, automated frameworks like Metasploit can be unreliable in specific environments. A manual approach was required to establish the initial shell.

## 4. Privilege Escalation (PrivEsc)

After gaining access as user `naruto` and pivoting to user `yashika`, I explored two primary escalation vectors:

### Vector 1: Sudo Rights

Checked for permissions using `sudo -l`. It was determined that user `yashika` had no sudo privileges, forcing the investigation of deeper system misconfigurations.

### Vector 2: Linux Capabilities (Success)

The core of the lab involved auditing the system for **Capabilities**.

* **Discovery:** Ran `getcap -r / 2>/dev/null` and found a critical misconfiguration:
* `/home/yashika/perl = cap_setuid+ep`


* **Exploitation:** The `cap_setuid` capability allows a binary to manipulate its own User ID. Since the `perl` binary had this set, I executed a Perl one-liner to switch the process UID to **0 (root)** and spawn a shell:
```bash
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash"'

```



## 5. Conclusion

System successfully compromised. The final flag was located in `/root/final.txt`.

**Key Lesson:** Linux Capabilities are a powerful but dangerous alternative to SUID. While intended to provide "least privilege," a single misplaced capability like `cap_setuid` on a versatile binary like Perl provides a direct path to full system takeover. Manual enumeration (checking both `sudo -l` and `getcap`) is essential when automated tools fail.

