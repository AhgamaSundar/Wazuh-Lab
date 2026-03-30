

# Wazuh Detection Lab


<img width="661" height="601" alt="Network_diagram drawio" src="https://github.com/user-attachments/assets/e8bef1ed-1e08-4807-b4f2-4d13fb0165eb" />

A home SOC lab for developing and testing detection queries against MITRE ATT&CK techniques using Wazuh as the primary SIEM. Built on VirtualBox with resource-constrained hardware in mind.

---

## Table of Contents

- [Overview](#overview)
- [Lab Goals](#lab-goals)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
  - [1. Create the NAT Network and Port Forwards](#1-create-the-nat-network-and-port-forwards)
  - [2. Deploy the Wazuh Server (Ubuntu)](#2-deploy-the-wazuh-server-ubuntu)
  - [3. Deploy Windows Server 2022 (Domain Controller)](#3-deploy-windows-server-2022-domain-controller)
  - [4. Deploy Windows 11 Target (Domain Endpoint)](#4-deploy-windows-11-target-domain-endpoint)
  - [5. Deploy Kali Linux (Attacker)](#5-deploy-kali-linux-attacker)
  - [6. Install Wazuh Agents on Windows Hosts](#6-install-wazuh-agents-on-windows-hosts)
  - [7. Install Atomic Red Team](#7-install-atomic-red-team)
- [Snapshots](#snapshots)
- [Lab Goals / Detection Work](#lab-goals--detection-work)

---

## Overview

This lab is deployed on Oracle VirtualBox on a Linux host. Due to resource constraints, the Ubuntu server runs both Wazuh and Splunk (Splunk is installed but disabled). The dual installation exists for comparison purposes — to understand the differences between a FOSS SIEM (Wazuh) and a commercial SIEM (Splunk).

Both Windows hosts have:
- Wazuh agent installed and reporting to the Ubuntu server
- Splunk Universal Forwarder installed (dormant while Splunk is disabled)

The Windows Server 2022 VM acts as an **Active Directory Domain Controller** at `192.168.10.7`. The Windows 11 endpoint at `192.168.10.100` is domain-joined, reflecting a realistic enterprise workstation setup. The Kali attacker is placed on the same subnet intentionally — this is a controlled lab and the flat network simplifies attack simulation and traffic capture.

---

## Lab Goals

- Simulate adversary techniques from the [MITRE ATT&CK Framework](https://attack.mitre.org/) using Atomic Red Team
- Develop and validate Wazuh detection rules and queries against those simulated attacks
- Compare detection capability between Wazuh (open source) and Splunk (commercial)
- Build a reference library of working detections mapped to ATT&CK technique IDs

---

## Architecture

| VM | OS | IP | vCPU | RAM | Role |
|---|---|---|---|---|---|
| splunkserver | Ubuntu 22.04 LTS | 192.168.10.10 (static) | 4 | 6 GB | Wazuh Manager + Indexer + Dashboard (+ Splunk, disabled) |
| winserver (DC) | Windows Server 2022 | 192.168.10.7 (static) | 2 | 4 GB | Active Directory Domain Controller |
| Target | Windows 11 | 192.168.10.100 (static) | 4 | 4.5 GB | Domain-joined Windows endpoint |
| Kali | Kali Linux (rolling) | DHCP (labnet) | 2 | 2 GB | Attacker |

**Network:** VirtualBox NAT Network named `labnet` — `192.168.10.0/24`

All VMs sit on the same NAT network and can communicate with each other and reach the internet. The host machine forwards ports **443** (Wazuh dashboard) and **8000** (Splunk) from the NAT network to allow browser access from the host without entering the VM.

```
Host browser → localhost:443  → 192.168.10.10:443  (Wazuh dashboard)
Host browser → localhost:8000 → 192.168.10.10:8000 (Splunk, when enabled)
```

The Windows endpoint (192.168.10.100) is joined to the AD domain hosted on the DC (192.168.10.7). Both Windows hosts run Wazuh agents and Splunk Universal Forwarders reporting to the Ubuntu server. Kali is on the same subnet intentionally for simplified attack simulation.

---

## Prerequisites

### Host Machine

- **OS:** Linux (tested on Arch)
- **RAM:** 16 GB minimum recommended (lab uses ~16.5 GB across all VMs)
- **Disk:** 120 GB+ free (Wazuh's OpenSearch backend is storage-heavy — expect 50–60 GB for the server VM alone)
- **CPU:** 8+ cores with virtualisation extensions enabled (check with `egrep -c '(vmx|svm)' /proc/cpuinfo`)

### Software

```bash
# Install VirtualBox
sudo apt update
sudo apt install virtualbox virtualbox-ext-pack -y

# Verify installation
vboxmanage --version
```

### ISOs Required

Download these before starting:

| ISO | Source |
|---|---|
| Ubuntu Server 22.04 LTS | https://ubuntu.com/download/server |
| Windows Server 2022 Evaluation | https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2022 |
| Windows 11 | https://www.microsoft.com/software-download/windows11 |
| Kali Linux | https://www.kali.org/get-kali/#kali-installer-images |

---

## Setup Guide

### 1. Create the NAT Network and Port Forwards

All VMs share a NAT network called `labnet`. Create it with port forwarding so you can access the Wazuh dashboard and Splunk from your host browser:

```bash
vboxmanage natnetwork add \
  --netname labnet \
  --network "192.168.10.0/24" \
  --enable \
  --dhcp on

# Port forward 443 → Wazuh dashboard
vboxmanage natnetwork modify --netname labnet \
  --port-forward-4 "wazuh-https:tcp:[]:443:[192.168.10.10]:443"

# Port forward 8000 → Splunk web UI
vboxmanage natnetwork modify --netname labnet \
  --port-forward-4 "splunk-web:tcp:[]:8000:[192.168.10.10]:8000"
```

Verify:

```bash
vboxmanage natnetwork list
```

After this, your host browser can reach:
- Wazuh: `https://localhost` (port 443)
- Splunk: `http://localhost:8000` (when enabled)

---

### 2. Deploy the Wazuh Server (Ubuntu)

#### 2a. Create the VM

```bash
vboxmanage createvm --name "splunkserver" --ostype Ubuntu_64 --register

vboxmanage modifyvm "splunkserver" \
  --memory 6000 \
  --cpus 4 \
  --vram 16 \
  --nic1 natnetwork \
  --nat-network1 labnet \
  --audio-driver none

# Create and attach a 60 GB disk (Wazuh needs space)
vboxmanage createhd --filename ~/VMs/splunkserver.vdi --size 61440

vboxmanage storagectl "splunkserver" --name "SATA" --add sata --controller IntelAhci
vboxmanage storageattach "splunkserver" \
  --storagectl "SATA" --port 0 --device 0 \
  --type hdd --medium ~/VMs/splunkserver.vdi

vboxmanage storagectl "splunkserver" --name "IDE" --add ide
vboxmanage storageattach "splunkserver" \
  --storagectl "IDE" --port 0 --device 0 \
  --type dvddrive --medium /path/to/ubuntu-22.04-live-server-amd64.iso
```

Boot the VM and complete the Ubuntu Server installation. Set a static IP during install or configure it afterward:

```bash
# On the Ubuntu VM — set static IP
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses:
        - 192.168.10.10/24
      nameservers:
        addresses: [8.8.8.8]
      routes:
        - to: default
          via: 192.168.10.1
```

```bash
sudo netplan apply
```

#### 2b. Install Wazuh (All-in-One)

```bash
# Download and run the Wazuh installer
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.x/config.yml
```

Edit `config.yml` to set the node IP to `192.168.10.10`, then:

```bash
sudo bash wazuh-install.sh -a
```

This installs the Wazuh Manager, Indexer, and Dashboard in one shot. Installation takes 10–20 minutes.

Save the admin credentials printed at the end — you'll need them for the dashboard.

Access the dashboard at: `https://192.168.10.10` (accept the self-signed cert warning)

#### 2c. Take a Snapshot

```bash
vboxmanage snapshot "splunkserver" take "wazuh-clean-install" \
  --description "Wazuh installed, no agents connected"
```

---

### 3. Deploy Windows Server 2022 (Domain Controller)

#### 3a. Create the VM

```bash
vboxmanage createvm --name "winserver" --ostype Windows2022_64 --register

vboxmanage modifyvm "winserver" \
  --memory 4096 \
  --cpus 2 \
  --vram 128 \
  --nic1 natnetwork \
  --nat-network1 labnet \
  --clipboard-mode bidirectional \
  --draganddrop bidirectional

vboxmanage createhd --filename ~/VMs/winserver.vdi --size 51200

vboxmanage storagectl "winserver" --name "SATA" --add sata --controller IntelAhci
vboxmanage storageattach "winserver" \
  --storagectl "SATA" --port 0 --device 0 \
  --type hdd --medium ~/VMs/winserver.vdi

vboxmanage storageattach "winserver" \
  --storagectl "SATA" --port 1 --device 0 \
  --type dvddrive --medium /path/to/SERVER_EVAL_x64FRE_en-us.iso
```

Install Windows Server 2022 — choose **Desktop Experience** (GUI). During setup, set a strong Administrator password.

#### 3b. Set a Static IP

After installation, open PowerShell as Administrator and set the static IP:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.7 -PrefixLength 24 -DefaultGateway 192.168.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

#### 3c. Promote to Domain Controller

```powershell
# Install AD DS role
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

# Promote to DC (creates a new forest — change the domain name as needed)
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName "lab.local" `
  -DomainNetbiosName "LAB" `
  -ForestMode "WinThreshold" `
  -DomainMode "WinThreshold" `
  -InstallDns:$true `
  -Force:$true
```

The server will reboot. After reboot, log in as `LAB\Administrator`.

#### 3d. Create a Domain User (for the endpoint to use)

```powershell
New-ADUser -Name "labuser" -AccountPassword (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) -Enabled $true -PasswordNeverExpires $true
Add-ADGroupMember -Identity "Domain Admins" -Members "labuser"
```

---

### 4. Deploy Windows 11 Target (Domain Endpoint)

#### 4a. Create the VM

```bash
vboxmanage createvm --name "Target" --ostype Windows11_64 --register

vboxmanage modifyvm "Target" \
  --memory 4516 \
  --cpus 4 \
  --vram 128 \
  --firmware efi \
  --tpm-type 2-0 \
  --nic1 natnetwork \
  --nat-network1 labnet \
  --clipboard-mode bidirectional

vboxmanage createhd --filename ~/VMs/Target.vdi --size 51200

vboxmanage storagectl "Target" --name "SATA" --add sata --controller IntelAhci --portcount 2
vboxmanage storageattach "Target" \
  --storagectl "SATA" --port 0 --device 0 \
  --type hdd --medium ~/VMs/Target.vdi

vboxmanage storageattach "Target" \
  --storagectl "SATA" --port 1 --device 0 \
  --type dvddrive --medium /path/to/Win11_23H2_English_x64.iso
```

> **Note:** Windows 11 enforces TPM 2.0 and Secure Boot checks during install. VirtualBox 7.x handles this with `--firmware efi` and `--tpm-type 2-0`. If the installer still blocks, apply these registry bypasses in Windows PE during setup:
>
> ```
> reg add HKLM\SYSTEM\Setup\LabConfig /v BypassTPMCheck /t REG_DWORD /d 1 /f
> reg add HKLM\SYSTEM\Setup\LabConfig /v BypassSecureBootCheck /t REG_DWORD /d 1 /f
> reg add HKLM\SYSTEM\Setup\LabConfig /v BypassRAMCheck /t REG_DWORD /d 1 /f
> ```

Create a local account during OOBE (skip the Microsoft account requirement by disconnecting the network or using `SHIFT+F10` → `oobe\bypassnro`).

#### 4b. Set a Static IP and Point DNS to the DC

Open PowerShell as Administrator:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.100 -PrefixLength 24 -DefaultGateway 192.168.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.10.7
```

#### 4c. Join the Domain

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

Enter `LAB\Administrator` (or `LAB\labuser`) credentials when prompted. The machine will reboot and join the domain. After reboot, log in as `LAB\labuser`.

---

### 5. Deploy Kali Linux (Attacker)

```bash
vboxmanage createvm --name "kali" --ostype Debian_64 --register

vboxmanage modifyvm "kali" \
  --memory 2048 \
  --cpus 2 \
  --vram 128 \
  --nic1 natnetwork \
  --nat-network1 labnet

vboxmanage createhd --filename ~/VMs/kali.vdi --size 40960

vboxmanage storagectl "kali" --name "SATA" --add sata --controller IntelAhci
vboxmanage storageattach "kali" \
  --storagectl "SATA" --port 0 --device 0 \
  --type hdd --medium ~/VMs/kali.vdi

vboxmanage storageattach "kali" \
  --storagectl "SATA" --port 1 --device 0 \
  --type dvddrive --medium /path/to/kali-linux-2024.x-installer-amd64.iso
```

Install Kali with default settings. The attacker is on `labnet` intentionally — this is a controlled lab environment.

---

### 6. Install Wazuh Agents on Windows Hosts

Do this on both the DC (`winserver`, 192.168.10.7) and the endpoint (`Target`, 192.168.10.100).

#### On each Windows VM (PowerShell as Administrator):

```powershell
# Download the Wazuh agent MSI (check https://packages.wazuh.com for latest version)
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi" `
  -OutFile "C:\wazuh-agent.msi"

# Install — set WAZUH_AGENT_NAME appropriately per host
# On the DC:
msiexec.exe /i C:\wazuh-agent.msi /q `
  WAZUH_MANAGER="192.168.10.10" `
  WAZUH_REGISTRATION_SERVER="192.168.10.10" `
  WAZUH_AGENT_NAME="winserver-dc"

# On the endpoint (Target):
msiexec.exe /i C:\wazuh-agent.msi /q `
  WAZUH_MANAGER="192.168.10.10" `
  WAZUH_REGISTRATION_SERVER="192.168.10.10" `
  WAZUH_AGENT_NAME="target-pc"

# Start the agent service
NET START WazuhSvc
```

#### Verify on the Wazuh Manager:

```bash
# On the Ubuntu server
sudo /var/ossec/bin/agent_control -l
```

Both agents should show as `Active`.

---

### 7. Install Atomic Red Team

Do this on both Windows hosts. Run PowerShell as Administrator:

```powershell
# Disable Windows Defender real-time protection (lab only — never do this in production)
Set-MpPreference -DisableRealtimeMonitoring $true

# Set execution policy
Set-ExecutionPolicy Bypass -Scope CurrentUser -Force

# Install Atomic Red Team
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force

# Import the module
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

#### Test a technique:

```powershell
# Example: T1059.001 - PowerShell execution
Invoke-AtomicTest T1059.001 -TestNumbers 1
```

Then check the Wazuh dashboard for the resulting alerts.

---

## Snapshots

Take snapshots at key points so you can roll back cleanly after running destructive Atomic Red Team tests:

```bash
# After clean OS install, before any agents
vboxmanage snapshot "winserver" take "clean-os" --description "Fresh Windows Server 2022, DC promoted, no agents"
vboxmanage snapshot "Target" take "clean-os" --description "Fresh Windows 11, domain-joined, no agents"

# After agents installed, before running any Atomic tests — this is your main restore point
vboxmanage snapshot "winserver" take "agents-installed"
vboxmanage snapshot "Target" take "agents-installed"

# Restore to clean state after a test run
vboxmanage snapshot "winserver" restore "agents-installed"
vboxmanage snapshot "Target" restore "agents-installed"
```

> Restore both VMs together after a test — if you restore only the endpoint and not the DC, AD state can get out of sync.

---

## Lab Goals / Detection Work

Detection queries and rules are being developed and will be documented here, mapped to MITRE ATT&CK technique IDs.

| Technique ID | Name | Status |
|---|---|---|
| T1059.001 | PowerShell | 🔄 In progress |
| T1003.001 | LSASS Memory Dump | 🔄 In progress |
| T1071.001 | Web Protocols C2 | 📋 Planned |
| T1548.002 | UAC Bypass | 📋 Planned |

> Wazuh custom rules live in `/var/ossec/etc/rules/local_rules.xml` on the Ubuntu server.
> After editing rules, reload with: `sudo systemctl restart wazuh-manager`
