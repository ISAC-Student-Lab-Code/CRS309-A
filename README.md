# Site A — CRS309

## Device port mapping

| Port | Destination / desired role | Unresolved details |
| --- | --- | --- |
| 1 | Separate Proxmox 10 Gb link | Host NIC, dedicated VLAN list and software interface name |
| 2 | Separate Proxmox 10 Gb link | Host NIC, dedicated VLAN list and software interface name |
| 3 | CRS310 first 10 Gb port | Software interface names and VLAN membership |
| Other ports | Unresolved | Peer, role, VLANs and enabled state |

Do not combine ports 1 and 2 into a bond. Confirm switch operating system/provider support and VLAN filtering behavior before implementation. The switch management VLAN remains unresolved.

## Status and confirmed scope

This is a new Site A build. Each device directory is an independent Git repository with its own configured remote. This README documents desired infrastructure behavior only. No Terraform code has been written and no infrastructure changes have been applied.

Sources are the Infrastructure Lab Design conversation (`6ae2488d-6d3c-83ea-bfcb-6d2d4c4fe98e`) and subsequent user corrections and copied planning material in this chat. The copied material establishes the network map below. Earlier assistant responses were unavailable, so numeric VM sizing and some physical mappings still require recovery. Unresolved details do not revoke earlier sizing approval.

Confirmed physical decisions:

- CRS309 ports 1 and 2 connect to separate Proxmox 10 Gb interfaces with dedicated VLAN assignments per link. There is no link aggregation. Per-link VLAN lists and physical NIC names remain unresolved.
- CRS309 port 3 connects to the CRS310 first 10 Gb port.
- The CRS310 second 10 Gb port must be administratively disabled in the build; this has not been applied.
- Proxmox management connects through CRS310 `ether2`, carrying tagged VLANs 25 and 26 on `vmbr0`.

## VLAN map and pfSense NIC assignments

| SDN network | VLAN | Subnet | Proxmox NIC | pfSense address |
| --- | --- | --- | --- | --- |
| `external` | Untagged | Unresolved | `net0` | WAN address TBD |
| `prodwin` | 20 | `172.16.100.0/25` | `net1` | `172.16.100.1/25` |
| `prodlinux` | 21 | `172.16.100.128/25` | `net2` | `172.16.100.129/25` |
| `devwin` | 22 | `172.16.101.0/25` | `net3` | `172.16.101.1/25` |
| `devlinux` | 23 | `172.16.101.128/25` | `net4` | `172.16.101.129/25` |
| `endpoint` | 24 | `172.16.102.0/26` | `net5` | `172.16.102.1/26` |
| `netmgmt` | 25 | `172.16.103.0/27` | `net6` | `172.16.103.1/27` |
| `virtmgmt` | 26 | `172.16.103.32/27` | `net7` | `172.16.103.33/27` |

Subnet network addresses above are derived from the supplied interface addresses and prefixes. All eight pfSense NICs use VirtIO. Proxmox SDN handles internal VLAN tagging; pfSense receives ordinary untagged Ethernet on each dedicated NIC. No guest VLAN subinterfaces or internal trunk NICs are required. `net0` connects directly to the external network; its underlying physical path remains unresolved. Untagged external does not imply VLAN 1 or permission to bridge external traffic into management.

## Proxmox management

| Setting | Build decision |
| --- | --- |
| VLAN | 26 — `virtmgmt` |
| Host address | `172.16.103.34/27` — **proposed**, not finalized |
| Default gateway | `172.16.103.33` — pfSense `net7` |
| Physical bridge | `vmbr0` |
| Host management interface | `vmbr0.26`; do not assign the management IP directly to `vmbr0` |
| Physical connection | CRS310 `ether2`; Proxmox physical NIC name unresolved |
| Switch port mode | Tagged trunk carrying VLANs 25 and 26 |

Proxmox external management connectivity depends on the local pfSense VM being operational. Configure pfSense to start automatically with startup priority and retain local Proxmox recovery access. Exact startup ordering/delays and recovery procedure remain unresolved.

