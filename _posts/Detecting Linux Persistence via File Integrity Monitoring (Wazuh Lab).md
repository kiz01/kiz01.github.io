# Detecting Linux Persistence via File Integrity Monitoring (Wazuh Lab)

A hands-on lab demonstrating how Wazuh detects unauthorized changes to critical Linux system and user files.

# Introduction

Modern system compromises rarely rely solely on network-level activity.
After gaining initial access, attackers interact directly with the host system to establish persistence, escalate privileges, and maintain control over time.

These objectives are commonly achieved through modifications to critical system and user-level files, including authentication databases, privilege configuration files, scheduled task definitions, and executable binaries. Such changes alter the state of the system in ways that persist beyond individual processes or sessions.

Unlike network traffic or process execution, these modifications represent durable artifacts of attacker activity. As a result, they provide a reliable surface for observing post-exploitation behavior, even in cases where traditional logs or runtime indicators are limited or absent.

**File Integrity Monitoring (FIM)** provides a mechanism to track these state changes by maintaining a baseline of file attributes and identifying deviations over time. By focusing on file-level modifications, FIM enables visibility into a class of attacker techniques that depend on altering system behavior through persistent changes.

In this lab, Wazuh was used to implement File Integrity Monitoring on a Linux system. A series of controlled modifications were performed to simulate common post-exploitation techniques, including SSH key-based persistence, privilege escalation through configuration changes, scheduled task insertion, and binary manipulation.

The objective of this work is to examine how these techniques manifest at the file system level and to demonstrate how monitoring a small set of high-value files can expose meaningful signals of system compromise.

This work focuses on file-level detection of post-exploitation techniques and does not cover runtime or network-based detection.

# Threat Model

To understand the effectiveness of file integrity monitoring, it is necessary to model how attackers interact with a system after initial access.

### Attacker Objectives

Following initial compromise, attackers typically aim to:

- establish persistence
- escalate privileges
- maintain long-term, reliable access
- reduce the likelihood of detection

These objectives are not achieved through a single action, but through a sequence of modifications that alter system behavior.

### System Constraints

Attackers operate under practical constraints:

- access must persist across sessions and reboots
- privilege escalation requires modification of authorization mechanisms
- execution must be triggered through existing system workflows (e.g., login, scheduled tasks)

As a result, attackers rely on modifying existing system components rather than introducing entirely new mechanisms.

### File System as an Attack Surface

Many of these objectives are implemented through changes to a small set of high-impact files:

| Category | Examples | Purpose |
| --- | --- | --- |
| Authentication | `/etc/passwd`, `/etc/shadow`, `~/.ssh/authorized_keys` | Control access |
| Privilege Control | `/etc/sudoers` | Elevate permissions |
| Execution Triggers | `/etc/cron.d/*`, `~/.bashrc`, `~/.profile` | Achieve persistence |
| System Binaries | `/usr/bin/*`, `/bin/*` | Modify or replace execution behavior |

These files directly influence how the system authenticates users, assigns privileges, and executes code.

### Observable Behavior

Modifying these files produces persistent, system-level changes:

- file content changes
- new file creation
- integrity (hash) mismatches
- deletion or replacement of critical files

Unlike transient events such as process execution or network traffic, these changes remain on disk and can be observed independently of how they were introduced.

### Implication for Detection

Because multiple attacker techniques converge on modifying the same set of high-value files, monitoring these locations provides a consistent and reliable detection surface.

Rather than attempting to observe every possible attacker action, it is more effective to monitor the system components that those actions must ultimately modify.

This model explains why File Integrity Monitoring is well-suited for detecting post-exploitation activity in Linux environments.

This approach prioritizes high-signal monitoring over exhaustive visibility, reducing noise while maintaining effective coverage.

# **Lab Architecture**

This lab environment was designed to observe how file-level changes propagate from a monitored system to a centralized analysis layer.

The setup consists of two primary components:

- **Monitored Endpoint (Ubuntu Host)** – Runs the Wazuh agent and serves as the system where controlled file modifications are performed.
- **Wazuh Manager (Ubuntu Server VM)** – Receives and processes file integrity events, providing a centralized view of observed changes.

### **Architecture Overview**

```bash
Ubuntu Host (Wazuh Agent)
        │
        │ File integrity events
        ▼
Ubuntu Server VM (Wazuh Manager + Dashboard)
```

