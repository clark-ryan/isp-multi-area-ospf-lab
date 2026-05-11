# IP Addressing Plan

## Address block allocation

| Block | Mask | Purpose |
|---|---|---|
| 10.0.0.0/24 | /24 | Loopbacks for ISP routers (one /32 per device) |
| 203.0.113.0/24 | /24 | ISP public addressing — P2P links, DNS LANs, customer uplinks (TEST-NET-3, reserved for documentation per RFC 5737) |
| 10.10.0.0/16 | /16 | Customer A — residential, VLAN-separated |
| 192.168.1.0/24 | /24 | Customer B — residential |
| 172.16.1.0/24 | /24 | Customer C — residential |
| 192.168.0.0/16 | /16 | Customer D — small business, VLAN-separated |

## ISP router loopbacks

| Device | Interface | IP/Mask | Purpose |
|---|---|---|---|
| isp-pe-1 | Lo0 | 10.0.0.1/32 | OSPF router-id, management target |
| isp-pe-2 | Lo0 | 10.0.0.2/32 | OSPF router-id, management target |
| isp-core-1 | Lo0 | 10.0.0.3/32 | OSPF router-id, management target |
| isp-core-2 | Lo0 | 10.0.0.4/32 | OSPF router-id, management target |
| isp-border-1 | Lo0 | 10.0.0.5/32 | OSPF router-id, future BGP router-id |

## ISP backbone P2P links

| Subnet | Link | Endpoints | OSPF Area |
|---|---|---|---|
| 203.0.113.8/30 | isp-pe-1 ↔ isp-core-1 | .9 = isp-pe-1 Gi0/2, .10 = isp-core-1 Gi0/1 | 0 |
| 203.0.113.12/30 | isp-pe-2 ↔ isp-core-2 | .13 = isp-pe-2 Gi0/2, .14 = isp-core-2 Gi0/1 | 0 |
| 203.0.113.16/30 | isp-core-1 ↔ isp-core-2 | .17 = isp-core-1 Gi0/2, .18 = isp-core-2 Gi0/2 | 0 |
| 203.0.113.20/30 | isp-core-1 ↔ isp-border-1 | .21 = isp-core-1 Gi0/3, .22 = isp-border-1 Gi0/0 | 0 |
| 203.0.113.24/30 | isp-core-2 ↔ isp-border-1 | .25 = isp-core-2 Gi0/3, .26 = isp-border-1 Gi0/1 | 0 |

## Customer-uplink P2P links

| Subnet | Customer | Endpoints | OSPF Area | Notes |
|---|---|---|---|---|
| 203.0.113.0/30 | Customer A | .1 = isp-pe-1 Gi0/0, .2 = cust-rtr-home-001 Gi0/0 | 1 | Passive on PE side |
| 203.0.113.4/30 | Customer B | .5 = isp-pe-1 Gi0/1, .6 = cust-rtr-home-002 Gi0/0 | 1 | Passive on PE side |
| 203.0.113.32/30 | Customer C | .33 = isp-pe-2 Gi0/0, .34 = cust-rtr-home-004 Gi0/0 | 2 | Passive on PE side |
| 203.0.113.36/30 | Customer D | .37 = isp-pe-2 Gi0/1, .38 = cust-rtr-business-001 Gi0/0 | 2 | Passive on PE side |

## DNS service LANs

| Subnet | Location | Gateway | Server IP | OSPF Area |
|---|---|---|---|---|
| 203.0.113.24/30 | Area 1 DNS | .25 = isp-pe-1 Fa0/2 | 203.0.113.26 (dns-area-1) | 1 |
| 203.0.113.40/30 | Area 2 DNS | .42 = isp-pe-2 Fa0/2 | 203.0.113.41 (dns-area-2) | 2 |

> **Note:** DNS LANs are currently on /30s, which permits only the gateway + one server. A future revision should migrate these to /29 segments to allow for redundant DNS, NTP, and syslog hosts.

## Customer A — Residential (cust-rtr-home-001)

| VLAN | Subnet | Gateway | Purpose |
|---|---|---|---|
| 11 | 10.10.10.0/24 | 10.10.10.1 (Gi0/1.11) | Trusted devices |
| 22 | 10.10.20.0/24 | 10.10.20.1 (Gi0/1.22) | IoT / untrusted devices |

WAN: Gi0/0 = 203.0.113.2/30. NAT/PAT overload outbound.

## Customer B — Residential (cust-rtr-home-002)

| Subnet | Gateway | Purpose |
|---|---|---|
| 192.168.1.0/24 | 192.168.1.1 (Gi0/1) | Home LAN |

WAN: Gi0/0 = 203.0.113.6/30. NAT/PAT overload outbound.

## Customer C — Residential (cust-rtr-home-004)

| Subnet | Gateway | Purpose |
|---|---|---|
| 172.16.1.0/24 | 172.16.1.1 (Gi0/1) | Home LAN |

WAN: Gi0/0 = 203.0.113.34/30. NAT/PAT overload outbound.

## Customer D — Small Business (cust-rtr-business-001)

| VLAN | Subnet | Gateway | Purpose |
|---|---|---|---|
| 10 | 192.168.10.0/24 | 192.168.10.1 (Gi0/1.10) | Workstations |
| 20 | 192.168.20.0/24 | 192.168.20.1 (Gi0/1.20) | Servers |
| 30 | 192.168.30.0/24 | 192.168.30.1 (Gi0/1.30) | Guest / wireless |

WAN: Gi0/0 = 203.0.113.38/30. NAT/PAT overload outbound.

## Addressing conventions

- **P2P links** use /30 with the lower-numbered IP on the higher-numbered router-id endpoint, by convention (not enforced).
- **Loopbacks** use /32 from the 10.0.0.0/24 block. Loopback IP = router-id.
- **Customer-facing PE interfaces** are configured `passive-interface` in OSPF — the network is advertised but no hellos are sent toward the customer.
- **DNS LANs** are in the same OSPF area as the PE that hosts them.
- **All customer-side networks** are private (RFC1918). Customer-edge routers perform NAT/PAT before traffic crosses into the ISP's 203.0.113.0/24 space.