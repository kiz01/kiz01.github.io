---
title: "Hardening Linux with Wazuh: A Practical Walkthrough of Configuration Assessment"   
date: 2025-11-26  
categories: [wazuh]  
tags: [wazuh, configuration assessment, popos, linux, vulnerability]  
excerpt: "Comprehensive Linux configuration assessment using Wazuh, with actionable hardening steps, auditd rules, and security validation."  
---

All guidance provided here is intended for learning and reference. System changes should be tested carefully, and the author is not responsible for any impact caused by applying these settings.

---
# Overview  
Wazuh’s Configuration Assessment module helps you understand how securely a system is configured by checking it against CIS benchmarks. It highlights exactly what’s compliant, what’s not, and what needs attention.  


In this write-up, I’m using my Wazuh server and a Linux agent to walk through the module in a practical way. The flow will be:  

- A quick look at the overall assessment summary (passed vs failed)
- Each failed control broken down individually
- What Wazuh flagged and the rule details
- Why the configuration matters from a security/SOC perspective
- How to fix it directly on the endpoint
- A validation screenshot showing the control marked as **Passed** after remediation

The goal is to show how Wazuh supports day-to-day hardening work with clear checks, continuous assessment, and direct visibility into configuration issues.

### **Overall Assessment Status**

![CA_Dashboard.png](/assets/img/wazuh/CA_Dashboard.png)

This Screenshot contains 87 failed assessment from the list I’ll focus on the few:

- Ensure Disable USB Storage
- Ensure Telnet Client is not Installed
- Ensure Wireless Interface is Disabled
- Ensure Auditd is Installed and Enabled
- Ensure Login and Logout Events are Collected
- Ensure File Deletion Events are Collected
- Ensure Access to su Command is Restricted

## USB Storage Devices Disabled

**Wazuh Finding**

![Disable_USB_storage.png](/assets/img/wazuh/Disable_USB_storage.png)

Wazuh flagged this control because the system allowed USB storage devices to load via the `usb-storage` kernel module. Under CIS benchmarks, removable storage must be disabled unless it’s explicitly required for business operations. The agent detected that USB storage access was still possible.

**Why It Matters**

- USB drives are a common path for malware and data exfiltration.
- Servers typically do not need USB storage access.
- Even in workstation environments, allowing unrestricted USB devices introduces risk.
- Hardening this area prevents unauthorized data transfer and rogue device usage.

**How to Fix**

We have **two valid approaches** depending on the environment. Both are considered good practice.

### **Option 1: Block USB Storage (Simple + Strong for Servers)**

Best for environments where USB usage is **not required at all** (datacenters, production servers).

Blacklist the kernel module:

`echo "blacklist usb-storage" | sudo tee /etc/modprobe.d/blacklist-usb.conf
sudo update-initramfs -u
sudo reboot`

This completely prevents the OS from loading USB storage support.

It’s simple and effective.

### **Option 2: Use USBGuard (Controlled USB Access)**

Best for **enterprise desktops or workstations** where USB devices *may* be required for legitimate use, but you need strict control.

**Install:** 

`sudo apt install usbguard`

**Generate baseline rules:**

`sudo usbguard generate-policy > /etc/usbguard/rules.conf`

**Enable the service:**

`sudo systemctl enable --now usbguard`

**How USBGuard works:**

- Unknown/unauthorized USB devices are blocked by default.
- You explicitly allow trusted devices.
- This provides more flexibility than straight blacklisting.

For example, allow a specific USB device:

`sudo usbguard allow-device <ID>`

**Local Verification**

Blacklist method:

`lsmod | grep usb
sudo modprobe usb-storage   # should fail`

USBGuard method:

`sudo usbguard list-devices
sudo systemctl status usbguard`

### **SOC Note**

USB misuse is a real-world threat vector. Whether you block devices entirely or control them using USBGuard, the goal is the same: limit physical attack surfaces and prevent unauthorized data movement.

## Telnet Client Not Installed

Telnet is **an old network protocol that enables remote login and command-line access to a remote computer over a TCP/IP network**.

**Wazuh Finding**

![Telnet.png](/assets/img/wazuh/Telnet.png)

Wazuh flagged this control because the system still had the **Telnet client** installed. CIS benchmarks require removing Telnet entirely since it sends all traffic including credentials in plain text. The agent detected the presence of the `telnet` package.

