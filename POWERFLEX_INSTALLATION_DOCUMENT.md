# Dell PowerFlex Installation Document
## PowerFlex 4.x / 5.x Rack and Appliance

**Version:** 1.0  
**Based on:** Dell PowerFlex Rack Upgrade Guide 4.x, Field Implementation Guides 4x/5x, PowerFlex 4.5.x Technical Overview  

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Requirements](#2-system-requirements)
3. [Prerequisites](#3-prerequisites)
4. [Installation Components](#4-installation-components)
5. [Software and Firmware](#5-software-and-firmware)
6. [PowerFlex Manager Deployment](#6-powerflex-manager-deployment)
7. [Management Cluster Types](#7-management-cluster-types)
8. [License Configuration](#8-license-configuration)
9. [Post-Installation Validation](#9-post-installation-validation)

---

## 1. Introduction

This document describes the installation requirements and components for Dell PowerFlex Rack and PowerFlex Appliance deployments. PowerFlex is a software-defined storage platform that can be deployed as:

- **PowerFlex Rack** — Rack-scale system with integrated management (PFMC KVM or ESXi)
- **PowerFlex Appliance** — Preconfigured storage appliance with bring-your-own networking

Supported consumption models:
- **Hyperconverged** — SDS + SDC on same nodes
- **Two-layer** — SDS nodes separate from SDC/compute nodes
- **Storage-only** — SDS nodes only (consumption from external hosts)
- **Compute-only** — SDC nodes only (consumption of remote storage)
- **PowerFlex File** — NAS (NFS/CIFS) nodes

---

## 2. System Requirements

### 2.1 Hardware Requirements

| Component | Requirements |
|-----------|--------------|
| **Compute/storage** | Minimum 4 nodes for storage or compute; 6 recommended |
| **Servers** | Dell PowerEdge R650, R750, R6525, R7525, R660, R760, R6625, R7625, R640, R740xd, R840 |
| **Network** | Two PowerFlex Manager–supported access switches or enterprise switches meeting specs |
| **Management** | 1, 3, or 5 management nodes (ESXi or KVM) for PFMP |
| **Cabling** | 4×10 GbE / 4×25 GbE / 4×100 GbE per node; 2×100 GbE uplinks per access switch |

### 2.2 Switch Requirements

- **Dell PowerSwitch** or **Cisco Nexus** (see Compatibility Matrix)
- **MTU 9216** on data networks
- **LACP** (802.3ad) or trunk support
- **vPC/VLT** for MLAG across switch pair

### 2.3 Operating System Requirements

| Platform | Supported Versions |
|----------|--------------------|
| VMware ESXi | 7.x, 8.x |
| RHEL | 8.x |
| SLES | 15.x |
| PowerFlex MVM OS | Customized SLES maintained by Dell Technologies |

### 2.4 PowerFlex Cluster Component Requirements

- **MDM cluster** — 3 or 5 nodes (1 Primary, 1+ Secondary, 1 Tie-Breaker)
- **Storage pools** — At least 2 nodes per protection domain for RF2
- **NTP** — Synchronized across all nodes
- **DNS** — Resolution for management and data networks

---

## 3. Prerequisites

### 3.1 Pre-Installation Checklist

- [ ] Servers installed and cabled per ABBA topology
- [ ] Switches configured (VLANs, port channels, MTU)
- [ ] NTP servers reachable
- [ ] DNS configured for PowerFlex nodes and management
- [ ] Customer services available: Active Directory (optional), DNS, NTP
- [ ] Physical infrastructure: power, cooling, core network access
- [ ] RCM bundle downloaded from Dell Support site
- [ ] Passwords retrieved from RCM portal for encrypted ZIPs
- [ ] VMware ESXi and vCSA images (from Broadcom) if deploying ESXi-based PFMC

### 3.2 Policy Settings (Pre-Upgrade / Maintenance)

For upgrades or maintenance, set:

| Policy | Value |
|--------|-------|
| Rebuild policy | Unlimited |
| Rebalance policy | Unlimited (during upgrade); Limit concurrent I/O=10 (post-upgrade) |
| Maintenance mode policy | Limit concurrent I/O=10 |

### 3.3 CloudLink Prerequisites (if used)

- CloudLink Center VMs: 6 GB vRAM, 4 vCPUs, 64 GB disk
- CloudLink Center cluster online and synchronized
- Backup CloudLink configuration
- CloudLink vault set to Auto Unlock before upgrade
- Verify `sleep 60` in `/opt/emc/extra/pre_run.sh` on encrypted SDS nodes

---

## 4. Installation Components

### 4.1 PowerFlex Software Stack

| Component | Role |
|-----------|------|
| **MDM** | Metadata Manager — cluster orchestration, topology, rebuild/rebalance |
| **SDS** | Storage Data Server — contributes disks, serves data |
| **SDC** | Storage Data Client — block device driver for hosts |
| **SDR** | Storage Data Replicator — replication I/O |
| **SDT** | Storage Data Target — NVMe host connections |
| **LIA** | Light Installation Agent — automated maintenance/upgrades |

### 4.2 PowerFlex Manager (PFMP)

- Kubernetes/RKE environment hosting management VMs
- PowerFlex Manager UI, REST API, lifecycle management
- Optional: vCenter, jump server, CloudLink, Secure Connect Gateway

### 4.3 Management Cluster Models

- **PowerFlex management cluster (KVM)** — SLES-based, BOSS + PowerFlex storage
- **PowerFlex management cluster (ESXi)** — Former PFMC 2.0; vSAN or PowerFlex storage
- **PowerFlex management cluster (vSAN)** — Former PFMC 1.0; vSAN for management data store

---

## 5. Software and Firmware

### 5.1 RCM and Upgrade Paths

- **RCM train** — Validated OS, storage, firmware, drivers
- Upgrade within train: skip up to 2 RCM versions
- Upgrade to new train: upgrade to end of current train first, then to new train
- RCM hops > 2 considered high risk; contact Dell Support

### 5.2 Download Sources

1. Dell Technologies Support: PowerFlex Rack RCM software
2. RCM Portal: Converged Platforms & Solutions Code Matrix — passwords for ZIP files
3. Broadcom: VMware ESXi and vCSA images (not in RCM bundle)

### 5.3 PowerFlex Management Cluster (KVM) Volumes

| Volume | Size (GB) | Purpose |
|--------|-----------|---------|
| SBD | 8 | Split-brain detection |
| LOCK | 8 | Lock device |
| pfmp | 2048 | PowerFlex management VMs |
| General | 1536 | CloudLink, additional VMs |

---

## 6. PowerFlex Manager Deployment

### 6.1 Deployment Options

1. **PowerFlex Manager–assisted switch configuration** — Full automation for validated switches (Cisco Nexus, Dell PowerSwitch)
2. **Customer-managed switch configuration** — Partial automation; manual switch config required

### 6.2 Network Automation Modes

| Mode | Switch Configuration | Node Networking |
|------|----------------------|-----------------|
| Full | PowerFlex Manager configures node-facing ports | Automated (VMs, bonds, VLANs, MTU) |
| Partial | Customer configures switches | PowerFlex Manager configures node OS networking |

### 6.3 Installer Workflow

1. Deploy installer VM or use co-resident on primary MDM node
2. Upload topology CSV (for automated deployment)
3. Deploy PowerFlex Manager cluster
4. Add protection domains, storage pools, SDS nodes
5. Map volumes to SDCs/host groups

---

## 7. Management Cluster Types

### 7.1 KVM-Based PFMC

- SLES hosts with KVM
- Management VMs on BOSS (for bootstrap) and PowerFlex (for production)
- Services: `lia`, `mdm`, `sds`, `scini`, `activemq` (MDMs)
- RKE2/Kubernetes for PowerFlex Manager

### 7.2 ESXi-Based PFMC

- VMware ESXi hosts
- vCenter, PowerFlex Manager, jump server, optional NSX/CloudLink
- vSAN or PowerFlex as shared storage for management VMs

---

## 8. License Configuration

### 8.1 PowerFlex License

- Upload via PowerFlex Manager or SCLI
- Required for production use

### 8.2 CloudLink License

- Validate license before upgrade
- Configure per CloudLink documentation

### 8.3 Dell SmartFabric OS10 EE License

- Required for Dell PowerSwitch in PowerFlex Manager–managed deployments
- Install per switch documentation

### 8.4 VMware Licensing

- License VMware products (vCenter, ESXi) per Broadcom terms

---

## 9. Post-Installation Validation

### 9.1 Health Checks

- Run inventory in PowerFlex Manager with no errors
- Verify all resource groups healthy
- Check MDM, SDS, SDC status

### 9.2 Service Verification (KVM PFMC)

```bash
sudo systemctl status lia
sudo systemctl status mdm
sudo systemctl status sds
sudo systemctl status scini
sudo systemctl status activemq  # MDMs only
```

### 9.3 Management VM Status

```bash
sudo crm cluster run "virsh list"
```

### 9.4 PowerFlex Manager Diagnostics

- Navigate to **Monitoring > Management Diagnostics > Diagnostics**
- Run diagnostics, review results

### 9.5 Protection Domain Activation

- Activate protection domain via PowerFlex Manager: Block > Protection Domain > Activate

---

## References

- Dell PowerFlex Rack: All Technical Documentation
- PowerFlex Rack with PowerFlex 4.x / 5.x Field Implementation Guide
- Dell PowerFlex 4.5.x Technical Overview
- Dell PowerFlex Appliance and PowerFlex Rack Compatibility Matrix
