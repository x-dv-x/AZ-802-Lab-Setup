---
tags: [az-802, vms, domain-controller, forest, trusts]
---
# CDC01 — Second Forest (corp.local)

A standalone forest root DC for **`corp.local`**, so there is a second forest to build trusts against. It backs the trust objectives (forest trusts, selective authentication, SID filtering, a one-way "admin forest" design), and the hands-on steps live in [[Forest-Trusts]].

> [!NOTE] Status
> Written during the build (2026-10-30). Anything that behaves differently when run gets corrected here and logged in [[Troubleshooting-Log]].

| | |
|---|---|
| Hostname | CDC01 |
| IP | 10.10.10.12 (static) |
| Domain | `corp.local` (NetBIOS `CORP`), its **own forest** |
| RAM | 2 GB |
| OS | Windows Server 2025 |

## Why a fresh VM and not a clone of DC02
> [!WARNING]
> A raw clone of a DC **is** that DC: same name, IP, machine SID and AD identity.
> - **Boot it while DC01 can reach it:** DC01 replicates with it as if it were DC02. Bring the real DC02 back later and you have a USN rollback.
> - **Demote it normally:** DC01 deletes DC02's DC metadata, which breaks the *real* DC02.
> - **Microsoft's supported DC cloning** (`DCCloneConfig.xml` + the *Cloneable Domain Controllers* group) needs VM-GenerationID and only produces another DC in the **same** domain, not a new forest.
>
> Even a plain member-server clone needs **sysprep** before it becomes a DC. The first DC of a new domain builds the domain SID from its machine SID, so two un-sysprepped clones promoted into two forests are reported to get matching domain SIDs, and the trust between them fails.
>
> Cheapest correct option: a fresh install from the ISO (or from a sysprepped golden image).

## Create the VM
```bash
virt-install --connect qemu:///system --name CDC01 --osinfo win2k25 \
  --memory 2048 --vcpus 2 --cpu host-passthrough --machine q35 --boot uefi \
  --disk path=/mnt/labs/CDC01.qcow2,size=50,bus=virtio,format=qcow2 \
  --disk path=/mnt/labs/Windows_server_2025.iso,device=cdrom \
  --disk path=/usr/share/virtio-win/virtio-win.iso,device=cdrom \
  --network network=az802-lab,model=virtio \
  --metadata genid_enable=yes \
  --graphics spice --video qxl --noautoconsole
```
- **`--connect qemu:///system`** puts the VM where the lab network lives. Without it you get *"Network not found: no network with matching name 'az802-lab'"* ([[Troubleshooting-Log]] T16).
- **`--metadata genid_enable=yes`** gives the VM a **VM-GenerationID**. A Windows Server 2012+ DC uses it to notice a snapshot revert and protect itself (new invocation ID, RID pool thrown away) instead of silently causing a USN rollback. Check it's there:
  ```bash
  virsh -c qemu:///system dumpxml CDC01 | grep genid
  ```
- **No `--tpm`.** Windows Server doesn't need one; only [[CLIENT01-Windows11]] does.
- **virtio drivers:** at Setup → *Load driver* → virtio-win DVD → `viostor\2k25\amd64` (use `2k22` if there's no `2k25` folder). After install, run `virtio-win-gt-x64.msi` from the same DVD for the NIC and the rest. See [[Creating-VMs]].

## Base config (before promotion)
```powershell
Test-Connection 10.10.10.12 -Count 2     # from another VM first: nothing should answer
Get-NetAdapter                           # confirm the interface name

Rename-Computer -NewName CDC01 -Restart
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.12 -PrefixLength 24 -DefaultGateway 10.10.10.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.12
Set-TimeZone -Id "Eastern Standard Time"
```
> [!IMPORTANT]
> **CDC01's DNS points at itself, never at DC01.** It will be the only DNS server for `corp.local`. DC01 only knows `lab.local`. Lookups between the forests go through **conditional forwarders**, set up in [[Forest-Trusts]].

## Promote: new forest
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
  -DomainName "corp.local" `
  -DomainNetbiosName "CORP" `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (Read-Host -AsSecureString "DSRM password") `
  -Force:$true
```
The DSRM password needs 8+ characters (see T5 in [[Troubleshooting-Log]]). After the reboot:
```powershell
Add-DnsServerForwarder -IPAddress 1.1.1.1      # internet = forwarder, not client DNS
Get-ADForest | Format-List Name, RootDomain, ForestMode, Domains
```

## Time (the trust depends on it)
CDC01 is the **PDC emulator of its own forest**, so the lab.local time hierarchy ([[Time-and-NTP]]) doesn't reach it. By default it trusts its own clock. Trusts authenticate with Kerberos, which fails past ~5 minutes of skew, so sync it to DC01:
```powershell
w32tm /config /manualpeerlist:"10.10.10.10" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
w32tm /query /status          # Source should be 10.10.10.10
```

## Snapshot the clean forest
```bash
virsh -c qemu:///system shutdown CDC01
virsh -c qemu:///system snapshot-create-as CDC01 corp-forest-clean
```
CDC01 is the only DC in its forest, so reverting it alone is fine. Reverting **one of DC01/DC02** alone is not (see [[Gotchas-and-Learnings]]).

## Running it
CDC01 isn't in a `lab` profile. The `storage` profile happens to be everything else the trust labs need:
```bash
lab up storage                         # DC01 FS01 CLIENT01
virsh -c qemu:///system start CDC01
```
That's 4 + 4 + 4 + 2 = **14 GB** of guests. Leave DC02 off. `lab down` stops everything that's running, CDC01 included.

## Removing it
Remove the trusts and the `corp.local` conditional forwarder on DC01 first (teardown in [[Forest-Trusts]]), then:
```bash
virsh -c qemu:///system destroy CDC01 2>/dev/null                       # if still running
virsh -c qemu:///system undefine CDC01 --nvram --snapshots-metadata --remove-all-storage   # deletes the disk too
```
