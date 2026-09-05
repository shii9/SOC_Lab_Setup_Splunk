# CENTRALIZED ENTERPRISE SECURITY OPERATIONS CENTER (SOC) HOME LAB
## Multi-OS Security Log Ingestion, Telemetry Forwarding, Noise Reduction, and Detection Engineering using Splunk Enterprise 10.4.2

---

### Executive Summary

Modern cybersecurity operations require centralized visibility across diverse operating systems and network boundaries. Traditional, decentralized log management—where system logs reside isolated on individual endpoints—fails to provide timely threat visibility, hinders forensic investigation, and increases mean time to detect (MTTD). 

This project details the design, deployment, configuration, and validation of a centralized Security Information and Event Management (SIEM) laboratory. Using **Splunk Enterprise 10.4.2** hosted on a Windows 11 platform as the central receiving, indexing, search, and alerting engine, security telemetry is collected near real-time over **TCP port 9997** from heterogeneous endpoints: a **Windows 10 Virtual Machine** and a **Kali Linux Virtual Machine**, alongside local **Windows 11 Host** logs.

The laboratory successfully establishes segregated index storage (`windows11`, `windows10`, `sh-kali`), implements custom Linux `auditd` noise reduction to prevent event flooding, enforces inbound host firewall restrictions, and validates end-to-end telemetry ingestion with over **9,435+ events processed**. Furthermore, actionable Splunk Processing Language (SPL) detection rules were engineered to identify brute-force logons, unauthorized user creations, privilege escalation attempts, and sensitive file integrity breaches.

---

## 1. Project Overview & Objectives

### 1.1 Project Background
In an enterprise environment, Security Operations Center (SOC) analysts must monitor thousands of endpoints. Without central aggregation, detecting lateral movement, privilege escalation, or multi-host attacks is nearly impossible. This project creates a fully functional, small-scale SOC telemetry pipeline to emulate enterprise SIEM operations within a virtualized home lab environment.

### 1.2 Core Objectives
- **Central SIEM Deployment**: Establish a single-instance Splunk Enterprise 10.4.2 server combining Indexer and Search Head capabilities on a Windows 11 host.
- **Heterogeneous Data Collection**: Ingest native Windows Event Logs (Security, System, Application, PowerShell, Defender) and Linux telemetry (`auth.log`, `auditd`, `kern.log`, `apt/dpkg`).
- **Data Pipeline Segregation**: Structure incoming data streams into logical indexes (`windows11`, `windows10`, `sh-kali`) for efficient querying and strict data hygiene.
- **Linux Audit Noise Optimization**: Fine-tune Linux kernel `auditd` rules to eliminate excessive system call logs while capturing high-value File Integrity Monitoring (FIM) and account database changes.
- **Detection Engineering**: Develop and test correlation rules using SPL to detect security anomalies and automate alerting workflows.

---

## 2. Comparative Analysis: Traditional vs. Centralized SIEM System

| Feature / Metric | Traditional Decentralized System | Centralized Splunk SIEM Lab | SOC Impact / Advantage |
| :--- | :--- | :--- | :--- |
| **Log Storage** | Dispersed locally across endpoints | Centralized Indexer database | Instant search access; prevents log tampering by attackers |
| **Event Correlation** | Manual log collection per machine | Automated cross-platform SPL rules | Correlates Windows & Linux events simultaneously |
| **Search Speed** | Hours/Days manually parsing files | Seconds using indexed SPL queries | Drastically reduces Mean Time to Detect (MTTD) |
| **Alerting Capabilities** | None or local email/popups | Automated alerts & threshold triggers | Real-time notification of brute-force & privilege abuse |
| **Visibility & Dashboards** | No visual aggregation | Real-time graphical UI & dashboards | Executive & operational situational awareness |
| **Audit Noise Control** | Raw un-filtered file logs | Filtered telemetry (`auditd` keys) | Prevents storage exhaustion and analyst burnout |

---

## 3. System Architecture & Network Design

### 3.1 High-Level Architecture Topology

