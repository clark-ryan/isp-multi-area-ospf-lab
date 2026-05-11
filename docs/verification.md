# Verification

Test outputs demonstrating that the network functions as designed. All output was captured from Cisco Packet Tracer 8.x with the running-configs in [`../configs/`](../configs/) loaded.

---

## 1. OSPF adjacencies

### 1.1 Provider Edge — `isp-pe-1`

```text
isp-pe-1#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.0.0.3          1   FULL/DR         00:00:31    203.0.113.6     GigabitEthernet0/2
```

A single FULL adjacency with `isp-core-1` over the Area 0 uplink. Customer-facing and DNS-LAN interfaces are configured `passive-interface`, so no Area 1 neighbors appear.

### 1.2 Backbone core — `isp-core-1`

```text
isp-core-1#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.0.0.1          1   FULL/BDR        00:00:35    203.0.113.5     GigabitEthernet0/0
10.0.0.4          1   FULL/DR         00:00:35    203.0.113.13    GigabitEthernet0/1
10.0.0.5          1   FULL/DR         00:00:35    203.0.113.18    GigabitEthernet0/2
```

Three FULL adjacencies — one to each backbone peer (`isp-pe-1`, `isp-core-2`, `isp-border-1`). Confirms the fully-meshed backbone is operating.

---

## 2. Inter-area routes on the backbone

```text
isp-core-1#show ip route ospf | include O IA
O IA    203.0.113.0 [110/2] via 203.0.113.5, 01:01:14, GigabitEthernet0/0
O IA    203.0.113.24 [110/2] via 203.0.113.5, 01:01:14, GigabitEthernet0/0
O IA    203.0.113.28 [110/2] via 203.0.113.5, 01:01:14, GigabitEthernet0/0
```

Three Area 1 prefixes appear as `O IA` routes via `isp-pe-1` — Customer A uplink, DNS Area 1 LAN, and Customer B uplink. The `O IA` tag is the visible proof that multi-area OSPF is operating.

> **Known issue:** Area 2 prefixes do not appear here despite being reachable end-to-end (verified in §5). Suspected cause: `isp-pe-2`'s customer-facing interfaces are configured in Area 0 rather than Area 2. To be corrected in the next revision.

---

## 3. Multi-area OSPF database on the ABR

```text
isp-pe-1#show ip ospf database
            OSPF Router with ID (10.0.0.1) (Process ID 1)

                Router Link States (Area 0)

Link ID         ADV Router      Age         Seq#       Checksum Link count
10.0.0.1        10.0.0.1        167         0x80000006 0x00a2f4 2
10.0.0.5        10.0.0.5        163         0x80000008 0x00f3c8 3
10.0.0.3        10.0.0.3        163         0x8000000a 0x0070c9 4
10.0.0.4        10.0.0.4        162         0x8000000a 0x00f631 4
10.0.0.2        10.0.0.2        162         0x80000009 0x00bd7d 5

                Net Link States (Area 0)
Link ID         ADV Router      Age         Seq#       Checksum
203.0.113.6     10.0.0.3        168         0x80000003 0x0042af
203.0.113.21    10.0.0.5        163         0x80000005 0x00c118
203.0.113.18    10.0.0.5        163         0x80000006 0x00d705
203.0.113.13    10.0.0.4        162         0x80000005 0x0008dc
203.0.113.10    10.0.0.4        162         0x80000006 0x001ec9

                Summary Net Link States (Area 0)
Link ID         ADV Router      Age         Seq#       Checksum
203.0.113.0     10.0.0.1        197         0x80000007 0x00db37
203.0.113.24    10.0.0.1        197         0x80000008 0x00e910
203.0.113.28    10.0.0.1        197         0x80000009 0x00bf35

                Router Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum Link count
10.0.0.1        10.0.0.1        206         0x80000006 0x0080b2 3

                Summary Net Link States (Area 1)
Link ID         ADV Router      Age         Seq#       Checksum
10.0.0.1        10.0.0.1        197         0x8000001b 0x0072bb
203.0.113.4     10.0.0.1        197         0x8000001c 0x008a6f
203.0.113.12    10.0.0.1        157         0x8000001d 0x0041ae
10.0.0.3        10.0.0.1        157         0x8000001e 0x0062c5
203.0.113.16    10.0.0.1        157         0x8000001f 0x0015d4
203.0.113.20    10.0.0.1        147         0x80000020 0x00f4ee
203.0.113.8     10.0.0.1        147         0x80000021 0x006b83
10.0.0.4        10.0.0.1        147         0x80000022 0x005ac7
10.0.0.5        10.0.0.1        147         0x80000023 0x004ed1
10.0.0.2        10.0.0.1        147         0x80000024 0x0074ac
203.0.113.32    10.0.0.1        147         0x80000025 0x007c55
203.0.113.36    10.0.0.1        147         0x80000026 0x00527a
203.0.113.40    10.0.0.1        147         0x80000027 0x00289f
```

