---
tags: [az-802, planning, networking, topology]
---
# Lab Topology

## Network
- **Domain:** `lab.local`
- **Subnet:** `10.10.10.0/24`
- **Lab network (libvirt):** `az802-lab`, NAT-forwarded, **no libvirt DHCP** (the DC serves DHCP)
- **Gateway (libvirt):** `10.10.10.1` — provides internet egress for the guests

```
        Internet
           |
      [ libvirt NAT ]  10.10.10.1
           |
   ========+====================  az802-lab  (10.10.10.0/24)
     |        |        |        |
   DC01     DC02     FS01     CLIENT01
  .10/PDC   .11      .20      DHCP .100-200
  DNS/DHCP  DNS    file/iSCSI  Win11 mgmt
  NTP src
```

## IP plan
| Host | IP | Role |
|------|----|----|
| libvirt gateway | 10.10.10.1 | NAT egress |
| [[DC01-Primary-DC]] | 10.10.10.10 (static) | PDC, DNS, DHCP, external NTP source |
| [[DC02-Second-DC]] | 10.10.10.11 (static) | 2nd DC, 2nd DNS |
| [[FS01-File-Server]] | 10.10.10.20 (static) | file server, iSCSI target |
| [[CLIENT01-Windows11]] | DHCP (.100–200) | RSAT / Windows Admin Center |
| [[LX01-Ubuntu-AD-Join]] | 10.10.10.30 (static) | Ubuntu Server, AD-joined via SSSD |

## RAM budget (32 GB host)
Reserve ~5 GB for Fedora → ~26 GB for VMs.

| VM | OS | RAM | Runs when |
|----|----|-----|-----------|
| DC01 | GUI → Core | 4 GB | always |
| DC02 | Core | 2 GB | always |
| FS01 | Core/GUI | 4 GB | storage + cluster |
| NODE1/NODE2 | Core | 3 GB ea | HA phase |
| CLIENT01 | Win 11 | 4 GB | management |

## LAB Scripts 


| Profile | RAM  | Machines               | For                         |
| ------- | ---- | ---------------------- | --------------------------- |
| min     | 6GB  | DC01 DC02              | core machines always needed |
| core    | 10GB | DC01 DC02 CLIENT01     | AD / DNS / DHCP / GPO       |
| storage | 12GB | DC01 FS01 CLIENT01     | storage & file services     |
| cluster | 14GB | DC01 FS01 NODE1 NODE2  | failover clustering         |
| hybrid  | 10GB | DC01 LX01 CLIENT01     | hybrid / linux              |
|         |      |                        |                             |

``` bash 
lab up core - # every day study set
lab down # clean stop everything
lab status 
```


> [!TIP]
> Manage headless Core servers from **Windows Admin Center** on CLIENT01 — one pane of glass, no GUI RAM tax. See [[Gotchas-and-Learnings]].

## DNS client settings (avoid "DNS islanding")
Each DC points **preferred → its partner, alternate → itself (127.0.0.1)**. Details in [[DNS-and-DHCP]].
