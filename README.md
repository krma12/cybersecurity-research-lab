# Cybersecurity Research Laboratory & Vulnerability Repository

Welcome to my cybersecurity research repository. The primary objective of this space is to document my continuous learning journey, rigorous laboratory deployments, and deep-dive technical analyses of critical, high-impact vulnerabilities. 

This repository serves as a professional portfolio showcasing my hands-on expertise in network auditing, infrastructure isolation, and kernel-level behavior analysis.

---

##  Project Directory & Lab Status

| Project / CVE | Target Protocol | Primary Impact | Status |
| :--- | :--- | :--- | :--- |
| [📂 EternalBlue](./01-eternalblue) | SMBv1 | Remote Code Execution | ![Completed](https://shields.io) |
| [📂 DoublePulsar](./02-doublepulsar) | SMBv1 | Fileless Kernel Backdoor | ![Completed](https://shields.io) |
| [📂 BlueKeep](./03-BlueKeep) | RDP | Pre-Auth RCE / Wormable | ![Completed](https://shields.io) |
| [📂 SMBGhost](./04-SMBGhostCVE2020-0796) | SMBv3 | Memory Corruption / RDoS | ![Completed](https://shields.io) |
| 📂 SIEM & EDR Log Monitoring | Threat Hunting | Defensive Auditing | ![In Progress](https://shields.io) |
| 📂 Linux Hardening & Scanning Lab | System Security | Host Hardening | ![Planned](https://shields.io) |


---

##  Research Focus & Core Interests

My security research is heavily focused on understanding system internals, protocol specifications, and memory management mechanics. I specialize in deploying controlled, multi-vector simulation environments to safely observe exploitation lifecycles and evaluate modern enterprise mitigation strategies.

### Primary Areas of Investigation:
*   **Kernel-Level Memory Corruption:** Analyzing flaws within operating system drivers, pool structures, and arithmetic bugs (e.g., Integer Overflows, Out-of-Bounds Primitives).
*   **Network Protocol Auditing:** Evaluating cryptographic implementations, data compression, and authentication handshakes within major enterprise layers (SMB, RDP, HTTP).
*   **Infrastructure Isolation (Lab Securing):** Designing restricted Host-Only and Internal network micro-segments within hypervisors to prevent telemetry leaks and ensure safe execution vectors.
*   **Remediation & Defensive Engineering:** Translating exploitation mechanics into actionable defensive strategies, including direct patching policies, host-hardening commands, and perimeter filtering.

---

##  Technical Skillset & Toolkit

### Offensive & Assessment Capabilities
*   **Network Mapping & Discovery:** Nmap, Masscan, Protocol Enumeration.
*   **Exploit Analysis Frameworks:** Metasploit Framework, Impacket Suite, Custom Python/PoC scripting.
*   **Environment Hypervisors:** VirtualBox, VMware Workstation (Isolated Host-Only infrastructure building).

### Defensive & Engineering Foundations
*   **Operating Systems:** Linux (Debian/Kali administration), Windows Architecture Internals.
*   **Enterprise Hardening:** Active Directory Group Policy management, Windows Registry auditing, PowerShell administration scripting.
*   **Core Concepts:** Cryptographic message digests, CIA Triad alignment, and Risk Management frameworks.

---

##  Methodological Approach & Disclaimer

Every project documented here follows strict professional ethical boundaries and standard industry principles:
1.  **Strict Isolation:** No tools or payloads are ever executed in contact with external networks or public infrastructures.
2.  **Zero Personal Leaks:** All configuration files, network traces (pcap), and reports are structurally sanitized to ensure absolute data privacy.
3.  **Defensive End-State:** An investigation is only considered complete once the corresponding software update (KB patch) or administrative workaround (PowerShell hardening) is fully verified.

*Disclaimer: All content, research entries, and deployment workflows hosted in this repository are developed exclusively for educational purposes, defensive auditing, and academic research.*

---
*“Understanding the mechanics of the breakdown is the first imperative step toward building unyielding infrastructure.”*
