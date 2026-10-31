---
tags: [az-802, services, active-directory, trusts, dns]
---
# Forest Trusts — lab.local ↔ corp.local

Trust practice between the main forest (`lab.local`, [[DC01-Primary-DC]]) and a second forest (`corp.local`, [[CDC01-Second-Forest]]). Covers the AZ-802 trust objectives: DNS prerequisites, forest trusts, trust direction, selective authentication, SID filtering, and the one-way "admin forest" pattern.

> [!NOTE] Status
> Written before the first run (2026-10-30). Results and any errors get added here and to [[Troubleshooting-Log]].

**Running VMs:** DC01, CDC01, FS01 (access tests), CLIENT01 (where you test from). DC02 can stay off. See *Running it* in [[CDC01-Second-Forest]].

## 0. Prerequisites
A trust needs three things in place. Check them before opening the wizard:

| Need | Why | Check |
|---|---|---|
| **Name resolution both ways** | each side must find the other's DCs (SRV records) | `Resolve-DnsName _ldap._tcp.dc._msdcs.<other domain> -Type SRV` |
| **Clocks within ~5 min** | trusts authenticate with Kerberos | `w32tm /stripchart /computer:10.10.10.10 /samples:3 /dataonly` on CDC01 |
| **Forest functional level 2003+** | forest trusts need it (any modern forest has it) | `(Get-ADForest).ForestMode` |

Plus credentials in **both** forests: Enterprise Admins, or Domain Admins of the forest root domain.

## 1. DNS: conditional forwarders both ways
```powershell
# On DC01: "for corp.local, ask CDC01"
Add-DnsServerConditionalForwarderZone -Name corp.local -MasterServers 10.10.10.12 -ReplicationScope Forest

# On CDC01: "for lab.local, ask DC01 / DC02"
Add-DnsServerConditionalForwarderZone -Name lab.local -MasterServers 10.10.10.10,10.10.10.11 -ReplicationScope Forest
```
`-ReplicationScope Forest` stores the forwarder in AD, so DC02 picks it up too when it's running. A **stub zone** or **secondary zone** also works. The point is just that each side can resolve the other.

Test from both DCs:
```powershell
Resolve-DnsName _ldap._tcp.dc._msdcs.corp.local -Type SRV     # on DC01 → cdc01.corp.local
Resolve-DnsName _ldap._tcp.dc._msdcs.lab.local  -Type SRV     # on CDC01 → dc01 (and dc02)
```

## 2. Two-way forest trust
**GUI** (on DC01): *Active Directory Domains and Trusts* → right-click `lab.local` → Properties → **Trusts** → **New Trust…**
1. Trust name: `corp.local`
2. Trust type: **Forest trust**
3. Direction: **Two-way**
4. Sides: **Both this domain and the specified domain** → enter `CORP\Administrator` credentials. (Picking "this domain only" means doing the other side on CDC01 with a shared trust password.)
5. Outgoing trust authentication level: **Forest-wide authentication** for now.
6. Confirm the outgoing **and** incoming trust at the end.

**Verify:**
```powershell
Get-ADTrust -Filter * | Format-Table Name, Direction, ForestTransitive, SelectiveAuthentication, SIDFilteringForestAware, SIDFilteringQuarantined
nltest /sc_verify:corp.local                                  # want: Status = 0 0x0 NERR_Success
netdom trust lab.local /domain:corp.local /verify /userO:LAB\Administrator /passwordO:* /userD:CORP\Administrator /passwordD:* /verbose
```
Expect `Direction = BiDirectional`, `ForestTransitive = True`. Run `netdom` **on DC01** and give credentials for **both** sides: `/userO` = lab.local (the trusting domain), `/userD` = corp.local. Leave out `/userO` and it uses the logged-on account, which fails with *"group look up on domain controller \\DC01.lab.local … Access is denied"* when that account isn't a lab.local admin. `/passwordX:*` prompts, so the password doesn't end up in PSReadLine history.

## 3. Access across the trust
A CORP user reaching a share on FS01 (lab.local).
```powershell
# On CDC01: a test user in corp.local
New-ADUser -Name alice -SamAccountName alice -AccountPassword (Read-Host -AsSecureString "alice password") -Enabled $true
Get-ADUser alice -Properties Enabled, PasswordLastSet | ft Name, Enabled, PasswordLastSet   # Enabled = True, PasswordLastSet filled in
# Sign-in fails later? Reset it here and retry:
# Set-ADAccountPassword alice -Reset -NewPassword (Read-Host -AsSecureString "new password"); Enable-ADAccount alice

# On FS01: a share CORP\alice can read
New-Item C:\TrustTest -ItemType Directory
New-SmbShare -Name TrustTest -Path C:\TrustTest -FullAccess "LAB\Domain Admins" -ReadAccess "CORP\alice"
icacls C:\TrustTest /grant "CORP\alice:(OI)(CI)R"

# On CLIENT01 (signed in as a LAB user): connect AS CORP\alice
net use * /delete /y                                  # see note below
net use \\fs01.lab.local\TrustTest /user:CORP\alice *
```
> [!TIP]
> `net use` to a server you're already connected to under a different account fails with **error 1219** ("multiple connections to a server … by the same user, using more than one user name"). Clear existing connections first, as above.

That this works at all is the lesson. FS01 resolves `CORP\alice` and authenticates her through its own DC, which follows the trust.

