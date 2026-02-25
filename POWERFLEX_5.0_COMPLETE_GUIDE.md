# Dell PowerFlex 5.0 — Complete Installation, Design & Administration Guide

A comprehensive reference covering installation, architecture, automation, networking, storage design, and system administration for Dell PowerFlex 5.0.x. PowerFlex 5.0 introduces a new generation with erasure coding (SAE), PDS/DGWT architecture, and enhanced capabilities.

---

## Table of Contents

1. [PowerFlex 5.0 Components & Architecture](#1-powerflex-50-components--architecture)
2. [What's New in 5.0 vs 4.x](#2-whats-new-in-50-vs-4x)
3. [Installation Guide & Steps](#3-installation-guide--steps)
4. [PowerFlex Manager (PxFM) 5.0 Installation](#4-powerflex-manager-pxfm-50-installation)
5. [vCenter Installation & Automation](#5-vcenter-installation--automation)
6. [Protection Domains, Device Groups & Storage Pools](#6-protection-domains-device-groups--storage-pools)
7. [Erasure Coding (SAE) & Storage Overhead](#7-erasure-coding-sae--storage-overhead)
8. [Volume Mapping, Thin Clones & Discovery](#8-volume-mapping-thin-clones--discovery)
9. [MDM Architecture & CLI Commands](#9-mdm-architecture--cli-commands)
10. [Network Design: VLAN, L2, L3](#10-network-design-vlan-l2-l3)
11. [Cabling, ABBA, CDP, LLDP, VDS](#11-cabling-abba-cdp-lldp-vds)
12. [Product Limits & System Requirements](#12-product-limits--system-requirements)
13. [Replication & Lifecycle Management](#13-replication--lifecycle-management)
14. [Backup & System Administration](#14-backup--system-administration)
15. [Architecture Diagrams](#15-architecture-diagrams)
16. [Official Dell Documentation Links](#16-official-dell-documentation-links)

---

## 1. PowerFlex 5.0 Components & Architecture

### Core Components (5.0 Architecture)

| Component | Acronym | Role |
|-----------|---------|------|
| **Metadata Manager** | MDM | HA cluster outside data path; supervises cluster health, coordinates rebalancing, rebuild |
| **PowerFlex Data Server** | PDS | Software storage processing service; manages erasure coding protection schemes and storage pool capacity; presents volumes to SDCs |
| **Device Gateway Target** | DGWT | Software service facilitating I/O to hardware devices (local and over network); communicates with PDSs and other DGWTs |
| **Storage Data Client** | SDC | Kernel driver providing block device access; connects to PDSs; translates SCSI ↔ PowerFlex protocol |
| **Storage Data Target** | SDT | NVMe-oF target for NVMe host connections |
| **Light Installation Agent** | LIA | Enables automated maintenance/upgrades; deployed on nodes |

### Storage Node (5.0)

- **Storage Node** = PDS + DGWT on a single node
- Each Storage Node belongs to exactly one Protection Domain
- Minimum **5 devices** per Storage Node (for erasure coding)
- Minimum **1.6 TB** capacity per node

### Logical Hierarchy (5.0)

```
System (PowerFlex Cluster)
└── Protection Domain (PD)
    └── Storage Node (PDS + DGWT)
        └── Device Group (storage devices + PMEM devices)
    └── Device Group
        └── Storage Pool
            └── Volume / Snapshot / Thin Clone
```

### Key Architectural Change from 4.x

- **4.x:** SDS (Storage Data Server) managed devices and mesh-mirror replication
- **5.0:** PDS manages erasure coding; DGWT manages device I/O; SAE (Scalable Availability Engine) provides distributed erasure coding

---

## 2. What's New in 5.0 vs 4.x

| Feature | 4.x | 5.0 |
|---------|-----|-----|
| **Data protection** | Mesh-mirror (RF 2/3) | Erasure coding (8+2, 2+2) |
| **Storage component** | SDS | PDS + DGWT |
| **Capacity efficiency** | 33–50% (RF 2/3) | Up to 80% (8+2) |
| **Availability** | High | Up to 10x 9s |
| **Compression** | Required NVDIMM (FG) | Enabled by default, no PMEM required |
| **Thin clones** | No | Yes |
| **Device groups** | No | Yes (storage + PMEM) |
| **Storage node** | N/A | Scale storage without compute |
| **Over-provisioning** | Limited | Unlimited by default; configurable control |
| **Volume granularity** | Coarser | 1 GB minimum and increment |
| **Write cache** | — | BBWC with persistent memory support |
| **Single component upgrade** | Full RCM/IC | Individual component upgrades |

---

## 3. Installation Guide & Steps

### Prerequisites

- NTP synchronized across all nodes
- DNS resolution for management and data networks
- Network connectivity (management, data, replication if used)
- Supported OS: RHEL 8.x/9.x, SLES 15.x, VMware ESXi 8.x (see [Dell PowerFlex Simple Support Matrix](https://elabnavigator.dell.com/eln/modernHomeSSM))
- Docker (SLES) or Podman (RHEL) for installer

### High-Level Installation Workflow

1. **Prepare environment**
   - Download PowerFlex Manager 5.0 bundle from Dell Support
   - Prepare CSV topology file (for automated deployment)
   - Configure NTP, DNS, networking

2. **Deploy PowerFlex Management Platform**
   - Deploy installer VM or use integrated management environment
   - Deploy PowerFlex Manager (PxFM) 5.0 cluster

3. **Deploy PowerFlex storage**
   - Prepare storage nodes (OS, drivers, networking)
   - Minimum 5 storage nodes for 2+2; 11 for 8+2
   - Deploy using CSV topology or PowerFlex Manager UI

4. **Post-installation**
   - Create protection domains, device groups, storage pools
   - Add storage nodes, create volumes, map to hosts

### CSV Topology File (Automation)

- Drives automated cluster deployment
- Key columns: node identity, component type (MDM, PDS, DGWT, SDC, TB), protection domain, network interfaces, device paths

---

## 4. PowerFlex Manager (PxFM) 5.0 Installation

### PxFM 5.0 Deployment Options

1. **VMware vSphere** — OVA/OVF deployment
2. **Integrated PowerFlex Management (non-KVM)** — Co-resident or dedicated
3. **Linux management nodes** — Dedicated VMs or bare metal

### PxFM 5.0 Usability Enhancements

- Enhanced event reporting and actionable alerts with remediation steps
- More detailed dashboard metrics
- Improved topology navigation
- Restructured Block tab: storage entities under one storage system tab

### PxFM Automation

- REST API and UI for automation
- vCenter integration for host discovery, SDC deployment, datastore provisioning
- CSV topology for fully automated cluster deployment

---

## 5. vCenter Installation & Automation

### vCenter Integration

- PowerFlex Manager 5.0 integrates with VMware vCenter for:
  - Host discovery
  - SDC deployment on ESXi
  - Datastore creation and VM provisioning
  - Network configuration (VDS port groups)
  - Certificate management for vCenter

### Automation Capabilities

- Add hosts via vCenter
- Deploy SDC on ESXi hosts automatically
- Create volumes and map to hosts
- Configure NVMe-oF or SCSI for block access

---

## 6. Protection Domains, Device Groups & Storage Pools

### Protection Domain (PD)

- Logical group of Storage Nodes that protect each other
- Each Storage Node belongs to exactly one PD
- **Minimum Storage Nodes:** 5 (2+2) or 11 (8+2)
- **Maximum capacity per PD:** 16 PB (8+2) or 4 PB (2+2)

### Device Group (New in 5.0)

- Collection of devices within a PD's Storage Nodes
- Two types: **storage devices** and **persistent memory devices** (SDPM/NVDIMM)
- Maximum: 1 storage device group + 1 PMEM device group per PD
- Provides underlying capacity for Storage Pool

### Storage Pool

- Volume container leveraging raw capacity of a Device Group
- Volumes distributed over all devices in the pool
- **1 storage pool per protection domain** (PowerFlex software)
- Minimum size: 1.6 TB × number of storage nodes in PD

### Fault Sets

- Fault set concepts apply for physical topology alignment (rack, row)
- Map fault sets to failure domains for resilience

---

## 7. Erasure Coding (SAE) & Storage Overhead

### Scalable Availability Engine (SAE)

- Distributed erasure coding for primary block storage
- Up to **80% capacity efficiency**
- Up to **10x 9s availability**
- Maintains scalable performance

### Protection Schemes

| Scheme | Data | Parity | Total | Min Storage Nodes | Max PD Capacity | Overhead |
|--------|------|--------|-------|-------------------|-----------------|----------|
| **8+2** | 8 | 2 | 10 | 11 | 16 PB | ~25% |
| **2+2** | 2 | 2 | 4 | 5 | 4 PB | ~100% |

### Storage Overhead

- **8+2:** 2/8 = 25% overhead → ~80% usable
- **2+2:** 2/2 = 100% overhead → ~50% usable (similar to RF2)

### How Erasure Coding Works

- Data split into fragments (data strips)
- Parity fragments calculated and stored on separate nodes
- If any fragment is lost, data reconstructed from remaining fragments

---

## 8. Volume Mapping, Thin Clones & Discovery

### Volume Mapping (SCLI)

```bash
# Map volume to SDC
scli --map_volume_to_sdc --volume_name <name> --sdc_name <sdc_name> --allow_multiple_mappings

# Unmap
scli --unmap_volume_from_sdc --volume_name <name> --sdc_name <sdc_name>

# Query mappings
scli --query_volume --volume_name <name>
```

### Thin Clones (New in 5.0)

- Writable copies of snapshots without consuming full storage
- Share data blocks with parent snapshot
- Remain unaffected if parent snapshot is deleted
- Managed via PowerFlex Manager

### Allocation Granularity

- **Minimum volume size:** 1 GB
- **Increment:** 1 GB (e.g., 10 GB → 11 GB)

### Discovery

- SDCs discover MDM via management IP
- NVMe-oF: persistent discovery, network sets, SDT configuration

---

## 9. MDM Architecture & CLI Commands

### MDM Cluster

- **Primary MDM** — Active metadata manager
- **Secondary MDM** — Standby
- **Tie-Breaker (TB)** — Quorum; standalone or collocated

### SCLI Login

```bash
scli --login --management_system_ip <MDM_IP> --username admin
```

### Key SCLI Commands

| Task | Command |
|------|---------|
| System limits | `scli --query_system_limits` |
| Cluster topology | `scli --query_cluster` |
| Switch primary | `scli --switch_mdm_ownership --new_primary_mdm_ip <IP>` |
| All PDSs/Storage Nodes | `scli --query_all_sds` (or PDS equivalents) |
| All SDCs | `scli --query_all_sdc` |
| Storage pool | `scli --query_storage_pool --protection_domain_name <PD> --storage_pool_name <SP>` |
| Volumes | `scli --query_all_volumes` |

---

## 10. Network Design: VLAN, L2, L3

### Typical VLAN Layout

| VLAN | Purpose | Example |
|------|---------|---------|
| Out-of-band | iDRAC/console | 101 |
| Management | PowerFlex Manager, MDM | 130 |
| Data1 | Server-side (PDS/DGWT) | 151 |
| Data2 | Client-side (SDC) or redundancy | 152 |
| Replication | Async replication | 160 |

### Best Practices

- Dedicate interfaces for management vs data
- Use jumbo frames (MTU 9000) on data networks
- Separate front-end (client) and back-end (server) traffic when possible

### Reference

- [Dell EMC PowerFlex Networking Best Practices](https://www.delltechnologies.com/asset/en-us/products/storage/industry-market/h18390-dell-emc-powerflex-networking-best-practices-wp.pdf)

---

## 11. Cabling, ABBA, CDP, LLDP, VDS

### Cabling (ABBA)

- Each node connects to Switch A and Switch B (A-B, B-A)
- Ensures no single point of failure
- Standard for PowerFlex Rack and Appliance

### CDP / LLDP

- Enable on VDS: `Set-VDSwitch -LinkDiscoveryProtocol LLDP` or `CDP`
- Operation: Advertise, Listen, Both, or Disabled

### VDS Port Group Settings

- Create port groups per VLAN
- Enable discovery (CDP/LLDP) for troubleshooting
- Configure teaming/failover per port group

---

## 12. Product Limits & System Requirements

### Key PowerFlex 5.0 Limits

| Capability | PowerFlex Software | PowerFlex Appliance |
|------------|-------------------|---------------------|
| Max system raw capacity | 8+2: 16 PB; 2+2: 4 PB | Same |
| Min storage nodes per PD | 8+2: 11; 2+2: 5 | Same |
| Max storage nodes | 128 | 128 |
| Max devices per node | 32 | 24 |
| Max devices per PD | 4096 | 768 |
| Device size | 480 GB – 16 TB | 1.92 TB – 16 TB |
| Volume size | 1 GB – 1 PB | Same |
| Max volumes/snapshots | 128,000 | Same |
| Max SDCs | 2,000 | 2,000 |
| Max NVMe hosts | 1,000 + 1,000 SDCs | Same |
| Max protection domains | 8 | 8 |
| Storage pools per PD | 1 | 1 |
| Over-provisioning factor | 100× storage pool | Same |

### NVMe over TCP Limits

- Max SDTs per system: 128
- Max NVMe hosts: 1024
- Max volumes per NVMe host: 1024
- Max paths per volume: 8

---

## 13. Replication & Lifecycle Management

### Lifecycle Mode

- Components can be placed in maintenance mode for upgrades
- Storage Node maintenance: drain data, perform work, exit maintenance

### Single Component Upgrade (New in 5.0)

- Upgrade individual components (e.g., security patches) without full RCM/IC upgrade

### LIA for Automation

- LIA enables automated maintenance and upgrades
- Config: `/opt/emc/scaleio/lia/cfg/conf.txt` (Linux)
- Token change: update `lia_token`, restart LIA service

### Replication

- Replication network: dedicated VLAN
- Direction and mapping per volume or consistency group

---

## 14. Backup & System Administration

### PowerFlex Manager Backup

- Backup to NFS share (local or remote)
- Includes configuration, not user data
- Restore recovers PxFM appliance state

### Volume Snapshots & Thin Clones

- Create snapshots via PowerFlex Manager or SCLI
- Thin clones: writable snapshot copies with shared blocks
- Secure snapshots supported

### System Administration Tasks

- Monitor cluster health (MDM, PDS, DGWT, SDC)
- Add/remove storage nodes, devices, volumes
- Rebalance and rebuild monitoring
- SupportAssist for proactive support

---

## 15. Architecture Diagrams

### High-Level PowerFlex 5.0 Architecture

```
                    ┌─────────────────────────────────────────┐
                    │       PowerFlex Manager (PxFM) 5.0      │
                    │         vCenter Integration / REST       │
                    └───────────────────┬─────────────────────┘
                                        │
        ┌───────────────────────────────┼───────────────────────────────┐
        │                               │                               │
        ▼                               ▼                               ▼
┌───────────────┐             ┌─────────────────┐             ┌───────────────┐
│  Primary MDM  │◄──────────►│ Secondary MDM   │             │ Tie-Breaker   │
└───────┬───────┘             └────────┬────────┘             └───────────────┘
        │                              │
        │     Management / Metadata    │
        └──────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   ┌────────┐      ┌────────┐      ┌────────┐
   │  PDS   │      │  PDS   │      │  PDS   │   ← Storage Nodes
   │ +DGWT  │      │ +DGWT  │      │ +DGWT  │     Protection Domain
   │ Node 1 │      │ Node 2 │      │ Node 3 │     Device Group → Storage Pool
   └───┬────┘      └───┬────┘      └───┬────┘
       │               │               │
       │    Data Path (Erasure Coded via SAE)
       │               │               │
       └───────────────┼───────────────┘
                       │
               ┌───────┴───────┐
               ▼               ▼
          ┌────────┐      ┌────────┐
          │  SDC   │      │ NVMe   │   ← Compute / ESXi
          │ Host 1 │      │ Host 2 │
          └────────┘      └────────┘
```

### 5.0 Data Flow (PDS + DGWT)

```
    SDC                    PDS                      DGWT                    Device
     │                      │                         │                        │
     │  I/O request         │                         │                        │
     │────────────────────►│                         │                        │
     │                      │  Erasure encode        │                        │
     │                      │  / decode              │                        │
     │                      │───────────────────────►│                        │
     │                      │                         │  Block I/O             │
     │                      │                         │──────────────────────►│
     │                      │                         │                        │
     │  Response            │  Response               │  Response              │
     │◄────────────────────│◄────────────────────────│◄───────────────────────│
```

### Erasure Coding (8+2)

```
   Data Block
       │
       ▼
   ┌─────────────────────────────────────┐
   │  Split into 8 data fragments         │
   │  + 2 parity fragments (erasure code) │
   └─────────────────────────────────────┘
       │
       ▼
   Distributed across 10+ Storage Nodes
   (min 11 for 8+2)
   ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ... ┌───┐
   │ D │ │ D │ │ D │ │ D │ │ P │     │ P │
   └───┘ └───┘ └───┘ └───┘ └───┘     └───┘
   Node1 Node2 Node3 Node4 Node5     Node10
```

---

## 16. Official Dell Documentation Links

| Topic | URL |
|-------|-----|
| PowerFlex 5.0.x Install Guide | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software_install_guide_5x/) |
| PowerFlex 5.0.x Technical Overview | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/flex-software-to-5x/powerflex-overview) |
| PowerFlex Manager 5.0.x User Guide | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/pfx_mgr_user_guide_5x/) |
| New Features in 5.0 | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/flex-software-to-5x/new-features-in-50) |
| Product Limits (5.0) | [KBA 000379528](https://www.dell.com/support/kbdoc/en-us/000379528/powerflex-5-0-x-product-min-and-max-limits) |
| PowerFlex Software Documentation | [KBA 000314729](https://www.dell.com/support/kbdoc/en-us/000314729/powerflex-software-documentation) |
| PowerFlex 5.0 Specification Sheet | [Dell Technologies PDF](https://www.delltechnologies.com/asset/en-us/products/storage/technical-support/powerflex-5-0-specification-sheet.pdf) |

---

*Document version: 1.0 | Based on PowerFlex 5.0.x documentation | For latest details, refer to Dell official manuals.*