### Data Flow

The monitoring workflow operates as follows:

1. The Wazuh agent monitors configured files and directories on the endpoint.
2. A baseline of file hashes and metadata is created.
3. When a file is modified, created, or deleted, the agent detects the change.
4. The event is sent to the Wazuh manager.
5. The manager generates alerts visible in the dashboard for investigation.

This separation between event generation (agent) and analysis (manager) enables consistent observation across multiple systems without relying on local visibility alone.

# Wazuh Dashboard and Agent Monitoring

The Wazuh dashboard provides a centralized interface for monitoring endpoint activity and investigating alerts generated by the monitoring system.

The screenshots below show:

- successful agent registration
- active event collection from the monitored endpoint

![***Figure 1:** Wazuh dashboard displaying active agent status*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/01_dashboard.png)

***Figure 1:** Wazuh dashboard displaying active agent status*

![***Figure 2:** File Integrity Monitoring module showing distribution and timeline of file change events (added, modified, deleted)*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/02_agent_status.png)

***Figure 2:** File Integrity Monitoring module showing distribution and timeline of file change events (added, modified, deleted)*

These confirm that the environment is correctly configured and capable of capturing file-level changes for analysis.

# File Integrity Monitoring Configuration

File Integrity Monitoring in Wazuh is implemented through the **Syscheck** module, which runs on the agent and is responsible for tracking changes to files over time.

Rather than monitoring the entire file system, this lab focuses on a targeted set of directories that represent high-value points of control within a Linux system.

### **Monitored Paths**

FIM configuration is defined in the ossec.conf file:

```bash
/var/ossec/etc/ossec.conf
```

For this lab, following directories were configured for monitoring.

```bash
<directories realtime="yes">/etc</directories>
<directories>/usr/bin,/usr/sbin</directories>
<directories>/bin,/sbin</directories>
<directories realtime="yes">/home/*/*.ssh</directories>
<directories realtime="yes">/home/*/*.bashrc</directories>
<directories realtime="yes">/home/*/.profile</directories>
```

### **Selection Rationale**

These paths were chosen because they map directly to core system functions that attackers commonly modify during post-exploitation.

| Category | Paths | Behavioral Significance |
| --- | --- | --- |
| Authentication | `/etc/passwd`, `/etc/shadow`, `~/.ssh` | Controls access and identity |
| Privilege Control | `/etc/sudoers` | Enables privilege escalation |
| Execution Triggers | `~/.bashrc`, `~/.profile`, `/etc/cron.d/*` | Allows automatic code execution |
| System Binaries | `/usr/bin`, `/bin`, `/sbin` | Defines system-level execution behavior |

These locations represent a **high-signal subset of the file system**, where unauthorized changes are more likely to indicate malicious activity than routine system operations.

This approach prioritizes coverage of attacker-controlled system state over exhaustive file system visibility.

### **Real-Time Monitoring Strategy**

Real-time monitoring was enabled selectively:

```
realtime="yes"
```

Applied to:

- `/etc`
- user-level persistence locations (`.ssh`, `.bashrc`, `.profile`)

These paths were prioritized because they:

- directly affect authentication and access
- enable immediate persistence
- are frequently modified during attacker activity

In contrast, large binary directories (`/usr/bin`, `/bin`) were not monitored in real time to avoid excessive event volume from legitimate system activity.

This reflects a trade-off:

- **real-time monitoring → faster detection**
- **selective scope → reduced noise**

### **Configuration Verification**

After applying the configuration, the agent was restarted:

```bash
sudo systemctl restart wazuh-agent
```

Once active, the agent began establishing a baseline and generating events for any detected deviations within the monitored paths.

This confirms that the system is correctly positioned to capture file-level changes introduced during subsequent test scenarios.

# Attack Simulations

To evaluate how file-level changes reflect attacker behavior, a series of controlled modifications were performed on the monitored system.

Each scenario targets a high-value file associated with authentication, privilege control, or execution. These modifications represent common post-exploitation techniques and allow observation of how such activity manifests at the file system level.

## 1. SSH Persistence

**Technique Overview**

One of the most reliable methods for maintaining access to a compromised Linux system is through SSH key-based persistence.

