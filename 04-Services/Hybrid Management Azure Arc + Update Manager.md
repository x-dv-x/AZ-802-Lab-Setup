---
tags: [az-802, services, hybrid, azure-arc, update-manager, patching]
---
Project your existing on-prem lab machines ([[DC02-Second-DC]], [[FS01-File-Server]], [[LX01-Ubuntu-AD-Join]]) into Azure with **Azure Arc**, then run **Azure Update Manager** against them — cloud-based patch management with no new cloud VMs. This covers the biggest chunk of the hybrid objectives using the lab you already have.

> [!NOTE]
> Nice mix for the exam: 2 Windows + 1 Linux means you see Update Manager's **unified cross-OS** dashboard.

## 💲 Cost (read first)
> [!IMPORTANT]
> - **Arc control plane = free** (onboarding, tagging, resource graph, RBAC).
> - **Azure Update Manager for Arc servers = up to $5/server/month**, but **prorated daily (~$0.16/day)** and **only charged on days the server is Connected AND an assessment/patch runs or it's tied to a schedule**.
> - So a few focused days across 3 boxes ≈ **$1–3 total**. The thing that racks up the full $5/server is leaving **periodic assessment ON all month** — don't.

## Prerequisites
- A resource group in a **Canada region** (e.g. Canada Central) to keep it tidy.
- Register providers (portal onboarding offers to do this): `Microsoft.HybridCompute`, `Microsoft.HybridConnectivity`, `Microsoft.GuestConfiguration`, `Microsoft.Maintenance`.
- Outbound **HTTPS 443** from the lab machines — the NAT egress ([[Networking]]) covers it.
- Azure RBAC: **Azure Connected Machine Onboarding** role (or Contributor).
- Local **admin/root** on each machine.

## 1. Generate the onboarding script
Portal → **Azure Arc → Machines → Add/Onboard**:
- **"Add a single server"** → interactive script (device-login per box).
- **"Add multiple servers"** → script using a **service principal** (no per-box login) — better for 3 machines, and good exam knowledge. Create the SP with the *Onboarding* role.
- Set subscription, resource group, region (Canada Central), connectivity = **Public endpoint**. It outputs a `.ps1` (Windows) / `.sh` (Linux).

## 2. Run it on each machine (admin/root)
```powershell
# DC02, FS01 — run the generated PowerShell elevated.
# It downloads AzureConnectedMachineAgent, installs, then runs: azcmagent connect ...
```
```bash
# LX01 — run the generated script with sudo.
sudo bash ./OnboardingScript.sh
```
> [!TIP]
> `azcmagent check` validates connectivity to the Arc endpoints **before** you connect — run it first if a box is stubborn.

## 3. Verify
```
azcmagent show        # Agent Status: Connected
```
Portal → Azure Arc → Machines → DC02 / FS01 / LX01 appear as **Connected** within a minute.

## 4. Azure Update Manager
Portal → **Azure Update Manager** → select your Arc machines:
- **Check for updates** (on-demand assessment) → missing patches for all three, Windows + Linux, in one dashboard.
- **One-time update** → install on demand.
- **Scheduled patching** → create a **maintenance configuration** (window) and associate the machines → recurring patching.

> [!TIP]
> This is the **WSUS replacement** conceptually. Contrast with [[Update-and-Debloat]]: on-prem you *disabled* updates via GPO; here you *orchestrate* them from the cloud. Same objective (patch management), two tools — exactly the on-prem-vs-hybrid framing the exam wants.

## 5. Teardown (keep it ~free)
- Don't leave **periodic assessment** on across the month.
- When done: `azcmagent disconnect` on each box, or **delete the resource group** in Azure — offboarding stops charges immediately.

## Where it leads
Once machines are Arc-connected, the same path opens **Azure Monitor** (AMA agent), **Azure Policy / machine configuration**, and **Defender for Cloud** against on-prem boxes — all optional, all billed per service.

## Exam mapping
Hybrid management core: Arc onboarding, Update Manager assessment + scheduled patching, hybrid patch orchestration. → [[AZ-802-Study-Mapping]]