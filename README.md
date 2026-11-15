
# 🛡️ Enterprise Multi-Zone Network & Windows Server 2025 Security Hardening (Graduation Project - Task 1)        




## 📌 Overview
**Task 1** of the **Automated Security Response System for Enterprise Networks** graduation project[cite: 13]. This foundational phase builds and secures an enterprise-grade, multi-zone network architecture in **GNS3**, protected at the perimeter by a **FortiGate Next-Generation Firewall (NGFW)** with **Virtual IPs (DNAT)**, hosting core enterprise services (**AD DS, DNS, Web, FTP, and File Sharing**) in a segmented **DMZ**, and enforcing **13 Security Policies** alongside **Least-Privilege Role-Based Access Control (RBAC)** across **Windows 10** and **Ubuntu Linux** endpoints[cite: 8, 9, 10, 13].

---

## 🏗️ Network Architecture & IP Addressing

![GNS3 Multi-Zone Topology](screenshots/01-gns3-topology.png)

The infrastructure is segmented into three isolated security zones governed by **FortiGate 7.4.12**[cite: 9]:

| Security Zone | FortiGate Interface | Gateway IP | Connected Nodes | Endpoint IPs | Security Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **External Zone (WAN)** | `Port1` | DHCP / NAT | `Cloud1` (`eth1`)[cite: 9] | External / ISP | Untrusted external network & internet access |
| **DMZ Zone** | `Port2` | `172.16.10.1/24`[cite: 9] | `Switch2` ➔ `WindowsServer2025V2-1`[cite: 9] | `172.16.10.30`[cite: 9] | Core Infrastructure & Enterprise Services (AD, DNS, Web, FTP, SMB)[cite: 8, 9] |
| **Trust Zone (LAN)** | `Port3` | `192.168.20.1/24`[cite: 9] | `Switch1` ➔ `Ubuntu64` & `Windows10x64`[cite: 9] | `.15` (Ubuntu)<br>`.25` (Win10)[cite: 9] | Hardened internal corporate workstations (Windows & Linux)[cite: 9] |

---

## 🔥 FortiGate NGFW: Interfaces, Firewall Policies & Virtual IPs

### 1. Zone Interfaces & Traffic Segmentation
Each FortiGate interface is assigned to an explicit security zone (`External`, `DMZ`, `Trust`) to prevent unauthorized lateral movement and enforce stateful packet inspection between segments[cite: 9].

![FortiGate Interfaces](screenshots/02-fortigate-interfaces.png)

### 2. Firewall Policies & NAT Rules
Strict access control policies regulate all inter-zone communication:

| Policy Name | Incoming Interface | Outgoing Interface | Source | Destination | Service | Action & NAT |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`Trust_to_Internet`** | `Port3` (Trust)[cite: 9] | `Port1` (External)[cite: 9] | `192.168.20.0/24` | `all` | `ALL` | **ACCEPT** (SNAT Enabled) |
| **`DMZ_to_Internet`** | `Port2` (DMZ)[cite: 9] | `Port1` (External)[cite: 9] | `172.16.10.0/24` | `all` | `ALL` | **ACCEPT** (SNAT Enabled) |
| **`Trust_to_DMZ_Services`** | `Port3` (Trust)[cite: 9] | `Port2` (DMZ)[cite: 9] | `192.168.20.0/24` | `172.16.10.30`[cite: 9] | `DNS, LDAP, Kerberos, SMB, HTTP, FTP` | **ACCEPT** (No NAT - Internal Routing) |
| **`WAN_to_DMZ_VIP`** | `Port1` (External)[cite: 9] | `Port2` (DMZ)[cite: 9] | `all` | `DMZ_Web_VIP` / `DMZ_FTP_VIP` | `HTTP, HTTPS, FTP` | **ACCEPT** (DNAT via Virtual IP) |

![FortiGate Firewall Policies](screenshots/03-fortigate-policies.png)

### 3. Virtual IPs (VIP / DNAT Port Forwarding)
To securely publish DMZ-hosted services to external clients without exposing the internal DMZ subnet (`172.16.10.0/24`), **FortiGate Virtual IPs (Static NAT / Port Forwarding)** were configured on `Port1`:

