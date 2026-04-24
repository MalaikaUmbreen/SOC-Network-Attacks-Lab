# 🛡️ SOC Home Lab — Network Attacks & Detection

![SOC Lab](https://img.shields.io/badge/SOC-Home%20Lab-blue?style=for-the-badge)
![Kali Linux](https://img.shields.io/badge/Kali-Linux-557C94?style=for-the-badge&logo=kali-linux)
![Wazuh](https://img.shields.io/badge/Wazuh-SIEM-00A651?style=for-the-badge)
![Windows 10](https://img.shields.io/badge/Windows%2010-Pro-0078D6?style=for-the-badge&logo=windows)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Lab%20Status-Completed-success?style=for-the-badge)

> **A complete home SOC lab simulating real-world network attacks monitored with Wazuh SIEM — covering Port Scanning, OS Fingerprinting, Service Enumeration, and SMB Enumeration with full MITRE ATT&CK mapping and professional incident reporting.**

---

## 📋 Table of Contents

- [Lab Overview](#-lab-overview)
- [Lab Environment](#-lab-environment)
- [Network Topology](#-network-topology)
- [Phase 1 — Port Scanning](#-phase-1--port-scanning-nmap)
- [Phase 2 — Service Enumeration](#-phase-2--service-enumeration)
- [Phase 3 — SMB Enumeration](#-phase-3--smb-enumeration)
- [Wazuh SIEM Detection](#-wazuh-siem-detection)
- [MITRE ATT&CK Mapping](#️-mitre-attck-mapping)
- [Key Findings](#-key-findings)
- [Repository Structure](#-repository-structure)
- [Disclaimer](#️-disclaimer)

---

##  Lab Overview

This lab demonstrates a **Real SOC analyst workflow** — performing network attacks from Kali Linux against a Windows 10 Pro victim, then detecting and analyzing them in Wazuh SIEM. The core goal is to correlate each attack tool with its SIEM alert and MITRE ATT&CK technique — exactly how a Tier 1 SOC analyst works.

<img width="446" height="356" alt="image" src="https://github.com/user-attachments/assets/ba36fa30-0094-4af9-a164-7244393e98c4" />

### Attacks Performed

| Phase | Attack Type | Tool Used | MITRE ID |
|---|---|---|---|
| 1 | Network Scanning | Nmap `-sn`, `-sS` | T1046 |
| 1 | OS Fingerprinting | Nmap `-O` | T1082 |
| 2 | Service Enumeration | Nmap NSE scripts | T1592 |
| 2 | Banner Grabbing | Netcat | T1592 |
| 2 | Web Enumeration | Nikto | T1592 |
| 3 | SMB Share Enumeration | smbclient, enum4linux | T1135 |
| 3 | SMB Vulnerability Check | Nmap smb-vuln scripts | T1210 |

### Results Summary

| Metric | Value |
|---|---|
| Total Wazuh Alerts Generated | **764** |
| Attack Sessions | 2 (Apr 19 & Apr 22, 2026) |
| Highest Severity Alert | Level 9 — Rule 92201 |
| MITRE Techniques Triggered | 6 |
| Log Files Capturing Attacks | 3 |

---

## 🖥️ Lab Environment

### Virtual Machines

| Role | OS | Notes |
|---|---|---|
| **Attacker** | Kali Linux | Nmap, enum4linux, smbclient, Nikto |
| **Victim** | Windows 10 Pro | Wazuh agent installed, SSH/RDP/SMB enabled |
| **SIEM** | Ubuntu — Wazuh Server | Log collection, alerts, threat hunting |
| **Firewall** | pfSense | Network segmentation layer |
| **DMZ Target** | Ubuntu-DMZ | Additional Linux target |

### Network Configuration

```
Attacker IP  : 192.168.XX.XX  (Kali Linux)
Victim IP    : 192.168.XX.XX  (Windows 10 Pro)
Wazuh Server : 192.168.XX.XX  (Ubuntu)
Network Type : VMware NAT / Host-Only
```

> ⚠️ All IP addresses are anonymized as `192.168.XX.XX` for security.

### Victim Setup (Windows 10 Pro — PowerShell as Admin)

```powershell
# Enable SMB
Set-Service -Name LanmanServer -StartupType Automatic
Start-Service -Name LanmanServer
Enable-NetFirewallRule -DisplayGroup "File and Printer Sharing"

# Create lab share
New-Item -ItemType Directory -Path "C:\LabShare"
New-SmbShare -Name "LabShare" -Path "C:\LabShare" -FullAccess Everyone

# Enable RDP
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Set-Service -Name TermService -StartupType Automatic
Start-Service -Name TermService

# Enable SSH
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
New-NetFirewallRule -Name sshd -DisplayName "OpenSSH Server" -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22

# Enable audit logging for Wazuh
auditpol /set /category:* /success:enable /failure:enable
```

### Verify Ports Are Listening

```powershell
netstat -an | findstr "LISTENING"
# Expected: 0.0.0.0:22, 0.0.0.0:135, 0.0.0.0:139, 0.0.0.0:445, 0.0.0.0:3389
```

---

## 🌐 Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    VMware Workstation                        │
│                                                             │
│  ┌─────────────┐         ┌──────────────────┐              │
│  │ Kali Linux  │─────────▶│  Windows 10 Pro  │             │
│  │  Attacker   │  Attack  │     Victim        │             │
│  │192.168.XX.XX│  Traffic │  192.168.XX.XX   │             │
│  └─────────────┘         │  Ports Open:      │             │
│                           │  22  (SSH)        │             │
│  ┌─────────────┐         │  135 (MSRPC)      │             │
│  │   pfSense   │         │  139 (NetBIOS)    │             │
│  │  Firewall   │         │  445 (SMB)        │             │
│  └─────────────┘         │  3389 (RDP)       │             │
│                           └────────┬─────────┘             │
│  ┌─────────────┐                   │ Logs forwarded         │
│  │ Ubuntu-DMZ  │         ┌─────────▼─────────┐             │
│  │ Extra Target│         │  Ubuntu-Wazuh      │             │
│  └─────────────┘         │  SIEM Server       │             │
│                           │  192.168.XX.XX    │             │
│                           └───────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔴 Phase 1 — Port Scanning (Nmap)

**Objective:** Discover live hosts and enumerate open ports on the target.

### Step 1.1 — Network Discovery (Ping Sweep)

```bash
nmap -sn 192.168.XX.XX/24
```

**Results:** 5 live hosts discovered including victim at `192.168.XX.XX`.

### Step 1.2 — Full Port Scan with Version & OS Detection

```bash
nmap -sV -sC -O -p- -T4 192.168.XX.XX -oN scan_results.txt
```

| Flag | Meaning |
|---|---|
| `-sV` | Detect service versions |
| `-sC` | Run default NSE scripts |
| `-O` | OS fingerprinting |
| `-p-` | Scan all 65535 ports |
| `-T4` | Aggressive timing |
| `-oN` | Save output to file |

**Ports Discovered Open:**

| Port | State | Service | Version |
|---|---|---|---|
| 22/tcp | Open | SSH | OpenSSH for Windows 7.7 |
| 135/tcp | Open | MSRPC | Microsoft Windows RPC |
| 139/tcp | Open | NetBIOS | Microsoft Windows netbios-ssn |
| 445/tcp | Open | SMB | Microsoft-DS |
| 3389/tcp | Open | RDP | MS-WBT-Server |

**OS Fingerprint:** Microsoft Windows 10 22H2 — **97% confidence**

### Step 1.3 — Stealth SYN Scan (IDS Alert Generator)

```bash
nmap -sS -p 22,80,139,445,3389 192.168.XX.XX
```

### Step 1.4 — UDP Scan

```bash
nmap -sU --top-ports 20 192.168.XX.XX
```

**UDP Services Found:** SNMP (161), NetBIOS (137/138), DHCP (67/68), Syslog (514)

---

##  Phase 2 — Service Enumeration

**Objective:** Gather detailed version and configuration info from open services.

### Step 2.1 — Banner Grabbing with Netcat

```bash
nc -nv 192.168.XX.XX 22
nc -nv 192.168.XX.XX 80
nc -nv 192.168.XX.XX 445
```

### Step 2.2 — Targeted NSE Scripts

```bash
# SSH — enumerate algorithms and auth methods
nmap -p 22 --script=ssh-auth-methods,ssh2-enum-algos 192.168.XX.XX

# HTTP — title, headers, common paths
nmap -sV -p 80 --script=http-title,http-headers,http-enum 192.168.XX.XX

# FTP — check anonymous login
nmap -p 21 --script=ftp-anon,ftp-syst 192.168.XX.XX
```

**SSH Results:** 10 key exchange algorithms, 5 host key algorithms (RSA, ECDSA, ED25519), 6 encryption algorithms detected.

### Step 2.3 — Web Enumeration with Nikto

```bash
nikto -h http://192.168.XX.XX
```

### Step 2.4 — Deep Version Detection (Ports 1–1000)

```bash
nmap -sV -p 1-1000 --version-intensity 9 192.168.XX.XX
```

**Results:** Full service strings captured — SSH, MSRPC, NetBIOS-SSN, Microsoft-DS.

---

## 🟣 Phase 3 — SMB Enumeration

**Objective:** Enumerate SMB shares, users, groups, and check for known vulnerabilities.

> SMB runs on **ports 139 and 445** — this phase generates the most Wazuh alerts.

### Step 3.1 — SMB OS Discovery

```bash
nmap -p 139,445 --script=smb-os-discovery 192.168.XX.XX
```

**Result:** Ports 139 (netbios-ssn) and 445 (microsoft-ds) confirmed open. MAC: 00:0C:29:C2:1F:B4 (VMware).

### Step 3.2 — Share Enumeration with smbclient

```bash
# List all shares via null session
smbclient -L //192.168.XX.XX -N

# Connect to specific shares
smbclient //192.168.XX.XX/IPC$ -N
smbclient //192.168.XX.XX/LabShare -N
```

### Step 3.3 — enum4linux (All-in-One SMB Recon)

```bash
enum4linux -a 192.168.XX.XX
```

**Dumps:** Users, groups, shares, password policy, OS info, RID brute force.

**Results:**
- Domain/Workgroup: `WORKGROUP`
- Users enumerated: administrator, guest, krbtgt, root
- File Server Service: **ACTIVE**
- Workstation Service: **ACTIVE**

### Step 3.4 — enum4linux-ng (Modern Version)

```bash
enum4linux-ng -A 192.168.XX.XX
```

### Step 3.5 — SMB Vulnerability Checks

```bash
# EternalBlue (MS17-010)
nmap -p 445 --script=smb-vuln-ms17-010 192.168.XX.XX

# All SMB vulnerability scripts
nmap -p 445 --script=smb-vuln-* 192.168.XX.XX

# SMB security mode check
nmap -p 445 --script=smb-security-mode 192.168.XX.XX
```

**EternalBlue Result:** `smb-vuln-ms17-010: false` — **NOT VULNERABLE** (patched system )

---

## 🟢 Wazuh SIEM Detection

### Agent Verification

```powershell
# On Windows 10 victim
Get-Service -Name WazuhSvc
# Status: Running 
```

### Live Log Monitoring (On Wazuh Server)

```bash
# Watch live alerts
sudo tail -f /var/ossec/logs/alerts/alerts.log

# Count alerts by rule
sudo grep "Rule:" /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn | head -20

# Find attacker IP across all log files
sudo find /var/ossec/logs/alerts/ -name "*.log" -exec grep -l "192.168.XX.XX" {} \;
```

### Alert Summary — 764 Total Alerts

| Count | Rule ID | Level | Description | Triggered By |
|---|---|---|---|---|
| **457** | 60107 | 4 | Failed attempt to perform privileged operation | Nmap + SMB scans |
| **231** | 92217 | 6 | Executable dropped in Windows root folder | SMB enumeration |
| **134** | 67027 | 3 | A process was created | NSE scripts |
| **46** | 92154 | 4 | taskschd.dll module loaded | Nmap NSE |
| **24** | 92201 | **9** | PowerShell created scripting file | enum4linux |
| **18** | 92031 | 3 | Discovery activity executed | Nmap scans |

### Wazuh Dashboard Filters

```
Threat Hunting → rule.id: 92031    (Nmap network discovery)
Threat Hunting → rule.id: 92217    (SMB enumeration)
Threat Hunting → rule.id: 60107    (All scan attempts)
MITRE ATT&CK  → T1046              (Network Service Scanning)
MITRE ATT&CK  → T1135              (Network Share Discovery)
```

### Key SOC Lesson — How to Find Attacker IP

> Raw grep on `alerts.log` shows **victim-side events only**.
> Attacker IP is stored inside **JSON fields** (`data.srcip`).
> Use the **Wazuh Dashboard** → Threat Hunting → expand any alert row → look for `data.srcip`.

```bash
# Attacker IP confirmed in these 3 log files:
/var/ossec/logs/alerts/2026/Apr/ossec-alerts-19.log
/var/ossec/logs/alerts/2026/Apr/ossec-alerts-22.log
/var/ossec/logs/alerts/alerts.log
```

---

## 🗺️ MITRE ATT&CK Mapping

| Attack Performed | Tactic | Technique | ID | Wazuh Rule |
|---|---|---|---|---|
| `nmap -sn` subnet scan | Discovery | Network Service Discovery | **T1046** | 92031 |
| `nmap -O` OS fingerprint | Discovery | System Information Discovery | **T1082** | 60107 |
| `enum4linux` SMB recon | Discovery | Network Share Discovery | **T1135** | 92217 |
| Service enumeration | Reconnaissance | Gather Victim Host Info | **T1592** | 67027 |
| Banner grabbing | Reconnaissance | Gather Victim Network Info | **T1590** | 67027 |
| SMB share access | Lateral Movement | SMB/Windows Admin Shares | **T1021.002** | 60107 |

### ATT&CK Tactic Flow

```
TA0043 Reconnaissance
    ├── T1590   Gather Victim Network Information
    └── T1592   Gather Victim Host Information

TA0007 Discovery
    ├── T1046   Network Service Scanning
    ├── T1082   System Information Discovery
    └── T1135   Network Share Discovery

TA0008 Lateral Movement (simulated)
    └── T1021.002   SMB/Windows Admin Shares
```

---

## 🔑 Key Findings

### Detections

1. **764 total alerts** generated across two attack sessions (April 19 & 22)
2. **Rule 92031** — Wazuh directly flagged Nmap network discovery scans
3. **Rule 92217 (Level 6)** — SMB enumeration caught as executable dropped in Windows root
4. **Rule 92201 (Level 9)** — enum4linux triggered highest severity alert in session
5. Attacker IP confirmed present in **3 separate log files**
6. All 6 MITRE ATT&CK techniques successfully triggered and mapped

### Defensive Recommendations

| Priority | Action |
|---|---|
| 🔴 High | Disable SMBv1: `Set-SmbServerConfiguration -EnableSMB1Protocol $false` |
| 🔴 High | Restrict ports 139/445 to authorized hosts only via firewall ACL |
| 🟡 Medium | Enable Wazuh active response to auto-block scanning source IPs |
| 🟡 Medium | Configure pfSense IDS/IPS rules (Snort or Suricata) |
| 🟢 Low | Deploy honeypot SMB shares for early enumeration detection |
| 🟢 Low | Conduct full CIS Benchmark remediation for Windows 10 Pro |

---
## 📁 Repository Structure

soc-network-attacks-lab/
│
├── README.md                             ← Lab overview, commands & MITRE mapping
│
├── reports/
│   ├── SOC-Network_Attacks.pdf           ← Full lab documentation (PDF)
│   ├── SOC_Incident_Report_Network_Attacks.docx  ← Professional SOC incident report
│   └── placeholder.md
│
└── screenshots/
    ├── Network_Attacks_Phases             ← Attack phase diagram
    ├── SIEM_logs.png                      ← Wazuh SIEM log evidence
    ├── Screenshot 2026-04-22 160341...    ← Nmap scan results
    ├── Screenshot 2026-04-22 160547...    ← Port scan output
    ├── Screenshot 2026-04-22 160602...    ← Service enumeration
    ├── Screenshot 2026-04-22 160706...    ← SMB enumeration
    ├── Screenshot 2026-04-22 161134...    ← enum4linux output
    ├── Screenshot 2026-04-22 161242...    ← Wazuh alerts dashboard
    ├── Screenshot 2026-04-22 161251...    ← 764 hits — Rule 60107
    ├── Screenshot 2026-04-23 125836...    ← Threat Hunting view
    ├── Screenshot 2026-04-23 125901...    ← Rule 92031 discovery alerts
    ├── Screenshot 2026-04-23 125932...    ← Rule 92217 SMB alerts
    ├── Screenshot 2026-04-23 130227...    ← Rule 67027 process creation
    ├── Screenshot 2026-04-23 130234...    ← MITRE ATT&CK T1046 mapped
    ├── Screenshot 2026-04-23 130246...    ← MITRE ATT&CK T1135 mapped
    ├── Screenshot 2026-04-23 130318...    ← Wazuh log terminal output
    ├── Screenshot 2026-04-23 131336...    ← Alert count verification
    ├── Screenshot 2026-04-23 135814...    ← Attacker IP log evidence
    ├── Screenshot 2026-04-23 135837...    ← CIS Benchmark violations
    ├── Screenshot 2026-04-23 140136...    ← enum4linux users dumped
    ├── Screenshot 2026-04-23 140147...    ← SMB shares enumerated
    ├── Wazu-Server-Terminal_logs.png      ← Wazuh server terminal log evidence
    └── placeholder.md


---

## 🔗 References

| Resource | URL |
|---|---|
| Nmap Documentation | https://nmap.org/docs.html |
| Wazuh Documentation | https://documentation.wazuh.com |
| MITRE ATT&CK Framework | https://attack.mitre.org |
| SMB Enumeration Guide | https://arnavtripathy98.medium.com/smb-enumeration-for-penetration-testing-e782a328bf1b |
| Wazuh Rules Reference | https://github.com/wazuh/wazuh-ruleset |
| enum4linux Tool | https://www.kali.org/tools/enum4linux |

---

## ⚠️ Disclaimer

> This lab is conducted entirely in a **controlled private home lab environment** for educational and training purposes only. All attacks are performed against virtual machines owned and operated by the researcher. Never perform these techniques against any system you do not own or have explicit written permission to test. Unauthorized use of these techniques is illegal.

---

## 👤 About This Lab

**Series:** SOC Home Lab Training
**Part:** Network Attacks (Part 2 of ongoing series)
**Previous Lab:** Brute Force Attacks
**Platform:** VMware Workstation — Windows 10 Pro + Kali Linux + Wazuh SIEM

*Building real SOC analyst skills through hands-on offensive and defensive security practice.*

---

![Level](https://img.shields.io/badge/Level-Intermediate-orange?style=flat-square)
![Series](https://img.shields.io/badge/Series-SOC%20Home%20Lab-blue?style=flat-square)
![Tools](https://img.shields.io/badge/Tools-Nmap%20%7C%20enum4linux%20%7C%20Wazuh-green?style=flat-square)