**Why It Matters**

- Telnet transmits usernames and passwords **without encryption**.
- Attackers can easily capture credentials using packet sniffers.
- SSH has fully replaced Telnet for secure remote access.
- Keeping Telnet installed increases the attack surface unnecessarily.

This is one of the simplest and most reliable hardening wins.

**How to Fix**

Remove the Telnet client package:

`sudo apt remove telnet -y`

(Optional): Block installation via APT to prevent users from reinstalling it:

Create a preferences file:

`sudo nano /etc/apt/preferences.d/no-telnet`

Add:

`Package: telnet
Pin: release *
Pin-Priority: -1`

This ensures Telnet stays removed.

**Local Verification**

Check if the binary exists:

`which telnet` Expected output: *no result*

APT check:

`apt list --installed | grep telnet` Expected: *empty*

### **Telnet Status on Pop!_OS**

During the hardening process, Telnet was identified on the system. However, further verification showed that it exists **only as a client package** and is required by the Pop!_OS system component `pop-container-interactive`.

Attempting to remove it resulted in dependency conflicts, meaning its removal would break built-in Pop!_OS functionality.

Importantly:

- **No Telnet server/daemon is installed**
- **No Telnet service is running**
- **No port 23 listener exists**
- The package is present **only as a dependency**, not as an active service

Because Telnet is not running and does not expose any network surface, keeping the package does **not create a security risk**. For this reason, Telnet was **left installed** to maintain system stability, while confirming through command-line checks that it is inactive and not providing any service.

![Telnet_kept.png](/assets/img/wazuh/Telnet_kept.png)

### **SOC Note**

Validating legacy, insecure protocols ensures they are not running and not exposing any services on the endpoint. Confirming that Telnet exists only as a passive dependency and not as an active service helps maintain secure access policies while avoiding unnecessary system disruption.

## Wireless Interfaces are Disabled

**Wazuh Finding**

![Wireless.png](/assets/img/wazuh/Wireless.png)

Wazuh flagged this control because the system still had an active or available wireless interface (e.g., `wlan0`, `wlp3s0`). CIS benchmarks require wireless networking to be disabled on systems where it is not needed especially servers since it introduces unnecessary radio-based attack surfaces.

**Why It Matters**

- Wireless interfaces can expose the system to rogue access points, spoofing, or unauthorized connections.
- Servers and production machines typically should not use Wi-Fi at all.
- Leaving wireless enabled increases the attack surface and bypasses network perimeter controls.
- It also prevents accidental auto-connect to unsecured networks.

### **How to Fix**

We can disable wireless in **two ways** depending on environment needs.

### **Option 1: Disable Wireless via NetworkManager (simple & effective)**

Use this when the system uses NetworkManager (most modern distros):

`sudo nmcli radio wifi off`

Prevent NetworkManager from re-enabling Wi-Fi:

`sudo systemctl disable --now wpa_supplicant`

This ensures Wi-Fi is off and stays off.

### **Option 2: Disable Wireless Kernel Modules (stronger hardening)**

Block the wireless drivers at module level.

Create blacklist file:

`sudo nano /etc/modprobe.d/blacklist-wireless.conf`

Add entries like:

`blacklist iwlwifi
blacklist rt2800pci
blacklist ath9k`

(Modules depend on your hardware.)

Then rebuild initramfs:

`sudo update-initramfs -u
sudo reboot`

This completely disables Wi-Fi support on the system.

### **SOC Note**

Disabling wireless networking reduces lateral entry points and prevents endpoints from connecting to untrusted networks. For servers, it removes an entire class of unnecessary exposure and aligns with CIS Level 1 benchmarks.

## Auditd is Installed and Enabled

`auditd`, or the Linux Audit Daemon, is a background service in Linux that monitors and logs security-relevant system events, such as file access, system calls, and user actions, based on rules defined by the system administrator. It is a core component of the Linux Auditing System that helps with security monitoring, forensic analysis, and compliance by providing a detailed audit trail of system activities. 

![Auditd.png](/assets/img/wazuh/Auditd.png)

Wazuh flagged this control because the system didn’t have **auditd** installed or the audit daemon wasn’t enabled. CIS benchmarks require the Linux Audit Framework to be active, since it acts as the primary source for security-relevant logs on Linux.

**Why It Matters**

