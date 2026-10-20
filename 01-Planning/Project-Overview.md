---
tags: [az-802, planning, overview]
---
# Project Overview

## Goal
Build a realistic, reproducible **Windows Server + Active Directory** lab on a Linux host to prepare for the **AZ-802 (Administering Windows Server)** certification — and to have concrete, hands-on infrastructure experience to point to.

AZ-802 is the single exam that replaced AZ-800 + AZ-801 (those retired in 2026). It became the **Windows Server Administrator Associate** credential. See [[AZ-802-Study-Mapping]] for how each lab phase maps to the exam objectives.

## Why a Linux host for a Windows cert?
Because the hypervisor doesn't matter for the objectives — the *guests* do. On Linux you run **KVM/QEMU + libvirt** and spin up Windows Server/Client VMs on top. The only Windows-only feature, **Hyper-V**, is practiced *nested* inside a Windows Server guest.

## Scope of the lab
- A single AD forest/domain: `lab.local`
- Two domain controllers ([[DC01-Primary-DC]], [[DC02-Second-DC]]) for replication / FSMO / Sites practice
- A file & storage server ([[FS01-File-Server]]) that doubles as the future iSCSI target for failover clustering
- A Windows 11 management client ([[CLIENT01-Windows11]]) running RSAT / Windows Admin Center
- Core services: [[Active-Directory]], [[DNS-and-DHCP]], [[Time-and-NTP]]

## Design principles
- **Server Core by default** to save RAM; GUI only while learning a console.
- **Snapshot before anything destructive** (FSMO seizure, cluster builds).
- **One authoritative clock** (the PDC) — see [[Time-and-NTP]].
- Keep the lab on its **own segmented subnet**, never bridged onto the home LAN (a lab DC runs DHCP — see [[Networking]]).

> [!NOTE]
> This vault documents the build in the order it actually happened, including the detours. The [[Troubleshooting-Log]] is the record of what broke.
