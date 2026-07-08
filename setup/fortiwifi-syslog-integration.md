# FortiWifi 71G Syslog Integration with Wazuh

This guide documents the steps taken to integrate FortiWifi 71G firewall logs with Wazuh, enabling network perimeter monitoring alongside endpoint monitoring.

---

## Overview

By forwarding syslog from the FortiWifi 71G to Wazuh, the SIEM now monitors three layers simultaneously:

| Layer | Source | What it detects |
|---|---|---|
| Network perimeter | FortiWifi 71G | Firewall traffic, IPS alerts, blocked connections |
| Endpoint (attacker) | Kali Linux agent | Attack tools, reconnaissance activity |
| Endpoint (target) | Metasploitable agent | Intrusion attempts, file changes, auth failures |

---

## Network Details

| Device | IP | Role |
|---|---|---|
| FortiWifi 71G | 10.1.1.1 | Firewall / syslog source |
| Wazuh Server | 10.1.1.3 | Syslog receiver |

---

## Step 1 — Configure FortiWifi Syslog

1. Log into the FortiWifi dashboard at `192.168.40.x`
2. Go to **Log & Report → Log Settings → Global Settings**
3. Click **Syslog Logging**
4. Enable syslog and enter the server address: `10.1.1.3`
5. Click **Apply**

> Note: The FortiWifi defaults to UDP port 514 which matches Wazuh's syslog listener.

---

## Step 2 — Open Firewall Port on Wazuh Server

```bash
sudo ufw allow 514/udp
sudo ufw reload
```

---

## Step 3 — Configure Wazuh to Receive Syslog

Edit the Wazuh manager config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this before the closing `</ossec_config>` tag:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.1.1.1</allowed-ips>
</remote>
```

Restart the Wazuh manager:

```bash
sudo systemctl restart wazuh-manager
```

---

## Step 4 — Verify Wazuh is Listening

```bash
sudo ss -ulnp | grep 514
```

Expected output:
```
UNCONN 0    0    0.0.0.0:514    0.0.0.0:*    users:(("wazuh-remoted",pid=XXXX,fd=4))
```

---

## Step 5 — Verify Logs Are Arriving

```bash
sudo tcpdump -i any port 514 -n
```

You should see packets arriving from `10.1.1.1` (FortiWifi) to `10.1.1.3:514` (Wazuh). Press **Ctrl+C** after 30 seconds.

---

## Wazuh FortiGate Decoder

Wazuh automatically detected and parsed the FortiGate log format using its built-in decoder:

```
decoder: fortigate-firewall-v6
rule file: 0391-fortigate_rules.xml
```

No manual decoder configuration was required — Wazuh ships with native FortiGate support.

---

## FortiWifi Events Detected

### 1. Normal Traffic Monitoring
**Rule 81633 — Fortigate: App passed by firewall**
- Every allowed connection is logged with full details
- Source IP, destination IP, destination country, application name, bytes transferred
- Compliance mapped to PCI DSS, GDPR, HIPAA, NIST-800-53

Example fields captured by Wazuh:
```
srcip: [internal LAN IP]
dstip: [destination IP]
dstcountry: United States
app: Microsoft.365.Portal
appcat: Collaboration
apprisk: elevated
action: accept
```

---

### 2. High Traffic Anomaly Detection
**Rule 81619 — Fortigate: Multiple high traffic events from same source**
- Triggered when same source IP generates 18+ connections within 45 seconds
- Automatically correlates events by source IP
- Compliance mapped to PCI DSS 10.6.1, GDPR IV.35.7.d, HIPAA 164.312.b, NIST-800-53 AU.6

---

### 3. IPS Attack Detection (level 11)
**Rule 81628 — Fortigate attack detected**

An nmap port scan from Kali Linux (`10.1.1.8`) against an external target was detected by the FortiWifi IPS engine and forwarded to Wazuh:

```
attack: Port.Scanning
srcip: 10.1.1.8 (Kali Linux)
dstip: 8.8.8.8
direction: outgoing
action: detected
severity: low
attackid: 43814
ref: http://www.fortinet.com/ids/VID43814
```

**This demonstrates dual-layer detection** — the same Kali attack was detected by:
- The **Wazuh endpoint agent** on Kali at the host level
- The **FortiWifi IPS** at the network perimeter level

---

## Alert Volume Management

Integrating FortiWifi syslog generates a high volume of low severity alerts from normal network traffic. To filter the dashboard to show only meaningful events:

**In Wazuh Security Events search bar:**
```
NOT rule.id: 81633
```

Or filter by rule level:
- Field: `rule.level`
- Operator: `is greater than`
- Value: `3`

This suppresses informational traffic logs while preserving high severity alerts like IPS detections.

---

## Compliance Mapping

FortiWifi alerts are automatically mapped to multiple compliance frameworks by Wazuh:

| Framework | Control |
|---|---|
| PCI DSS | 10.6.1 |
| GDPR | IV.35.7.d |
| HIPAA | 164.312.b |
| NIST-800-53 | AU.6 |

---

## Screenshots

- `fortigate-syslog-integration.png` — Dashboard showing FortiGate events confirming integration
- `fortigate-high-traffic.png` — Multiple high traffic events from same source
- `fortigate-compliance-mapping.png` — Compliance framework mapping on FortiGate alert
- `fortigate-attack-detected-detail.png` — Full alert detail of FortiWifi IPS detection
- `fortigate-attack-detected-dashboard.png` — FortiWifi IPS detecting Kali port scan (rule 81628, level 11)

---

## Result

The FortiWifi 71G is now fully integrated with Wazuh, providing complete network perimeter visibility. Every connection passing through the firewall is logged, parsed, and correlated with endpoint events — replicating an enterprise-grade defense in depth monitoring architecture.