Instead of relying on passwords, attackers add their public key to the `authorized_keys` file, allowing passwordless authentication. This method is particularly effective because it leverages legitimate access mechanisms and does not require repeated credential use.

**Command used**

```bash
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```

A test SSH key was added to simulate unauthorized access.

**Observed Change**

The modification resulted in a change to:

```
/home/user/.ssh/authorized_keys
```

This represents a direct alteration of the system’s authentication configuration.

### **FIM Detection**

The change was captured as a file modification event.

```nasm
File modified
/home/user/.ssh/authorized_keys
```

![***Figure 3:** Injected SSH key in `authorized_keys` establishing persistent remote access*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/04_ssh_persistence.png)

***Figure 3:** Injected SSH key in `authorized_keys` establishing persistent remote access*

This confirms a change in system baseline.

**Security Interpretation**

The `authorized_keys` file defines which identities are permitted to authenticate via SSH.

Unauthorized modification of this file enables persistent, passwordless access and represents a high-confidence indicator of compromise.

From a behavioral perspective, this technique is significant because it:

- persists across reboots
- does not rely on active processes
- leverages legitimate authentication mechanisms

Because this technique requires modifying a specific file, it produces a durable artifact that can be reliably detected through file integrity monitoring.

## 2. Privilege Escalation via `sudoers`

**Technique Overview**

Privilege escalation in Linux systems often involves modifying authorization mechanisms that control access to elevated privileges.

The `/etc/sudoers` file defines which users are allowed to execute commands as other users, including root. By altering this file, an attacker can grant themselves unrestricted administrative access without needing to exploit additional vulnerabilities.

**Command Used**

```bash
sudo visudo
```

A test rule was added to simulate unauthorized privilege escalation.

**Observed Change**

The modification affected a critical system configuration file:

```
/etc/sudoers
```

This file directly governs privilege boundaries within the system.

![***Figure 4:** Modification of `/etc/sudoers` granting elevated privileges to a user*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/05_privilidge_esclation.png)

***Figure 4:** Modification of `/etc/sudoers` granting elevated privileges to a user*

### **FIM Detection**

The change was detected as:

```
File modified
/etc/sudoers
Integrity checksum changed
```

This indicates that the file content deviated from its baseline state.

**Security Interpretation**

The `/etc/sudoers` file defines privilege boundaries within the system.

Unauthorized modification allows an attacker to grant elevated access without further exploitation, making it a high-impact privilege escalation mechanism.

This technique is significant because it:

- provides persistent administrative control
- integrates with legitimate system functionality
- alters core authorization logic

Because such changes must be explicitly defined in configuration, unexpected modifications represent a strong indicator of compromise.

## 3. Backdoor User Creation

**Technique Overview**

Creating new user accounts is a straightforward method for maintaining persistent access to a compromised system.

Instead of modifying existing credentials, attackers introduce a new identity under their control. This approach is effective because it blends with legitimate system functionality and does not rely on exploiting vulnerabilities after initial access.

**Command Used**

```bash
sudo useradd backdooruser
```

**Observed Changes**

The creation of a new user results in modifications to multiple authentication-related files:

```
/etc/passwd
/etc/shadow
```

These files together define user identities and associated authentication data.

![*Figure 5: Entry added to `/etc/passwd` for new user*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/06_backdoor_user.png)

*Figure 5: Entry added to `/etc/passwd` for new user*

![*Figure 6: Corresponding entry created in `/etc/shadow`*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/06_backdoor_user2.png)

*Figure 6: Corresponding entry created in `/etc/shadow`*

### **FIM Detection**

The activity was detected as multiple file modification events:

```bash
File modified
/etc/passwd
File modified
/etc/shadow
```

These changes triggered File Integrity Monitoring alerts.

**Security Interpretation**

User creation modifies core authentication files (`/etc/passwd` and `/etc/shadow`), introducing a new identity into the system.

This produces correlated changes across multiple high-value files, increasing detection confidence.

This technique is effective because it:

- creates a legitimate system account
- enables persistent access
- does not depend on active sessions

Simultaneous modifications to these files represent a strong signal of account manipulation.

## 4. Cron Job Persistence

**Technique Overview**

Cron-based persistence leverages the system scheduler to execute commands at defined intervals.

Instead of requiring manual interaction or continuous access, attackers can configure tasks that run automatically in the background. This makes cron jobs an effective mechanism for maintaining execution on a compromised system over time.

