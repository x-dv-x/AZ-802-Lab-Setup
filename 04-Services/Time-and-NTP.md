---
tags: [az-802, services, time, ntp, kerberos]
---
# Time & NTP — The Domain Time Hierarchy

> [!IMPORTANT]
> **Time is load-bearing in AD.** Kerberos rejects auth (and domain joins) if clocks differ by more than ~5 minutes. VMs on a busy host *do* drift.

## The design
- **One authoritative clock:** the **PDC emulator** ([[DC01-Primary-DC]]) → external NTP.
- **Everyone else:** time type **`NT5DS`**, following the domain hierarchy (automatic on join).

## Configure the PDC → external source
```powershell
w32tm /config /manualpeerlist:"pool.ntp.org" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
```
> [!NOTE]
> `w32tm /resync` → **"no time data available"** just means there's no source configured yet. Set the external peer first (above). To *just fix the clock now* without a source: `Set-Date "MM/dd/yyyy h:mm tt"`.

## Sync the rest from the PDC
```powershell
w32tm /resync /rediscover
w32tm /query /source        # should show the DC / domain, NOT "Local CMOS Clock"
w32tm /query /configuration | findstr /i "Type"   # members = NT5DS
```

## ⚠️ The #1 time trap: timezone ≠ time sync
> [!WARNING]
> If `w32tm /query /source` shows your DC **but the clock still reads wrong**, it is almost always a **timezone** problem, not a sync problem. w32time only ever syncs **UTC**; the displayed wall-clock = UTC + timezone.
>
> **Diagnose:** `w32tm /stripchart /computer:dc01.lab.local /samples:3`. A tiny offset (e.g. +0.26s) with a whole-hours display difference = timezone, confirmed.
>
> **Fix (per machine — timezone is local, never synced):**
> ```powershell
> Set-TimeZone -Id "Eastern Standard Time"
> ```
> Make `Set-TimeZone` a standard line in your post-install checklist for **every** VM. Full story in [[Troubleshooting-Log]].

## Keep it from drifting back on reboot
Windows guests want a **localtime** hardware clock on KVM:
```bash
virsh -c qemu:///system dumpxml DC01 | grep -i clock   # want offset='localtime'
# if 'utc': virsh edit DC01 → <clock offset='localtime'> → reboot
```