```
                       ┌──────────────────────────────────────┐
                       │          MONITORED ENDPOINTS         │
                       └──────────────────┬───────────────────┘
                                          │
            ┌─────────────────────────────┼─────────────────────────────┐
            ▼                             ▼                             ▼
   ┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
   │  Windows 10 VM  │           │  Kali Linux VM  │           │ Windows 11 Host │
   │ (Splunk UF)     │           │ (Splunk UF)     │           │ (Local Input)   │
   └────────┬────────┘           └────────┬────────┘           └────────┬────────┘
            │                             │                             │
            │ TCP 9997                    │ TCP 9997                    │ Direct Read
            └──────────────────────┬──────┴─────────────────────────────┘
                                   ▼
                   ┌──────────────────────────────────────┐
                   │  SPLUNK ENTERPRISE ON WINDOWS 11     │
                   │  • Receiving Layer (Port 9997)       │
                   │  • Indexer (Parsing & Storage)       │
                   │  • Search Head (Web UI Port 8000)    │
                   └──────────────────┬───────────────────┘
                                      │
            ┌─────────────────────────┼─────────────────────────┐
            ▼                         ▼                         ▼
   ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
   │ index=windows10 │       │  index=sh-kali  │       │ index=windows11 │
   └────────┬────────┘       └────────┬────────┘       └────────┬────────┘
            └─────────────────────────┼─────────────────────────┘
                                      ▼
                           ┌─────────────────────┐
                           │   SPL Search Engine │
                           └──────────┬──────────┘
                                      ▼
                           ┌─────────────────────┐
                           │ SOC Analyst Triage  │
                           └─────────────────────┘
```

### 3.2 Component Flow & Processing Pipeline

```mermaid
flowchart LR
    A["Log Sources<br/>(Win 10 | Kali | Win 11)"] --> B["Agent / Collector<br/>(Universal Forwarder / Local Input)"]
    B --> C["Receiving Layer<br/>(TCP Port 9997)"]
    C --> D["Indexer<br/>(Parse, Timestamp, Index)"]
    D --> E["Search Head<br/>(SPL, Dashboards, Reports)"]
    E --> F["Detection & Alerting<br/>(Rules & Thresholds)"]
    F --> G["SOC Analyst<br/>(Triage, Investigate, Respond)"]
```

### 3.3 Network & Infrastructure Configuration

| Machine Name | Operating System | IP Address | Subnet / Networking | Primary Role |
| :--- | :--- | :--- | :--- | :--- |
| **Splunk Server Host** | Windows 11 Pro x86-64 | `<Splunk_Server_IP>` | Host Adapter / LAN | SIEM Receiver, Indexer & Search Head |
| **Windows 10 Endpoint** | Windows 10 Enterprise VM | Dynamic (DHCP) | Hypervisor NAT Gateway | Monitored Windows Desktop |
| **Kali Linux Endpoint** | Kali Linux 2024.x x86-64 | `192.168.X.X/24` | Virtual Subnet (Routed) | Monitored Linux Security VM |

### 3.4 Service Port Mapping

- **Port 8000 (TCP)**: Web UI browser access for SOC analysts (`http://<Splunk_Server_IP>:8000`).
- **Port 8089 (TCP)**: Splunk REST API and management service.
- **Port 9997 (TCP)**: Encrypted Splunk-to-Splunk telemetry receiving listener.

---

## 4. Ingestion Design & Telemetry Mapping

### 4.1 Index Organization & Sourcetype Definitions

To maximize query efficiency and strictly enforce data separation, telemetry is routed to designated indexes:

| Index Name | Source Host | Monitored Channels / Log Paths | Assigned Sourcetypes |
| :--- | :--- | :--- | :--- |
| `windows11` | Windows 11 Host | Security, System, Application, PowerShell | `WinEventLog:Security`, `WinEventLog:System`, etc. |
| `windows10` | Windows 10 VM | Security, System, Application, PowerShell, Defender | `WinEventLog:Security`, `XmlWinEventLog:Sysmon` |
| `sh-kali` | Kali Linux VM | `/var/log/auth.log`, `/var/log/audit/audit.log`, `kern.log`, `apt/dpkg` | `linux_secure`, `linux_audit`, `linux_kernel`, `dpkg`, `apt_history` |

