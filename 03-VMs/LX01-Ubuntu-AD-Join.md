---
tags: [az-802, vms, linux, ubuntu, sssd, realmd, hybrid-identity]
---
# LX01 — Ubuntu Server Joined to the AD Domain

The hybrid piece: an **Ubuntu Server** VM joined to `lab.local` via **realmd + SSSD**, so **domain users can SSH in with their AD credentials**. Mirrors a real environment (Linux servers + AD identity + Windows clients) and is the cross-platform skill most Windows-only admins lack.

> [!NOTE]
> **Naming:** libvirt domain = `LX01`, guest hostname = `lx01.lab.local`, static IP `10.10.10.30`. If you named the VM something else at `virt-install` time, that name is what `virsh`/`lab up` refer to — the hostname and IP below are independent of it.

> [!NOTE]
> On KVM this VM is the easy one — Linux has **native virtio** drivers, so use `--disk bus=virtio --network model=virtio` with no driver hassle (unlike the Windows guests in [[Creating-VMs]]). 1–2 GB RAM is plenty.

## 1. Prerequisites (the same two lessons as the Windows joins)
> [!IMPORTANT]
> **DNS must point at the DCs**, or `realm discover` finds nothing — identical to [[DNS-and-DHCP]]. And **time must be in Kerberos tolerance** (~5 min) — identical to [[Time-and-NTP]].

Static IP + DNS via netplan (`/etc/netplan/00-installer-config.yaml`):
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [10.10.10.30/24]
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses: [10.10.10.10, 10.10.10.11]
        search: [lab.local]
```
```bash
sudo netplan apply
sudo hostnamectl set-hostname lx01.lab.local
```

Point time at the DC (chrony) so Kerberos is happy:
```bash
# /etc/chrony/chrony.conf  -> add:
#   server 10.10.10.10 iburst
sudo systemctl restart chrony
chronyc sources
```

## 2. Install the join stack
```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss \
  adcli samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user chrony
```
> [!TIP]
> When `krb5-user` prompts for the default realm, enter **`LAB.LOCAL`** (uppercase — Kerberos realms are case-sensitive).

## 3. Discover and join
```bash
realm discover lab.local            # should list the domain + required packages
sudo realm join --verbose -U Administrator lab.local
realm list                          # confirm: domain joined, sssd configured
id Administrator@lab.local          # should resolve to a uid/gid
```

## 4. Make it usable
Short usernames + auto-create home dirs — edit `/etc/sssd/sssd.conf`:
```ini
use_fully_qualified_names = False
fallback_homedir = /home/%u
```
```bash
sudo pam-auth-update --enable mkhomedir
sudo systemctl restart sssd
```

Allow logins and grant sudo to a domain group:
```bash
sudo realm permit --all                     # lab convenience (or: realm permit user@lab.local)

# sudo for an AD group (make a "LinuxAdmins" group in AD first):
echo '%linuxadmins ALL=(ALL) ALL' | sudo tee /etc/sudoers.d/linuxadmins
```
> [!CAUTION]
> AD group names with **spaces** must be escaped in sudoers, e.g. `%domain\ admins ALL=(ALL) ALL`. Prefer a dedicated space-free group like `LinuxAdmins`.

## 5. Verify end to end
```bash
id someuser@lab.local
su - someuser            # (short name works after step 4) -> home dir auto-created
# from CLIENT01 or anywhere on the lab net:
ssh someuser@10.10.10.30
```

## Common failures (same family as [[Troubleshooting-Log]])
- `realm discover` returns nothing → DNS not pointing at the DC. Fix netplan nameservers.
- Join fails with a clock/KRB error → time skew. Fix chrony (above).
- Login works but no home dir → `mkhomedir` not enabled (step 4).
- `realm join` warns about reverse DNS → create a PTR/reverse zone on the DC (cosmetic in a lab).

## Why this matters for a hybrid role
> [!TIP]
> One line that proves cross-platform range: *"Windows Server + AD for identity, Ubuntu for the workloads, with SSSD binding Linux logins to the same AD accounts — one identity plane across both."* That's an enterprise hybrid environment in miniature.

## Where it leads
- File interop: add **Samba/NFS** between LX01 and [[FS01-File-Server]].
- LX01 becomes the natural **syslog / SIEM collector** in [[Purple-Team-Range]].
