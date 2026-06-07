# Spine-Leaf Network Implementation

> A Cisco Modeling Labs (CML) lab implementing a production-style **Spine-Leaf data center fabric** using VXLAN/EVPN over IS-IS underlay and iBGP overlay — with multi-tenant VRF support.

![Spine-Leaf Topology](Spine-Leaf%20Network.png)

---

## Overview

This lab models a 2-tier spine-leaf architecture commonly deployed in modern data center environments. It demonstrates how overlay technologies (VXLAN + EVPN) can be layered on top of a routed underlay (IS-IS) to provide scalable, multi-tenant Layer 2 and Layer 3 connectivity across the fabric.

All devices are pre-configured and boot into a fully operational state when loaded into CML.

---

## Topology

### Devices

| Role | Hostname | Platform | Loopback (VTEP) | IOS-XE Version |
|------|----------|----------|-----------------|----------------|
| Spine | S2 | CSR1000V | `1.1.1.2/32` | 17.3 |
| Spine | S3 | CSR1000V | `1.1.1.3/32` | 17.3 |
| Leaf | L2 | Cat8000V | `2.2.2.2/32` | 17.13 |
| Leaf | L3 | Cat8000V | `2.2.2.3/32` | 17.13 |
| Leaf | L4 | Cat8000V | `2.2.2.4/32` | 17.13 |

### Point-to-Point Underlay Links

| Link | Interface (Spine) | IP (Spine) | Interface (Leaf) | IP (Leaf) |
|------|-------------------|------------|------------------|-----------|
| S2 ↔ L2 | GigabitEthernet2 | `10.2.2.0/31` | GigabitEthernet2 | `10.2.2.1/31` |
| S2 ↔ L3 | GigabitEthernet3 | `10.2.3.0/31` | GigabitEthernet2 | `10.2.3.1/31` |
| S2 ↔ L4 | GigabitEthernet4 | `10.2.4.0/31` | GigabitEthernet2 | `10.2.4.1/31` |
| S3 ↔ L2 | GigabitEthernet2 | `10.3.2.0/31` | GigabitEthernet3 | `10.3.2.1/31` |
| S3 ↔ L3 | GigabitEthernet3 | `10.3.3.0/31` | GigabitEthernet3 | `10.3.3.1/31` |
| S3 ↔ L4 | GigabitEthernet4 | `10.3.4.0/31` | GigabitEthernet3 | `10.3.4.1/31` |

### Host-Facing Ports

| Leaf | Interface | VLAN / Service Instance | Tenant Subnet |
|------|-----------|------------------------|---------------|
| L2 | GigabitEthernet4 | Service Instance 10 (untagged) | `192.168.10.0/24` |
| L2 | GigabitEthernet5 | Service Instance 20 (untagged) | `192.168.20.0/24` |
| L3 | GigabitEthernet4 | Service Instance 10 (untagged) | `192.168.10.0/24` |
| L4 | GigabitEthernet4 | Service Instance 20 (untagged) | `192.168.20.0/24` |

---

## Technology Stack

### Underlay — IS-IS

All devices participate in a single IS-IS Level-2-only domain (Area `49.0001`, ASN equivalent), using wide metrics and point-to-point links. IS-IS provides loop-free reachability between all loopback addresses, which serve as VTEP and BGP router-IDs.

| Device | IS-IS NET |
|--------|-----------|
| S2 | `49.0001.0000.0000.1002.00` |
| S3 | `49.0001.0000.0000.1003.00` |
| L2 | `49.0001.0000.0000.2002.00` |
| L3 | `49.0001.0000.0000.2003.00` |
| L4 | `49.0001.0000.0000.2004.00` |

### Overlay — iBGP EVPN (AS 65000)

BGP AS `65000` runs across all devices using loopback peering. The two spine nodes (S2, S3) act as **Route Reflectors**, reflecting EVPN routes to all leaf peers. Leaf nodes peer only to spines — no leaf-to-leaf direct BGP sessions.

