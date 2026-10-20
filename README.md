# AZ-802 Home Lab — A Windows Server / Active Directory Lab on Linux (KVM)

A fully documented home lab for studying **Microsoft AZ-802 (Administering Windows Server)** — built on a **Fedora + KVM/QEMU/libvirt** host, with a multi-DC Active Directory domain, DNS, DHCP, a file server, and a Windows 11 management client.

This repo is also an **Obsidian vault**. It documents not just the happy path but every failure hit along the way and how each was fixed — because that's the part most guides leave out.

## What's inside

- A reproducible build: host setup → networking → VMs → AD → core services.
- A [Troubleshooting Log](05-Lessons/Troubleshooting-Log.md) of real errors, root causes, and fixes.
- Distilled [Gotchas & Learnings](05-Lessons/Gotchas-and-Learnings.md).
- A roadmap to extend the lab into a [purple-team range](06-Future/Purple-Team-Range.md).

## The rig

| | |
|---|---|
| Host OS | Fedora Linux |
| CPU / RAM | Intel i7 / 32 GB |
| GPU | NVIDIA RTX 4050 Ti (not used by the lab) |
| Hypervisor | KVM/QEMU + libvirt |
| Lab storage | external 500 GB NVMe (or internal — see notes) |
| Guests | Windows Server 2025 (eval) + Windows 11 Enterprise/Pro (eval) |

## How to use this

- **In Obsidian:** open this folder as a vault. The notes are linked with `[[wikilinks]]` and the graph view shows how everything connects. Start at [[Index]].
- **On GitHub:** browse from this README. Note that `[[wikilinks]]` show as plain text on GitHub (they're only clickable inside Obsidian); the standard links in this README do work here.

## Start here

➡️ **[[Index]]** — the map of content and recommended reading order.

## Disclaimer

This is an **isolated lab** on private hardware. Several steps (disabling updates, debloating, generic product keys, weak passwords) are **lab-only shortcuts** and must never be applied to production or internet-facing systems.

## License

[CC0 1.0 Universal](LICENSE) — dedicated to the public domain. Copy it, adapt it, build your lab from it; no attribution required (though a star or a link back is always welcome).