**Command Used**

```bash
sudo nano /etc/cron.d/backdoor
```

A cron entry was added to simulate periodic execution of a malicious command.

**Observed Change**

The attack resulted in the creation of a new scheduled task file:

```bash
/etc/cron.d/backdoor
```

This file defines a recurring execution rule within the system scheduler.

![*Figure 7: Malicious cron job added under `/etc/cron.d`*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/07_cron_persistence.png)

*Figure 7: Malicious cron job added under `/etc/cron.d`*

### **FIM Detection**

The activity was detected as a file creation event:

```
File added
/etc/cron.d/backdoor
```

### **Security Interpretation**

Files in `/etc/cron.d`define scheduled execution within the system.

Creating a new cron job introduces a persistent execution mechanism that operates independently of user sessions.

This technique is significant because it:

- enables recurring command execution
- maintains persistence without interaction
- modifies system execution behavior

Since cron jobs are file-based, their creation produces a clear and observable artifact.

## 5. Binary Tampering

**Technique Overview**

System binaries are a fundamental part of the operating system and are generally assumed to be trusted.

Attackers may modify or introduce binaries within system directories to alter system behavior, execute malicious code, or maintain persistence. This technique is often associated with rootkits, where legitimate functionality is replaced or extended to conceal malicious activity.

**Command Used**

```bash
sudo touch /usr/bin/testbinary
echo "test" | sudo tee -a /usr/bin/testbinary
```

A test binary was created to simulate unauthorized modification within a system directory.

**Observed Change**

The action resulted in the creation of a new file in a system binary path:

```
/usr/bin/testbinary
```

This location is typically reserved for trusted executables managed by the operating system.

![*Figure 8: New binary created in `/usr/bin`*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/08_binary_tampering.png)

*Figure 8: New binary created in `/usr/bin`*

![*Figure 9: File presence confirmed in system path*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/08_binary_tampering2.png)

*Figure 9: File presence confirmed in system path*

### **FIM Detection**

The activity was detected as:

```bash
File added
/usr/bin/testbinary
```

**Security Interpretation**

Directories such as `/usr/bin`, `/bin`, and `/sbin` represent trusted execution paths.

Unauthorized additions or modifications introduce executable code into high-trust locations.

This technique is significant because it:

- leverages implicit trust in system binaries
- can affect multiple users and processes
- allows malicious code to execute as legitimate commands

Unexpected changes in these directories represent a strong signal of potential tampering.

## 6. Shell Persistence via `.bashrc`

**Technique Overview**

Shell initialization files such as `.bashrc` are executed whenever a new shell session is started.

Attackers can leverage this behavior to achieve persistence by embedding commands that execute automatically during user login or shell initialization. Unlike system-level mechanisms, this technique operates within the context of a specific user.

**Command Used**

```bash
nano ~/.bashrc
```

A test command was added to simulate automatic execution during shell startup.

**Observed Change**

The modification affected a user-level initialization file:

```
/home/user/.bashrc
```

This file is executed each time a shell session is initiated.

![*Figure 10: Malicious command added to `.bashrc` for execution on shell startup*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/09_bashrc_persistence.png)

*Figure 10: Malicious command added to `.bashrc` for execution on shell startup*

### **FIM Detection**

The activity was detected as:

```
File modified
/home/user/.bashrc
```

**Security Interpretation**

The `.bashrc` file defines commands executed during shell initialization.

Modifying this file enables user-level persistence by triggering execution during normal activity.

This technique is significant because it:

- executes automatically on login or shell start
- does not require elevated privileges
- blends with legitimate configuration changes

Although it may generate higher noise, unauthorized changes remain a useful indicator of persistence.

# Detection Results

After executing the simulated scenarios, file integrity monitoring captured a series of events corresponding to each modification performed on the system.

These events reflect how different post-exploitation techniques manifest as observable changes at the file system level.

**Observed Event Timeline**

![*Figure 11: Timeline of file integrity events generated during attack simulations*](Detecting%20Linux%20Persistence%20via%20File%20Integrity%20Mon/10_detection_summary.png)

*Figure 11: Timeline of file integrity events generated during attack simulations*

### **Summary of Detected Activity**

The following events were observed:

```
File modified: /home/user/.ssh/authorized_keys
File modified: /etc/sudoers
File modified: /etc/passwd
File modified: /etc/shadow
File added: /etc/cron.d/backdoor
File added: /usr/bin/testbinary
File modified: /home/user/.bashrc
```

### **Analysis of Results**

Across all scenarios, a consistent pattern emerges:

different attacker techniques result in modifications to a relatively small set of high-value files.

These files correspond to key system functions:

- authentication (`/etc/passwd`, `/etc/shadow`, `.ssh`)
- privilege control (`/etc/sudoers`)
- execution pathways (`cron`, `.bashrc`)
- trusted binaries (`/usr/bin`)

Despite variations in technique, the underlying behavior converges on altering system state through these components.

**Key Observations**

- Multiple attack techniques target the same core files or directories
- File-level changes provide durable evidence of attacker activity
- Both system-level and user-level persistence mechanisms are observable
- Correlated changes (e.g., `/etc/passwd` and `/etc/shadow`) increase detection confidence

### **Implication**

These results demonstrate that monitoring a focused set of critical files provides broad visibility into post-exploitation activity.

Rather than attempting to detect every possible action, observing changes to core system components offers a reliable and scalable detection approach.

This highlights that effective detection can be achieved through strategic monitoring of system state rather than exhaustive visibility into all system activity.

## **Understanding Event Types**

File Integrity Monitoring events represent changes to the state of monitored files.

While the event types themselves are simple, their significance depends on **where** the change occurs and **how it relates to system behavior**.

### **Core Event Types**

| Event Type | Meaning | Observed Examples |
| --- | --- | --- |
| File Added | New file introduced | `/etc/cron.d/backdoor`, `/usr/bin/testbinary` |
| File Modified | Existing file changed | `/etc/sudoers`, `/etc/passwd`, `.bashrc` |
| Checksum Changed | File content altered | `/etc/sudoers` |
| File Deleted | File removed | (not directly simulated) |

### **Interpretation in Context**

The same event type can have very different implications depending on the file involved.

- **File Added**
    - In system directories → may indicate persistence or unauthorized code execution
    - Example: cron job or binary introduction
- **File Modified**
    - In authentication or privilege files → high-impact changes
    - Example: `/etc/sudoers`, `/etc/passwd`
- **Checksum Changed**
    - Strong signal of content tampering
    - Particularly important for binaries and configuration files
- **File Deleted**
    - May indicate cleanup or evasion
    - Often requires correlation with prior events

### **Key Insight**

Event types alone do not determine severity.

The security relevance emerges from the **combination of event type and file context**.

For example:

- `File modified` on `/etc/sudoers` → high impact
- `File modified` on `.bashrc` → lower impact, higher noise
- `File added` in `/usr/bin` → potential execution risk

### **Implication**

Effective analysis of file integrity events requires prioritizing:

- **high-value file paths**
- **correlated changes across files**
- **deviations from expected system behavior**

Rather than treating all events equally, focusing on context allows meaningful signals to be distinguished from routine system activity.

This highlights that effective detection is not based on event volume, but on identifying meaningful changes within critical system components.

# Security Insights

This lab demonstrates that post-exploitation activity consistently manifests as changes to a small set of critical system components.

Rather than requiring broad visibility across all system activity, effective detection can be achieved by focusing on how attackers alter system state.

### **High-Value Files Provide Strong Detection Coverage**

A limited set of files governs core system behavior, including authentication, privilege control, and execution.

Examples include:

```
/etc/passwd
/etc/shadow
/etc/sudoers
~/.ssh/authorized_keys
~/.bashrc
```

Across all simulated scenarios, attacker actions converged on these locations. This indicates that monitoring a focused subset of high-impact files can provide broad coverage of common attack techniques.

### **Persistence Mechanisms Leave Durable Artifacts**

Most persistence techniques require modifying files to survive across sessions and reboots.

Observed examples:

```
SSH key insertion → ~/.ssh/authorized_keys
Cron job creation → /etc/cron.d/backdoor
Shell persistence → ~/.bashrc
```

These modifications create persistent, file-level artifacts that remain observable regardless of how the change was introduced.

### **Detection is Driven by System State, Not Activity**

Several simulated techniques did not rely on suspicious processes or network behavior.

Instead, they modified existing system files:

```
/etc/sudoers
~/.ssh/authorized_keys
```