### 4.2 High-Value Windows Security Event ID Reference

The lab explicitly captures critical Windows Security Event IDs essential for threat hunting:

```
+----------+-------------------------------------------------------------+
| Event ID | Security Meaning & SOC Investigation Relevance               |
+----------+-------------------------------------------------------------+
|   4624   | Successful user logon authentication                        |
|   4625   | Failed user logon authentication (Brute-force indicator)    |
|   4648   | Logon attempted using explicit credentials (runas)          |
|   4672   | Special privileges assigned to new logon (Admin rights)     |
|   4688   | New process creation (Command line execution tracking)      |
|   4720   | User account created                                        |
|   4725   | User account disabled                                       |
|   4726   | User account deleted                                        |
|   4732   | Member added to local security group (Privilege escalation) |
|   4740   | User account locked out                                     |
|   7045   | New Windows service installed (Persistence mechanism)       |
+----------+-------------------------------------------------------------+
```

---

## 5. Detailed Step-by-Step Setup Guide

### 5.1 Central Splunk Server Installation (Windows 11)

1. Download and run `splunk-10.4.2-x64-release.msi`.
2. Complete setup and set Administrator credentials.
3. Access Splunk Web interface at `http://localhost:8000`.
4. Navigate to **Settings** -> **Forwarding and receiving** -> **Configure receiving**.
5. Click **New Receiving Port**, enter `9997`, and save.
6. Open PowerShell as Administrator and create the inbound host firewall rule:
   ```powershell
   New-NetFirewallRule -DisplayName "Splunk Forwarder 9997" -Direction Inbound -Protocol TCP -LocalPort 9997 -Action Allow
   Get-NetTCPConnection -LocalPort 9997 -State Listen
   ```

### 5.2 Local Windows 11 Ingestion Configuration

Define direct file inputs in `C:\Program Files\Splunk\etc\system\local\inputs.conf`:

```ini
[WinEventLog://Security]
disabled = 0
index = windows11

[WinEventLog://System]
disabled = 0
index = windows11

[WinEventLog://Application]
disabled = 0
index = windows11

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
index = windows11
```

### 5.3 Windows 10 VM Universal Forwarder Deployment

1. Install Splunk Universal Forwarder `splunkforwarder-10.4.2-x64-release.msi`.
2. Connect agent to central receiver:
   ```cmd
   cd "C:\Program Files\SplunkUniversalForwarder\bin"
   .\splunk.exe add forward-server <Splunk_Server_IP>:9997
   .\splunk.exe list forward-server
   ```
3. Deploy `inputs.conf` to `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`:
   ```ini
   [WinEventLog://Security]
   disabled = 0
   index = windows10

   [WinEventLog://System]
   disabled = 0
   index = windows10

   [WinEventLog://Application]
   disabled = 0
   index = windows10

   [WinEventLog://Microsoft-Windows-PowerShell/Operational]
   disabled = 0
   index = windows10
   ```

### 5.4 Kali Linux VM Forwarder Setup & Audit Tuning

1. **Install Universal Forwarder Debian Package**:
   ```bash
   wget -O splunkforwarder-10.4.2-linux-amd64.deb "https://download.splunk.com/products/universalforwarder/releases/10.4.2/linux/splunkforwarder-10.4.2-33c3bf42cd73-linux-amd64.deb"
   sudo dpkg -i splunkforwarder-10.4.2-linux-amd64.deb
   ```
2. **Start and Enable Service**:
   ```bash
   sudo /opt/splunkforwarder/bin/splunk start --accept-license
   sudo /opt/splunkforwarder/bin/splunk enable boot-start
   sudo /opt/splunkforwarder/bin/splunk add forward-server <Splunk_Server_IP>:9997
   ```
3. **Configure `rsyslog` for Auth Telemetry**:
   ```bash
   sudo apt update && sudo apt install rsyslog -y
   sudo systemctl enable --now rsyslog
   ```