## 4. Selective authentication
By default a forest trust lets *every* CORP user authenticate to *every* lab.local machine (permissions still apply). Selective authentication flips that: nobody from CORP gets in anywhere unless a computer explicitly allows them.

**Turn it on** (the trusting side, lab.local): Domains and Trusts → `lab.local` Properties → Trusts → **Domains trusted by this domain (outgoing trusts)** → `corp.local` → Properties → **Authentication** → **Selective authentication**.

Alternatively: `netdom trust lab.local /domain:corp.local /SelectiveAUTH:Yes /userO:LAB\Administrator /passwordO:* /userD:CORP\Administrator /passwordD:*`

1. Re-run the `net use` from step 3 → it should now **fail** with an "authentication firewall" style logon error. Share and NTFS rights are unchanged; the trust itself blocks her.
2. Grant the right on the target computer: *AD Users and Computers* → View → **Advanced Features** → `FS01` computer object → Properties → **Security** → Add `CORP\alice` (or a CORP group) → tick **Allowed to Authenticate**.
3. Re-run the `net use` → works again.

Same block on **sign-in**: signing in to CLIENT01 as `CORP\alice` shows *"The computer you are signing into is protected by an authentication firewall. The specified account is not allowed to authenticate to the computer."* That's the expected result, not an error. Every computer she touches needs the right: CLIENT01 to sign in, FS01 for the share.

PowerShell instead of the Security tab (on DC01):
```powershell
$dn  = (Get-ADComputer CLIENT01).DistinguishedName
$sid = ([System.Security.Principal.NTAccount]"CORP\alice").Translate([System.Security.Principal.SecurityIdentifier])
$acl = Get-Acl "AD:$dn"
$acl.AddAccessRule((New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
  $sid, "ExtendedRight", "Allow", [guid]"68b1d179-0d15-4d4f-ab71-46152e79a7bc")))   # GUID = Allowed to Authenticate
Set-Acl "AD:$dn" $acl
```

**Takeaway:** with selective authentication, access needs **Allowed to Authenticate on the computer object** *and* the normal share/NTFS permissions.

## 5. SID filtering (just look)
SID filtering is **on by default** for forest and external trusts. It drops SIDs from the other forest that don't belong to it (e.g. a forged SID-history entry claiming lab.local's Domain Admins). Confirm it's on, then **leave it on**:
```powershell
Get-ADTrust -Filter * | Format-List Name, SIDFilteringForestAware, SIDFilteringQuarantined
```
Turning it off is only for migrations that rely on SID history, and it's a classic privilege-escalation path across a trust.

## 6. One-way trust: the "admin forest" pattern
The retired ESAE / red-forest design: a locked-down admin forest whose accounts manage production, while production accounts can't touch the admin forest. Here **corp.local = admin forest**, **lab.local = production**.

1. Remove the two-way trust (see Teardown below).
2. New Trust from DC01 → `corp.local` → **Forest trust** → **One-way: outgoing** (lab.local trusts corp.local, so *users in corp.local can authenticate in lab.local*) → both sides → **Selective authentication**.
3. Test both directions:
   - `CORP\alice` → `\\fs01.lab.local\TrustTest` (with Allowed to Authenticate on FS01) → **works**.
   - A LAB user → `dir \\cdc01.corp.local\SYSVOL` → **fails**. corp.local doesn't trust lab.local.

**Direction memory hook:** the **trusting** side holds the resources, the **trusted** side holds the accounts. Access flows **opposite** to the trust arrow.

## Teardown
```powershell
# Trust: Domains and Trusts → lab.local → Trusts → select corp.local → Remove
#        → "Yes, remove the trust from both the local domain and the other domain" (CORP creds)
Get-ADTrust -Filter *                                     # on DC01 and CDC01: should be empty

# DNS
Remove-DnsServerZone -Name corp.local -Force              # on DC01
Remove-DnsServerZone -Name lab.local  -Force              # on CDC01

# Test share
Remove-SmbShare -Name TrustTest -Force                    # on FS01
```

## If it fails
- **"The specified domain either does not exist or could not be contacted"** in the wizard → DNS. Re-run the `Resolve-DnsName` SRV checks from step 1 on **both** DCs.
- **Trust validates, but logons fail with time / Kerberos errors** → clock skew. Fix CDC01's time source ([[CDC01-Second-Forest]], [[Time-and-NTP]]).
- **`netdom … /verify`: "group look up on domain controller \\DC01.lab.local … Access is denied"** while both sessions open fine → the trust is OK. netdom used the logged-on account for lab.local. Run it on DC01 with `/userO:LAB\Administrator /passwordO:*`.
- **CORP\alice can't sign in or connect, but the trust verifies** → check the account on CDC01 before the trust: `Get-ADUser alice -Properties Enabled, PasswordLastSet`. `Read-Host -AsSecureString` asks once with no confirmation, so a typo becomes the password. Reset it from CDC01 (T17 in [[Troubleshooting-Log]]).
- **`net use` error 1219** → existing connection under another account. Run `net use * /delete /y`.
- **Authentication firewall error** after enabling selective authentication → that's it working. Grant *Allowed to Authenticate* on the target computer.

Related: [[Active-Directory]] · [[DNS-and-DHCP]] · [[Troubleshooting-Log]]
