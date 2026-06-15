# SRT210 Assignment 1

**Student Name:** Arian Nili
**Student ID:** 173189226  
**Date:** 06/15/2026  
**Environment:** VirtualBox 7.2.8, Windows Server 2022, Ubuntu 26.04 LTS

---

## 1. Windows Network Topology 

| Segment                | Subnet              | Mask             | Usable addresses      |
|------------------------|---------------------|------------------|-----------------------|
| Gateway ↔ Relay        | `192.168.76.0/29`   | 255.255.255.248  | `.1` – `.6`           |
| Relay ↔ Node           | `192.168.76.80/28`  | 255.255.255.240  | `.81` – `.94`         |

| VM               | Interface role               | IP address                     | How assigned                  |
|------------------|------------------------------|--------------------------------|-------------------------------|
| windows-GW       | NAT (to internet)            | `10.0.2.15`                    | VirtualBox DHCP               |
|                  | Internal (to Relay)          | `192.168.76.1/29`              | Static (manual)               |
| windows-Relay    | Internal (to Gateway)        | `192.168.76.2/29`              | DHCP from GW (MAC reservation)|
|                  | Bridged (to Node)            | `192.168.76.81/28`             | Static                        |
| windows-Node     | Bridged (to Relay)           | `192.168.76.82/28`             | DHCP from Relay (MAC reserv.) |

**Physical connections:**  
- Relay ↔ Node: Realtek USB GbE Family Controller adapter connected by a physical CAT6 cable.  
- Host‑only adapters (Adapter 4 on GW) used for SSH from the K1001 PC.

---

## 2. Windows Server 2022 Routing Lab

### 2.1 Virtual Machine Creation

| VM            | OS Edition                                     | RAM  | CPUs | Disk  | Cloned from   |
|---------------|------------------------------------------------|------|------|-------|---------------|
| windows-GW    | Server 2022 Standard Evaluation (Core)         | 4 GB | 2    | 50 GB | Fresh install |
| windows-Relay | Server 2022 Standard Evaluation (Core)         | 4 GB | 2    | 50 GB | Full clone of GW, **generate new MACs** |
| windows-Node  | Server 2022 Standard Evaluation (Desktop Exp.) | 8 GB | 2    | 50 GB | Fresh install |

**Critical clone step:** After cloning windows‑Relay, right‑click → Clone → “Generate new MAC addresses for all network adapters”. Otherwise MAC conflicts break networking.

### 2.2 Virtual Network Adapter Configuration (all VMs powered off)

#### windows‑GW (Gateway)
| Adapter | Attached to      | Name / Promiscuous      | Notes                     |
|---------|------------------|-------------------------|---------------------------|
| 1       | NAT              | -                       | internet out              |
| 2       | Internal Network | `intnet`                | to Relay                  |
| 3       | Disabled         | -                       | -                         |
| 4       | Host‑Only Adapter| VirtualBox Host‑Only    | for SSH from K1001 PC    |

#### windows‑Relay
| Adapter | Attached to      | Name / Promiscuous      | Notes                         |
|---------|------------------|-------------------------|-------------------------------|
| 1       | Internal Network | `intnet`                | to Gateway                    |
| 2       | Bridged          | Realtek USB GbE Family Controller, Allow All | to Node (physical cable)      |

#### windows‑Node
| Adapter | Attached to      | Name / Promiscuous      | Notes                         |
|---------|------------------|-------------------------|-------------------------------|
| 1       | Bridged          | Realtek USB GbE Family Controller #2, Allow All | to Relay (physical cable)     |

> **Promiscuous Mode = Allow All** is mandatory on bridged adapters of Relay and Node because the Relay acts as a router and must forward frames not destined to its own MAC.

---

### 2.3 Gateway (windows‑GW) Configuration

All commands run as Administrator in PowerShell.

#### Step GW‑1 – Identify interfaces
```powershell
Get-NetIPConfiguration
```
* NAT Adapter shows `10.0.2.15` (internet)
* Internal adapter shows `169.254.x.x` (no DHCP yet) – note its name, e.g. `"Ethernet 2"`

#### Step GW‑2 – Internet confirmation
```powershell
ping 1.1.1.1
```
Must get replies.

