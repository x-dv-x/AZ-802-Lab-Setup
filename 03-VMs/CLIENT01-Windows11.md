---
tags: [az-802, vms, windows-11, client]
---
# CLIENT01 — Windows 11 Management Client

Runs **RSAT** and **Windows Admin Center** to manage the headless Core servers.

## Getting past OOBE's forced Microsoft account
At the "Unlock your Microsoft experience" / network screen:
1. **Shift + F10** (laptop: `Shift + Fn + F10`) → a command prompt opens over Setup.
2. Run `oobe\bypassnro` → it reboots; now there's an **"I don't have internet" → Continue with limited setup** → create a **local account**.

If `bypassnro` is gone on a very new build:
```text
taskkill /F /IM oobenetworkconnectionflow.exe
```
Or simplest for a VM — **detach the NIC**, boot with no network, and OOBE falls back to local setup:
```bash
virsh -c qemu:///system detach-interface CLIENT01 network --mac <mac> --live
# re-attach after you reach the desktop
```

## ⚠️ The edition trap
> [!WARNING]
> **Windows 11 Home cannot join a domain.** If "Domain or workgroup" is greyed out and `Add-Computer` throws **"The request is not supported"**, check the edition. Home physically lacks domain join.
>
> **Fix A — rebuild** from the Windows 11 **Enterprise** eval ISO (correct for a lab client).
> **Fix B — upgrade to Pro** in place with Microsoft's public generic Pro key (changes SKU, not activation; lab is fine):
> ```powershell
> changepk.exe /productkey VK7JG-NPHTM-C97JM-9MPGT-3V66T
> ```
> Always grab **Enterprise/Pro** eval for any domain-joined lab client. Full story in [[Troubleshooting-Log]].

## Join the domain (once Pro/Enterprise)
```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.10.10.10,10.10.10.11
Add-Computer -DomainName lab.local -Credential (Get-Credential LAB\Administrator) -Restart
```

## Tidy it up
Set the timezone ([[Time-and-NTP]]) and strip the consumer bloat ([[Update-and-Debloat]]).