4. **Deploy Noise-Controlled `auditd` Rules**:
   To prevent high event volume caused by broad syscall tracking, target high-risk files in `/etc/audit/rules.d/splunk_soc.rules`:
   ```ini
   # Account database changes
   -w /etc/passwd -p wa -k account_changes
   -w /etc/shadow -p wa -k account_changes
   -w /etc/group -p wa -k account_changes
   -w /etc/gshadow -p wa -k account_changes

   # Privilege configuration
   -w /etc/sudoers -p wa -k sudo_changes
   -w /etc/sudoers.d/ -p wa -k sudo_changes

   # SSH and persistence monitoring
   -w /etc/ssh/sshd_config -p wa -k ssh_config_changes
   -w /etc/crontab -p wa -k cron_changes

   # Lab directory file integrity monitoring
   -w /home/sh-kali/Desktop/ -p wa -k user_file_changes
   ```
   Apply audit rules:
   ```bash
   sudo augenrules --load && sudo auditctl -l
   ```
5. **Set `/opt/splunkforwarder/etc/system/local/inputs.conf`**:
   ```ini
   [monitor:///var/log/auth.log]
   disabled = false
   index = sh-kali
   sourcetype = linux_secure

   [monitor:///var/log/audit/audit.log]
   disabled = false
   index = sh-kali
   sourcetype = linux_audit

   [monitor:///var/log/kern.log]
   disabled = false
   index = sh-kali
   sourcetype = linux_kernel

   [monitor:///var/log/dpkg.log]
   disabled = false
   index = sh-kali
   sourcetype = dpkg

   [monitor:///var/log/apt/history.log]
   disabled = false
   index = sh-kali
   sourcetype = apt_history
   ```
   Restart forwarder: `sudo /opt/splunkforwarder/bin/splunk restart`.

---

## 6. Verification & Testing Suite

The system functionality was validated through 5 formal test cases:

```
+---------+------------------------------+---------------------------------------+---------------------------------------+--------+
| Test ID | Test Scenario                | Input / Action                        | Expected Result                       | Status |
+---------+------------------------------+---------------------------------------+---------------------------------------+--------+
|  TC-01  | TCP Listener Check           | netstat / Get-NetTCPConnection        | Port 9997 listed in LISTEN state      | PASS   |
|  TC-02  | Forwarder Connectivity       | splunk list forward-server on endpoints| Active forward to <Splunk_Server_IP>:9997  | PASS   |
|  TC-03  | Windows Event Ingestion      | Generate failed logon on Win 10 VM    | EventCode=4625 appears in windows10   | PASS   |
|  TC-04  | Kali Auth Log Ingestion      | Run invalid `sudo` command on Kali    | "authentication failure" in sh-kali   | PASS   |
|  TC-05  | Auditd FIM Verification      | `touch /home/sh-kali/Desktop/test.txt`| `key="user_file_changes"` in sh-kali  | PASS   |
+---------+------------------------------+---------------------------------------+---------------------------------------+--------+
```

### Ingestion Volume Proof
Executing `index=* | stats count by host index` confirmed **9,435 total ingested events** across all three hosts.

---

## 7. SOC Detection Engineering & SPL Rule Suite

### Use Case 1: Windows Brute Force Authentication Detection
- **Objective**: Detect potential brute-force or password spraying attacks targeting Windows user accounts.
- **SPL Detection Logic**:
  ```spl
  index=windows10 EventCode=4625 
  | stats count by TargetUserName, WorkstationName, src_ip 
  | where count >= 5
  ```
- **Triage Workflow**: Verify if username is valid, inspect source IP, cross-reference Event Code 4624 for subsequent successful logon.

### Use Case 2: Linux Failed Sudo / Privilege Abuse
- **Objective**: Identify unauthorized privilege escalation attempts by non-root users on Kali Linux.
- **SPL Detection Logic**:
  ```spl
  index=sh-kali sourcetype=linux_secure "authentication failure" 
  | table _time, host, process, message
  ```
- **Triage Workflow**: Correlate user ID (`auid`) with SSH logon session to determine if a compromised user account is attempting escalation.