| VIP Name | External Interface | External IP / Port | Mapped Internal IP | Mapped Port | Published Service |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`VIP_DMZ_Web_Server`** | `Port1` (WAN)[cite: 9] | `Port1 IP : 80 / 443` | `172.16.10.30`[cite: 9] | `80 / 443` | Enterprise Web Server (IIS)[cite: 8] |
| **`VIP_DMZ_FTP_Server`** | `Port1` (WAN)[cite: 9] | `Port1 IP : 21` | `172.16.10.30`[cite: 9] | `21` | Enterprise FTP Server[cite: 8] |

![FortiGate Virtual IPs](screenshots/04-fortigate-virtual-ips.png)

---

## 🖥️ Hosted Enterprise Services (DMZ Zone)

The DMZ server (`WindowsServer2025V2-1` — `172.16.10.30`) hosts core organizational infrastructure and network services accessible under controlled firewall policies[cite: 8, 9]:

* **Active Directory Domain Services (AD DS) & DNS (`faresismael.com`):** Centralized identity management, Kerberos authentication, internal name resolution, and Group Policy distribution[cite: 4, 5, 9].
* **Web Server (IIS — HTTP/HTTPS):** Hosts the internal/external corporate web portal reachable from both Trust workstations and external clients via FortiGate VIP[cite: 8].
* **FTP Server (File Transfer Protocol):** Configured with user authentication and directory isolation for secure file uploads/downloads across zones[cite: 8].
* **File Sharing Server (SMB / Network Shares):** Configured centralized department shared folders protected with granular **NTFS** and **Share-level permissions** so only authorized domain users can read/write corporate files[cite: 8].

![DMZ Enterprise Services](screenshots/05-dmz-web-ftp-fileshare.png)

---

## 🏢 Active Directory Hierarchy & Privilege Separation (RBAC)

Within **`faresismael.com`**, objects are organized under the root container **`ECE_Grad_Project`** across three dedicated Organizational Units (OUs)[cite: 3, 4, 5]:

![Active Directory OU Structure](screenshots/06-ad-ou-structure.png)

1. **`Project_Team` (Standard Users):** Contains team/employee accounts (`fares`, `mina`, etc.) governed by strict endpoint restrictions.
2. **`Project_Computers` (Workstations):** Contains domain-joined client machines (`Windows10x64`)[cite: 9].
3. **`IT_Department` (Technical Support):** Contains `IT_Support_Group` and dedicated IT admin accounts (`Fares-IT`, `ROIT`).

### 🔐 Least-Privilege Administration (No `Domain Admins` Over-Privileging)
To prevent privilege escalation and protect the Domain Controller, IT support accounts are explicitly excluded from `Domain Admins` and granted scoped permissions via two mechanisms:
* **Delegation of Control:** Delegated authority exclusively on `Project_Team` OU, allowing `IT_Support_Group` to inspect user details (`Read all user information`) and reset employee passwords without accessing the Domain Controller[cite: 1, 2].
* **Restricted Groups (`GPO_12`):** Elevated `IT_Support_Group` to **Local Administrator** solely on `Project_Computers` workstations (while preserving `Domain Admins`), enabling full local maintenance without domain-level control[cite: 3, 4, 5].

| Delegation of Control & Account Lockout | Restricted Groups (`Local Administrators`) |
| :---: | :---: |
| ![Delegation and Lockout](screenshots/08-delegation-and-lockout.png) | ![Restricted Groups Configuration](screenshots/09-it-restricted-groups.png) |

---

## 📜 Security Hardening Policies (12 GPOs + Domain Lockout)

![Group Policy Management](screenshots/07-gpo-list.png)

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
| **`GPO_11_BitLocker_Encryption`** | `Project_Computers` | Computer | Configures BitLocker OS drive encryption policy (with non-TPM support for virtualized environments)[cite: 6]. |
| **`GPO_12_IT_Local_Admin`** | `Project_Computers` | Computer | Grants `IT_Support_Group` Local Administrator rights on workstations via `Restricted Groups`[cite: 5]. |
| **`Account Lockout Policy`** | `Default Domain Policy`[cite: 10] | Domain | Protects against brute-force attacks by locking accounts for **30 minutes** after **5 invalid logon attempts**[cite: 7, 10]. |

---

## ✅ End-to-End Testing & Verification

