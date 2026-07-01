# Attack: Network Reconnaissance (Nmap)

## Overview
An aggressive Nmap scan was performed from Kali Linux against Metasploitable 2 to enumerate open ports, running services, and the operating system. Wazuh detected the scan across multiple rule categories.

---

## Attack Details

| Field | Value |
|---|---|
| Attacker | Kali Linux (10.1.1.8) |
| Target | Metasploitable 2 (10.1.1.5) |
| Tool | Nmap |
| MITRE ATT&CK Tactic | Discovery |
| MITRE ATT&CK Technique | T1046 — Network Service Discovery |

---

## Command Used

```bash
nmap -A -sV -O 10.1.1.5
```

**Flags explained:**
- `-A` — Aggressive scan (OS detection, version detection, script scanning)
- `-sV` — Service version detection
- `-O` — OS fingerprinting

---

## Wazuh Detections

| Rule ID | Level | Description |
|---|---|---|
| 2551 | 10 | Connection to rshd from unprivileged port — possible network scan |
| 5706 | 6 | sshd: insecure connection attempt (scan) |
| 31101 | 5 | Web server 400 error code |
| 5602 | 3 | telnetd: Remote host established a telnet connection |
| 11401 | 3 | vsftpd: FTP session opened |
| 11201 | 3 | ProFTPD: FTP session opened |

---

## Screenshot

![vsftpd Exploit Detections](../screenshots/vsftpd-exploit-detections.png)

---

## Key Takeaway

The Nmap scan triggered alerts across multiple services simultaneously — SSH, FTP, Telnet, and the web server — demonstrating how Wazuh correlates activity across different log sources to detect reconnaissance activity.