The output is partitioned by area, which is the most direct evidence that multi-area OSPF is working. Area 0 contains five Router LSAs (one per backbone-participating router); Area 1 contains exactly one (just `isp-pe-1` itself, since customer routers do not run OSPF). The Summary LSAs originated by `isp-pe-1` carry inter-area reachability across the boundary in both directions — three Area 1 prefixes summarized into Area 0, and thirteen Area 0 prefixes summarized into Area 1.

---

## 4. End-to-end reachability — Customer A → Customer D

```text
C:\>ping 203.0.113.37

Pinging 203.0.113.37 with 32 bytes of data:

Request timed out.
Reply from 203.0.113.37: bytes=32 time<1ms TTL=250
Reply from 203.0.113.37: bytes=32 time<1ms TTL=250
Reply from 203.0.113.37: bytes=32 time<1ms TTL=250

Ping statistics for 203.0.113.37:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

ICMP echo from a VLAN 22 host in Customer A reaches `cust-rtr-business-001`'s WAN interface in Area 2. First packet times out during ARP resolution; subsequent replies succeed.

---

## 5. Path tracing — Customer A → Customer D

```text
C:\>tracert 203.0.113.37

Tracing route to 203.0.113.37 over a maximum of 30 hops:

  1   1 ms      0 ms      0 ms      10.10.20.1
  2   *         0 ms      0 ms      203.0.113.1
  3   0 ms      0 ms      0 ms      203.0.113.6
  4   0 ms      0 ms      0 ms      203.0.113.13
  5   0 ms      0 ms      0 ms      203.0.113.9
  6   0 ms      0 ms      0 ms      203.0.113.37
```

Six clean hops crossing three OSPF areas (1 → 0 → 2):

| Hop | Address | Device |
|---|---|---|
| 1 | 10.10.20.1 | `cust-rtr-home-001` (VLAN 22 gateway) |
| 2 | 203.0.113.1 | `isp-pe-1` (Area 1 customer-facing) |
| 3 | 203.0.113.6 | `isp-core-1` |
| 4 | 203.0.113.13 | `isp-core-2` |
| 5 | 203.0.113.9 | `isp-pe-2` |
| 6 | 203.0.113.37 | `cust-rtr-business-001` (Customer D WAN) |

---

## 6. OSPF convergence after backbone link failure

Continuous ping from Customer A to Customer D (203.0.113.37) while the `isp-core-1` ↔ `isp-core-2` link is administratively shut down.

```text
C:\>ping -t 203.0.113.37

Pinging 203.0.113.37 with 32 bytes of data:

Reply from 203.0.113.37: bytes=32 time<1ms TTL=250
Reply from 203.0.113.37: bytes=32 time<1ms TTL=250
Reply from 203.0.113.37: bytes=32 time<1ms TTL=250
Request timed out.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Request timed out.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Request timed out.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Request timed out.
Reply from 203.0.113.1: Destination host unreachable.
Reply from 203.0.113.1: Destination host unreachable.
Request timed out.
Reply from 203.0.113.37: bytes=32 time<1ms TTL=249
Reply from 203.0.113.37: bytes=32 time<1ms TTL=249
Reply from 203.0.113.37: bytes=32 time<1ms TTL=249
```

Three phases are visible. **Pre-failure:** steady replies, TTL = 250. **Reconvergence:** `isp-pe-1` responds with "Destination host unreachable" while SPF recomputes — the router knows its next-hop is invalid rather than silently dropping. **Post-failure:** pings recover with **TTL = 249**, indicating the new path is one hop longer (traffic now traverses `isp-border-1` instead of the direct core-to-core link). Total convergence ≈ 5–10 seconds.

---

## 7. NAT/PAT translations on the customer edge

```text
cust-rtr-home-001#show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
udp 203.0.113.2:1025   10.10.10.11:1025   203.0.113.26:53    203.0.113.26:53
udp 203.0.113.2:1026   10.10.10.11:1026   203.0.113.26:53    203.0.113.26:53
tcp 203.0.113.2:1025   10.10.10.11:1025   203.0.113.41:80    203.0.113.41:80
```

Three concurrent conversations from inside-local 10.10.10.11 are PAT-overloaded onto the single inside-global address 203.0.113.2 — two DNS lookups (UDP/53) and one HTTP request (TCP/80). The ISP-side routers never see the RFC1918 source.

---
