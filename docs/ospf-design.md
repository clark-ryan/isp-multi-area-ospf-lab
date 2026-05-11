# OSPF Design Rationale

This document explains the OSPF architecture used in the ISP simulation: what each area represents, where the area boundaries fall, why specific routers were chosen as ABRs, and how design choices were made.

## Design goals

The OSPF deployment was designed to satisfy three goals:

1. **Scalability** — the architecture should support adding additional customer access regions without requiring backbone reconfiguration.
2. **Fault containment** — topology changes in one region should not trigger SPF recomputation across the entire AS.
3. **Operational clarity** — each router should have a clearly defined role (access, transit, or border) that maps directly to its OSPF configuration.

These goals push the design toward multi-area OSPF with hierarchical addressing, rather than a flat single-area deployment.

## Area architecture

The AS is partitioned into three OSPF areas:

### Area 0 — Backbone

Contains the three transit-only routers: `isp-core-1`, `isp-core-2`, and `isp-border-1`. These routers form a fully-meshed triangle, giving OSPF multiple equal-cost paths for any traffic crossing the backbone.

No customers, servers, or non-transit services attach to Area 0 directly. This is deliberate — keeping the backbone "pure" means that backbone routers run a smaller, more stable LSDB and don't carry policy logic that belongs at the edge.

`isp-border-1` is the future eBGP speaker (Phase 3). Placing it in Area 0 today means external routes redistributed into OSPF will originate in the backbone, which is conventional and avoids redistributing externals into a non-zero area (a known source of suboptimal routing).

### Area 1 — Customer access (east)

Contains `isp-pe-1`, the DNS LAN segment for that region, and the customer-uplink interfaces toward Customer A and Customer B. Customer-edge routers themselves do not participate in OSPF — they receive a default route via static configuration.

Area 1 exists as a separate area (rather than being collapsed into Area 0) so that customer churn — link flaps, customer adds/removes, ACL changes — does not propagate into the backbone's SPF calculation. The ABR at the edge of Area 1 hides Area 1's internal topology from Area 0, advertising only summary information.

### Area 2 — Customer access (west)

Symmetric to Area 1. Contains `isp-pe-2`, the DNS LAN segment for that region, and the customer uplinks toward Customer C and Customer D.

Two access areas (rather than just one) was a deliberate choice to demonstrate multi-area behavior. With only one non-zero area, the architecture would still technically be "multi-area" but wouldn't exercise the inter-area routing path between two non-backbone areas — which is the most interesting property of hierarchical OSPF.

## ABR placement

`isp-pe-1` and `isp-pe-2` are configured as Area Border Routers. Each PE has interfaces in two areas: its access area (1 or 2) and Area 0 (the uplink to the cores).

Why the PEs are ABRs:

- They are the natural seam between customer-edge complexity (varied customers, ACLs, NAT, passive interfaces) and the simpler backbone.
- Co-locating the ABR with the PE function means a single device handles both the OSPF area boundary and the policy boundary, which simplifies operations.
- The PEs are not transit routers in the backbone sense — they sit one layer out from the core — so even if they fail, the backbone remains intact.

An alternative design would put the ABRs in the core (with the access area extending into the PE), but this would mix policy responsibilities and increase the LSDB size on the core routers. The chosen design keeps responsibilities crisply separated.

## Passive-interface strategy

The following interface types are configured `passive-interface` in OSPF:

| Interface type | Rationale |
|---|---|
| Customer-facing PE interfaces | Customer routers do not speak OSPF; sending hellos here is wasted traffic and a minor security exposure. |
| DNS LAN interfaces | DNS servers are not OSPF speakers; same reasoning. |
| Loopback interfaces | Nothing to peer with on a loopback. |

Without passive-interface on these, OSPF would emit hello packets that no one answers, polluting captures and potentially exposing the routing protocol to malicious participation from a compromised customer host.

The prefixes on passive interfaces are still advertised into OSPF via the `network` statement — passive only suppresses hellos, not advertisement. This is what makes customer subnets reachable from elsewhere in the AS.

## Router-ID strategy

Every OSPF router has a manually configured router-id that matches its loopback /32:

| Router | Router-ID |
|---|---|
| isp-pe-1 | 10.0.0.1 |
| isp-pe-2 | 10.0.0.2 |
| isp-core-1 | 10.0.0.3 |
| isp-core-2 | 10.0.0.4 |
| isp-border-1 | 10.0.0.5 |

Manually pinning the router-id (rather than letting OSPF pick automatically based on interface IPs) prevents the router-id from changing when interfaces flap, which would otherwise cause unnecessary topology churn. Loopback interfaces are also always up, so OSPF adjacencies sourced from them are more stable than those sourced from physical interfaces.

In Phase 3, these same loopback /32s will serve as the BGP router-id and the source address for iBGP/eBGP sessions.

## LSA flow

Three LSA types are most relevant to this design:

- **Type 1 (Router LSA):** generated by every OSPF router, flooded within a single area. Each access area's Router LSAs stay confined to that area.
- **Type 2 (Network LSA):** generated by the DR on multi-access segments. In this design, multi-access OSPF segments exist on the DNS LANs (where the PE elects DR for itself, since the DNS server doesn't participate).
- **Type 3 (Summary LSA):** generated by ABRs to carry inter-area reachability. When Customer A wants to reach Customer C, the route from A's perspective is `O IA via isp-pe-1`, which traces back through Type 3 LSAs originated by both ABRs.

There are no Type 5 (External) LSAs in the current design because no routes are redistributed into OSPF. Type 5s will appear in Phase 3 when BGP routes from the transit AS are redistributed into the OSPF domain — at which point Area 0 will carry them and any stub areas would block them.

## Convergence considerations

The backbone triangle gives any single core failure two alternate paths. SPF recomputation in the access areas is not affected by core link failures — only the path traversing the failed link is recomputed by the cores themselves.

Tested failure scenario: shutting down the link between `isp-core-1` and `isp-core-2` caused traffic between Customer A and Customer C to reroute via `isp-border-1` within ~3–5 seconds (default OSPF dead-interval is 40s but the link-state change triggers immediate SPF). See `screenshots/08-convergence-link-failure.png` for the demonstration.

## Trade-offs made

A few deliberate choices worth noting:

- **All areas are normal areas**, not stubs. The decision to keep all areas as normal keeps the design simple and avoids reconfiguration when BGP is added; a future revision could make Areas 1 and 2 stub areas with default-route injection from the ABRs.
- **Single ABR per access area.** This is a single point of failure for each access region. A production design would dual-home each access area to two ABRs. This was omitted to keep the topology readable; the convention extends naturally to multiple ABRs.
- **Customer-edge routers do not participate in OSPF.** This is industry standard — customers receive a default route via static config. Running OSPF with customers would expose the ISP's internal topology and trust the customer's advertisements, both of which are operationally undesirable.


---