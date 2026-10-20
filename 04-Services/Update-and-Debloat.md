---
tags: [az-802, services, windows-update, debloat, gpo]
---
# Windows Update Control & Debloating

> [!CAUTION]
> Everything here is a **lab-only** convenience. Never disable updates or strip apps on production.

## Stop update churn via GPO (the right way, 1 place)
On [[DC01-Primary-DC]]:
```powershell
New-GPO -Name "Lab - Disable Auto Updates" | New-GPLink -Target "DC=lab,DC=local"
```
Edit → **Computer Config → Admin Templates → Windows Components → Windows Update → Configure Automatic Updates → Disabled**. The registry value it writes:
```powershell
$p = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU"
New-Item $p -Force | Out-Null
Set-ItemProperty $p -Name NoAutoUpdate -Value 1 -Type DWord
```
Push it: `gpupdate /force` on each box.

> [!NOTE]
> Disabling the **service** alone (`wuauserv`) isn't durable — the **Update Medic (WaaSMedicSvc)** flips it back. The GPO/registry `NoAutoUpdate` route is what actually holds.

## Debloat
**Tools (current, work on 26H2):** Win11Debloat (Raphire) and ChrisTitusTech WinUtil.
```powershell
# Win11Debloat (has a GUI + CLI params for sysadmins):
& ([scriptblock]::Create((irm "https://debloat.raphi.re/")))
```
> [!WARNING]
> Those are `irm <url> | iex` — remote code straight into admin PowerShell, the exact behavior you'd write detections for. **Snapshot first**, and ideally download the release ZIP and read the `.ps1` instead of piping blind.

**Native way (inspectable + GPO-deployable):**
```powershell
Get-AppxPackage | Select Name
Get-AppxPackage *Clipchamp* | Remove-AppxPackage
# strip from the image so NEW domain profiles don't get it back:
Get-AppxProvisionedPackage -Online |
  Where-Object DisplayName -like "*Clipchamp*" | Remove-AppxProvisionedPackage -Online
```
> [!CAUTION]
> **Do not remove:** Microsoft Store, **App Installer** (winget), Windows Terminal, VCLibs/.NET framework packages, Edge WebView2 — other software depends on them.

Honest expectation: debloating saves little RAM (~100–200 MB). The win is a clean, managed-looking client.
