# Dell PowerFlex 4.6 — Complete Installation, Design & Administration Guide

A comprehensive reference covering installation, architecture, automation, networking, storage design, and system administration for Dell PowerFlex 4.6.x.

---

## Table of Contents

1. [PowerFlex Components & Architecture](#1-powerflex-components--architecture)
2. [Installation Guide & Steps](#2-installation-guide--steps)
3. [PowerFlex Manager (PxFM) Installation](#3-powerflex-manager-pxfm-installation)
4. [vCenter Installation & Automation](#4-vcenter-installation--automation)
5. [Fault Sets & Protection Domains](#5-fault-sets--protection-domains)
6. [Storage Pool Best Practices](#6-storage-pool-best-practices)
7. [Fine vs Coarse Volume Granularity](#7-fine-vs-coarse-volume-granularity)
8. [Volume Mapping & Discovery](#8-volume-mapping--discovery)
9. [MDM Architecture & CLI Commands](#9-mdm-architecture--cli-commands)
10. [Network Design: VLAN, L2, L3](#10-network-design-vlan-l2-l3)
11. [Cabling, ABBA, CDP, LLDP, VDS](#11-cabling-abba-cdp-lldp-vds)
12. [RAID Architecture & Storage Overhead](#12-raid-architecture--storage-overhead)
13. [Replication](#13-replication)
14. [PowerFlex Lifecycle Management](#14-powerflex-lifecycle-management)
15. [Backup & System Administration](#15-backup--system-administration)
16. [Architecture Diagrams](#16-architecture-diagrams)

---

## 1. PowerFlex Components & Architecture

### Core Components

| Component | Acronym | Role |
|-----------|---------|------|
| **Metadata Manager** | MDM | HA cluster outside data path; supervises cluster health, coordinates rebalancing, rebuild/reprotect |
| **Storage Data Server** | SDS | Software on nodes contributing disks; abstracts local storage, maintains pools, presents volumes to SDCs |
| **Storage Data Client** | SDC | Kernel driver providing block device access; connects to every SDS in pool; translates SCSI ↔ PowerFlex protocol |
| **Light Installation Agent** | LIA | Enables automated maintenance/upgrades; deployed on nodes; uses token auth |

### Logical Hierarchy

```
System (PowerFlex Cluster)
└── Protection Domain (PD)
    └── Fault Set (optional)
        └── SDS (Storage Data Server)
    └── Storage Pool (SP)
        └── Device (physical disk)
        └── Volume
```

### L2 vs L3 (Two-Layer Deployment)

- **L2 (Two-layer):** SDS nodes separate from SDC/compute nodes; typical for large scale.
- **L3 (Three-layer):** SDS, SDC, and MDM on separate node types.
- **Hyperconverged:** SDS + SDC on same nodes.

---

## 2. Installation Guide & Steps

### Prerequisites

- NTP synchronized across all nodes
- DNS resolution for management and data networks
- Network connectivity (management, data, replication if used)
- Compatible OS: RHEL 8.x, SLES 15.x, VMware ESXi 7.x/8.x
- Docker (SLES) or Podman (RHEL) for installer

### High-Level Installation Workflow

1. **Prepare environment**
   - Download PowerFlex Manager bundle from Dell Support
   - Prepare CSV topology file (for automated deployment)
   - Configure NTP, DNS, networking

2. **Deploy PowerFlex Management Platform**
   - Deploy installer VM or use co-resident on primary MDM node
   - Deploy PowerFlex Manager (PxFM) cluster

3. **Deploy PowerFlex storage**
   - Prepare storage nodes (OS, drivers, networking)
   - Deploy using CSV topology or PowerFlex Manager UI

4. **Post-installation**
   - Create protection domains, storage pools
   - Add SDSs, create volumes, map to hosts

### CSV Topology File (Automation)

The CSV topology file drives automated cluster deployment. Key columns:

- Node identity (hostname, IP)
- Component type (MDM, SDS, SDC, TB)
- Protection domain, fault set
- Network interfaces and IPs
- Device paths for SDS

**Reference:** [Prepare the CSV topology file](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software-install-upgrade-46x/prepare-the-csv-topology-file)

---

## 3. PowerFlex Manager (PxFM) Installation

### PxFM Deployment Options

1. **VMware vSphere** — OVA/OVF deployment
2. **Co-resident** — On boot device of PowerFlex storage nodes
3. **Linux management nodes** — Dedicated VMs or bare metal

### Steps to Deploy PxFM Installer

```bash
# On installer node (will become primary MDM or external VM)
# SLES:
zypper install docker python3-docker-compose

# RHEL:
yum install podman-docker podman-plugins

# Create directory and extract bundle
sudo mkdir -p /opt/dell/pfmp
cd /opt/dell/pfmp
sudo tar -xvf PowerFlex-Manager-bundle-4.6.x-Build-xxx.tgz
```

### PxFM Automation

- PowerFlex Manager provides REST API and UI for automation
- vCenter integration enables automated host discovery, SDC deployment, datastore provisioning
- CSV topology file enables fully automated cluster deployment without manual node-by-node setup

---

## 4. vCenter Installation & Automation

### vCenter Integration

- PowerFlex Manager integrates with VMware vCenter for:
  - Host discovery
  - SDC deployment on ESXi
  - Datastore creation and VM provisioning
  - Network configuration (VDS port groups)

### Automation Capabilities

- Add hosts via vCenter
- Deploy SDC on ESXi hosts automatically
- Create volumes and map to hosts
- Configure NVMe-oF or SCSI for block access

---

## 5. Fault Sets & Protection Domains

### Protection Domain (PD)

- Logical group of SDSs that protect each other
- Each SDS belongs to exactly one PD
- Data copies are never placed on the same fault unit (SDS)

### Fault Set

- Subset of SDSs within a PD
- Used to align with physical topology (rack, row, datacenter)
- Ensures replicas are spread across fault sets for resilience

### Best Practices

- Use fault sets when you have multiple racks/rows
- Map fault sets to physical failure domains
- Minimum 3 fault sets recommended for full protection
- Avoid single fault set spanning entire cluster

### CSV Examples (Fault Sets)

- Four data networks with fault sets
- Two front-end + two back-end with fault sets

---

## 6. Storage Pool Best Practices

### Storage Pool Design

- Storage pool = set of devices within a PD
- Volumes are striped across all devices in the pool

### Best Practices

1. **SDS addition to storage pool**
   - Add SDSs gradually; allow rebalance to complete
   - Ensure devices are prepared (no partitions, correct block size)

2. **Device selection**
   - Use same device type/size within a pool when possible
   - Avoid mixing SSD and HDD in same pool unless intentional

3. **Capacity planning**
   - Account for RF (replication factor) overhead
   - Fine granularity has additional metadata overhead (NVDIMM required)

---

## 7. Fine vs Coarse Volume Granularity

| Aspect | Fine Granularity (FG) | Coarse/Medium Granularity (MG) |
|--------|------------------------|--------------------------------|
| **Use case** | Space efficiency, heavy snapshots | High performance |
| **NVDIMM** | Required | Not required |
| **Compression** | Supported (inline) | Not supported |
| **Metadata** | Higher (persistent memory + RAM) | Lower |
| **Chunk size** | Smaller | Larger |

### Fine Granularity Requirements

- NVDIMM or SDPM as DAX (acceleration) device
- Calculate persistent memory and RAM for SDS: see Dell Install Guide

---

## 8. Volume Mapping & Discovery

### Volume Mapping (SCLI)

```bash
# Map volume to SDC
scli --map_volume_to_sdc --volume_name <name> --sdc_name <sdc_name> --allow_multiple_mappings

# Unmap
scli --unmap_volume_from_sdc --volume_name <name> --sdc_name <sdc_name>

# Query mappings
scli --query_volume --volume_name <name>
```

### Discovery

- SDCs discover MDM via management IP
- Persistent discovery: configure `--persistent_dsc_port` and network sets for NVMe-oF

### Volume Best Practices

- Use host groups for consistent mapping across clusters
- Unmap before SDC removal
- Use `--allow_multiple_mappings` for multi-path scenarios

---

## 9. MDM Architecture & CLI Commands

### MDM Cluster

- **Primary MDM** — Active metadata manager
- **Secondary MDM** — Standby
- **Tie-Breaker (TB)** — Quorum; can be standalone or collocated

### SCLI Login

```bash
scli --login --management_system_ip <MDM_IP> --username admin
```

### Key SCLI Commands

| Task | Command |
|------|---------|
| Cluster topology | `scli --query_cluster` |
| Switch primary | `scli --switch_mdm_ownership --new_primary_mdm_ip <IP>` |
| All SDSs | `scli --query_all_sds` |
| Specific SDS | `scli --query_sds --sds_name <name>` |
| All SDCs | `scli --query_all_sdc` |
| Storage pool | `scli --query_storage_pool --protection_domain_name <PD> --storage_pool_name <SP>` |
| SDS devices | `scli --query_sds_device_info --sds_name <name> --all_devices` |
| Volumes | `scli --query_all_volumes` |
| Add SDS | `scli --add_sds --sds_name <name> --sds_ip <IP> ...` |
| Add device to SDS | `scli --add_sds_device --sds_name <name> --device_path <path> --storage_pool_name <SP>` |

---

## 10. Network Design: VLAN, L2, L3

### Typical VLAN Layout

| VLAN | Purpose | Example |
|------|---------|---------|
| Out-of-band | iDRAC/console | 101 |
| Management | PowerFlex Manager, MDM | 130 |
| Data1 | Server-side (SDS) | 151 |
| Data2 | Client-side (SDC) or redundancy | 152 |
| Replication | Async replication | 160 |
| NAS (if File) | NAS management/data | 170+ |

### L2 vs L3

- **L2:** Direct VLAN access; typical for data path
- **L3:** Routed; possible with RPQ for simplified designs
- PowerFlex IP multi-pathing reduces need for LACP in many cases

### Best Practices

- Dedicate interfaces for management vs data
- Use jumbo frames (MTU 9000) on data networks
- Separate front-end (client) and back-end (server) traffic when possible

### Reference

- [Dell EMC PowerFlex Networking Best Practices](https://www.delltechnologies.com/asset/en-us/products/storage/industry-market/h18390-dell-emc-powerflex-networking-best-practices-wp.pdf)
- [PowerFlex 4.x Network Design Options](https://powerflex.me/2024/09/17/powerflex-4-x-exploring-networking-options/)

---

## 11. Cabling, ABBA, CDP, LLDP, VDS

### Cabling (ABBA)

- **ABBA** refers to redundant cabling: each node connects to Switch A and Switch B (A-B, B-A)
- Ensures no single point of failure; each NIC to different switch
- Standard for PowerFlex Rack and Appliance deployments

### CDP / LLDP

- **CDP (Cisco Discovery Protocol)** and **LLDP (Link Layer Discovery Protocol)** used for switch/port discovery
- Enable on VDS for topology visibility: `Set-VDSwitch -LinkDiscoveryProtocol LLDP` or `CDP`
- Operation: Advertise, Listen, Both, or Disabled

### VDS Port Group Settings

- Create port groups per VLAN (Management, Data1, Data2, Replication)
- Use VLAN tagging (trunk) or access per port group
- Enable discovery (CDP/LLDP) for troubleshooting
- Configure teaming/failover per port group

---

## 12. RAID Architecture & Storage Overhead

### PowerFlex Data Layout

- PowerFlex uses **mesh-mirror** (replication), not traditional RAID
- Replication Factor (RF) 2 or 3
- Data striped across devices; copies on different fault units

### Storage Overhead

| RF | Usable % | Overhead |
|----|----------|----------|
| 2 | 50% | 2x |
| 3 | 33% | 3x |

### Fine Granularity Overhead

- Additional metadata for compression and snapshots
- Requires NVDIMM; RAM and persistent memory sizing per SDS

---

## 13. Replication

### Async Replication

- Replication network: dedicated VLAN
- Direction and mapping configured per volume or consistency group
- RPO/RTO depend on replication lag and failover procedure

### Replication Topology

- Primary ↔ Secondary site
- Replication traffic isolated on dedicated network

---

## 14. PowerFlex Lifecycle Management

### Lifecycle Mode

- Components can be placed in maintenance mode for upgrades
- SDS maintenance: drain data, perform work, exit maintenance

### Upgrade Paths

- 3.6.x → 4.5.x (core) + 4.6.x (management)
- 4.5.x → 4.5.x (core) + 4.6.x (management)
- PowerFlex Manager upgrade separate from core

### LIA for Automation

- LIA enables automated maintenance and upgrades
- Config: `/opt/emc/scaleio/lia/cfg/conf.txt` (Linux)
- Token change: update `lia_token`, restart LIA service

---

## 15. Backup & System Administration

### PowerFlex Manager Backup

- Backup to NFS share (local or remote)
- Includes configuration, not user data
- Restore recovers PxFM appliance state

### Volume Snapshots

- Create snapshots via PowerFlex Manager or SCLI
- Use for consistency groups when needed

### System Administration Tasks

- Monitor cluster health (MDM, SDS, SDC)
- Add/remove SDS, devices, volumes
- Rebalance and rebuild monitoring
- SupportAssist for proactive support

### Reference

- [PowerFlex Manager Backup and Restore](https://www.dell.com/support/kbdoc/en-us/000184942)
- [PowerFlex 4.6.x Administration Guide](https://www.dell.com/support/manuals/en-us/scaleio/powerflex_sw_admin_guide_46/)

---

## 16. Architecture Diagrams

### High-Level PowerFlex Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           PowerFlex Manager (PxFM)       │
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
   │  SDS   │      │  SDS   │      │  SDS   │   ← Protection Domain
   │ Node 1 │      │ Node 2 │      │ Node 3 │     Storage Pool
   └───┬────┘      └───┬────┘      └───┬────┘
       │               │               │
       │    Data Path (Front-end/Back-end)
       │               │               │
       └───────────────┼───────────────┘
                       │
               ┌───────┴───────┐
               ▼               ▼
          ┌────────┐      ┌────────┐
          │  SDC   │      │  SDC   │   ← Compute / ESXi
          │ Host 1 │      │ Host 2 │
          └────────┘      └────────┘
```

### Network Topology (Simplified)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Management VLAN (130)                        │
│  PxFM ◄──► MDM ◄──► SDS ◄──► SDC                                │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  Data VLANs (151, 152)                           │
│  SDC ◄═══════════════ Data Path ═══════════════► SDS             │
│       (Client/Server traffic, jumbo frames)                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  Replication VLAN (160)                          │
│  SDS ◄══════════ Replication ══════════► Remote Site SDS        │
└─────────────────────────────────────────────────────────────────┘
```

### ABBA Cabling (Redundant)

```
        Switch A                    Switch B
           │                            │
     ┌─────┴─────┐                ┌─────┴─────┐
     │  Port 1   │                │  Port 1   │
     └─────┬─────┘                └─────┬─────┘
           │                            │
           │    Node 1 NIC1      Node 1 NIC2
           │         \                /
           │          \              /
           │           ▼            ▼
           │         ┌──────────────┐
           │         │   Node 1     │
           │         │  (SDS/SDC)   │
           │         └──────────────┘
           │
     A-B: NIC1→SwitchA, NIC2→SwitchB
     B-A: Ensures no single switch failure
```

---

## Official Dell Documentation Links

| Topic | URL |
|-------|-----|
| PowerFlex 4.6.x Install & Upgrade Guide | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software-install-upgrade-46x/) |
| PowerFlex 4.6.x Administration Guide | [Dell Support](https://www.dell.com/support/manuals/en-us/scaleio/powerflex_sw_admin_guide_46/) |
| PowerFlex Components (Architecture) | [Dell Support](https://www.dell.com/support/manuals/en-us/powerflex-appliance-r650/flex_app_archg_4x/powerflex-components) |
| CSV Topology File | [Prepare CSV](https://www.dell.com/support/manuals/en-us/scaleio/pfx_p_software-install-upgrade-46x/prepare-the-csv-topology-file) |
| SCLI Login (4.x) | [KBA 000350923](https://www.dell.com/support/kbdoc/en-lc/000350923/4-x-how-to-login-to-scli-on-powerflex) |
| Networking Best Practices | [h18390 PDF](https://www.delltechnologies.com/asset/en-us/products/storage/industry-market/h18390-dell-emc-powerflex-networking-best-practices-wp.pdf) |

---

*Document version: 1.0 | Based on PowerFlex 4.5.x/4.6.x documentation | For latest details, refer to Dell official manuals.*
