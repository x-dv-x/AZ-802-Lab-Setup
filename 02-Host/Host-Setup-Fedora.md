---
tags: [az-802, host, fedora, kvm, libvirt]
---
# Host Setup — Fedora + KVM/QEMU/libvirt

## Install the virtualization stack
```bash
sudo dnf install -y @virtualization edk2-ovmf swtpm virtio-win
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt "$USER"      # log out/in for the group to apply
```
- `edk2-ovmf` = UEFI firmware (Server 2025 / Win 11 want UEFI)
- `swtpm` = emulated **TPM 2.0** (required for the Win 11 client)
- `virtio-win` = paravirtual drivers, installed to `/usr/share/virtio-win/` (on Fedora you get this from the repo; other distros download it by hand)

## Enable nested virtualization (Intel)
Needed so a Windows Server guest can run **Hyper-V**.
```bash
cat /sys/module/kvm_intel/parameters/nested      # recent Fedora often ships this ON (Y)
# if it shows N:
echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm-intel.conf
sudo modprobe -r kvm_intel && sudo modprobe kvm_intel   # or reboot
```
Then set **host-passthrough** CPU on the VM that will run Hyper-V.

> [!CAUTION]
> **Use `qemu:///system`, not the user session.** If you create VMs in virt-manager's *QEMU/KVM (system)* connection but your shell `virsh` defaults to the session URI, they see two different VM sets — and `virsh` will claim your VM "doesn't exist." Fix:
> ```bash
> export LIBVIRT_DEFAULT_URI=qemu:///system
> ```
> This caused a real "failed to get domain" error — see [[Troubleshooting-Log]].

## Next
- [[Storage-Pool]] — where the VM disks live
- [[Networking]] — the lab network
