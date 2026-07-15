# Phase 1: Server Provisioning & Infrastructure Hardening

This document tracks the initialization, provisioning, and security baseline establishment for the remote cloud host architecture.

## 1. System Architecture Profile
* **Provider:** Amazon Web Services (AWS) EC2
* **Instance Type:** `t3.small` / `t3.micro` burstable performance instance
* **Compute Profile:** 2.0 GiB Physical RAM | 1 vCPU
* **Storage Profile:** 8.0 GB Elastic Block Store (EBS) Root Volume
* **Operating System:** Ubuntu Server 24.04 LTS (Noble Numbat)

---

## 2. Environment Fixes & Terminal Emulation
Connecting via modern GPU-accelerated terminal emulators (e.g., `kitty`) causes environment mismatch warnings (`unknown terminal type`) on remote hosts lacking terminfo definitions.

* **Mitigation:** Enforced a fallback to standard `xterm` terminal emulation.
* **Commands Executed:**
    ```bash
    export TERM=xterm
    echo 'export TERM=xterm' >> ~/.bashrc
    ```

---

## 3. SSH Daemon Hardening (`sshd_config`)
To secure the open network interface against automated brute-force attacks, the SSH daemon configuration (`/etc/ssh/sshd_config`) was modified.

* **Configurations Applied:**
    * `PermitRootLogin no`: Hard blocks direct administrative root access via SSH.
    * `MaxAuthTries 3`: Drops connection state after three consecutive failed cryptographic/auth requests.
* **Verification & Reload:**
    ```bash
    sudo sshd -t
    sudo systemctl reload sshd
    ```

---

## 4. Host-Based Network Firewall (UFW)
Implemented an operating-system-level firewall layer using the Uncomplicated Firewall (UFW) to act as a secondary defense boundary independent of cloud provider security groups.

* **Policy Constraints:**
    * Default Incoming: `DENY`
    * Default Outgoing: `ALLOW`
    * Explicit Exception: Port 22/TCP (`SSH`)
* **Commands Executed:**
    ```bash
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw enable
    sudo ufw status verbose
    ```

---

## 5. Memory Management & Swap Allocation
With a restricted physical footprint of 2.0 GiB RAM, deploying complex Dockerized application layers runs the risk of triggering the Linux kernel's Out-Of-Memory (OOM) Killer. A 2.0 GiB swap space was provisioned on disk to serve as an emergency overflow paging file.

* **Resource Trade-Off:** Allocating a 2.0 GiB file reduces available root storage space from 4.6 GiB down to roughly 2.6 GiB, requiring optimal container storage hygiene going forward.
* **Execution & Persistence Steps:**
    ```bash
    # Allocate and secure block storage
    sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
    sudo chmod 600 /swapfile
    
    # Format and mount paging file
    sudo mkswap /swapfile
    sudo swapon /swapfile
    
    # Register to File System Table for boot persistence
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
    ```
* **Post-Audit Verification:**
    ```bash
    free -h
    df -h
    ```
