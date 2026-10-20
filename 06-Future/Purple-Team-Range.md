---
tags: [az-802, future, purple-team, siem, offensive]
---
# Future — Purple-Team Range

> [!NOTE]
> Parked for **after** the AZ-802 build. The AD lab you built **is** the target environment — nothing gets wasted. On your own hardware/network, attacking your own VMs: legitimate security self-study.

**Goal:** attack the domain from a Kali box and watch it land in a SIEM — break AD *and* detect the break.

## The one real change: NAT → routed
The `az802-lab` network is NAT today, so the physical LAN can't reach in. Switch it to `route` mode and add a static route (`10.10.10.0/24 via <host LAN IP>`) on the router, or on just the attacker + SIEM hosts.
> [!CAUTION]
> Do **not** bridge the VMs onto the home LAN — the DC's DHCP will wreck your real network. Keep the lab segmented. → [[Networking]]

## Attacker (Kali, e.g. a Raspberry Pi)
- ARM-friendly: nmap, netexec/CrackMapExec, Impacket, BloodHound collectors, Responder.
- **Offload hash cracking to a real GPU** (e.g. the host's RTX 4050 Ti) — pointless on a Pi.
- Target chain vs your AD: recon → AS-REP roast / Kerberoast → BloodHound → lateral movement → DCSync.

## Telemetry (push via GPO — your [[Active-Directory]] skill, reused)
- **Sysmon** (SwiftOnSecurity / Olaf Hartong config)
- **PowerShell script-block logging** + **process command-line auditing** (Event ID 4688)

> [!TIP]
> [[LX01-Ubuntu-AD-Join]] is a natural **syslog / SIEM collector** host — reuse it rather than standing up another box.

## SIEM (light → heavy)
- **Splunk Free** — easiest; Universal Forwarders → indexer (watch the daily cap).
- **Wazuh** — free, agents + built-in rules; best "learn blue team," hungrier.
- **Security Onion / ELK** — full SOC stack, heaviest.

## Safety rails
- The attacker box is **dual-homed** — keep exploits pointed at `10.10.10.0/24`.
- **Never port-forward** any of this to the internet.
- **Snapshot before each attack.**

## Why bother
"I can both compromise AD *and* detect the compromise" is the purple-team profile that gets hired.
