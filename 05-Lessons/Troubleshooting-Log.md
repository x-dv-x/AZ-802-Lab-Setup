---
tags: [az-802, lessons, troubleshooting, errors]
---
# 🔧 Troubleshooting Log

Every wall hit during the build, with **symptom → cause → fix**. This is the note that saves someone else a weekend.

---
## T1 — `virsh`: "failed to get domain 'DC02'"
**Symptom:** `virsh shutdown DC02` → `error: failed to get domain 'DC02'`.
**Cause:** Usually a name mismatch (libvirt names are case-sensitive) — or **user-session vs system libvirt**. VMs made in virt-manager's *system* connection are invisible to a `virsh` defaulting to the session URI.
**Fix:**
```bash
virsh list --all                      # see the EXACT name
virsh -c qemu:///system list --all     # the system instance
export LIBVIRT_DEFAULT_URI=qemu:///system
```
Related: [[Host-Setup-Fedora]]

---
## T2 — Windows Setup shows no disk / Windows shows no NIC
**Symptom:** "no drives found" at install, or `Get-NetAdapter` returns nothing.
**Cause:** **virtio** device with no driver in Windows.
**Fix:** Load the driver from the virtio-win DVD (`vioscsi`/`viostor` for disk, `NetKVM` for NIC), **or** switch to `e1000e` / `sata` which Windows drives natively:
```bash
virsh edit <vm>   # <model type='virtio'/> -> <model type='e1000e'/>
```
Related: [[Creating-VMs]]

---
## T3 — "First-boot CPU usage is insane!"
**Symptom:** A fresh Server VM pegs cores; host CPU graph looks alarming.
**Cause:** Normal post-install churn — Windows Update, Defender's first scan, `TiWorker`, `SearchIndexer`. Only 2–3 cores were actually busy out of 16.
**Fix:** Wait 20–60 min. Confirm in Task Manager → Details. To cut Windows' idle overhead on KVM, add **Hyper-V enlightenments**:
```bash
virsh dumpxml DC01 | grep -A12 hyperv   # check if present
```
```xml
<hyperv mode='custom'>
  <relaxed state='on'/><vapic state='on'/>
  <spinlocks state='on' retries='8191'/>
  <vpindex state='on'/><synic state='on'/><stimer state='on'/>
</hyperv>
```
Also install **qemu-guest-agent + balloon** driver from virtio-win guest tools.

---
## T4 — DC promotion "stuck" at *Waiting for DNS installation to finish*
**Symptom:** Promotion sits forever on that line.
**Cause:** **Not hung — I/O-bound.** AD DS `fsync`s every DB commit; on external USB NVMe each flush was ~53 ms. Confirmed with:
```bash
virsh -c qemu:///system domstats DC02 --vcpu --block
# blocking.cur=yes (vCPU waiting), huge block.0.fl.times, wr.reqs slowly climbing
```
**Fix:** Wait it out. Then move disks to **internal NVMe** or set `cache='writeback'`. **Do not** kill a promo mid-flight — you'll be doing `ntdsutil` metadata cleanup.
Related: [[Storage-Pool]]

---
## T5 — DSRM password rejected during promo
**Symptom:** "DSRM password does not meet minimum length."
**Cause #1:** `Read-Host -AsSecureString "shortpw"` — the string is the **prompt**, not the value; it promoted with an *empty* password.
**Cause #2:** 7 chars < default 8-char minimum.
**Fix:** Type it at the prompt (use `"DSRM password"` as the prompt text) or build it explicitly, 8+ chars:
```powershell
$dsrm = ConvertTo-SecureString '<strong-DSRM-passphrase>' -AsPlainText -Force
```
> The **DSRM password** is the local recovery/safe-mode password for that DC — separate from the domain admin password. Write it down.

---
## T6 — `-NoGlobalCatalogue` parameter not found
**Cause:** Typo. It's `-NoGlobalCatalog` (US spelling). A new DC is a GC by default, so you can just omit it.

---
## T7 — `Add-Computer`: "argument is null or empty" on -Credential
**Cause:** Passed a bare string `LAB\Administrator`. `-Credential` wants a **PSCredential object**.
**Fix:**
```powershell
Add-Computer -DomainName lab.local -Credential (Get-Credential LAB\Administrator) -Restart
```
Related: [[FS01-File-Server]]

---
## T8 — `Add-Computer`: "The request is not supported" / Domain option greyed out
**Cause:** The client is **Windows 11 Home**, which cannot join a domain.
**Fix:** Upgrade to Pro (`changepk.exe /productkey VK7JG-NPHTM-C97JM-9MPGT-3V66T`) or rebuild from the **Enterprise** eval ISO.
Related: [[CLIENT01-Windows11]]

---
## T9 — DHCP authorization "Failed — Error 20079"
**Cause:** DHCP is **already authorized** in AD; you can't authorize twice.
**Fix:** Nothing — it's fine. `Restart-Service DHCPServer` so the created security groups apply.
Related: [[DNS-and-DHCP]]

---
## T10 — Clock wrong even though time "syncs"
**Symptom:** `w32tm /query /source` = `dc01.lab.local`, but the clock is hours off.
**Cause:** **Timezone, not sync.** `w32tm` only syncs UTC. Stripchart showed +0.26s offset with a 3-hour *display* gap = wrong timezone.
**Fix:** `Set-TimeZone -Id "Eastern Standard Time"` on each machine. Timezone is local and **never** propagates from the DC.
Related: [[Time-and-NTP]]

---
## T11 — `w32tm /resync`: "no time data available"
**Cause:** The PDC has no external time source configured, so there's nothing to sync from.
**Fix:** Point it at `pool.ntp.org` (see [[Time-and-NTP]]), or `Set-Date` directly for an immediate fix.

---
## T12 — Timezone fan-out loop does nothing
**Symptom:** `Get-ADComputer | ForEach { Invoke-Command ... }` silently affects nothing.
**Cause:** `-ErrorAction SilentlyContinue` hid the real errors: WinRM not enabled, loop includes the local box, or short name vs FQDN.
**Fix:** Drop `SilentlyContinue` to see errors; exclude `$env:COMPUTERNAME`; use `.DNSHostName`; `Enable-PSRemoting` on targets. Honestly, for 3 machines just run `Set-TimeZone` on each.

---
## T13 — OOBE forces a Microsoft account (Win 11)
**Fix:** `Shift+F10` → `oobe\bypassnro`; or `taskkill /F /IM oobenetworkconnectionflow.exe`; or detach the NIC and boot offline.
Related: [[CLIENT01-Windows11]]