#### Step GW‑3 – Static IP on Internal interface
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet 2" 
-IPAddress 192.168.76.1 -PrefixLength 29
```
Verify: `Get-NetIPConfiguration -InterfaceAlias "Ethernet 2"`
#### Step GW‑4 – Enable IP forwarding (registry)
```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\
Parameters" -Name IPEnableRouter -Value 1
```
**Staged – requires reboot later.**

#### Step GW‑5 – Add return route to Node network (most commonly missed)
```powershell
New-NetRoute -DestinationPrefix 192.168.76.80/28 
-NextHop 192.168.76.2 -InterfaceAlias "Ethernet 2"
```
Without this, replies from internet to Node are dropped at Gateway.

#### Step GW‑6 – Install RRAS (routing role)

```powershell
Install-WindowsFeature -Name Routing -IncludeManagementTools
Install-RemoteAccess -VpnType RoutingOnly   # May require reboot first – see Troubleshooting
```
#### Step GW‑7 – Reboot

```powershell
Restart-Computer
```
After reboot, verify forwarding:

```powershell
Get-NetIPInterface | Select-Object InterfaceAlias, AddressFamily, Forwarding
```
Both IPv4 entries must show `Enabled`.

#### Step GW‑8 – Create NAT translation

```powershell
New-NetNat -Name LabNAT -InternalIPInterfaceAddressPrefix 192.168.76.0/24
Get-NetNat   # should show Active = True
```
#### Step GW‑9 – Firewall rules (allow essential traffic)

```powershell
# ICMP only from internal network (Lab 4 requirement)
New-NetFirewallRule -Name "Allow-ICMPv4-Internal" -DisplayName "Allow ICMPv4 from Internal Only" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow -RemoteAddress 192.168.76.0/24
# DHCP server inbound
New-NetFirewallRule -Name "Allow-DHCP-In" -DisplayName "Allow DHCP Inbound" -Protocol UDP -LocalPort 67 -Direction Inbound -Action Allow
# SSH from anywhere (school PC)
New-NetFirewallRule -Name "Allow-SSH-In" -DisplayName "Allow SSH Inbound" -Protocol TCP -LocalPort 22 -Direction Inbound -Action Allow
# Full internal trust (Relay can talk freely)
New-NetFirewallRule -Name "Allow-Internal-In" -DisplayName "Allow Internal Network" -Direction Inbound -Action Allow -RemoteAddress 192.168.76.0/24
```
#### Step GW‑10 – Set default inbound policy to Block (Lab 4 strict mode)

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Block
```
#### Step GW‑11 – Install DHCP Server role

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```
#### Step GW‑12 – Disable rogue detection (non‑domain fix)

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\DHCPServer\Parameters" -Name "DisableRogueDetection" -Value 1 -Type DWord
Restart-Service DHCPServer
Get-DhcpServerSetting   # must show IsAuthorized = True
```
#### Step GW‑13 – Create DHCP scope for Gateway–Relay link

```powershell
Add-DhcpServerv4Scope -Name "GW-to-Relay" -StartRange 192.168.76.2 -EndRange 192.168.76.6 -SubnetMask 255.255.255.248 -State Active
```
#### Step GW‑14 – MAC reservation for Relay’s Internal adapter

```powershell
Add-DhcpServerv4Reservation -ScopeId 192.168.76.0 -IPAddress 192.168.76.2 -ClientId "08-00-27-68-02-92" -Description "Relay Internal"
```
Replace MAC with actual address from VirtualBox → Relay → Adapter 2 → Advanced.

#### Step GW‑15 – Scope options (default gateway, DNS)

```powershell
Set-DhcpServerv4OptionValue -ScopeId 192.168.76.0 -Router 192.168.76.1 -DnsServer 1.1.1.1
```
#### Step GW‑16 – Restart DHCP service

```powershell
Restart-Service DHCPServer
Get-Service DHCPServer   # Running
```
#### Step GW‑17 – Install OpenSSH server

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```
---
### 2.4 Relay (windows‑Relay) Configuration

#### Step R‑1 – Identify interfaces by MAC

```powershell
Get-NetAdapter
```
Match MACs with VirtualBox settings:

-   Adapter 2 (Internal) → `"Ethernet 2"`
    
-   Adapter 3 (Bridged) → `"Ethernet 3"`
    

#### Step R‑2 – Static IP on Bridged interface (facing Node)

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet 3" -IPAddress 192.168.76.81 -PrefixLength 28
```
#### Step R‑3 – Enable IP forwarding (same registry key as GW)

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name IPEnableRouter -Value 1
```
#### Step R‑4 – Allow ICMP inbound (for testing)

```powershell
New-NetFirewallRule -Name "Allow-ICMPv4-In" -DisplayName "Allow ICMPv4 Ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```
#### Step R‑5 – Reboot

