---
tags: [az-802, vms, virt-install, iso]
---
# Creating the VMs

## ISOs you need
- **Windows Server 2025** (and/or 2022) — 180-day evaluation (extend with `slmgr -rearm`)
- **Windows 11 Enterprise** — 90-day evaluation → see the [[CLIENT01-Windows11]] edition warning
- **virtio-win** — on Fedora already at `/usr/share/virtio-win/virtio-win.iso`

## virt-install example (UEFI + TPM)
```bash
osinfo-query os | grep -iE 'win2k2|win11'   # pick the closest code; win2k22 if win2k25 absent

virt-install --connect qemu:///system \
  --name DC01 --osinfo win2k25 \
  --memory 4096 --vcpus 2 --cpu host-passthrough --machine q35 --boot uefi \
  --disk path=/mnt/az802lab/DC01.qcow2,size=50,bus=virtio,format=qcow2 \
  --disk path=/path/to/WindowsServer_EVAL.iso,device=cdrom \
  --disk path=/usr/share/virtio-win/virtio-win.iso,device=cdrom \
  --network network=az802-lab,model=virtio \
  --tpm backend.type=emulator,backend.version=2.0,model=tpm-crb \
  --graphics spice --video qxl
```

## Boot-device menu — which entry?
On first boot the UEFI menu lists several entries:
- **UEFI QEMU DVD-ROM (QM0000x)** → your install media. Two will show (Windows ISO + virtio-win). The virtio-win ISO is **not bootable**, so if one doesn't launch Setup, pick the other.
- **UEFI Misc Device** → the blank virtual hard disk (install *target*, nothing to boot yet).
- **PXE / HTTP** → network boot — not what you want.
- Watch for "Press any key to boot from CD…" and tap fast, or it falls through.

## virtio vs e1000e / SATA
> [!WARNING]
> With a **virtio** disk or NIC, Windows Setup shows **no drive** / Windows shows **no network adapter**, because Windows has no built-in virtio driver.
> - **Disk:** at Setup, *Load driver* → virtio-win DVD → `vioscsi`/`viostor` for your OS version.
> - **NIC:** load `NetKVM`, **or** just use `--network model=e1000e` and `--disk bus=sata`, which Windows drives natively with zero fuss (slightly slower — fine for a lab).

See the full symptom/fix in [[Troubleshooting-Log]].

## Tip: build one, clone the rest
Install a **Server Core** "golden" VM, **generalize it with `sysprep /generalize /oobe /shutdown`**, then clone for DC02/FS01/nodes. Without sysprep every clone has the same machine SID, which matters most for anything you'll promote to a DC. Remember to set the **timezone** on every clone — it is *not* inherited or synced ([[Time-and-NTP]]).

> [!WARNING]
> Clone the **golden image**, never an existing **DC**. A DC clone carries that DC's AD identity. See [[CDC01-Second-Forest]].

## First-boot CPU looks insane?
That's normal post-install churn (Windows Update, Defender's first scan, `TiWorker`, `SearchIndexer`) — self-limiting, leave it ~20–60 min. Details + the Hyper-V enlightenment tweak in [[Troubleshooting-Log]].
