---
tags: [az-802, lessons, troubleshooting, errors, powershell]
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

---
## T14 — `Set-Alias` with arguments: "The term 'gpupdate /force' is not recognized"
> **General PowerShell lesson.** This isn't specific to the lab. It applies to *any* shortcut you want for a command plus its arguments (`gpupdate /force`, `ipconfig /flushdns`, `Get-Service -Name W32Time`…).

**Symptom:**
```powershell
Set-Alias -Name gpu -Value "gpupdate /force"
gpu
# gpu : The term 'gpupdate /force' is not recognized as the name of a cmdlet, function,
# script file, or operable program.   (CommandNotFoundException)
```
**Cause:** A PowerShell **alias is just another name for a single command**. It **can't hold arguments**. `-Value` is treated as one literal command name, so PowerShell went looking for a program called `gpupdate /force`, space and all. (The same is true for cmdlets, not just `.exe`s.)
**Fix:** Use a **function** whenever the shortcut needs arguments:
```powershell
function gpu { gpupdate /force }                 # fixed arguments
function gpu { gpupdate /force @args }           # ...and pass any extras through, e.g. gpu /target:computer
```
Use an alias only for a bare rename (`Set-Alias -Name gpu -Value gpupdate`, then type `gpu /force`).

**Make it stick:** aliases and functions typed into a session vanish when the window closes. Put them in your profile:
```powershell
if (!(Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
Add-Content $PROFILE 'function gpu { gpupdate /force @args }'
. $PROFILE                                       # load it now
```
> If the profile doesn't load on a Windows **client** (CLIENT01), the cause is the default execution policy (`Restricted`), which blocks every script, including `$PROFILE`. Fix with `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`. Servers already default to `RemoteSigned`.

**Rule of thumb:** alias = **rename**. Function = **rename + arguments/logic**.

---
## T15 — GPO startup script (`.ps1`) never runs
> **General Group Policy lesson.** It applies to any PowerShell script deployed as a startup, shutdown, logon or logoff script.

**Symptom:** In *Computer Configuration → Policies → Windows Settings → Scripts → Startup*, the **Scripts** tab was set to:
- Name: `C:\Users\Administrator\Documents\export-pwsh-his.ps1`
- Parameters: `-ExecutionPolicy Bypass`

The machine reboots and nothing happens.

**Cause #1, wrong tab / wrong fields:** the **Scripts** tab launches *Name* like a double-click and hands it *Parameters*. Windows doesn't run a `.ps1` on double-click (it opens it in Notepad), and `-ExecutionPolicy Bypass` is a switch for **powershell.exe**, not for the script. So PowerShell is never started.

**Cause #2, the path:** startup scripts run **as SYSTEM, on each target computer, before anyone signs in**. `C:\Users\Administrator\Documents\…` only exists on the box where the script was written. Every other computer looks for that path on its **own** C: drive and finds nothing.

**Fix:** put the script in the GPO itself (SYSVOL), then launch it through PowerShell. Pick **one** of these:
- **Scripts tab:** Name = `powershell.exe`, Parameters = `-NoProfile -ExecutionPolicy Bypass -File "\\lab.local\SYSVOL\lab.local\Policies\{GPO-GUID}\Machine\Scripts\Startup\export-pwsh-his.ps1"`
- **PowerShell Scripts tab** (made for `.ps1`): Name = the script file. Parameters = **only the script's own arguments**, never powershell.exe switches like `-ExecutionPolicy`.

To get the script into SYSVOL: Startup → Properties → **Show Files…**. That opens the GPO's `Machine\Scripts\Startup` folder in SYSVOL, so copy the `.ps1` there. It replicates with the GPO, and every computer can read it as `Domain Computers`.
```powershell
gpupdate /force          # then REBOOT: startup scripts only run at boot (logon scripts only at sign-in)
gpresult /r /scope computer   # is the GPO in "Applied Group Policy Objects"?
```
Event Viewer → *Applications and Services Logs → Microsoft → Windows → GroupPolicy → Operational* shows script start/finish and errors.