```powershell
Restart-Computer
```
After reboot, verify forwarding and test internet:

```powershell
Get-NetIPInterface | Select-Object InterfaceAlias, AddressFamily, Forwarding
ping 1.1.1.1   # must succeed – this confirms Gateway NAT works
```
#### Step R‑6 – Install DHCP Server role

```powershell
Install-WindowsFeature -Name DHCP -IncludeManagementTools
```
#### Step R‑7 – Disable rogue detection (again, non‑domain)

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\DHCPServer\Parameters" -Name "DisableRogueDetection" -Value 1 -Type DWord
Restart-Service DHCPServer
Get-DhcpServerSetting   # True
```
#### Step R‑8 – Create DHCP scope for Relay–Node segment

```powershell
Add-DhcpServerv4Scope -Name "Relay-to-Node" -StartRange 192.168.76.82 -EndRange 192.168.76.94 -SubnetMask 255.255.255.240 -State Active
```
#### Step R‑9 – MAC reservation for Node

```powershell
Add-DhcpServerv4Reservation -ScopeId 192.168.76.80 -IPAddress 192.168.76.82 -ClientId "08-00-27-59-D8-F3" -Description "Node"
```
Replace MAC with Node’s Adapter 3 MAC.

#### Step R‑10 – Scope options (gateway = Relay itself, DNS = 1.1.1.1)

```powershell
Set-DhcpServerv4OptionValue -ScopeId 192.168.76.80 -Router 192.168.76.81 -DnsServer 1.1.1.1
```
#### Step R‑11 – Restart DHCP service

```powershell
Restart-Service DHCPServer
```
#### Step R‑12 – Switch Internal interface to DHCP client (critical order)

```powershell
Remove-NetIPAddress -InterfaceAlias "Ethernet 2" -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet 2" -DestinationPrefix 0.0.0.0/0 -Confirm:$false
Set-NetIPInterface -InterfaceAlias "Ethernet 2" -Dhcp Enabled
ipconfig /release "Ethernet 2"
ipconfig /renew "Ethernet 2"
Get-NetIPConfiguration -InterfaceAlias "Ethernet 2"
```
Must show `IPv4Address 192.168.76.2`, `IPv4DefaultGateway 192.168.76.1`.

#### Step R‑13 – Install sshd (same as GW)

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
```
Set-Service -Name sshd -StartupType Automatic
----------

### 2.5 Node (windows‑Node) Configuration

Node has Desktop Experience (GUI). Use PowerShell as Administrator.

#### Step N‑1 – Identify interface

```powershell
Get-NetAdapter
```
Only one enabled (Bridged). Note name, e.g., `"Ethernet"`.

#### Step N‑2 – Switch to DHCP client

```powershell
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix 0.0.0.0/0 -Confirm:$false
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
ipconfig /renew "Ethernet"
```
#### Step N‑3 – Verify IP configuration

```powershell
ipconfig
```
Expected: `192.168.76.82`, mask `255.255.255.240`, gateway `192.168.76.81`.

#### Step N‑4 – Allow ICMP inbound (for testing)

```powershell
New-NetFirewallRule -Name "Allow-ICMPv4-In" -DisplayName "Allow ICMPv4 Ping" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```
----------

### 2.6 Verification (run from Node)

```cmd

ping 192.168.76.81   # Relay
ping 192.168.76.1    # Gateway
ping 1.1.1.1         # Internet
ping google.com      # DNS
tracert 1.1.1.1      # Should show Relay → Gateway → 10.0.2.2 → ...
```
**Hop‑by‑hop reasoning:**

-   If ping to Relay fails → physical cable or bridged promiscuous mode.
    
-   If ping to Gateway fails → Relay forwarding not enabled or Gateway return route missing.
    
-   If ping to 1.1.1.1 fails but Gateway works → NAT missing on Gateway (`Get-NetNat`).
    
----------

### 2.7 Lab 4 Firewall Strict Mode Verification

**From school PC (host) – Host‑Only network `192.168.56.x`:**

-   `ping 192.168.56.x` → **timed out** (ICMP blocked by default).
    
-   `ssh Administrator@192.168.56.x` → **successful login** (TCP/22 allowed).
    

