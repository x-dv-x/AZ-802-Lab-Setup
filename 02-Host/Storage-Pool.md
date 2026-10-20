---
tags: [az-802, host, storage, performance]
---
# Storage Pool — External NVMe

## Create the pool
```bash
lsblk -f
# ---- FORMATTING ERASES THE DISK ----
sudo mkfs.xfs -L az802lab /dev/sdX1        # xfs or ext4; see btrfs note
sudo mkdir -p /mnt/az802lab
sudo mount /dev/sdX1 /mnt/az802lab
sudo chown -R "$USER":"$USER" /mnt/az802lab

virsh pool-define-as az802 dir --target /mnt/az802lab
virsh pool-build az802 && virsh pool-start az802 && virsh pool-autostart az802
```

## Fedora/SELinux gotcha
SELinux is enforcing. If a VM won't start with a permission error:
```bash
sudo semanage fcontext -a -t virt_image_t "/mnt/az802lab(/.*)?"
sudo restorecon -Rv /mnt/az802lab
sudo ausearch -m avc -ts recent          # read denials
```

## btrfs note
If you format btrfs, disable copy-on-write on the VM dir (`chattr +C` on an empty dir) or qcow2 performance tanks. xfs/ext4 avoid this.

## ⚠️ The big performance lesson
> [!WARNING]
> **AD DS on external USB storage is painfully slow.** The NTDS (ESENT/JET) database calls `fsync` on every transaction commit, and promotion is thousands of commits. On an external USB NVMe, flushes measured **~53 ms each** — so DC promotion *looked* frozen at "Waiting for DNS installation to finish" when it was actually just grinding the database to disk.
>
> Diagnosed with `virsh domstats <vm> --block` (watch `block.0.wr.reqs` climb + huge `fl.times`). Full story in [[Troubleshooting-Log]].

**Fixes:**
- Prefer the **internal NVMe** if it has room — removes the bottleneck and the USB-autostart caveat.
- Or set disk cache to `writeback` (faster flushes via host page cache): in `virsh edit <vm>`, `<driver name='qemu' type='qcow2' cache='writeback'/>`. Trade-off: data loss risk if the drive disconnects — fine for a lab, never production.

## USB caveats
- Keep the SSD on your fastest port; never unplug while VMs run.
- If it's absent at boot, autostart fails — re-attach and `virsh pool-start az802`.
