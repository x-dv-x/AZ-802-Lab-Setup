---
tags: [az-802, host, networking, libvirt]
---
# Networking — libvirt lab network

Keep the **default NAT** network (`virbr0`) for internet/updates. Add a routed lab network where the **DC owns DNS + DHCP** so AD behaves like production.

## Define `az802-lab`
`az802-lab.xml`:
```xml
<network>
  <name>az802-lab</name>
  <forward mode='nat'/>
  <bridge name='virbr10' stp='on' delay='0'/>
  <ip address='10.10.10.1' netmask='255.255.255.0'>
    <!-- intentionally NO <dhcp> block: the Windows DC serves DHCP here -->
  </ip>
</network>
```
```bash
virsh net-define az802-lab.xml
virsh net-start az802-lab
virsh net-autostart az802-lab
```

## The IP plan
Gateway `10.10.10.1` (libvirt, egress) · DC01 `.10` · DC02 `.11` · FS01 `.20` · clients DHCP `.100–200`. Full table in [[Lab-Topology]].

> [!CAUTION]
> **Never bridge the lab VMs directly onto your home LAN.** [[DC01-Primary-DC]] runs DHCP — a rogue DHCP server on your real network will hand addresses to your actual devices and break your home. A separate routed subnet is both safer *and* more realistic. (This matters again in [[Purple-Team-Range]].)

> [!NOTE]
> Because the lab network has **no libvirt DHCP**, every machine needs either a static IP (servers) or the DC's DHCP scope (clients). A NIC with no address is expected until the DC's DHCP is up — see [[DNS-and-DHCP]].