This highlights an important distinction:

- traditional monitoring → observes activity
- file integrity monitoring → observes **state changes**

By focusing on system state, FIM provides visibility into techniques that may not generate conventional alerts.

### **Signal Quality Depends on Context**

Not all monitored files produce equally meaningful signals.

- system-level files (`/etc/sudoers`, `/etc/passwd`) → high signal, low frequency
- user-level files (`.bashrc`) → higher noise, more frequent changes

Effective monitoring requires:

- prioritizing high-impact paths
- understanding normal system behavior
- filtering low-value noise

### **Layered Visibility is Necessary**

File Integrity Monitoring provides strong visibility into persistent changes, but it does not capture all forms of attacker activity.

It is most effective when combined with:

- authentication and audit logging
- process monitoring
- network-level visibility

This layered approach ensures that both **state changes and runtime behavior** are observable.

## Key Takeaway

The results of this lab show that many attacker techniques converge on modifying a small number of critical files.

By focusing on these high-value components and interpreting changes in context, file integrity monitoring can provide a reliable and scalable method for detecting post-exploitation activity.

This suggests that effective detection does not require monitoring everything, but understanding which parts of the system attackers cannot avoid modifying.

# Limitations

While File Integrity Monitoring provides strong visibility into system state changes, it has inherent limitations that affect how results should be interpreted.

### **1. Lack of Attribution and Context**

FIM identifies *what changed*, but not *who* or *how* the change occurred.

Example:

```
File modified: /etc/passwd
```

This does not reveal:

- which user performed the action
- which process initiated the change
- whether the modification was intentional or malicious

As a result, additional data sources (e.g., audit logs) are required to provide attribution and context.

### **2. No Visibility into Runtime Behavior**

FIM observes persistent changes to disk, but does not capture in-memory or runtime activity.

It cannot detect:

- malicious processes running without modifying files
- reverse shells operating in memory
- network-based attacks

This highlights a fundamental limitation:

- FIM → observes **state**
- other systems → observe **behavior**

### **3. Signal-to-Noise Trade-off**

Not all file changes are malicious.
Legitimate system activity can generate frequent modifications, especially in user-level or application-managed files.

Examples include:

```
/etc/cups/subscriptions.conf
temporary system files
```

This creates noise in the form of:

- false positives
- alert fatigue

Effective use of FIM requires:

- prioritizing high-value paths
- excluding low-risk locations
- understanding baseline system behavior

### **4. Dependency on Coverage and Configuration**

FIM visibility is limited to monitored paths and configuration choices.

Detection gaps can occur if:

- critical directories are not included
- real-time monitoring is disabled
- the monitoring agent is tampered with

Additionally, advanced evasion techniques may involve:

- restoring original file contents after execution
- operating outside monitored locations

# Key Takeaway

File Integrity Monitoring is effective for detecting persistent changes to system state, but it does not provide complete visibility on its own.

To achieve comprehensive detection, it must be combined with:

- audit and authentication logging
- process and command monitoring
- network visibility

This reinforces that FIM is most effective as part of a **layered detection strategy**, rather than a standalone solution.

These limitations define the boundary of what file integrity monitoring can and cannot observe, making it essential to interpret its signals within a broader detection context.

# Conclusion

This lab examined how common post-exploitation techniques manifest as changes to system state, and how those changes can be observed through file integrity monitoring.

Across all simulated scenarios, a consistent pattern emerged:

different techniques ultimately rely on modifying a small set of critical files related to authentication, privilege control, and execution.

```
/etc/passwd
/etc/shadow
/etc/sudoers
~/.ssh/authorized_keys
/etc/cron.d/*
/usr/bin/*
~/.bashrc
```

These modifications create durable artifacts that remain observable regardless of how the action was performed.

This demonstrates a key principle:

- **effective detection does not require monitoring all system activity, but understanding which components attackers cannot avoid modifying.**

At the same time, the results highlight important trade-offs.

While real-time monitoring improves detection speed, meaningful analysis depends on context, prioritization, and distinguishing legitimate changes from malicious ones.

File Integrity Monitoring provides strong visibility into persistent system changes, but is most effective when combined with complementary data sources such as audit logs and process monitoring.

Ultimately, this approach shows that focusing on system state offers a reliable and scalable method for identifying post-exploitation activity in Linux environments.