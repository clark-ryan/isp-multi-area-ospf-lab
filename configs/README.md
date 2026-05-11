# Device Configurations

Full Cisco IOS running-configs for every router in the ISP Multi-Area OSPF lab.
For project context, architecture, and verification, see the [main README](../README.md).

## Device inventory

| File | Hostname | Role | OSPF Area | Loopback | Notable features |
|---|---|---|---|---|---|
| `isp-pe-1.txt` | isp-pe-1 | Provider Edge / ABR | 0 & 1 | 10.0.0.1/32 | OSPF ABR, passive customer interfaces, multi-area summarization |
| `isp-pe-2.txt` | isp-pe-2 | Provider Edge / ABR | 0 & 2 | 10.0.0.2/32 | OSPF ABR, passive customer interfaces, multi-area summarization |
| `isp-core-1.txt` | isp-core-1 | Backbone Core | 0 | 10.0.0.3/32 | Pure transit, OSPF only |
| `isp-core-2.txt` | isp-core-2 | Backbone Core | 0 | 10.0.0.4/32 | Pure transit, OSPF only |
| `isp-border-1.txt` | isp-border-1 | Border (future eBGP) | 0 | 10.0.0.5/32 | OSPF backbone router; eBGP peering planned for Phase 3 |
| `cust-rtr-home-001.txt` | cust-rtr-home-001 | Customer Edge | 1 | — | Inter-VLAN routing (VLAN 11 & 22), NAT/PAT, static default |
| `cust-rtr-home-002.txt` | cust-rtr-home-002 | Customer Edge | 1 | — | NAT/PAT, static default |
| `cust-rtr-home-004.txt` | cust-rtr-home-004 | Customer Edge | 2 | — | NAT/PAT, static default |
| `cust-rtr-business-001.txt` | cust-rtr-business-001 | Customer Edge | 2 | — | Inter-VLAN routing (VLAN 10/20/30), NAT/PAT |

## Conventions used in these configs

- **Credentials redacted.** All passwords, secrets, and community strings have been replaced with `password123` placeholders. These configs are documentation, not deployable artifacts.
- **Passive interfaces** on PE routers: customer-facing and DNS-LAN interfaces are configured `passive-interface` in OSPF to advertise the prefixes without sending hellos.
- **Naming convention:** ISP devices use `isp-<role>-<n>`; customer-edge routers use `cust-rtr-<type>-<id>`.
- **IOS version:** Cisco IOS as packaged with Packet Tracer 8.x (2911 ISR, 2960 switch images).

## Suggested reading order

If you only have time to look at a couple of configs, start here:

1. **`isp-pe-1.txt`** — the most interesting config in the project. Demonstrates OSPF multi-area design (interfaces in Area 0 and Area 1), passive-interface usage, and how an ABR is configured in practice.
2. **`isp-core-1.txt`** — a clean example of a pure transit router. Notice the deliberate simplicity: no customer interfaces, no policy, just OSPF and routed point-to-point links.
3. **`cust-rtr-business-001.txt`** — router-on-a-stick inter-VLAN routing with three VLANs plus NAT/PAT. Shows customer-edge configuration patterns.

## What these configs demonstrate

- **OSPF design:** multi-area configuration, area-type assignment, passive-interface, loopback as router-id
- **IP services:** NAT/PAT, DHCP, static routing on the customer edge
- **VLAN configuration:** 802.1Q trunking, sub-interface inter-VLAN routing
- **IOS hygiene:** consistent naming, full interface descriptions, structured config sections

## Loading into Packet Tracer

To reproduce the lab, open `../packet-tracer/isp-multi-area-ospf.pkt`. Each device's running-config in that file matches the corresponding `.txt` in this folder.