
Install **Microsoft Entra Connect** on a lab member server and sync on-prem AD users into an Entra tenant. This is the hands-on that makes the "directory synchronization" theory click — you watch your [[Active-Directory]] users appear in the cloud.

> [!IMPORTANT] **This exercise is essentially free.** Entra Connect is just an agent + **Entra ID Free** tier — no Azure VM, no compute credit burned. The thing that _does_ cost money is **Entra Domain Services** (a different feature) — **do not spin that up**, it's billed hourly and drains credit fast. Read those units; lab this one.

## Prerequisites

- An Entra tenant where **you are Global Administrator** (your _personal_-account tenant, not an org tenant — see note below).
- A **domain-joined Windows Server** to host Connect — [[FS01-File-Server]] works. (Best practice is a member server, not a DC; a DC works in a pinch for a lab.)
- **Outbound HTTPS (443)** to Entra — the lab's NAT egress covers this ([[Networking]]).
- Correct **time** ([[Time-and-NTP]]) and the server pointing DNS at the DCs ([[DNS-and-DHCP]]).

> [!CAUTION] If "I don't have access to Entra ID" — you probably signed up with a **school/org account**, so your subscription sits in that org's tenant where you're not admin. Re-sign-up with a **personal** Microsoft account (use the school email only for student verification). Confirm: portal → **Microsoft Entra ID** → Overview shows _your_ tenant and you as Global Admin.

## The `.local` gotcha

> [!WARNING] `lab.local` is **non-routable and can't be verified in Entra**. Users still sync, but their cloud UPN defaults to `user@<tenant>.onmicrosoft.com`. That's fine for the lab. To get matching UPNs you'd add a _routable_ UPN suffix (a public domain you own) in **AD Domains and Trusts** and verify it in Entra — skip unless you already own a domain.

## Steps

### 1. Pre-flight (optional but good practice)

Run **IdFix** against the directory to catch duplicate/invalid attributes before syncing.

### 2. Install Entra Connect

On the member server:

1. Download **Microsoft Entra Connect** (MSI) from Microsoft.
2. Run it → accept terms → **Express Settings** (this enables **Password Hash Sync** + syncs everything).
3. Sign in with your **Entra Global Administrator** (tenant) credentials.
4. Sign in with on-prem **Enterprise Admin** (`LAB\Administrator`) — it creates the AD DS connector (sync) account.
5. Let it configure and run the initial sync.

### 3. Verify the sync

```powershell
# on the Connect server — force/inspect sync:
Start-ADSyncSyncCycle -PolicyType Initial     # full sync
Get-ADSyncScheduler                            # default 30-min cycle
```

Then in the portal: **Entra ID → Users** → your lab users appear, marked **Directory synced / On-premises sync enabled = Yes**. **Entra ID → Entra Connect** shows last sync time. _This is the payoff moment._

### 4. Explore the integration models (the exam's core concept)

Express gave you **Password Hash Sync (PHS)** — cloud validates auth against a synced hash. Re-run Connect → **Customize** to try:

- **Pass-through Authentication (PTA)** — auth validated on-prem via lightweight agents.
- **Seamless SSO** — Kerberos-based silent sign-in on domain-joined devices.
- (**Federation / AD FS** — know it conceptually; don't build it in the lab.)

### 5. Day-to-day

```powershell
Start-ADSyncSyncCycle -PolicyType Delta       # push a change you just made on-prem
```

## Lighter alternative

**Entra Connect cloud sync** — a thin agent with config in the cloud (the modern direction). Easier for a lab; same outcome for the sync objectives. Full **Connect Sync** (above) is what the classic AZ-802 material walks through.

## Teardown

Uninstall Entra Connect from the server when done. The synced cloud users remain until you disable directory sync in the tenant — fine to leave in a lab.

## Exam mapping

Covers the hybrid-identity slice: integration models (PHS / PTA / federation / Entra DS), Entra Connect install & config, Seamless SSO. → [[AZ-802-Study-Mapping]]