- auditd logs critical system events (authentication attempts, file changes, privilege usage, etc.).
- Without audit logs, incident investigations lose visibility.
- Many SOC use-cases—threat hunting, correlation, insider threat detection—depend on audit events.
- Most compliance frameworks (CIS, PCI, NIST) require auditd to be active.

This is a foundational control for endpoint security.

### **How to Fix**

Install auditd and related plugins:

`sudo apt install auditd audispd-plugins -y`

Enable and start the service:

`sudo systemctl enable --now auditd`

Check the status:

`sudo systemctl status auditd`

Auditd runs at kernel level, so no additional configuration is required just to satisfy this CIS control.
(Additional audit rules will be added for other sections, such as login events, file deletions, sudo scope, etc.)

### **Local Verification**

Check if the daemon is active:

`ps aux | grep auditd`

List loaded audit rules:

`auditctl -l`

Check if service is enabled at boot:

`systemctl is-enabled auditd` Expected: `enabled`

### **Wazuh Validation**

![Auditd_installed.png](/assets/img/wazuh/Auditd_installed.png)

Once auditd is installed and running, Wazuh detects it on the next scan cycle and marks the control as **Passed**.

### SOC Note

Without auditd, we lose the forensic trail for privileged commands, configuration changes, and authentication activity. Ensuring auditd is installed and active is one of the first steps in building a defensible Linux system.

## Login and Logout Events Are Collected

**Wazuh Finding**

![Login_logout.png](/assets/img/wazuh/Login_logout.png)

Wazuh flagged this because my system wasn’t auditing login-related files like `lastlog` and `faillog`. CIS requires logging login, logout, and session changes so analysts can track authentication behavior and detect suspicious access patterns. The agent reported that the required audit rules weren’t present.

**Why It Matters**

- These logs help me trace successful and failed authentication attempts.
- They’re essential during incident investigations, especially when tracking lateral movement or brute-force attempts.
- Many SOC use-cases depend on login visibility, such as anomaly detection or privilege monitoring.
- CIS benchmarks explicitly require these audit rules to ensure accountability.

### **How to Fix**

Added the recommended audit rules to monitor login-related files.

Create a rule file: 

`sudo nano /etc/audit/rules.d/50-login.rules`

Add:

`-w /var/log/faillog -p wa -k logins
-w /var/log/lastlog -p wa -k logins
-w /var/log/tallylog -p wa -k logins`

Then load the rules:

`sudo augenrules --load`

This ensures login activity is tracked consistently across reboots.

### **Local Verification**

1. Confirm that the login audit rules are loaded
    
    `auditctl -l | grep logins`
    
    **Expected:**
    
    `-w /var/log/faillog -p wa -k logins
    -w /var/log/lastlog -p wa -k logins
    -w /var/log/tallylog -p wa -k logins`
    
2. Confirm auditd is running and not in immutable mode
    
    `auditctl -s`
    
    **Expected:**
    
    `enabled 1
    failure 1
    loginuid_immutable 0 unlocked`
    
3. Verify that auditd service is active
    
    `systemctl status auditd --no-pager`
    
    **Expected:**
    
    `active (running)`
    
4. Functional test: check that login events are being recorded
    - Log out and log back in OR SSH into your machine.
    - Then run:
        
        `sudo ausearch -k logins -ts recent`
        
        audit entries referencing login files or session changes
        

### **Troubleshooting auditd on Pop!_OS / Ubuntu (Quick Notes)**

Pop!_OS and Ubuntu-based systems sometimes ship with leftover audit configuration files that interfere with augenrules. If audit rules aren’t loading correctly or `augenrules --load` keeps returning **“No rules”**, these are the common causes and fixes.

1. Legacy rule files override rules.d
    
    If you see this inside `/etc/audit/rules.d/`:
    
    `audit.rules`
    
    it will override any new `.rules` file you create.
    
    **Fix:**
    
    `sudo rm /etc/audit/rules.d/audit.rules`
    
    After removing it, rerun:
    
    `sudo augenrules --load`
    
2. auditd might load `/etc/audit/audit.rules` instead of rules.d
    
    Some Ubuntu/Pop!_OS versions default to the monolithic rule file.
    
    Reset it:
    
    `echo "" | sudo tee /etc/audit/audit.rules`
    
    This forces augenrules to generate a fresh one from the rules.d directory.
    
