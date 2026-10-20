---
tags: [az-802, services, dns, dhcp]
---
# DNS & DHCP

## DNS (AD-integrated)
On the PDC after `Install-ADDSForest`, point the DC at itself and forward externally:
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
Add-DnsServerForwarder -IPAddress 1.1.1.1 -PassThru
```
Second DC uses **preferred = partner, alternate = self** (anti-islanding) — see [[DC02-Second-DC]]. Forwarders are **per-server** and don't replicate.

> [!TIP]
> Clients must use the **DCs** as DNS (`10.10.10.10,10.10.10.11`), never a public resolver — domain join relies on SRV records only the DCs hold. A wrong-DNS client is the #1 cause of "domain could not be contacted."

## DHCP
```powershell
# authorize + scope (on DC01)
Add-DhcpServerInDC
Add-DhcpServerv4Scope -Name "LabScope" -StartRange 10.10.10.100 -EndRange 10.10.10.200 -SubnetMask 255.255.255.0
Set-DhcpServerv4OptionValue -ScopeId 10.10.10.0 -DnsServer 10.10.10.10,10.10.10.11 -Router 10.10.10.1 -DnsDomain lab.local

# verify
Get-DhcpServerInDC
Get-DhcpServerv4Scope
```

> [!NOTE]
> **DHCP authorization "Failed — Error 20079 / already present"** is harmless: it means DHCP is *already* authorized in AD, you can't do it twice. Just `Restart-Service DHCPServer` so the created security groups apply. See [[Troubleshooting-Log]].
