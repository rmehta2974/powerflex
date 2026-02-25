# Dell PowerFlex Cabling Diagram
## PowerFlex 4.x / 5.x Rack and Appliance

**Version:** 1.0  
**Based on:** Dell PowerFlex Rack/Appliance Documentation, Network Planning Guide  

---

## Table of Contents

1. [ABBA Cabling Principle](#1-abba-cabling-principle)
2. [Single Node Cabling (ABBA)](#2-single-node-cabling-abba)
3. [Rack-Wide Topology](#3-rack-wide-topology)
4. [Access Switch Connectivity](#4-access-switch-connectivity)
5. [PowerFlex Management Controller Cabling](#5-powerflex-management-controller-cabling)
6. [Dual Network (Management + Data)](#6-dual-network-management--data)
7. [Port and Cable Summary](#7-port-and-cable-summary)
8. [Physical Layer Best Practices](#8-physical-layer-best-practices)

---

## 1. ABBA Cabling Principle

**ABBA** ensures redundant connectivity: each node connects to **both** switches with different NICs, so a single switch failure does not isolate the node.

```
        Switch A                    Switch B
           │                            │
     ┌─────┴─────┐                ┌─────┴─────┐
     │  Port Ch  │                │  Port Ch  │
     │  (or      │                │  (or      │
     │  Trunk)   │                │  Trunk)   │
     └─────┬─────┘                └─────┬─────┘
           │                            │
           │    Node NIC1        Node NIC2
           │         \                /
           │          \              /
           │           \            /
           │            \          /
           │             \        /
           │              \      /
           │               \    /
           │                \  /
           │                 \/
           │         ┌──────────────┐
           │         │   Node 1     │
           │         │  NIC1  NIC2  │
           │         │   SDS/SDC    │
           │         └──────────────┘
           │
     A-B: NIC1 → Switch A, NIC2 → Switch B
     Ensures no single point of failure
```

**Rule:** NIC1 to Switch A, NIC2 to Switch B (or vice versa). Never cable both NICs to the same switch.

---

## 2. Single Node Cabling (ABBA)

### 2.1 Hyperconverged / Compute-Only Node (4×25 GbE or 4×100 GbE)

```
    ┌─────────────────┐              ┌─────────────────┐
    │   Access        │              │   Access        │
    │   Switch A      │              │   Switch B      │
    │                 │              │                 │
    │  Port 1 ◄───────┼──────────────┼── NIC1 (25/100G) │
    │  Port 2 ◄───────┼──────────────┼── NIC2 (25/100G)│
    └────────┬────────┘              └────────┬────────┘
             │                                │
             │         ┌─────────────────────┴─────────┐
             │         │                               │
             │    NIC3 (25/100G) ──────────────────► Port 1
             │    NIC4 (25/100G) ──────────────────► Port 2
             │         │                               │
             └─────────┼───────────────────────────────┘
                       │
                       │   ┌─────────────────────────┐
                       └───│  PowerFlex Node         │
                           │  (Hyperconverged /      │
                           │   Compute-only)         │
                           │  NIC1,2 → Switch A      │
                           │  NIC3,4 → Switch B      │
                           └─────────────────────────┘
```

### 2.2 Storage-Only Node (4×25 GbE or 4×100 GbE)

Same physical pattern as hyperconverged:
- NIC1, NIC2 → Switch A (Bond0 or trunk)
- NIC3, NIC4 → Switch B (Bond1 or trunk)

---

## 3. Rack-Wide Topology

```
                                    [Customer Core / Aggregation]
                                               │
                                    ┌──────────┴──────────┐
                                    │  2×100 GbE uplinks  │
                                    └──────────┬──────────┘
                                               │
              ┌────────────────────────────────┼────────────────────────────────┐
              │                                │                                │
              ▼                                ▼                                ▼
    ┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
    │  Management      │              │  Access         │              │  Access         │
    │  Switch          │              │  Switch A        │              │  Switch B       │
    │  (1 GbE OOB)     │              │  (25/100 GbE)    │              │  (25/100 GbE)   │
    └────────┬─────────┘              └────────┬─────────┘              └────────┬────────┘
             │                                │                                │
             │ iDRAC / OOB                    │                                │
             │ (VLAN 101)                     │ VLT/vPC                        │
             │                                │ interconnect                    │
             │                         ┌──────┴──────┐                           │
             │                         │             │                           │
             │                         ▼             ▼                           │
             │                  ┌──────────┐  ┌──────────┐  ┌──────────┐
             │                  │ Node 1   │  │ Node 2   │  │ Node 3   │  ...
             └──────────────────│ NIC OOB  │  │ NIC OOB  │  │ NIC OOB  │
                                │ NIC1→A   │  │ NIC1→A   │  │ NIC1→A   │
                                │ NIC2→B   │  │ NIC2→B   │  │ NIC2→B   │
                                └──────────┘  └──────────┘  └──────────┘
```

---

## 4. Access Switch Connectivity

### 4.1 Port Channel (LACP) to Node

```
    Access Switch A                          Access Switch B
    
    ┌────────────────────────┐               ┌────────────────────────┐
    │ Port 1 ────────────────┼─── NIC1 ──────┤ Node                    │
    │ Port 2 ────────────────┼─── NIC2       │ (Bond0 / Port Channel)  │
    │         (LACP Po1)     │               │                         │
    └────────────────────────┘               │ NIC3 ───────────────────┼── Port 1
                                             │ NIC4 ───────────────────┼── Port 2
                                             │         (LACP Po2)      │
                                             └─────────────────────────┘
```

- Port 1+2 on Switch A form Port Channel 1 (to NIC1+NIC2)
- Port 1+2 on Switch B form Port Channel 2 (to NIC3+NIC4)
- vPC/VLT between Switch A and B for redundancy

### 4.2 Trunk to Node (No LACP)

```
    Access Switch A                          Access Switch B
    
    ┌────────────────────────┐               ┌────────────────────────┐
    │ Port 1 (Trunk) ────────┼─── NIC1 ──────┤ Node                    │
    │ VLANs: 151,152,...     │               │ (Standalone trunk)     │
    └────────────────────────┘               │                         │
                                             │ NIC2 ───────────────────┼── Port 1 (Trunk)
                                             │ VLANs: 151,152,...      │
                                             └─────────────────────────┘
```

---

## 5. PowerFlex Management Controller Cabling

### 5.1 PFMC 2.0 (3–5 Nodes, ESXi)

Each PFMC node:
- **Front-end (fe_dvSwitch):** Management, PowerFlex mgmt, vCenter – 25 GbE
- **Back-end (be_dvSwitch):** vCenter HA, SDS data – 25 GbE
- **OOB (oob_dvSwitch):** iDRAC – Access VLAN 101

```
    PFMC Node 1              PFMC Node 2              PFMC Node 3
    ┌──────────┐             ┌──────────┐             ┌──────────┐
    │ NIC1→A   │             │ NIC1→A   │             │ NIC1→A   │
    │ NIC2→B   │             │ NIC2→B   │             │ NIC2→B   │
    │ NIC3→A   │             │ NIC3→A   │             │ NIC3→A   │
    │ NIC4→B   │             │ NIC4→B   │             │ NIC4→B   │
    └──────────┘             └──────────┘             └──────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │  Access Switch A    Access Switch B  │
                    └───────────────────────────────────┘
```

### 5.2 PFMC KVM (SLES-Based)

- Similar physical cabling: 4×25 GbE per node, ABBA to access switches
- OOB (1 GbE) to management switch for iDRAC

---

## 6. Dual Network (Management + Data)

When management and data use separate physical networks:

```
    [Mgmt Network]                    [Data Network]
    
    ┌─────────────┐                   ┌─────────────┐
    │ Mgmt Switch │                   │ Data Switch │
    │     A       │                   │     A       │
    └──────┬──────┘                   └──────┬──────┘
           │                                 │
           │                          ┌──────┴──────┐
           │                          │              │
           │                   ┌─────┴─────┐  ┌─────┴─────┐
           │                   │ Data Sw B │  │ Data Sw B │
           │                   └─────┬─────┘  └─────┬─────┘
           │                         │              │
           │    ┌────────────────────┼──────────────┼────────────┐
           │    │                    │              │            │
           └────┼── OOB (1G)         │              │            │
                │                    │              │            │
                ▼                    ▼              ▼            │
           ┌─────────────────────────────────────────────────┐
           │  PowerFlex Node                                   │
           │  OOB NIC → Mgmt Switch                            │
           │  NIC1,NIC2 → Data Switch A (Bond0)                │
           │  NIC3,NIC4 → Data Switch B (Bond1)                │
           └─────────────────────────────────────────────────┘
```

---

## 7. Port and Cable Summary

| Component | Ports/Cables | Type | Purpose |
|-----------|--------------|------|---------|
| **PowerFlex node** | 4×25 GbE or 4×100 GbE | SFP28/DAC or QSFP28 | Data, management, vMotion |
| **PowerFlex node** | 1×1 GbE | Cat5/Cat6 | iDRAC OOB |
| **Access switch** | 2×100 GbE | QSFP28 | Uplink to aggregation |
| **Access switch** | 2×100 GbE | QSFP28 | VLT/vPC interconnect |
| **Access switch** | 1×1 GbE | Cat5/Cat6 | Management |

---

## 8. Physical Layer Best Practices

| Practice | Description |
|----------|-------------|
| **ABBA** | Never cable both NICs of a bond to the same switch |
| **Labeling** | Label cables at both ends (node, switch, port) |
| **Length** | Use appropriate DAC/cable length; avoid excessive slack |
| **Bend radius** | Follow cable manufacturer spec for fiber |
| **Zones** | Separate Zone A and Zone B power for redundancy |
| **Documentation** | Maintain cabling spreadsheet (node, NIC, switch, port) |
| **LLDP/CDP** | Enable on switch ports for topology discovery |
| **MTU** | Ensure 9216 end-to-end on data path |

---

## Reference Diagrams (Simplified)

### Minimal 4-Node Hyperconverged

```
    [Switch A]              [Switch B]
        │                       │
    ┌───┴───┐               ┌───┴───┐
    │ N1,N2 │               │ N1,N2 │
    │ N3,N4 │               │ N3,N4 │
    └───┬───┘               └───┬───┘
        │   Node1 Node2        │
        │   Node3 Node4        │
        └──────────┬───────────┘
                   │
            [4 Hyperconverged Nodes]
```

---

## References

- Dell PowerFlex Appliance Network Planning Guide 4.x
- Dell PowerFlex Rack Field Implementation Guide
- Dell PowerFlex 4.6 Complete Guide (Cabling, ABBA, CDP, LLDP)
