---
tags: [az-802, vms, domain-controller, replication]
---
# DC02 — Second Domain Controller

Adds redundancy and unlocks the AD objectives that need two DCs: **replication, FSMO transfer/seizure, Sites & Services, RODC**.

## Prerequisite
> [!IMPORTANT]
> Before promotion, DC02's DNS **must point at DC01 (10.10.10.10)** — that's how it finds the domain.
> ```powershell
> Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10
> nltest /dsgetdc:lab.local       # must return DC01
> ```

## Promote (as a DC *and* a 2nd DNS server)
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSDomainController `
  -DomainName "lab.local" `
  -Credential (Get-Credential "LAB\Administrator") `
  -InstallDns:$true `
  -SiteName "Default-First-Site-Name" `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
  -Force:$true
```
Because DNS zones are AD-integrated, `-InstallDns:$true` makes DC02 a working second DNS server automatically.

## Fix DNS client settings after promo (anti-islanding)
```powershell
# DC02: preferred = partner, alternate = self
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10,127.0.0.1
Add-DnsServerForwarder -IPAddress 1.1.1.1 -PassThru
```

## Verify replication
```powershell
repadmin /replsummary          # 0 failures
New-ADUser -Name "repltest" -Server DC01
repadmin /syncall /AdeP
Get-ADUser repltest -Server DC02   # should now exist
```

## Failures hit here (see [[Troubleshooting-Log]])
- `virsh shutdown DC02` → "failed to get domain" — **qemu:///system** URI issue
- DSRM password rejected — `Read-Host` arg is the *prompt*, not the value; and 7 chars < 8-char minimum
- Promotion "stuck" at *Waiting for DNS installation* — actually **I/O-bound on USB storage** ([[Storage-Pool]])
- `-NoGlobalCatalogue` → typo; the parameter is `-NoGlobalCatalog`

## Then practice
FSMO, Sites & Services, RODC → [[Active-Directory]].
