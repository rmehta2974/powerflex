# Dell PowerFlex Network Best Practices and VLAN Guide
## PowerFlex 4.x Appliance and Rack

**Version:** 1.0  
**Based on:** Dell PowerFlex Appliance Network Planning Guide 4.x, PowerFlex 4.6 Complete Guide  

---

## Table of Contents

1. [Overview](#1-overview)
2. [Network Architecture Options](#2-network-architecture-options)
3. [VLAN Reference](#3-vlan-reference)
4. [VLAN Setup by Node Type](#4-vlan-setup-by-node-type)
5. [Switch Requirements](#5-switch-requirements)
6. [LACP and Trunking](#6-lacp-and-trunking)
7. [MTU and Jumbo Frames](#7-mtu-and-jumbo-frames)
8. [Multi-VLAN and Multi-Subnet](#8-multi-vlan-and-multi-subnet)
9. [VMware NSX Ready](#9-vmware-nsx-ready)
10. [Best Practices Summary](#10-best-practices-summary)

---

## 1. Overview

PowerFlex requires properly designed networks for management, hypervisor, vMotion, storage data, and replication. This guide provides VLAN assignments, switch configuration, and best practices based on Dell PowerFlex Appliance Network Planning Guide 4.x.

**Key principles:**
- Dedicated VLANs for each function
- Redundant connectivity (ABBA to switch pair)
- MTU 9216 on data networks
- LACP preferred for load balancing and resiliency

---

## 2. Network Architecture Options

### 2.1 Access-Aggregation

```
                    [Customer Core]
                           │
              ┌────────────┼────────────┐
              │            │            │
        [Aggregation] [Aggregation] [Management]
              │            │            │
        ┌─────┴─────┐ ┌─────┴─────┐     │
        │ Access A  │ │ Access B  │     │
        └─────┬─────┘ └─────┬─────┘     │
              │      MLAG   │           │
              └──────┬──────┘           │
                     │                  │
            [PowerFlex Nodes]    [Management VMs]
```

- Access switches connect to aggregation
- PowerFlex Manager configures node-facing ports
- Customer configures aggregation, management, uplinks

### 2.2 Leaf-Spine

```
              [Spine 1]    [Spine 2]
                   │    │    │    │
              ┌────┴────┴────┴────┴────┐
              │     [Border-Leaf]      │
              └───────────┬────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   [Leaf A]          [Leaf B]          [Management]
        │                 │
        └────────┬────────┘
                 │
        [PowerFlex Nodes]
```

- Leaf switches connect to spine and border-leaf
- PowerFlex Manager configures leaf node-facing ports

---

## 3. VLAN Reference

### 3.1 Standard VLAN Assignments

| VLAN | Name / Purpose | Layer | MTU | Used By |
|------|----------------|-------|-----|---------|
| **101** | Out-of-band (OOB) / iDRAC | L3 | 1500 | Node iDRAC, switch mgmt |
| **103** | vCenter HA | L2 | 1500 | vCenter Active/Passive/Witness |
| **105** | Hypervisor management | L2 | 1500 | ESXi mgmt, vCenter, jump server |
| **106** | vMotion | L2 | 9000 | VMware vMotion |
| **130** | PowerFlex management | L3 | 1500 | PFMP VMs, vCenter, CloudLink, SCG |
| **140** | PFMC 2.0 – pfmc-sds-mgmt | L3 | 1500 | SVM-1..5 SDS to PowerFlex Manager |
| **141** | PFMC 2.0 – pfmc-sds-data1 | L2 | 9216 | SDS-SDS, SDS-SDC, SDS-PFMP |
| **142** | PFMC 2.0 – pfmc-sds-data2 | L2 | 9216 | SDS-SDS, SDS-SDC |
| **143** | PFMC 2.0 – pfmc-sds-data3 | L2 | 9216 | SDS-SDS, SDS-SDC |
| **150** | flex-stor-mgmt | L2 | 1500 | Storage management |
| **151** | flex-data1 | L2 | 9216 | Primary storage data |
| **152** | flex-data2 | L2 | 9216 | Secondary storage data |
| **153** | flex-data3 | L2 | 9216 | Tertiary (optional) |
| **154** | flex-data4 | L2 | 9216 | Quaternary (optional) |
| **161** | Replication 1 | L2 | 9216 | Async replication |
| **162** | Replication 2 | L2 | 9216 | Async replication |
| **250** | NAS (untagged) | L2 | 9216 | PowerFlex File (NAS) |
| **251** | NAS VLAN 251 | L2 | 9216 | PowerFlex File |
| **252** | NAS VLAN 252 | L2 | 9216 | PowerFlex File |

### 3.2 Example Multi-VLAN Ranges (Customer-Specific)

| Default VLAN | Example Range | Purpose |
|--------------|---------------|---------|
| 101 | 210–224 | OOB / hardware management |
| 105 | 240–254 | Hypervisor management |
| 106 | 270–284 | vMotion |
| 130 | — | PowerFlex management (multi-VLAN) |
| 140 | 300–314 | PFMC 2.0 management |
| 141–143 | — | PFMC data |
| 150–154 | — | Storage data |
| 161–162 | — | Replication |

---

## 4. VLAN Setup by Node Type

### 4.1 PowerFlex Management Controller 2.0 (PFMC 2.0)

| Virtual Switch | Mode | Speed | LACP | VLANs | Load Balancing |
|----------------|------|-------|------|------|----------------|
| fe_dvSwitch (front-end) | Trunk | 25 Gb | Active | 105, 130, 140, 150 | LAG-Active Src/dest IP TCP/UDP |
| be_dvSwitch (back-end) | Trunk | 25 Gb | Active | 103, 141–143, 151–154 | — |
| oob_dvSwitch (OOB) | Access | 25 Gb | NA | 101 | NA |

### 4.2 PowerFlex Hyperconverged Nodes

| Virtual Switch | Mode | Speed | LACP | VLANs | Load Balancing |
|----------------|------|-------|------|------|----------------|
| cust_dvSwitch | Trunk | 25/100 Gb | ON | 105–106, 150 | Route based on IP hash |
| flex_dvSwitch | Trunk | 25/100 Gb | ON | 151–154, 161–162 | Route based on IP hash |

### 4.3 PowerFlex Compute-Only Nodes

| Virtual Switch | Mode | Speed | LACP | VLANs | Load Balancing |
|----------------|------|-------|------|------|----------------|
| cust_dvSwitch | Trunk | 25/100 Gb | ON | 105–106, 150 | Route based on IP hash |
| flex_dvSwitch | Trunk | 25/100 Gb | ON | 151–154 | Route based on IP hash |

### 4.4 PowerFlex Storage-Only Nodes

| Bond | Mode | Speed | LACP | VLANs | Load Balancing |
|------|------|-------|------|------|----------------|
| Bond0 | Trunk | 25/100 Gb | Active | 150, 151, 153, 161 | Mode 4 (LACP) |
| Bond1 | Trunk | 25/100 Gb | Active | 152, 154, 162 | Mode 4 (LACP) |

**Alternative (Individual trunks):**
- Per-NIC VLAN, Bond0/Bond1 with Mode 0 (RR), Mode 1 (Active Backup), or Mode 6 (ALB)
- For Mode 6: link up/down delay ≥ 100 ms

### 4.5 PowerFlex File Nodes

| Bond | VLANs |
|------|-------|
| Bond0 | 150, 151, 152, 153, 154 |
| Bond1 | 250 (untagged), 251, 252 |

---

## 5. Switch Requirements

### 5.1 Customer-Managed Switches

| Requirement | Specification |
|-------------|---------------|
| Latency | Sub-millisecond round-trip |
| Ports per node | Minimum 4×25 GbE SFP28 |
| Uplinks per switch | Minimum 2×100 GbE QSFP28 |
| MTU | 9000+ (9216 recommended for data) |
| VLANs | 802.1Q, multiple VLANs |
| Trunk | 802.1Q encapsulation |
| LACP | 802.3ad with LACP fallback |
| MLAG | vPC (Cisco) or VLT (Dell); 100G supported |
| Spanning tree | 802.1w Rapid STP; BPDU guard, Root guard |
| Port types | Port-fast/edge, Normal, Network |

### 5.2 PowerFlex Manager–Validated Switches

- Cisco Nexus (see Compatibility Matrix)
- Dell PowerSwitch (see Compatibility Matrix)
- Full automation: node-facing ports and node OS networking
- Partial: node OS only; manual switch config

### 5.3 Pre-Deployment Configuration

- Redundant connections to access switches (VLT/vPC)
- Management IPs configured
- VLT/vPC on interconnect
- **MTU=9216** on uplinks and server ports for PowerFlex data
- **LLDP** enabled on server-facing ports
- **SNMP** enabled, community `public`, trap destination = PowerFlex Manager
- Layer-2 connectivity of PowerFlex data and management VLANs to PFMC

---

## 6. LACP and Trunking

### 6.1 Recommended: Port Channel with LACP

- **Mode:** Active (LACP)
- **Switch:** vPC (Cisco) or VLT (Dell)
- **Channel group:** mode active
- **Node load balancing:** Route based on IP hash (VMware); LAG-Active Src/dest IP TCP/UDP

### 6.2 Alternative: Trunk (No LACP)

- Individual trunk interfaces per VLAN
- Load balancing: Originating virtual port, Physical NIC load, Source MAC hash
- No vPC/VLT required; per-NIC VLAN on storage nodes

### 6.3 Storage-Only: Bond Modes

| Mode | Name | vPC/VLT Required |
|------|------|-------------------|
| 4 | LACP | Yes |
| 0 | Round Robin | No |
| 1 | Active Backup | No |
| 6 | Adaptive LB | No (link up/down delay ≥ 100 ms) |

---

## 7. MTU and Jumbo Frames

| Network | MTU | Notes |
|---------|-----|-------|
| Management, OOB | 1500 | Standard |
| vMotion | 9000 | Recommended |
| PowerFlex data (141–154, 161–162) | **9216** | Required for performance |
| NAS data (250–252) | 9216 | For file services |

**Critical:** MTU 9216 must be consistent across entire data path (switch ports, uplinks, server NICs).

---

## 8. Multi-VLAN and Multi-Subnet

### 8.1 Supported

- All network types except: data, vSAN, NSX overlay
- Aggregation: SVI + trunk allowed VLAN list
- Multi-subnet: additional IP per subnet, first-hop redundancy as needed

### 8.2 Requirements

- Management VM on PFMC must have **VLAN 130** for multi-VLAN
- All PowerFlex controller nodes must have **VLAN 130**
- OOB management for multi-VLAN on all access/leaf switches

---

## 9. VMware NSX Ready

### 9.1 NSX Network Requirements

- Additional VLANs for NSX overlay
- See PowerFlex Appliance Network Planning Guide: "VLAN setup summary for VMware NSX ready nodes"
- Separate overlay and transport networks

### 9.2 VLAN Setup Summary for NSX

- Per NSX documentation and PowerFlex compatibility matrix
- Overlay network isolated from PowerFlex data

---

## 10. Best Practices Summary

| Area | Recommendation |
|------|----------------|
| **Switches** | Use two dedicated switches for PowerFlex; plan extra ports for growth |
| **Cabling** | ABBA: NIC1→Switch A, NIC2→Switch B per node |
| **MTU** | 9216 on all data networks end-to-end |
| **LACP** | Prefer LACP with vPC/VLT over standalone trunks |
| **VLANs** | Separate management, data, replication |
| **Management** | VLAN 130 for PowerFlex management in multi-VLAN |
| **OOB** | Dedicated OOB (101) for iDRAC and switch management |
| **Replication** | Dedicated VLANs 161–162; isolate from production data |
| **NAS** | VLANs 250–252 for PowerFlex File |
| **Expansion** | Ensure 100 GbE QSFP28 ports for uplinks when scaling |

---

## References

- Dell PowerFlex Appliance with PowerFlex 4.x Network Planning Guide
- Dell PowerFlex Appliance and PowerFlex Rack Compatibility Matrix
- Dell PowerFlex Networking Best Practices (h18390)
