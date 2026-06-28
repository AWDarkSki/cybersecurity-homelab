# Wazuh Server Installation Guide

This guide documents the steps taken to deploy a Wazuh all-in-one server on Ubuntu Server 24.04.4 LTS inside VMware Workstation Pro.

---

## VM Specifications

| Resource | Value |
|---|---|
| RAM | 8 GB |
| CPUs | 2 |
| Disk | 100 GB |
| OS | Ubuntu Server 24.04.4 LTS |
| Network | Bridged |
| IP Address | 10.1.1.3 (static) |

> **Important:** Use at least 100 GB of disk space. The Wazuh indexer and vulnerability detection database require significant storage. Running out of disk space mid-install will corrupt the setup and require a full reinstall.

---

## Step 1 — Create the VM in VMware Workstation Pro

1. Open VMware Workstation Pro and click **Create a New Virtual Machine**
2. Select **Typical**
3. Point to your Ubuntu Server 24.04.4 LTS ISO
4. Set RAM to **8 GB**
5. Set CPUs to **2**
6. Set disk size to **100 GB**
7. Set network adapter to **Bridged: Connected directly to the physical network**
8. Do NOT check "Replicate physical network connection state"
9. Click **Finish**

---

## Step 2 — Install Ubuntu Server 24.04.4 LTS

During the Ubuntu installation wizard:

- **OpenSSH Server:** Enable this — allows SSH access after install
- **Proxy:** Leave blank
- **Storage:** Use entire disk (LVM group is fine to leave enabled)
- **Active Directory:** Skip — not needed for this lab
- **Username/Password:** Create a memorable username and password
- **Server name:** Set to something recognizable e.g. `wazuh-server`

---

## Step 3 — Prepare the System

After Ubuntu is installed and you are logged in, update the system:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg apt-transport-https
```

Confirm disk space is sufficient before proceeding:

```bash
df -h
```

You should have at least **30 GB free** before running the Wazuh installer.

---

## Step 4 — Run the Wazuh All-in-One Installer

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

The `-a` flag installs all components on a single host:
- **Wazuh Manager** — core analysis engine, agents connect on port 1514
- **Wazuh Indexer** — OpenSearch-based alert storage on port 9200
- **Filebeat** — ships alerts from manager to indexer
- **Wazuh Dashboard** — web UI accessible on HTTPS port 443

> This process takes 10–15 minutes. The indexer bootstrap step will appear to hang for several minutes — this is normal. Do not interrupt the process.

At the end of the install, credentials are printed to the terminal:

```
User: admin
Password: <generated_password>
```

**Copy these immediately.** If you miss them, retrieve them with:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

---

## Step 5 — Verify Services Are Running

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should show `active (running)`.

---

## Step 6 — Open Firewall Ports

```bash
sudo ufw allow 443/tcp    # Dashboard
sudo ufw allow 1514/tcp   # Agent data
sudo ufw allow 1515/tcp   # Agent enrollment
sudo ufw allow 55000/tcp  # Wazuh API
sudo ufw reload
```

---

## Step 7 — Set a Static IP

To ensure agents can always reach the Wazuh server, assign a static IP:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace contents with:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 10.1.1.3/24
      routes:
        - to: default
          via: 10.1.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

Apply the changes:

```bash
sudo netplan apply
ip a
```

---

## Step 8 — Configure Vulnerability Detection

Edit the Wazuh manager config:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this before the closing `</ossec_config>` tag:

```xml
<vulnerability-detection>
  <enabled>yes</enabled>
  <index-status>yes</index-status>
  <feed-update-interval>60m</feed-update-interval>
</vulnerability-detection>
```

Restart the manager:

```bash
sudo systemctl restart wazuh-manager
```

---

## Step 9 — Configure Active Response

Add this to `ossec.conf` before the closing `</ossec_config>` tag:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5712</rules_id>
  <timeout>180</timeout>
</active-response>
```

Restart the manager:

```bash
sudo systemctl restart wazuh-manager
```

This automatically blocks any IP that triggers rule 5712 (SSH brute force) for 180 seconds.

---

## Step 10 — Access the Dashboard

From any browser on the same network navigate to:

```
https://10.1.1.3
```

Accept the self-signed certificate warning and log in with the admin credentials from Step 4.

---

## Troubleshooting

### Installation failed / services not found
The most common cause is running out of disk space mid-install. Check with `df -h` and ensure at least 30 GB is free. If the install failed, fully clean up before retrying:

```bash
sudo dpkg --purge --force-all wazuh-manager wazuh-indexer wazuh-dashboard filebeat
sudo rm -rf /var/lib/dpkg/info/wazuh-manager.*
sudo rm -rf /var/lib/dpkg/info/wazuh-indexer.*
sudo rm -rf /var/lib/dpkg/info/wazuh-dashboard.*
sudo rm -rf /var/ossec /etc/wazuh-* /usr/share/wazuh-* /var/lib/wazuh-*
sudo rm -f /etc/apt/sources.list.d/wazuh.list /usr/share/keyrings/wazuh.gpg
sudo apt clean && sudo apt autoremove -y && sudo apt update
sudo reboot
```

Then re-run the installer.

### prerm maintainer script failing with exit status 127
Blank out the broken script and force remove:

```bash
sudo tee /var/lib/dpkg/info/wazuh-manager.prerm << 'SCRIPT'
#!/bin/bash
exit 0
SCRIPT
sudo chmod +x /var/lib/dpkg/info/wazuh-manager.prerm
sudo dpkg --remove --force-remove-reinstreq wazuh-manager
sudo dpkg --purge --force-all wazuh-manager
```

### VM gets wrong IP after reboot
Go to **VMware → Edit → Virtual Network Editor → VMnet0** and set the bridged adapter to your specific physical network interface rather than Automatic.

### Vulnerability detection warnings about deprecated tag
Replace `<vulnerability-detector>` with `<vulnerability-detection>` in ossec.conf — the tag was renamed in newer Wazuh versions.

---

## Result

A fully operational Wazuh SIEM/XDR server accessible at `https://10.1.1.3` with the manager, indexer, and dashboard all running on a single Ubuntu 24.04 VM, with vulnerability detection and active response configured.