**From Node (`192.168.76.82`):**

-   `ping 192.168.76.1` → **success** (internal network fully allowed).
    

**Toggle commands on Gateway (for demo):**

```powershell

Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Block   # Strict mode
Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Allow  # Open mode
Get-NetFirewallProfile | Select-Object Name, DefaultInboundAction
```
----------

## 3. Ubuntu Routing Lab (Ubuntu 26.04) – Labs 3 & 4

Parallel topology with identical IP scheme (Y=76). Three VMs: `ubuntu-gw`, `ubuntu-relay`, `ubuntu-node`. All Ubuntu Server 26.04 LTS (no GUI).

### 3.1 Network Adapter Mapping (VirtualBox)

#### ubuntu‑GW (Gateway)
| Adapter | Attached to      | Name / Promiscuous      | Notes                     |
|---------|------------------|-------------------------|---------------------------|
| 1       | NAT              | -                       | internet out              |
| 2       | Internal Network | `intnet`                | to Relay                  |
| 3       | Disabled         | -                       | -                         |
| 4       | Host‑Only Adapter| VirtualBox Host‑Only    | for SSH from K1001 PC    |

#### ubuntu‑Relay
| Adapter | Attached to      | Name / Promiscuous      | Notes                         |
|---------|------------------|-------------------------|-------------------------------|
| 1       | Internal Network | `intnet`                | to Gateway                    |
| 2       | Bridged          | Realtek USB GbE Family Controller, Allow All | to Node (physical cable)      |

#### ubuntu‑Node
| Adapter | Attached to      | Name / Promiscuous      | Notes                         |
|---------|------------------|-------------------------|-------------------------------|
| 1       | Bridged          | Realtek USB GbE Family Controller #2, Allow All | to Relay (physical cable)     |

### 3.2 Gateway (ubuntu-gw) Configuration

#### Set static IP on internal interface

Edit `/etc/netplan/00-installer-config.yaml`:

```yaml
network:
 version: 2
 ethernets:
 enp0s3:   # NAT – leave as DHCP
 dhcp4: true
 enp0s8:   # Internal
 dhcp4: false
 addresses:
 - 192.168.76.1/29
 nameservers:
 addresses: [1.1.1.1]
```
Apply: `sudo netplan apply`

#### Enable IP forwarding (permanent)

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
#### Add return route to Node network

```bash
sudo ip route add 192.168.76.80/28 via 192.168.76.2 dev enp0s8
```
Make persistent: create `/etc/systemd/network/10-return-route.network` or add to `/etc/rc.local`.

#### Install and configure NAT with iptables

```bash
sudo apt update
sudo apt install iptables-persistent
sudo iptables -t nat -A POSTROUTING -s 192.168.76.0/24 -o enp0s3 -j MASQUERADE
sudo netfilter-persistent save
```
#### Install Kea DHCP server 

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea-dhcp4-server -y
```
Edit `/etc/kea/kea-dhcp4.conf:

```text
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s8" ]
    },
    "lease-database": {
      "type": "memfile",
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/tmp/kea-dhcp4-ctrl.sock"
    },
    "subnet4": [
      {
        "id": 1,
        "subnet": "192.168.76.0/29",
        "pools": [
          {
            "pool": "192.168.76.2 - 192.168.76.6"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.76.1"
          },
          {
            "name": "domain-name-servers",
            "data": "1.1.1.1"
          }
        ],
        "reservations": [
          {
            "hw-address": "08:00:27:68:02:92",
            "ip-address": "192.168.76.2"
          }
        ]
      }
    ],
    "loggers": [
      {
        "name": "kea-dhcp4",
        "severity": "INFO",
        "output_options": [
          {
            "output": "stdout"
          }
        ]
      }
    ],
    "ddns-send-updates": false,
    "ddns-override-client-update": false,
    "ddns-override-no-update": false
  }
}
```

#### Firewall (ufw) – Lab 4 strict mode

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.76.0/24 to any
sudo ufw allow 22/tcp   # SSH from K1001 PC
sudo ufw enable
```
#### Enable SSH

```bash
sudo apt install openssh-server
sudo systemctl enable ssh --now
```
### 3.3 Relay (ubuntu-relay) Configuration

#### Static IP on bridged interface (enp0s10)

Netplan configuration:

```yaml
network:
 version: 2
 ethernets:
 enp0s8:   # Internal – will get DHCP
 dhcp4: true
 enp0s10:  # Bridged to Node
 dhcp4: false
 addresses:
 - 192.168.76.81/28
