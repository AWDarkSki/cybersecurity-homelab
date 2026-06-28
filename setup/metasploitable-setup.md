# Metasploitable 2 Agent Setup Guide

This guide documents the steps taken to import Metasploitable 2 into VMware and install a Wazuh agent with multiple legacy compatibility workarounds.

---

## VM Specifications

| Resource | Value |
|---|---|
| OS | Metasploitable 2 (Ubuntu 8.04, Kernel 2.6.24) |
| Architecture | i386 (32-bit) |
| IP Address | 10.1.1.5 |
| Network | Bridged |
| RAM | 512 MB |
| Role | Vulnerable target machine |
| Wazuh Agent Version | 4.3.0 (legacy compatibility) |

---

## Step 1 — Download Metasploitable 2

Download from the official Rapid7 page:

```
https://information.rapid7.com/download-metasploitable-2017.html
```

Fill out the short form to receive the download link. The file is a ZIP containing a `.vmdk` virtual disk file.

---

## Step 2 — Import into VMware Workstation Pro

Metasploitable 2 comes as a `.vmdk` file, not an OVA, so it requires manual VM creation:

1. In VMware go to **File → New Virtual Machine → Custom**
2. Click through until **Guest Operating System** — select **Linux → Ubuntu 64-bit**
3. When asked about the disk select **Use an existing virtual disk**
4. Browse to and select the extracted `.vmdk` file
5. Set RAM to **512 MB**
6. Set network adapter to **Bridged: Connected directly to the physical network**
7. Click **Finish**

---

## Step 3 — Boot and Log In

Power on the VM. Metasploitable boots to a login prompt automatically.

Default credentials:
- **Username:** `msfadmin`
- **Password:** `msfadmin`

> Do NOT connect Metasploitable to the internet or expose it outside your lab network. It is intentionally vulnerable and will be compromised immediately if exposed.

---

## Step 4 — Check IP Address

```bash
ifconfig
```

Note the IP address — should be on the `10.1.1.x` subnet.

---

## Step 5 — Install the Wazuh Agent

Metasploitable 2 has several compatibility issues with modern Wazuh agents that required the following workarounds:

| Issue | Cause | Solution |
|---|---|---|
| SSL connection error | Ubuntu 8.04 has outdated SSL libraries | Use HTTP instead of HTTPS |
| amd64 package fails | Metasploitable is 32-bit (i386) | Download i386 package |
| Wazuh 4.14 fails to start | Kernel 2.6.24 too old for modern agent | Use Wazuh 4.3.0 instead |
| systemctl not found | Ubuntu 8.04 uses SysV init, not systemd | Use wazuh-control script directly |

Run this command to download and install the compatible agent:

```bash
wget http://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.3.0-1_i386.deb && sudo WAZUH_MANAGER='10.1.1.3' WAZUH_AGENT_NAME='metasploitable' dpkg -i ./wazuh-agent_4.3.0-1_i386.deb
```

> Note: `http://` is used instead of `https://` due to SSL compatibility issues on Ubuntu 8.04.

---

## Step 6 — Start the Wazuh Agent

Metasploitable 2 uses the old SysV init system — `systemctl` and `service` commands are not available. Use the Wazuh control script directly:

```bash
sudo /var/ossec/bin/wazuh-control start
```

Verify all processes are running:

```bash
sudo /var/ossec/bin/wazuh-control status
```

You should see these all listed as running:
- `wazuh-execd`
- `wazuh-agentd`
- `wazuh-syscheckd`
- `wazuh-logcollector`
- `wazuh-modulesd`

---

## Step 7 — Enable Auto-start on Boot

Since systemctl is unavailable, register with SysV init to auto-start on boot:

```bash
sudo update-rc.d wazuh-agent defaults 95 5
```

---

## Step 8 — Enable SSH Log Monitoring

By default Wazuh does not monitor the SSH auth log on this system. Add it manually:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following inside the `<ossec_config>` block:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

Save and restart the agent:

```bash
sudo /var/ossec/bin/wazuh-control restart
```

---

## Step 9 — Configure File Integrity Monitoring

Edit the ossec.conf on Metasploitable to monitor sensitive directories:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find the `<syscheck>` section and add:

```xml
<directories check_all="yes">/etc/passwd</directories>
<directories check_all="yes">/etc/shadow</directories>
<directories check_all="yes">/tmp</directories>
<directories check_all="yes">/root</directories>
<directories check_all="yes">/bin</directories>
```

Restart the agent:

```bash
sudo /var/ossec/bin/wazuh-control restart
```

To manually trigger a FIM scan from the Wazuh server:

```bash
sudo /var/ossec/bin/agent_control -r -u 002
```

---

## Step 10 — Verify in Wazuh Dashboard

In the Wazuh dashboard go to **Agents** — Metasploitable should appear with a green **Active** status showing:
- Agent ID: 002
- Agent Name: metasploitable
- OS: Ubuntu 8.04
- Version: Wazuh v4.3.0

---

## Known Limitations

- **Vulnerability Detection:** Wazuh 4.3.0 on Ubuntu 8.04 does not support syscollector, so package inventory is not reported and CVEs cannot be automatically detected on this agent
- **Wazuh Agent Version:** Must use 4.3.0 — newer versions fail to start on kernel 2.6.24
- **No systemctl:** All agent management must be done via `/var/ossec/bin/wazuh-control`
- **Legacy SSH:** Connecting via SSH from modern systems requires legacy algorithm flags

---

## SSH Connection from Kali (legacy flags required)

```bash
ssh -o "KexAlgorithms=diffie-hellman-group1-sha1" -o "HostKeyAlgorithms=ssh-rsa" -o "MACs=hmac-sha1" msfadmin@10.1.1.5
```

---

## Result

Metasploitable 2 is enrolled as an active Wazuh agent at `10.1.1.5`, monitored by the Wazuh server at `10.1.1.3`. SSH authentication events, failed logins, file integrity changes, and system activity are all forwarded to the Wazuh dashboard and mapped to MITRE ATT&CK tactics.