**Checklist for any GPO script:**

- [ ] **Right half of the GPO.** Startup/shutdown = *Computer* scripts, so link the GPO to the OU holding the **computer** object. Logon/logoff = *User* scripts, so link it to the OU holding the **user**. Put a script in the wrong half and it's silently ignored.
- [ ] **Script in SYSVOL.** Use Show Files…, or a share that **Domain Computers** can read. Local paths and admin-only shares fail.
- [ ] **Launched the right way.** Use the PowerShell Scripts tab, **or** the Scripts tab with `powershell.exe -File …`. The Name field isn't a command line.
- [ ] **Triggered.** Reboot for startup scripts, sign out and back in for logon scripts. `gpupdate` alone doesn't run them.
- [ ] **Right account.** Startup/shutdown scripts run as **SYSTEM**, logon/logoff scripts as **the user**. `$env:USERPROFILE`, `$HOME`, HKCU and **PSReadLine history** belong to whoever runs the script, so a startup script that exports a *user's* history gets SYSTEM's (empty) history. Use a **logon/logoff** script for that.

Related: T14 above (PowerShell alias vs function)

---
## T16 — `virt-install`: "Network not found: no network with matching name 'az802-lab'"
**Symptom:** creating a new VM fails right after allocating the disk:
```
Allocating 'CDC01.qcow2'          |  50 GB  00:00:02
Removing disk 'CDC01.qcow2'       |         00:00:00
ERROR    Network not found: no network with matching name 'az802-lab'
Domain installation does not appear to have been successful.
If it was, you can restart your domain by running:
  virsh --connect qemu:///session start CDC01
```
**Cause:** same root as T1. Run as a normal user with no `--connect`, `virt-install` uses the **user session** instance (`qemu:///session`). That instance has no lab network; `az802-lab` and every lab VM live in **`qemu:///system`**. The last line of the error gives it away: `qemu:///session`.

**Fix:** tell it which libvirt to use, and re-run. The failed run deletes its own disk, so there's nothing to clean up.
```bash
virt-install --connect qemu:///system --name CDC01 ...         # one-off
echo 'export LIBVIRT_DEFAULT_URI=qemu:///system' >> ~/.zshrc   # permanent (~/.bashrc for bash)
virsh -c qemu:///system net-list --all                         # the network IS there
```
**Applies to:** any libvirt tool (`virsh`, `virt-install`, `virt-clone`). If a VM or network "doesn't exist", check the URI before anything else.

Related: T1, [[Creating-VMs]], [[CDC01-Second-Forest]]

---
## T17 — Trusted-forest user can't sign in across a forest trust
**Symptom:** `CORP\alice` (created in the second forest, [[CDC01-Second-Forest]]) can't sign in to CLIENT01 in `lab.local`. The trust verifies fine.

**Cause:** the account, not the trust. Resetting alice's password on CDC01 fixed it. The likely story: the password was set with `Read-Host -AsSecureString`, which asks **once** and never confirms, so a typo became the real password. Also worth knowing: if the password fails the complexity rules, `New-ADUser` still creates the account but leaves it **disabled** with no password set.

**Fix:** on the account's own DC (the trusted forest), check the account first, then reset it:
```powershell
Get-ADUser alice -Properties Enabled, PasswordLastSet, LockedOut | ft Name, Enabled, PasswordLastSet, LockedOut
Set-ADAccountPassword alice -Reset -NewPassword (Read-Host -AsSecureString "new password")
Enable-ADAccount alice
```
**Lesson:** when a cross-forest logon fails, rule out the boring causes (password, disabled, locked out) **before** debugging the trust. The CLIENT01 Security log (event **4625**, Status/Sub Status) tells you which one it is. `0xC000006A` = bad password, `0xC0000072` = disabled, `0xC0000234` = locked out, `0xC0000413` = blocked by selective authentication.

Related: [[Forest-Trusts]]