- Address Family: `l2vpn evpn`
- Community: Extended communities enabled on all peers
- Route Reflection: S2 and S3 reflect for leaves `2.2.2.2`, `2.2.2.3`, `2.2.2.4`

### VXLAN Data Plane

Each leaf runs an NVE (Network Virtualization Edge) interface sourced from `Loopback0`. Tunnel replication mode is **ingress replication** (unicast BUM flooding).

| VNI | Purpose | Bridge Domain | Tenant |
|-----|---------|--------------|--------|
| `10010` | L2 stretch — VLAN 10 | BD 10 | TENANT-1 |
| `10020` | L2 stretch — VLAN 20 | BD 20 | TENANT-1 |
| `50000` | L3 VNI (IRB/inter-subnet routing) | BD 50 | TENANT-1 |

### Multi-Tenancy — VRF TENANT-1

All leaf nodes carry VRF `TENANT-1`, which provides inter-subnet routing between VLAN 10 (`192.168.10.0/24`) and VLAN 20 (`192.168.20.0/24`). The L3 VNI (`50000`) stitches the VRF across the fabric via EVPN Type-5 routes.

- RD format: `<loopback>:1` per device (e.g., `2.2.2.2:1` on L2)
- Route-Targets: import/export `1:1` for the L3 VNI
- Anycast Gateway MAC: `0000.aaaa.bbbb` (same MAC on all leaves for seamless VM mobility)
- BDI10 / BDI20 on each leaf act as distributed default gateways: `192.168.10.254` / `192.168.20.254`

---

## Files

| File | Description |
|------|-------------|
| `Spine-Leaf Network.yaml` | CML lab topology file — import directly into CML |
| `Spine-Leaf Network.png` | Topology diagram |

---

## Prerequisites

- Cisco Modeling Labs (CML) 2.x or later
- Cisco image licenses for:
  - **CSR1000V** (Spine nodes — IOS-XE 17.3)
  - **Cat8000V** (Leaf nodes — IOS-XE 17.13, `network-advantage` + `dna-advantage`)
- Sufficient host RAM (~2–4 GB per node recommended)

---

## Getting Started

**1. Clone the repo**
```bash
git clone https://github.com/Krishn-G/Spine-Leaf_Network_Implementation.git
```

**2. Import into CML**

In the CML dashboard, navigate to **Import Lab** and upload `Spine-Leaf Network.yaml`. All node configurations are embedded in the topology file and will be pre-loaded on first boot.

**3. Start the lab**

Start all nodes. Allow 2–3 minutes for IOS-XE to fully initialize. IS-IS adjacencies and BGP sessions come up automatically.

**4. Verify underlay**

SSH into any leaf and confirm IS-IS neighbors:
```
L2# show isis neighbors
```

**5. Verify EVPN overlay**

Check BGP EVPN table on a leaf:
```
L2# show bgp l2vpn evpn summary
L2# show bgp l2vpn evpn
```

**6. Verify VXLAN tunnels**

```
L2# show nve peers
L2# show l2vpn evpn mac ip
```

**7. Test end-to-end connectivity**

Attach hosts to the leaf access ports and ping across VLANs to validate inter-subnet routing through the L3 VNI.

---

## Key Concepts Demonstrated

- **Spine-Leaf (Clos) fabric** — non-blocking, equal-cost multipath architecture
- **IS-IS as underlay** — fast convergence, loop-free loopback reachability
- **VXLAN encapsulation** — Layer 2 extension over Layer 3 routed fabric
- **EVPN control plane** — BGP-based MAC/IP advertisement, eliminating flood-and-learn
- **Symmetric IRB** — distributed anycast gateway for inter-VLAN routing without a centralized router
- **VRF / multi-tenancy** — traffic isolation between tenants across a shared physical fabric
- **BGP Route Reflection** — spine nodes as RRs to reduce iBGP mesh complexity

---

## License

This project is licensed under the [MIT License](LICENSE).