### Use Case 3: Linux Account Database Modification (Tampering)
- **Objective**: Detect unauthorized creation, modification, or deletion of local Linux accounts (`/etc/passwd`, `/etc/shadow`).
- **SPL Detection Logic**:
  ```spl
  index=sh-kali sourcetype=linux_audit key="account_changes" 
  | table _time, host, exe, name, auid
  ```
- **Triage Workflow**: High Severity Alert. Validate binary (`exe`) executing write action. Legitimate changes should trace back to `/usr/sbin/useradd` or `/usr/bin/passwd` during approved maintenance.

### Use Case 4: Windows Local User Account Creation
- **Objective**: Alert on newly created local administrator or back-door accounts on Windows endpoints.
- **SPL Detection Logic**:
  ```spl
  index=windows10 EventCode=4720 
  | table _time, TargetUserName, SubjectUserName, host
  ```
- **Triage Workflow**: Identify `SubjectUserName` (who created the account) and check if account creation followed change management approval.

### Use Case 5: File Integrity Monitoring (FIM) on Desktop Directory
- **Objective**: Monitor sensitive desktop directory modifications on Linux workstation.
- **SPL Detection Logic**:
  ```spl
  index=sh-kali sourcetype=linux_audit key="user_file_changes" 
  | table _time, host, key, name, SYSCALL, auid
  ```

---

## 8. Troubleshooting & Problem Resolution Matrix

| Problem Scenario | Root Cause Analysis | Corrective Resolution Action |
| :--- | :--- | :--- |
| **PowerShell Access Denied** | Execution attempted under standard user privileges | Relaunch PowerShell using **Run as Administrator**. |
| **Kali `nc` inverse host lookup warning** | Missing reverse DNS entry for `<Splunk_Server_IP>` | Benign warning; confirm output reports `open` on 9997. |
| **Forwarder Active Status Inactive** | Host firewall blocking port 9997 or service down | Verify `New-NetFirewallRule` and confirm service status. |
| **Missing `/var/log/auth.log` on Kali** | Systemd journald default storage | Install `rsyslog` service (`sudo apt install rsyslog -y`). |
| **Audit Log Volume Flooding** | Broad `-a always,exit -S execve` global rule | Remove global command tracking; deploy focused audit key rules. |

---

## 9. Security & Operational Hardening Best Practices

1. **Network Segmentation**: Isolate laboratory virtual machines using Host-Only or NAT network adapters to prevent external exposure.
2. **Firewall Scoping**: Restrict inbound firewall rules on port 9997 to explicit VM subnets (`192.168.X.X/24`) rather than `Any`.
3. **Credential Security**: Store Splunk credentials securely and never commit plain-text passwords into version control.
4. **Time Synchronization**: Keep all monitored machines synchronized via NTP to ensure accurate cross-platform event correlation timelines.

---

## 10. Future Enhancements & Project Roadmap

- **Microsoft Sysmon Integration**: Deploy Sysmon with Olaf Hartong modular XML configurations to capture process line parentage, DNS queries, and network connection events.
- **Network Threat Detection (NIDS)**: Integrate Suricata or Zeek to ship packet metadata into Splunk.
- **Active Directory Domain Controller VM**: Introduce Windows Server 2022 to log Kerberos tickets and Active Directory attack vectors (Kerberoasting).
- **MITRE ATT&CK Mapping**: Tag custom SPL alerts directly to MITRE ATT&CK technique IDs (e.g., T1078, T1136, T1068).

---

## 11. Conclusion

This project successfully implemented a resilient, multi-OS centralized SOC Home Lab using Splunk Enterprise 10.4.2. By combining native Windows event logging, Linux `rsyslog`, and targeted `auditd` FIM telemetry, the lab achieved complete end-to-end visibility into security events across host and virtual environments. The deployment of custom SPL detection rules proves that effective SIEM operations depend not only on log ingestion, but on disciplined data filtering, index segregation, and actionable alert engineering.

---
*Report Compiled & Validated for SOC Engineering Portfolio.*