## Repository ownership and integration

| Repository | Responsibility | Handoff |
| --- | --- | --- |
| `crs309-a` | Proposed: CRS309 physical interfaces and VLAN forwarding | Per-port VLAN lists and interface mapping |
| `crs310-a` | Proposed: CRS310 physical interfaces and VLAN forwarding | Uplink, disabled second 10 Gb port and `ether2` tagged trunk |
| `prox9-a` | Confirmed: create seven internal SDN VNets and pfSense VM with eight NICs; host networking belongs here | VNet names, VLAN/bridge paths, NIC indexes and stable MACs |
| `pfsense-a` | Confirmed: guest interface assignments, addresses, DHCP, routing, NAT and firewall rules after installation and accessibility | Guest configuration and connectivity requirements |

Each pfSense NIC references its appropriate SDN network/VNet name rather than a physical bridge plus a guest NIC VLAN tag. Creation/ownership of the underlying `external` attachment, exact SDN zones and bridge mappings remain unresolved. Do not assume all networks share one physical uplink.

Use separate state per repository and avoid overlapping resource ownership. Keep pfSense VM hardware in `prox9-a` and guest configuration in `pfsense-a`. Human owners, repository URLs, provider choices/versions and state backends remain unresolved.

## VM sizing and storage

All previously proposed VM sizes were approved and all VM disks must be thin provisioned. Numeric vCPU, RAM and disk values, the complete VM inventory and storage backend are still unavailable. pfSense's eight VirtIO NICs are confirmed. Do not invent replacement sizing or assume a storage format/backend. Verify thin allocation on the selected backend and monitor actual physical free space. Discard/TRIM policy and capacity thresholds remain unresolved.

## Prerequisites and unresolved inputs

- Recover the approved VM inventory and numeric sizing; choose a thin-capable storage backend and VM images.
- Record Proxmox physical NIC names, per-10-Gb-link VLAN lists, bridge/zone mapping, and switch software interface names. Confirm negotiated link speeds and the full cable schedule.
- Confirm the proposed Proxmox host address; other device management addresses remain unresolved.
- Establish WAN addressing, upstream gateway and physical external connection.
- Confirm pfSense edition/version, DHCP scopes, DNS service, routing details, NAT mode, firewall policy and any VPN requirements. The interface addresses above are confirmed; service policy is not supplied.
- Validate supported Terraform provider capabilities and authentication for each platform before implementation; record device/software versions, state backend/locking, named owners and recovery access.

## Safety and validation

Documentation is not authorization to apply changes. Keep secrets and state out of source control, restrict backend access and retain backups. Review eventual plans for correct targets, ownership, replacements/deletions and management reachability. Maintain a tested local console path while commissioning switches, host networking and pfSense.

Future acceptance checks, not completed infrastructure tests:

- Verify each physical link and tagged/untagged VLAN membership against its agreed port contract, including exclusion of unintended VLANs. Independent Proxmox links do not imply automatic failover.
- Verify CRS309 port 3 reaches the CRS310 first 10 Gb port, and the second 10 Gb port is administratively disabled.
- Verify CRS310 `ether2` carries tagged VLANs 25/26 and host management uses `vmbr0.26`; validate the proposed host address only after approval.
- Match seven internal VNets and all eight VirtIO pfSense NICs to the table; record stable MAC-to-guest-interface mappings and verify guest-facing traffic is untagged.
- Verify pfSense internal addresses, Proxmox gateway reachability, approved DHCP/DNS, routing, NAT and permitted/denied firewall flows once policy is specified.
- Test pfSense autostart/startup priority, management recovery when pfSense is unavailable, and stable NIC assignments after reboot.
- Match VM hardware to the recovered approved inventory and verify thin disk allocation and capacity monitoring.

Once implementation exists, validate its formatting/configuration and review its saved plan before an authorized deployment. Record observed acceptance results; README checks alone do not establish a successful build.