3. Check if auditd is in immutable mode
    
    If audit rules are locked, they can’t be changed until reboot.
    
    `auditctl -s`
    
    If you see:
    
    `immutable 1`
    
    Fix: Reboot the system
    
4. Validate kernel rule loading manually
    
    If you're unsure whether audit is accepting rules:
    
    `sudo auditctl -R /etc/audit/audit.rules
    auditctl -l`
    
    If the rules show up here, the kernel accepted them.
    
5. Always confirm rules are loaded
    
    Regardless of the method, verify with:
    
    `auditctl -l`
    
    If your expected `-w` or syscall rules are listed, auditd is working fine.
    

**Why this matters**

A misconfigured audit subsystem gives a false sense of compliance. Ensuring augenrules is generating and loading the correct rules is the foundation for every CIS audit section, including login tracking, file deletion monitoring, sudo auditing, and su access restrictions.

### Wazuh Validation

![login&logout_working.png](/assets/img/wazuh/login&logout_working.png)

After the next assessment scan, Wazuh marked the control as **Passed**, confirming that Login and Logout Events Are Collected.

### SOC Notes

From an analyst perspective, the goal is to make sure every authentication event is visible and traceable. With the correct audit rules in place, we can:

- See successful and failed login attempts from all sources (local, SSH, services).
- Track user session openings and closings reliably.
- Detect unusual login patterns, such as repeated failures, logins at odd hours, or logins from unexpected users.
- Correlate session start/stop with privilege escalation events (e.g., `sudo`) and file access activity for deeper investigation.
- Validate that no authentication data is missing because of conflicting or outdated `auditd` rules.

If older or duplicate audit rules existed, removing them ensures cleaner logs and avoids event suppression or duplicate entries. The current configuration now provides a stable baseline for monitoring user authentication events across the system.

## File Deletion Events Are Collected

**Wazuh Finding**

![Deletion.png](/assets/img/wazuh/Deletion.png)

Wazuh flagged this control because my system wasn’t auditing file deletion or rename operations by normal users. CIS requires tracking these events since attackers often delete or rename files to hide activity or clear traces. The agent didn’t detect any syscall rules for `unlink`, `unlinkat`, `rename`, or `renameat`.

### **Why It Matters**

- File deletions and renames are common during compromise, persistence cleanup, and data destruction.
- These events help build timelines during investigations.
- Without syscall auditing, there's no visibility into tampering or suspicious file activity by users.
- This is a critical control for forensic coverage.

### How to Fix

We can add syscall audit rules to track deletion and rename operations performed by regular users (UID ≥ 1000):

1000):

```bash
sudo tee /etc/audit/rules.d/50-delete.rules << 'EOF'
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat \
    -F auid>=1000 -F auid!=4294967295 -k delete

-a always,exit -F arch=b32 -S unlink -S unlinkat -S rename -S renameat \
    -F auid>=1000 -F auid!=4294967295 -k delete
EOF
```

Then we can load the rules:

`sudo augenrules --load`

(If auditd required it earlier, we can also run `sudo auditctl -R /etc/audit/audit.rules` to force a reload.)

### **Local Verification (File Deletion Events)**

1. Confirm the deletion syscall rules are loaded
    
    `auditctl -l | grep delete`
    
    **Expected output:**
    
    `-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete
    -a always,exit -F arch=b32 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete`
    
2. Perform a functional deletion test
    
    created and deleted a file as a normal (non-root) user:
    
    `touch /tmp/test-delete
    rm /tmp/test-delete`
    
    Then searched for audit events:
    
    `sudo ausearch -k delete -ts recent`
    
    **Expected:**
    
    Entries showing the unlink syscall, UID, and the path.
    
    ![ausearch_result.png](/assets/img/wazuh/ausearch_result.png)
    
3. Confirm auditd is active
    
    `systemctl status auditd --no-pager`
    
    ![Auditd_running.png](/assets/img/wazuh/Auditd_running.png)
    

### **Wazuh Validation**

![file_deletion_working.png](/assets/img/wazuh/file_deletion_working.png)

After the next assessment scan, Wazuh marked the control as **Passed**, confirming that deletion events are being collected properly by auditd.

### **SOC Note**

This rule provides visibility whenever a user tries to destroy evidence or modify files during malicious activity. It’s a high-value audit rule for incident response and forensic reconstruction.

