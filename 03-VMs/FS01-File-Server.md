---
tags: [az-802, vms, file-server, iscsi, storage]
---
# FS01 — File & Storage Server

The member server for storage objectives, and the **iSCSI target** that will provide shared storage for the future failover cluster.

## Join the domain (rename + join)
```powershell
# DNS must point at the DC first, or discovery fails:
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10,10.10.10.11

Add-Computer -DomainName lab.local -NewName "FS01" -Credential (Get-Credential LAB\Administrator) -Restart
```
> [!CAUTION]
> `-Credential LAB\Administrator` (a bare string) **fails** with "argument is null or empty." You must pass a **PSCredential object**: `(Get-Credential LAB\Administrator)`. See [[Troubleshooting-Log]].

## Pre-join sanity checks
```powershell
nltest /dsgetdc:lab.local                 # DC discoverable?
w32tm /stripchart /computer:10.10.10.10 /samples:3 /dataonly   # time sane? (Kerberos)
```

## Storage objectives to build here
- **Storage Spaces** pool + virtual disk
- **iSCSI target** (reused as cluster shared storage)
- SMB shares, **DFS-N + DFS-R**, **data deduplication**, FSRM quotas

> [!TIP]
> Put the **iSCSI target role on FS01, not on a DC.** Shared storage served from the file server, with the cluster nodes as initiators — clean roles, and how you'd build it for real.

## Hybrid extension
Pair with [[LX01-Ubuntu-AD-Join]] for **Samba/NFS interop** between a Windows file server and a Linux client/server — a realistic cross-platform storage scenario.
