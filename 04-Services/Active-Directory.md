---
tags: [az-802, services, active-directory, fsmo, replication]
---
# Active Directory — Forest, Replication, FSMO, Sites, RODC

Domain: `lab.local` · DCs: [[DC01-Primary-DC]], [[DC02-Second-DC]]
Trusts with a second forest: [[CDC01-Second-Forest]] → [[Forest-Trusts]]

## Replication
```powershell
repadmin /replsummary          # health — want 0 failures
repadmin /showrepl
repadmin /syncall /AdeP        # force a sync
Get-ADDomainController -Filter * | ft Name,Site,IPv4Address,IsGlobalCatalog
```

## FSMO roles (transfer, then seizure)
```powershell
netdom query fsmo             # who holds what

# graceful transfer to DC02:
Move-ADDirectoryServerOperationMasterRole -Identity DC02 `
  -OperationMasterRole PDCEmulator,RIDMaster,InfrastructureMaster,SchemaMaster,DomainNamingMaster

# seizure (holder dead) — add -Force
```
> [!CAUTION]
> If you **seize** roles while DC01 is off, **do not boot DC01 back up** with its old roles — that's a split-brain. Snapshot both DCs, practice the seizure, then roll back. This is exactly what [[Gotchas-and-Learnings|snapshots]] are for.

> [!IMPORTANT]
> The **external NTP source follows the PDC role**, but *not automatically*. If you move the PDC emulator, reconfigure the new holder as the time source — see [[Time-and-NTP]].

## Sites & Services
```powershell
New-ADReplicationSite   -Name "Branch-Site"
New-ADReplicationSubnet -Name "10.10.20.0/24" -Site "Branch-Site"
New-ADReplicationSiteLink -Name "HQ-Branch" -SitesIncluded "Default-First-Site-Name","Branch-Site" -Cost 100 -ReplicationFrequencyInMinutes 15
Move-ADDirectoryServer -Identity DC02 -Site "Branch-Site"
```

## RODC (needs its own small VM)
```powershell
Install-ADDSDomainController -DomainName lab.local -Credential (Get-Credential LAB\Administrator) `
  -ReadOnlyReplica:$true -SiteName "Branch-Site" -InstallDns:$true `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM") -Force
```
The exam-relevant concept to explore is the **Password Replication Policy** — which accounts' credentials the RODC may cache.