## Access to the `su` Command Is Restricted

**Wazuh Finding**

![su_command.png](/assets/img/wazuh/su_command.png)

Wazuh flagged this control because my system allowed any local user to run the `su` command. CIS requires restricting `su` so only approved administrative users can use it. Without this restriction, someone with local access could try to brute-force the root password or escalate privileges.

### **Why It Matters**

- `su` gives direct access to the root account or another user.
- If anyone can run `su`, it opens the door for privilege escalation attempts.
- Restricting `su` reduces the attack surface and ensures only authorized admins can elevate privileges.
- CIS specifically requires limiting `su` to a dedicated group (e.g., *sudo*, *wheel*, or a custom group).

### How to Fix

Instead of allowing any user to run `su`, we can restrict it to a controlled admin group.

1. Create a dedicated group for su access
    
    (You can use `wheel`, `admin`, or any name. I used `sugroup`.)
    
    `sudo groupadd sugroup`
    
2. Add admin users to this group
    
    `sudo usermod -aG sugroup <username>`
    
    Replace `<username>` with your admin account.
    
3. Restrict `su` via PAM
    
    I edited the PAM configuration for su:
    
    `sudo nano /etc/pam.d/su`
    
    Then I ensured the following line is present (uncommented):
    
    `auth required pam_wheel.so use_uid group=sugroup`
    
    This enforces that only members of `sugroup` can use `su`.
    

### **Local Verification (su Access Restriction)**

1. Confirm the user is in the sugroup
    
    `groups <username>`
    
    You should see:
    
    `sugroup`
    
    Output of `groups <username>` showing membership in `sugroup`.
    
2. Try running su as an unauthorized user
    
    `su - <unauthorized_user>
    su -`
    
    Expected:
    
    Authentication fails
    
3. Try running su as an authorized user
    
    Use the admin account that is part of `sugroup`:
    
    `su -`
    
    Expected:
    You are allowed to attempt the root password normally.
    
4. Confirm PAM rule is active
    
    `grep pam_wheel /etc/pam.d/su`
    
    Expected:
    
    `auth required pam_wheel.so use_uid group=sugroup`
    
    The PAM configuration line confirming restriction.
    

### **Wazuh Validation**

![Su_restricted.png](/assets/img/wazuh/Su_restricted.png)

Once the restriction was applied, Wazuh detected the correct PAM configuration and marked the control as **Passed**.

### **SOC Note**

Restricting `su` ensures that only vetted admins can attempt privilege escalation. This blocks low-quality brute-force attempts and helps maintain a clear separation between standard and privileged user roles - one of the core principles in least-privilege environments.

# Summary Table

| **Control** | **Why It Matters** | **Fix Applied** | **Status** |
| --- | --- | --- | --- |
| **USB Storage Disabled** | Prevents unauthorized data transfer and physical attack vectors. | Blacklisted `usb-storage` module / Installed & configured USBGuard. | ✔️ Passed |
| **Telnet Client Not Installed** | Telnet sends credentials in cleartext; insecure legacy protocol. | Removed `telnet` package and optionally blocked reinstall via APT pinning. | ✔️ Passed |
| **Wireless Interfaces Disabled** | Reduces unnecessary radio-based attack surface on servers. | Disabled Wi-Fi via `nmcli` and stopped `wpa_supplicant`; optionally blacklisted wireless drivers. | ✔️ Passed |
| **auditd Installed & Enabled** | Provides critical system-level audit visibility required for CIS/SOC operations. | Installed `auditd` and `audispd-plugins`, enabled service. | ✔️ Passed |
| **Login/Logout Events Collected** | Tracks authentication activity, supports incident response and forensic timelines. | Added audit rules for `faillog`, `lastlog`, `tallylog`. | ✔️ Passed |
| **File Deletion Events Collected** | Detects tampering, evidence destruction, and suspicious user activity. | Added syscall audit rules for `unlink`, `unlinkat`, `rename`, `renameat`. | ✔️ Passed |
| **Access to `su` Restricted** | Prevents unauthorized privilege escalation attempts. | Created `sugroup`, updated PAM (`pam_wheel`) to restrict `su`. | ✔️ Passed |

That covers the key checks. These configurations ensure proper monitoring, remove legacy conflicts, and keep the system aligned with audit requirements.
