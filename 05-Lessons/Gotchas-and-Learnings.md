---
tags: [az-802, lessons, principles]
---
# 💡 Gotchas & Learnings (Distilled)

The principles behind the specific fixes in [[Troubleshooting-Log]].

## Active Directory
- **Time is load-bearing.** One authoritative clock (PDC → external NTP), everyone else `NT5DS`. Kerberos dies past ~5 min skew. → [[Time-and-NTP]]
- **Timezone ≠ time sync.** AD syncs UTC only; set the timezone on every machine by hand.
- **DNS must point at the DCs** on every domain member, or joins and logons fail.
- **Snapshot before destructive AD ops** (FSMO seizure, RODC). Seizing roles then rebooting the old holder = split-brain.
- The **external NTP config does not follow the PDC role** — reconfigure after moving it.
- **Revert DC snapshots in pairs.** Reverting one DC of a multi-DC domain on its own (with no VM-GenerationID) is a USN rollback: its partners ignore its changes and nothing warns you. Shut both down, snapshot both, revert both.
- **Never clone a DC to get a "new" DC or forest.** The copy *is* the original DC as far as AD is concerned. Build fresh. → [[CDC01-Second-Forest]]
- **A second forest has its own time hierarchy.** Its PDC trusts its own clock until you point it somewhere, and trusts are Kerberos, so set it up before building a trust. → [[Forest-Trusts]]

## KVM / libvirt
- **Stick to `qemu:///system`.** Mixed URIs make VMs "disappear" (T1) and make `virt-install` fail with "Network not found" (T16). Pass `--connect qemu:///system` or export `LIBVIRT_DEFAULT_URI`.
- **Give new DC VMs a VM-GenerationID** (`virt-install --metadata genid_enable=yes`) so snapshot reverts are detected by Windows.
- **virtio needs drivers**; `e1000e` + `sata` work out of the box (slightly slower, fine for labs).
- **AD on USB storage is brutally slow** (fsync). Internal NVMe or `cache='writeback'`.
- Windows guests want **`<clock offset='localtime'>`**.
- A "frozen" installer/promo is often just **slow I/O** — check `virsh domstats` before killing anything.

## Windows
- **Home can't join a domain.** Always Pro/Enterprise for lab clients.
- `-Credential` wants a **PSCredential object**, not a string.
- `Read-Host`'s argument is the **prompt**, not a value.
- **Shift+F10** in OOBE/Setup is the universal escape hatch.
- Disabling the Update **service** isn't durable (Update Medic); use **GPO/registry**.

## Workflow
- **Build once, clone** (golden Core image) — **sysprep it first**, and re-set timezone on each clone.
- **Manage Core headless** via Windows Admin Center / RSAT from [[CLIENT01-Windows11]] to save RAM.
- **Read the actual error.** `-ErrorAction SilentlyContinue` hides the thing you need to see.
