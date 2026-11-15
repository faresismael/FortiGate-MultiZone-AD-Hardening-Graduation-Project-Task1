# 🛡️ Enterprise Multi-Zone Network & Windows Server 2025 Security Hardening (Graduation Project - Task 1)

![GNS3](https://img.shields.io/badge/Simulator-GNS3-blue?style=flat-square)
![FortiGate](https://img.shields.io/badge/Firewall-FortiGate%20v7.4.12-red?style=flat-square)
![Windows Server](https://img.shields.io/badge/OS-Windows%20Server%202025-0078D6?style=flat-square)
![Ubuntu](https://img.shields.io/badge/Client-Ubuntu%20Linux-E95420?style=flat-square)
![Windows 10](https://img.shields.io/badge/Client-Windows%2010%20Pro-00ADEF?style=flat-square)

## 📌 Project Overview
This repository documents **Task 1** of our Electronics & Communications Engineering Graduation Project (**Automated Security Response System for Enterprise Networks**). This foundational phase builds and hardens an enterprise-grade, segmented network architecture using **GNS3**, protected at the perimeter by a **FortiGate Next-Generation Firewall (NGFW)** and governed internally by a **Windows Server 2025 Active Directory Domain Controller (`faresismael.com`)**.

Beyond basic multi-zone routing, this implementation enforces **Endpoint Hardening (13 Security Policies)**, **Role-Based Access Control (RBAC)** adhering to the *Principle of Least Privilege*, full **Policy Enforcement Verification** (CMD, Control Panel, Removable DVD/USB, BitLocker, Account Lockout), and **Cross-Platform Client Connectivity** across both **Windows 10** and **Ubuntu Linux**.

---

## 🏗️ Network Architecture & IP Addressing

![GNS3 Multi-Zone Topology](screenshots/01-gns3-topology.png)

The topology is segmented into three isolated security zones controlled by **FortiGate 7.4.12**:

| Security Zone | FortiGate Interface | Gateway IP | Connected Devices | Endpoint IPs | Role & Security Level |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **External Zone (WAN)** | `Port1` | DHCP / NAT | `Cloud1` (`eth1`) | External / ISP | Untrusted internet access via NAT |
| **DMZ Zone** | `Port2` | `172.16.10.1/24` | `Switch2` ➔ `WindowsServer2025V2-1` | `172.16.10.30` | Hosted Core Infrastructure (AD DS, DNS, GPO) |
| **Trust Zone (LAN)** | `Port3` | `192.168.20.1/24` | `Switch1` ➔ `Ubuntu64` & `Windows10x64` | `.15` (Ubuntu)<br>`.25` (Win10) | Internal corporate workstations (Windows & Linux) |

---

## 🔥 FortiGate Firewall & Traffic Control
* **Zone Segmentation:** Strict interface isolation between **Trust**, **DMZ**, and **External** zones.
* **Firewall Policies & NAT:**
  * Controlled **Trust ➔ DMZ** policy permitting domain-joined workstations (`192.168.20.0/24`) to authenticate and pull Group Policies from the Domain Controller (`172.16.10.30`).
  * Secure **Trust / DMZ ➔ External** policies with Source NAT (SNAT) enabled for internet access and system updates on Ubuntu and Windows clients.

![FortiGate Configuration](screenshots/02-fortigate-interfaces.png)

---

## 🏢 Active Directory Structure & Least Privilege (RBAC)

Within the **`faresismael.com`** domain, objects are structured under the root container **`ECE_Grad_Project`** across three dedicated Organizational Units (OUs):

![Active Directory OU Structure](screenshots/03-ad-ou-structure.png)

1. **`Project_Team` (Standard Users):** Contains team/employee accounts (`fares`, `mina`, etc.) subjected to strict endpoint restrictions.
2. **`Project_Computers` (Workstations):** Contains domain-joined client machines (`Windows10x64`).
3. **`IT_Department` (Technical Support):** Contains `IT_Support_Group` and dedicated IT support accounts (`Fares-IT`, `ROIT`).

### 🔐 Privilege Separation (Avoiding `Domain Admins` Over-Privileging)
Instead of granting IT support staff high-risk `Domain Admins` privileges, granular access was engineered using two mechanisms:
* **Delegation of Control:** Delegated authority exclusively on `Project_Team` OU, allowing `IT_Support_Group` to read user attributes and reset forgotten passwords without touching the Domain Controller.
* **Restricted Groups (`GPO_12`):** Elevated `IT_Support_Group` to **Local Administrator** solely on `Project_Computers` workstations (while preserving `Domain Admins`), enabling full local troubleshooting while keeping the domain completely isolated.

| Delegation of Control & Account Lockout | Restricted Groups (`Local Administrators`) |
| :---: | :---: |
| ![Delegation and Lockout](screenshots/05-delegation-and-lockout.png) | ![Restricted Groups Configuration](screenshots/06-it-restricted-groups.png) |

---

## 📜 Implemented Security Policies (12 GPOs + Domain Lockout)

![Group Policy Management](screenshots/04-gpo-list.png)

### A. User-Side Hardening (`Project_Team` OU)
| GPO Name | Scope | Security Objective |
| :--- | :--- | :--- |
| **`GPO_01_Block_CMD`** | User | Disables Command Prompt (`cmd.exe`) and batch script execution. |
| **`GPO_02_Block_Control_Panel`** | User | Blocks access to Control Panel and Windows Settings to prevent system tampering. |
| **`GPO_03_Block_Network_Properties`** | User | Locks TCP/IP and NIC properties to prevent IP/DNS spoofing or firewall bypass. |
| **`GPO_04_Block_USB_Storage`** | User | Denies read/write access to removable USB drives to prevent data exfiltration. |
| **`GPO_05_Block_Task_Manager`** | User | Disables Task Manager to prevent users from terminating security/monitoring processes. |
| **`GPO_06_Block_Registry`** | User | Blocks Registry Editor (`regedit.exe`) to protect core OS security keys. |
| **`GPO_07_Block_CD_DVD`** | User | Restricts CD/DVD optical media access as an additional DLP layer. |
| **`GPO_08_Block_PowerShell`** | User | Blocks Windows PowerShell execution to mitigate unauthorized scripting attacks. |

### B. Computer-Side & Domain-Wide Policies
| Policy Name | Target | Scope | Security Objective |
| :--- | :--- | :--- | :--- |
| **`GPO_09_Logon_Security_Banner`** | `Project_Computers` | Computer | Displays an authorized-use legal warning banner prior to Windows logon. |
| **`GPO_10_Disk_Quota_5GB`** | `Project_Computers` | Computer | Enforces a strict **5 GB per-user NTFS disk quota** to prevent storage exhaustion. |
| **`GPO_11_BitLocker_Encryption`** | `Project_Computers` | Computer | Configures BitLocker OS drive encryption policy (with non-TPM support for virtualized environments). |
| **`GPO_12_IT_Local_Admin`** | `Project_Computers` | Computer | Grants `IT_Support_Group` Local Administrator rights on workstations via `Restricted Groups`. |
| **`Account Lockout Policy`** | `Default Domain Policy` | Domain | Protects against brute-force attacks by locking accounts for **30 minutes** after **5 invalid logon attempts**. |

---

## ✅ Policy Enforcement & Verification

All security controls were practically tested on domain-joined clients to guarantee enforcement:
* **Endpoint Restrictions (`Project_Team`):** Confirmed immediate access denial when attempting to launch `CMD`, `Control Panel`, `PowerShell`, `Task Manager`, or access **CD/DVD** and **USB** drives.
* **Drive Encryption (`BitLocker`):** Verified BitLocker policy deployment and drive encryption management directly via Control Panel (`GPO_11`).
* **Brute-Force Protection (`Account Lockout`):** Verified automatic account lockout after 5 consecutive failed login attempts and administrative unlock from AD DS.
* **IT Support Elevation (`Fares-IT`):** Verified seamless login with administrative privileges on `Windows10x64` without Domain Admin rights.
* **Ubuntu Linux Client (`Ubuntu64`):** Verified network reachability to the gateway (`192.168.20.1`), DMZ server (`172.16.10.30`), and external internet via FortiGate NAT.

| System Tools Blocked (`CMD` / `Control Panel`) | Removable Media Blocked (`DVD` / `USB`) |
| :---: | :---: |
| ![System Tools Blocked](screenshots/07-win10-block-cmd-controlpanel.png) | ![DVD and USB Blocked](screenshots/08-win10-block-dvd-usb.png) |

| BitLocker Policy Verification (`GPO_11`) | IT Support Full Admin Access (`Fares-IT`) |
| :---: | :---: |
| ![BitLocker Verification](screenshots/09-bitlocker-verification.png) | ![IT Admin Access](screenshots/10-win10-it-admin-access.png) |

![Ubuntu Connectivity Verification](screenshots/11-ubuntu-connectivity.png)

---

## 🛠️ Tech Stack & Tools
* **Network Simulation:** GNS3 v2.2.52 + GNS3 VM
* **Firewall:** Fortinet FortiGate VM (v7.4.12)
* **Server OS:** Windows Server 2025 (AD DS, DNS, Group Policy Management)
* **Client OS:** Windows 10 Pro (x64) & Ubuntu Linux (x64)

---

## Future Enhancements

* **Management VLAN (VLAN 40)** — A dedicated out-of-band VLAN for secure device management (SSH to firewalls/switches), isolated from user and server data traffic and restricted by ACLs to an admin host.
* **SIEM/XDR & MITRE ATT&CK Integration (Wazuh + Sysmon)** — Deploying Wazuh agents and Sysmon across Windows and Linux endpoints for centralized log ingestion, real-time alerting, and MITRE ATT&CK technique mapping.
* **AI Risk Scoring Engine** — Building an intelligent risk analysis module that evaluates incoming SIEM alerts and assigns dynamic severity scores (0–100) to prioritize critical threats.
* **Automated Incident Response & SOAR (Ansible & Python)** — Automating containment playbooks to trigger zero-touch attacker IP blocking on FortiGate/iptables, endpoint quarantine, and automated recovery.
* **Software-Defined Networking (SDN)** — Integrating Ryu Controller and Open vSwitch (OVS) for programmable flow-level traffic isolation.
* **Red Team Attack Simulation (Kali Linux)** — Validating the full detection-to-recovery pipeline (`Attack → Detect → Alert → Automate → Isolate → Recover`) using Hydra, Nmap, and Metasploit.

---

## Author

**Fares Ismael** | HCIA-Security

[LinkedIn](https://www.linkedin.com/in/fares-ismael-058976425/)