```
Apply.

#### Enable IP forwarding

Same as Gateway: `net.ipv4.ip_forward=1` in sysctl.

#### Install DHCP server for Node segment

Edit `/etc/kea/kea-dhcp4.conf` (second subnet):

```conf
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "enp0s10" ]
    },
    "lease-database": {
      "type": "memfile",
      "name": "/var/lib/kea/kea-leases4.csv"
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/tmp/kea-dhcp4-ctrl.sock"
    },
    "subnet4": [
      {
        "id": 2,
        "subnet": "192.168.76.80/28",
        "pools": [
          {
            "pool": "192.168.76.82 - 192.168.76.94"
          }
        ],
        "option-data": [
          {
            "name": "routers",
            "data": "192.168.76.81"
          },
          {
            "name": "domain-name-servers",
            "data": "1.1.1.1"
          }
        ],
        "reservations": [
          {
            "hw-address": "08:00:27:59:D8:F3",
            "ip-address": "192.168.76.82"
          }
        ]
      }
    ],
    "loggers": [
      {
        "name": "kea-dhcp4",
        "severity": "INFO",
        "output_options": [
          {
            "output": "stdout"
          }
        ]
      }
    ],
    "ddns-send-updates": false,
    "ddns-override-client-update": false,
    "ddns-override-no-update": false
  }
}
 hardware 
```
Restart.

#### Firewall – allow ICMP for testing

```bash
sudo ufw allow proto icmp from any to any
```
### 3.4 Node (ubuntu-node) Configuration

No static IP – use DHCP on bridged interface. Netplan:

```yaml
network:
 version: 2
 ethernets:
 enp0s3:
 dhcp4: true
