---
tags: [az-802, homelab, moc, windows-server, active-directory]
---

# 🗺️ Index — AZ-802 Home Lab

The map of content for this vault. New here? Follow the order below.

## 1. Planning
- [[Project-Overview]] — what this lab is and why it exists
- [[Lab-Topology]] — network, IP plan, VM list, RAM budget
- [[AZ-802-Study-Mapping]] — exam domains mapped to lab phases

## 2. Host (Fedora / KVM)
- [[Host-Setup-Fedora]] — packages, nested virt, libvirt
- [[Storage-Pool]] — external NVMe as a libvirt pool (+ a hard-won performance lesson)
- [[Networking]] — NAT + isolated lab network, the IP plan

## 3. Building the VMs
- [[Creating-VMs]] — ISOs, virt-install, UEFI/TPM, virtio vs e1000e, boot-device selection
- [[DC01-Primary-DC]] — first domain controller
- [[DC02-Second-DC]] — second DC + replication
- [[FS01-File-Server]] — file server / future iSCSI target
- [[CLIENT01-Windows11]] — Windows 11 management client
- [[LX01-Ubuntu-AD-Join]] — Ubuntu Server joined to AD (hybrid identity)
- [[CDC01-Second-Forest]] — a second forest (`corp.local`) to build trusts against

## 4. Core services
- [[Active-Directory]] — forest, replication, FSMO, Sites & Services, RODC
- [[Forest-Trusts]] — conditional forwarders, forest trust, selective authentication, one-way admin-forest pattern
- [[DNS-and-DHCP]] — AD-integrated DNS, forwarders, DHCP scope + authorization
- [[Time-and-NTP]] — the domain time hierarchy (and the #1 time trap)
- [[Update-and-Debloat]] — taming Windows Update + stripping bloat

## 5. Lessons (the good stuff)
- [[Troubleshooting-Log]] — every error hit, with cause + fix
- [[Gotchas-and-Learnings]] — the distilled principles

## 6. Where it goes next
- [[Purple-Team-Range]] — turning the lab into an attack/detect range

---
> [!TIP]
> If you only read two notes, read [[Troubleshooting-Log]] and [[Gotchas-and-Learnings]]. That's the stuff no tutorial tells you.
