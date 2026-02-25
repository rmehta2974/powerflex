# Dell PowerFlex Logical Build Implementation Guide
## Factory Logical Build for PowerFlex 4.x / 5.x Rack

**Version:** 1.0  
**Based on:** Dell PowerFlex Rack Factory Logical Build Guide 4x (Nov 2025, Rev 4.0) and 5x (Nov 2025, Rev 1.0)  

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Deployment Flow Overview](#2-deployment-flow-overview)
3. [Getting Started with the Logical Build](#3-getting-started-with-the-logical-build)
4. [VAST Environment Setup](#4-vast-environment-setup)
5. [Switch Configuration](#5-switch-configuration)
6. [Network Architecture](#6-network-architecture)
7. [VLAN Requirements](#7-vlan-requirements)
8. [PowerFlex Management Cluster (KVM)](#8-powerflex-management-cluster-kvm)
9. [PowerFlex Manager Configuration](#9-powerflex-manager-configuration)
10. [Resource Groups and Deployment](#10-resource-groups-and-deployment)
11. [Post-Deployment Tasks](#11-post-deployment-tasks)
12. [Supported Switches and Defaults](#12-supported-switches-and-defaults)
13. [Handoff to Field](#13-handoff-to-field)

---

## 1. Introduction

This guide documents the **factory logical build** process for Dell PowerFlex Rack (PowerFlex 4.x and 5.x). The logical build is performed in the factory using **VAST** (Virtual Automated System Test) before shipping to the customer. After factory completion, the **Dell PowerFlex Rack Field Implementation Guide** is used for power-on, customer network integration, and finalization at the customer site.

### Target Audience

- Build teams  
- Deployment and installation personnel  
- Factory engineers configuring PowerFlex racks

### Key Differences: 4.x vs 5.x

| Aspect | PowerFlex 4.x | PowerFlex 5.x |
|--------|---------------|---------------|
| Multi-VLAN/subnet | Management cluster and production | Production only |
| Terminology | SDS, SDR, SDT | PDS, DGWT (some references) |
| File nodes | Supported in 4.x | Removed in 5.x LBG |

---

## 2. Deployment Flow Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FACTORY LOGICAL BUILD (This Guide)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. VAST / Cell Router / Environment                                        │
│  2. Verify System Data (Service Catalog, Inventory, Configuration)           │
│  3. Sync Topology → Temp Cabling + Console Info → Manufacturing              │
│  4. Switch Upgrade (Cisco Nexus / Dell PowerSwitch)                          │
│  5. Configure Cisco Nexus or Dell PowerSwitch Network                        │
│  6. Configure IPI (G5, G6, G7)                                               │
│  7. Configure PowerFlex Nodes (management + production)                       │
│  8. Configure PowerFlex Management Cluster (KVM)                             │
│  9. Deploy PowerFlex Manager Production                                      │
│ 10. Deploy CloudLink, UCC Edge (optional)                                    │
│ 11. Deploy Resource Groups (storage-only, compute-only)                       │
│ 12. Post-Deployment, Backup                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              FIELD IMPLEMENTATION (Separate Guide)                            │
│  Power On → Customer Network → Licenses → Alerts → Test Plan                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Getting Started with the Logical Build

### 3.1 VAST Prerequisites

- Log in to VAST UI  
- Locate build in Schedule  
- Click **System ID** or **System Name**  
- Click **LAUNCH ORCHESTRATION SERVICE**

### 3.2 Chapters Applicable to Factory Build

| Chapter | Use in Factory |
|---------|----------------|
| Switch upgrade | Yes |
| Cisco Nexus management network | Yes |
| Cisco Nexus access-aggregation network | Yes |
| Cisco Nexus leaf-spine network | Yes |
| Dell PowerSwitch management network | Yes |
| Dell PowerSwitch access-aggregation network | Yes |
| Configuring Cisco Nexus network | Yes |
| Configuring Cisco nexus management switch | Yes |
| Configure Dell PowerSwitch network | Yes |
| Configuring IPI G7, G6, G5 | Yes |
| Configure PowerFlex nodes | Yes (management + production only) |
| **All other topics** | Reference only; continue from Field Guide after power-on |

---

## 4. VAST Environment Setup

### 4.1 Configure Cell Router and VAST

1. Click **Pre Build** in Orchestration.  
2. Select:  
   - Configure Cisco Router  
   - Prepare VAST For Build  
   - Deploy Ubuntu VM  
3. Click **Start Provision** and wait for completion.  
4. Monitor via Console and % Complete.

### 4.2 Verify System Data

| Tab | Action |
|-----|--------|
| **Service Catalog** | Validate opportunity product identifier; select **PowerFlex Rack System > OFS Controller Node PowerFlex OS configured by VAST** |
| **Inventory** | Click Discovery → SYNC (from OMS); verify BOM; Elevation → SYNC; verify Floor Plan |
| **Options** | Verify and update per LCS |
| **Configuration** | Verify LCS; use C3 LCS toggle if needed |
| **Topology** | SYNC for port map; send temp cabling + console info to manufacturing |

### 4.3 Verify Firmware (RCM)

1. Go to **Firmware** tab.  
2. Select RCM version from LCS.  
3. Select latest Addendum.  
4. **Orchestration** → **Prepare RCM files for Automation** → **Start Provision**.

### 4.4 Verify Environment

1. Go to **Environment** tab.  
2. Check Manufacturing Infrastructure.  
3. Click **GENERATE LAYER 2 IP's**.

### 4.5 Prepare VAST

1. Locate build in Schedule → Edit.  
2. Fill all required fields (*), Save.  
3. First-time: click **DEPLOY ORCHESTRATION SERVICE** (~30 min).  
4. When complete: **LAUNCH ORCHESTRATION SERVICE**.

---

## 5. Switch Configuration

### 5.1 Upgrade Order

**Access-aggregation:**
1. Management switches  
2. Access switches  
3. Aggregation switches  

**Leaf-spine:**
1. Management switches  
2. Leaf switches  
3. Border-leaf switches  
4. Spine switches  

### 5.2 Switch Upgrade via PowerFlex Manager

1. Log in to PowerFlex Manager.  
2. **Resources** → select switch → **Non-Compliant with Default Catalog** → **Update Resources**.  
3. **Apply Updates** > **Yes**.  
4. Upgrade one switch at a time.

### 5.3 Cisco Nexus Manual Upgrade

- Minimum ONIE for ONIE firmware updater: 3.40.1.1-6  
- NX-OS: upgrade via 9.3.10 before 10.3.x if on older version  
- `copy running-config startup-config` before upgrade  
- Cisco Software Central for firmware  

### 5.4 Dell PowerSwitch

- PowerFlex Manager for automation  
- PCIe firmware on S5200: manual upgrade  
- DIAG OS installation/update before automation Part 2  

### 5.5 Configuration Workflow

1. Initialize switch (if needed)  
2. Upgrade firmware (if needed)  
3. Configure basic requirements  
4. Configure VLANs  
5. Configure access-aggregation or leaf-spine  
6. Configure first installer node access ports  
7. Configure customer uplinks  

---

## 6. Network Architecture

### 6.1 Options

- **Access-aggregation** — Typical; simpler integration  
- **Leaf-spine** — For scale or east/west bandwidth  

### 6.2 Supported Switches

| Type | Cisco Nexus | Dell PowerSwitch |
|------|-------------|------------------|
| Management | 92348GC-X, 9348GC-FX3 | S4148T-ON |
| Access / Leaf | 93180YC-FX3, 93240YC-FX2, 9336C-FX2, 9364C-GX | S5248F-ON |
| Aggregation / Spine | 9336C-FX2, 9364C-GX | S5232F-ON |
| Mgmt aggregation | 93180YC-FX3 | — |
| Customer access (dual net) | 93240YC-FX2, 93180YC-FX3 | — |

### 6.3 Temporary Cabling

| Link | Cable Type |
|------|------------|
| Cisco 92348GC-X to 92348GC-X | CAT6 Ethernet |
| Dell S4148T-ON to S4148T-ON | CAT6 Ethernet |
| PXE | CAT6 Ethernet |
| Dell router | CAT6 Ethernet |
| Cisco 92348GC-X to access/aggregation/leaf | 40G to 40G |

---

## 7. VLAN Requirements

### 7.1 PowerFlex Management Cluster (KVM)

| VLAN | Traffic | Network Type | Components |
|------|---------|--------------|------------|
| 101 | OOB / hardware mgmt | General | iDRAC, MVMs, jump VM, ingress |
| 103 | vCenter HA | General | vCenter (optional) |
| 105 | Hypervisor mgmt | General | ESXi, vCenter (if ESXi extension) |
| 140 | PFMC (KVM) mgmt | Hypervisor / PF | KVM nodes, PFMP MVMs, CloudLink, jump, SCG |
| 141 | PFMC data 1 | PowerFlex data | SDS-SDS, SDS-SDC for PFMC |
| 142 | PFMC data 2 | PowerFlex data | — |
| 143 | vMotion (mgmt) | VMware vMotion | ESXi (if extension) |
| 150 | PowerFlex mgmt | Hypervisor / PF | Production MVM, SVM |
| 151–154 | PowerFlex data 1–4 | PowerFlex data | Production SDS-SDS, SDS-SDC |

### 7.2 Production Storage/Compute Nodes

| VLAN | Purpose | Properties |
|------|---------|------------|
| 101 | OOB | L2/L3, MTU 1500/9216 |
| 105 | Hypervisor mgmt | L3, MTU 1500/9216 |
| 150 | PowerFlex mgmt | L3, MTU 1500/9216 |
| 151–154 | PowerFlex data | L2, MTU 9216 |

### 7.3 Replication (Optional)

| VLAN | Purpose | Properties |
|------|---------|------------|
| 161 | Replication 1 | L3, MTU 9216, routable to peer |
| 162 | Replication 2 | L3, MTU 9216, routable to peer |

### 7.4 Routing

- **VLANs 105, 140, 150** must be routable to each other.  
- **MTU=9216** on all ports/link-aggregation carrying PowerFlex data.  
- Two data networks standard; four only for specific designs (e.g., high performance, trunk).

---

## 8. PowerFlex Management Cluster (KVM)

### 8.1 Datastore and VM Details

| Volume | Size (GB) | Purpose |
|--------|-----------|---------|
| SBD | 8 | Split-brain detection |
| LOCK | 8 | Lock device |
| pfmp | 2048 | PowerFlex management VMs |
| General | 1536 | CloudLink, additional VMs |

### 8.2 Networking Prerequisites

- Redundant links with VLT (Dell) or vPC (Cisco)  
- MTU=9216 on PowerFlex data VLANs  
- VLAN 105, 140, 150 routable to each other  

### 8.3 iDRAC Configuration (Manual)

1. Connect to server (KVM/crash cart).  
2. Power on, F2 → BIOS (password: `emcbios`).  
3. **iDRAC Settings > Network**.  
4. Configure: DNS name, domain; IPv4 static (DHCP off); subnet, gateway.  
5. Disable IPv6.  

---

## 9. PowerFlex Manager Configuration

### 9.1 Initial Setup

1. Log in to PowerFlex Manager.  
2. Perform initial setup.  
3. Enable Dell Technologies connectivity services.  
4. Configure compliance.  
5. Specify installation type.  
6. Change password; create non-root user; generate SSH key pairs.  

### 9.2 Settings

- Configure repositories  
- Configure networking  
- Configure license management  
- Resource group deployment  
- Discover resources (nodes, switches via SSH keys)  

### 9.3 Templates

- Clone existing template or use sample templates  
- Publish template  
- Deploy resource groups  

---

## 10. Resource Groups and Deployment

### 10.1 Deploy Storage-Only and Compute-Only Nodes

1. Configure PowerFlex Manager for KVM cluster.  
2. Add networks on KVM host for MVMs.  
3. Deploy storage-only and compute-only nodes.  
4. Deploy resource groups.  
5. Verify resource group status.  
6. Verify non-root user.  

### 10.2 Switch to PowerFlex Node Settings

- Switch from generic to PowerFlex-specific node settings in PowerFlex Manager when deploying production nodes.  

### 10.3 Supported Modes

- Supported modes depend on deployment type (storage-only, compute-only, replication, NVMe-oF).  

---

## 11. Post-Deployment Tasks

### 11.1 Verify and Finalize

- Verify spare capacity  
- Add volumes to resource group (if applicable)  
- Configure CPU state for performance  
- Validate syslog in CloudLink Center (if used)  
- Redistribute MDM cluster  
- Configure alert connector  
- Configure syslog forwarding  
- Download compliance report  

### 11.2 Optional

- Secure Component Verification, TPM  
- NVMe over TCP template  
- Replication  
- NSX Ready (4.x)  

### 11.3 Backup

- Follow Backup tasks chapter before handoff.  

---

## 12. Supported Switches and Defaults

### 12.1 LCS / Configuration Questions (Dell PowerSwitch)

| Question | Answer |
|----------|--------|
| Has VPC | YES (VLT used for Dell) |
| Aggregation VPC Domain ID | LCS or default 60 |
| Management STP | LCS |
| Production STP | LCS |
| ToR VPC Domain ID | LCS |

### 12.2 LCS / Configuration Questions (Cisco Nexus)

| Question | Answer |
|----------|--------|
| Has VPC | YES |
| Aggregation VPC Domain ID | LCS or 60 |
| Management STP | LCS |
| Production STP | LCS |
| Management UDLD | LCS (optional) |
| Production UDLD | LCS (optional) |
| ToR/Leaf VPC Domain ID | LCS |
| Mgmt Aggregation VPC Domain ID | LCS |

### 12.3 Leaf-Spine (Cisco)

- BGP AS, Anycast MAC, L2/L3 VNI IDs  
- OSPF Process ID  
- EVPN Tenant VRF VLAN IDs and VRF names  
- Loopback IPs for spine/leaf/border-leaf  

### 12.4 Default Passwords (Change at Customer Site)

| Subsystem | Username | Default Password |
|-----------|----------|------------------|
| Cisco Nexus | admin, pflex | VMwar3!! |
| Dell PowerSwitch | admin, pflex | admin |
| CloudLink Center | secadmin, root | VMwar3123!! |
| Embedded OS jump server | root, admin | admin |
| Embedded OS storage host | admin, pflex | VMwar3!! |
| iDRAC | admin, pflex | VMwar3!! |
| PowerFlex Manager UI | admin | Admin123! |
| PowerFlex Manager appliance | delladmin | delladmin |
| Management VM | delladmin | delladmin |
| vCSA | administrator@vsphere.local | VMwar3!! |
| Policy Manager | admin | @\):/GjZmPcuE4xT |
| IPI G5 | admin | g5acadia |
| IPI G3 | admin | acadia |

---

## 13. Handoff to Field

### 13.1 Factory Completes

- Configure PowerFlex management nodes  
- Configure PowerFlex rack production nodes  
- Complete switch, IPI, and PFMC (KVM) configuration  
- Deploy PowerFlex Manager production  
- Deploy resource groups  
- Run backup tasks  

### 13.2 Field Continues With

- **Dell PowerFlex Rack Field Implementation Guide**  
- Power on sequence  
- Customer network integration  
- License configuration  
- Events and alerts  
- Test plan  

### 13.3 Customer Access Switches

- In dual-network designs, customer access switches are configured by the customer after delivery.  

---

## References

- Dell PowerFlex Rack with PowerFlex 4.x Factory Logical Build Guide (November 2025)
- Dell PowerFlex Rack with PowerFlex 5.x Factory Logical Build Guide (November 2025)
- Dell PowerFlex Rack Field Implementation Guide 4.x / 5.x
- Dell PowerFlex Appliance Network Planning Guide
- Dell Code and Compatibility Knowledge Hub (formerly RCM Portal)
- Dell PowerFlex Rack: All Technical Documentation