```
Apply, then verify:

```bash
ip addr show enp0s3   # should show 192.168.76.82/28
ip route show default  # gateway 192.168.76.81
```
### 3.5 Ubuntu Verification (same hop‑by‑hop tests)

```bash
ping -c 4 192.168.76.81
ping -c 4 192.168.76.1
ping -c 4 1.1.1.1
ping -c 4 google.com
traceroute -n 1.1.1.1
```
----------

## 4. Problems Encountered and Solutions

### 4.1 Windows: `Install-RemoteAccess` command not found

**Problem:** After `Install-WindowsFeature -Name Routing`, the cmdlet was not available.  
**Cause:** The Routing role installation requires a reboot before the RemoteAccess PowerShell module becomes accessible.  
**Solution:** Rebooted the Gateway before running `Install-RemoteAccess`. After reboot, the command worked.

### 4.2 DHCP client on Node: “IP address already in use on the network”

**Problem:** Node obtained `192.168.76.82` but Windows disabled the interface with a duplicate address error, even though no other device used that IP.  
**Diagnosis:** Checked Relay DHCP leases: `Get-DhcpServerv4Lease -ScopeId 192.168.76.80 -AllLeases` showed `AddressState = InactiveReservation`. The reservation existed but had never been successfully leased.  
**Solution:**

1.  Removed the reservation: `Remove-DhcpServerv4Reservation -ScopeId 192.168.76.80 -IPAddress 192.168.76.82 -Confirm:$false`
    
2.  Re‑added the reservation with the same MAC.
    
3.  Restarted DHCP service on Relay.
    
4.  On Node: `ipconfig /release` and `ipconfig /renew`.  
    **Still had error.**
    
5.  Disabled Duplicate Address Detection on Node:
    
    ```powershell
    Set-NetIPInterface -InterfaceAlias "Ethernet" -AddressFamily IPv4 -DadTransmits 0
    ```
    Then rebooted Node. After reboot, `ipconfig /renew` succeeded and the interface stayed up.
      
    **Root cause:** VirtualBox’s bridged mode with promiscuous “Allow All” can cause ARP probes to be reflected, tricking Windows DAD into thinking the IP is taken. Disabling DAD in an isolated lab is safe.
    

### 4.3 Ubuntu: DHCP server not responding on Relay

**Problem:** Node received APIPA address (169.254.x.x) instead of `192.168.76.82`.  
**Cause:**  `kea-dhcp4-server` was bound to wrong interface; also needed to disable rogue detection (not a concept on Linux, but service wasn’t listening on bridged interface).  
**Solution:**

-   Verified `"interfaces": [ "enp0s10" ]` in `/etc/kea/kea-dhcp4.conf`.
    
-   Checked that `enp0s10` had the static IP `192.168.76.81/28`.
    
-   Restarted service and checked `sudo systemctl status kea-dhcp4-server`.
    
-   Also ensured `ufw` allowed DHCP (UDP port 67) – though by default DHCP server creates its own iptables rule.
    

### 4.4 Windows Relay could not ping Gateway after switching Internal to DHCP

**Problem:** After `Set-NetIPInterface -Dhcp Enabled`, Relay got `169.254.x.x`.  
**Cause:** Gateway’s DHCP firewall rule missing or rogue detection still enabled.  
**Solution:** On Gateway, confirmed `Get-DhcpServerSetting` showed `IsAuthorized = True`. Re‑ran `Set-ItemProperty` for `DisableRogueDetection` and restarted DHCP. Also verified `Allow-DHCP-In` firewall rule existed. Then on Relay, forced renewal: `ipconfig /renew "Ethernet 2"`. Worked.

### 4.5 Ubuntu: Default route missing on Relay after DHCP

**Problem:** Relay got IP `192.168.76.2` from Gateway’s DHCP but no default route.  
**Cause:**  `kea-dhcp4-server` on Gateway did not include `option-data` in the subnet declaration.  
**Solution:** Added `option-data 192.168.76.1;` to the subnet block and restarted DHCP. Then on Relay, `sudo dhclient -r enp0s8 && sudo dhclient enp0s8`.

----------

## 5. Lab 4 Firewall Summary
### 1. Expected Behavior – Strict Firewall Mode
| Source | Destination | Service / Protocol | Windows Result | Linux (UFW) Result |
|----------------------------|------------------|--------------------|-----------------------|-----------------------|
| School PC (`192.168.56.x`) | Gateway | ICMP (ping) | ❌ Blocked (timeout) | ❌ Blocked (timeout) |
| School PC (`192.168.56.x`) | Gateway | SSH (TCP/22) | ✅ Allowed (login) | ✅ Allowed (login) |
| Node (`192.168.76.82`) | Gateway | ICMP (ping) | ✅ Allowed (reply) | ✅ Allowed (reply) |
| Node (`192.168.76.82`) | Internet (any) | Any (HTTP, DNS, …) | ✅ Allowed (outbound) | ✅ Allowed (outbound) |
| Relay (`192.168.76.2`) | Gateway | DHCP (UDP/67) | ✅ Allowed (lease) | ✅ Allowed (lease) |
**Note:** All traffic from inside `192.168.76.0/24` is fully permitted (Windows `Allow-Internal-In` rule; Linux `ufw allow from 192.168.76.0/24`).
---
## 2. Verification Steps
### From the K1001 PC (Host)
Run these commands on the **host** machine (or another VM on the same Host‑Only network):
```bash
ping 192.168.56.101          # Should time out (replace with Gateway's Host‑Only IP)
ssh administrator@192.168.56.101   # Should succeed (password prompt)
```

**How to verify on Windows:**

-   Check current default action: `Get-NetFirewallProfile | Select Name, DefaultInboundAction`
    
-   Toggle: `Set-NetFirewallProfile -Profile Domain,Public,Private -DefaultInboundAction Block/Allow`
    

**On Ubuntu (ufw):**

-   `sudo ufw status verbose` shows default incoming policy.
    
-   Toggle: `sudo ufw default deny incoming` / `sudo ufw default allow incoming`.
    
----------

## 6. Conclusion

The routing lab was successfully implemented for **Y = 76** on both Windows Server 2022 and Ubuntu 26.04. All three VMs (Gateway, Relay, Node) can communicate through the intended chain, and the Node reaches the internet via NAT on the Gateway. Lab 4’s strict firewall policy blocks unsolicited inbound traffic except SSH, while allowing full internal communication and outbound connections. The encountered problems (missing RRAS cmdlet, DHCP inactive reservation, Ubuntu DHCP interface binding) were documented and resolved. The configuration is reproducible by following the steps above.

**Final verification from Node (Windows):**

```cmd
C:\> ping 192.168.76.81   → replies 
C:\> ping 192.168.76.1    → replies 
C:\> ping 1.1.1.1         → replies 
C:\> tracert 1.1.1.1      → 1: 192.168.76.81, 2: 192.168.76.1, 3: 10.0.2.2
```
