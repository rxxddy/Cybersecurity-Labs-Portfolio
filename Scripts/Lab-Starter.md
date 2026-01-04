# Lab-Starter: Pentest Environment Automation

A Python-based automation utility designed to streamline the initialization of penetration testing laboratories. This script handles directory structure creation, target discovery, and automated service enumeration.

## Overview

During CTF challenges or HTB/VulnHub labs, repetitive tasks like setting up workspaces and running initial scans can be time-consuming. `lab_starter.py` automates the "Reconnaissance" phase, ensuring a consistent filesystem structure for reporting and data management.

## Features

- **Automated Workspace:** Creates a dedicated lab directory with subfolders for `nmap`, `exploits`, `scans`, and `notes`.
- **Target Discovery:** Performs an ICMP/ARP sweep (`nmap -sn`) to identify live hosts within a specified subnet.
- **Dynamic IP Resolution:** Supports both full IP entry and suffix-only input (e.g., entering `112` for a `192.168.56.0/24` network).
- **Service Enumeration:** Executes a full-port scan (`-p-`) with OS detection and script scanning (`-A`), saving results directly to the workspace.

## The Script

```python
import os

# Pentest Lab Automation Tool
# Handles filesystem setup and initial enumeration

def main():
    print("=== Lab Environment Starter ===")

    # Define machine name and root directory
    machine_name = input("\nEnter machine name: ").strip()
    if not machine_name:
        print("[-] Error: Name required")
        return

    # Filesystem initialization
    base_dir = f"/home/kali/Documents/labs/{machine_name}"
    folders = ["nmap", "exploits", "scans", "notes"]
    
    if not os.path.exists(base_dir):
        os.makedirs(base_dir)
        # Generate subdirectories
        for folder in folders:
            os.makedirs(os.path.join(base_dir, folder))
        print(f"[+] Workspace initialized: {base_dir}")
    else:
        print(f"[*] Directory exists. Skipping creation...")

    # Template generation
    summon_path = os.path.join(base_dir, "summon.md")
    with open(summon_path, "w") as f:
        f.write(f"# Lab: {machine_name}\n\n- IP: \n- Creds: \n- Status: In Progress\n")
    print(f"[+] summon.md generated")

    # Network configuration
    default_net = "192.168.56.0/24"
    net_input = input(f"\nNetwork Range (Default {default_net}): ").strip()
    network = net_input if net_input else default_net

    # Host discovery
    print(f"\n[*] Running discovery on: {network}")
    os.system(f"nmap -sn {network}")

    # Target selection
    print("\n" + "="*25)
    target_suffix = input("Target IP or suffix (e.g. 112): ").strip()
    
    # Resolve target IP address
    if "." in target_suffix:
        target_ip = target_suffix
    else:
        prefix = ".".join(network.split(".")[0:3])
        target_ip = f"{prefix}.{target_suffix}"

    # Service enumeration
    print(f"\n[*] Target: {target_ip}")
    print("[*] Initiating full scan (-A -p-)... Output: scans/full_scan.txt\n")
    
    scan_file = os.path.join(base_dir, "scans", "full_scan.txt")
    
    # Execute Nmap with output redirection
    os.system(f"nmap -A -p- {target_ip} -oN {scan_file}")

    print(f"\n[!] Enumeration complete. Log: {scan_file}")

if __name__ == "__main__":
    main()

```

## Technical Workflow

1. **Path Management:** Uses `os.path.join` for cross-platform path compatibility.
2. **Network Parsing:** Splits the network string to resolve partial IP addresses, allowing for faster target selection during local network labs.
3. **Output Persistent:** Uses Nmap's `-oN` (Normal Output) flag to ensure all scan data is preserved for future reference or report writing.

## Usage

1. Clone the repository.
2. Run the script: `python3 lab_starter.py`.
3. Follow the interactive prompts to define the target and scan scope.