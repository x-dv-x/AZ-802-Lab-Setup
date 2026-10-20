---
tags: [az-802, vms, domain-controller, active-directory]
---
# DC01 — Primary Domain Controller

The first DC: holds all FSMO roles initially, is the DNS + DHCP server, and is the **authoritative time source** for the domain.

## 1. Static IP (lab network has no DHCP yet)
```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.10 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 1.1.1.1   # temporary, pre-AD
```
> [!NOTE]
> `New-NetIPAddress` / `Set-DnsClientServerAddress` write to the registry — they **persist across reboots**. No need to reapply.

## 2. Promote to a forest
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "lab.local"
```
After promotion, flip DNS to itself and add a forwarder — see [[DNS-and-DHCP]].

## 3. DNS / DHCP
Zones are **AD-integrated**. Authorize DHCP and create the scope `10.10.10.100–200`. Full steps: [[DNS-and-DHCP]].

## 4. Time
As the **PDC emulator**, DC01 must point at an external NTP source; everyone else follows it. See [[Time-and-NTP]].

## Related
- [[DC02-Second-DC]] replicates from here
- [[Active-Directory]] for FSMO / replication practice