All firewall policies, Virtual IPs, DMZ services, and GPO restrictions were validated across both Windows and Linux clients:

1. **Standard User Endpoint Hardening (`Project_Team`):**
   * Confirmed immediate access denial when attempting to launch `CMD`, `Control Panel`, `PowerShell`, `Task Manager`, or open **CD/DVD** and **USB** drives.
   * Verified **BitLocker** policy application (`GPO_11`) in Control Panel and **Account Lockout** enforcement after 5 failed password attempts[cite: 6, 10].
2. **IT Support Elevation (`Fares-IT`):**
   * Logged into `Windows10x64` (`192.168.20.25`) using `Fares-IT`; verified bypass of user restrictions and full Local Admin access (`CMD`, `Control Panel`, `gpresult /h`) without Domain Admin rights[cite: 5, 9].
3. **DMZ Services & FortiGate VIP Verification:**
   * Verified internal access from Trust clients to the **Web Server**, **FTP Server**, and **SMB File Share** (`\\172.16.10.30`), as well as external access through FortiGate **Virtual IP (VIP)** port forwarding[cite: 8, 9].
4. **Ubuntu Linux Client (`Ubuntu64`):**
   * Verified IP configuration (`192.168.20.15`), gateway connectivity (`192.168.20.1`), DMZ service access (`172.16.10.30`), and external internet access via FortiGate SNAT[cite: 9].

| System Tools Blocked (`CMD` / `Control Panel`) | Removable Media Blocked (`DVD` / `USB`) |
| :---: | :---: |
| ![System Tools Blocked](screenshots/10-win10-block-cmd-controlpanel.png) | ![DVD and USB Blocked](screenshots/11-win10-block-dvd-usb.png) |

| BitLocker Policy Verification (`GPO_11`) | IT Support Full Admin Access (`Fares-IT`) |
| :---: | :---: |
| ![BitLocker Verification](screenshots/12-bitlocker-verification.png) | ![IT Admin Access](screenshots/13-win10-it-admin-access.png) |

| DMZ Services & VIP Testing (`Web` / `FTP` / `SMB`) | Ubuntu Linux Connectivity (`192.168.20.15`) |
| :---: | :---: |
| ![Services Client Test](screenshots/14-services-client-test.png) | ![Ubuntu Connectivity](screenshots/15-ubuntu-connectivity.png) |

---

## 🛠️ Tech Stack & Tools
* **Network Simulation:** GNS3 v2.2.52 + GNS3 VM[cite: 9]
* **Next-Gen Firewall:** Fortinet FortiGate VM (v7.4.12)[cite: 9]
* **Server OS & Services:** Windows Server 2025 (AD DS, DNS, GPO, IIS Web, FTP, SMB File Sharing)[cite: 8, 9]
* **Client OS:** Windows 10 Pro (x64) & Ubuntu Linux (x64)[cite: 9]

---

## Future Enhancements

* **Management VLAN (VLAN 40)** — A dedicated out-of-band VLAN for secure device management (SSH to firewalls/switches), isolated from user and server data traffic and restricted by ACLs to an admin host[cite: 11].
* **SIEM/XDR & MITRE ATT&CK Integration (Wazuh + Sysmon)** — Deploying Wazuh agents and Sysmon across Windows and Linux endpoints for centralized log ingestion, real-time alerting, and MITRE ATT&CK technique mapping[cite: 13].
* **AI Risk Scoring Engine** — Building an intelligent risk analysis module that evaluates incoming SIEM alerts and assigns dynamic severity scores (0–100) to prioritize critical threats[cite: 13].
* **Automated Incident Response & SOAR (Ansible & Python)** — Automating containment playbooks to trigger zero-touch attacker IP blocking on FortiGate/iptables, endpoint quarantine, and automated recovery[cite: 13].
* **Software-Defined Networking (SDN)** — Integrating Ryu Controller and Open vSwitch (OVS) for programmable flow-level traffic isolation[cite: 13].
* **Red Team Attack Simulation (Kali Linux)** — Validating the full detection-to-recovery pipeline (`Attack → Detect → Alert → Automate → Isolate → Recover`) using Hydra, Nmap, and Metasploit[cite: 13].

---

## Author

**Fares Ismael** | Sphinx University

[LinkedIn](https://www.linkedin.com/in/fares-ismael-058976425/)
