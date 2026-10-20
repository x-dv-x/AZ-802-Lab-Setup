---
tags: [az-802, planning, study, exam]
---
# AZ-802 — Exam Domains Mapped to the Lab

> [!IMPORTANT]
> Always cross-check against the live **"skills measured"** page on Microsoft Learn — Microsoft re-weights it periodically.

| Exam theme | Where in this lab |
|------------|-------------------|
| AD DS (deploy/manage, multi-DC, FSMO, sites, RODC) | [[Active-Directory]], [[DC01-Primary-DC]], [[DC02-Second-DC]] |
| DNS / DHCP / IPAM | [[DNS-and-DHCP]] |
| Storage & file services (Storage Spaces, iSCSI, DFS, dedup) | [[FS01-File-Server]] |
| VMs & containers (Hyper-V nested, containers) | *HA/virtualization phase (planned)* |
| High availability (failover clustering, Storage Replica) | *cluster phase — NODE1/NODE2 (planned)* |
| Securing Windows Server (LAPS, JEA, BitLocker, firewall) | *security phase (planned)* |
| Backup & DR (WSB, Azure Backup) | [[Update-and-Debloat]] (update mgmt) + *backup phase (planned)* |
| Monitoring & troubleshooting (WAC, PerfMon, dcdiag) | [[Troubleshooting-Log]] |
| Hybrid (Azure Arc, Update Manager, Azure Monitor) | *needs a free Azure account — planned* |

## Suggested order
1. Host + networking + DC01/DC02 → [[Host-Setup-Fedora]], [[DC01-Primary-DC]], [[DC02-Second-DC]]
2. DNS, DHCP, time → [[DNS-and-DHCP]], [[Time-and-NTP]]
3. Storage & file services → [[FS01-File-Server]]
4. Nested Hyper-V + containers
5. Failover clustering + replication
6. Security hardening
7. Backup/DR + monitoring
8. Hybrid/Azure

> [!TIP]
> Run a practice exam after step 4 and again after step 8; aim for 85%+ before booking.
